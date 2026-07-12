# Empty Logs and a Burning Core: Tracing a Silent k3s Failure Three Layers Down

*A backup that never ran, a CPU core that never rested — and why they turned out to be the same bug.*

---

<!-- TODO hero image: images/k3s-leaked-goroutine_small.jpg (~400KB) — suggestion: terminal screenshot of the pprof -peek output, Tokyo Night palette, windowed frame -->

---

## The Starting Point

It began as a routine morning check on the cluster. Five nodes Ready, nothing on fire — except the `frigate` namespace, where the nightly backup CronJob had a row of `Error` pods and one cloudflared replica stuck in `Pending` for two days.

The backup job mirrors Frigate's recordings to a second node with rsync. It is the kind of job you write once, see it complete, and stop thinking about. That was the first mistake, but not in the way I expected.

## An Exit Code and Nothing Else

The failed pods all looked identical: exit code 2, dead in exactly 10 seconds.

`kubectl logs` returned nothing.

Not an error — nothing. Zero lines from a shell script that starts with `set -e` and has echo statements everywhere. Which means it died on the only line whose output was deliberately thrown away:

<pre><code>set -e
apk add --no-cache openssh-client rsync >/dev/null 2>&1   # first line, output discarded
...
echo "[$(date)] Starting rsync mirror..."                  # never reached
</code></pre>

The trap: the job installed rsync *at runtime*, on every run. It almost works. The image stays 100 MB smaller, the script stays one file, and on a healthy cluster nobody ever notices the dependency. But it quietly makes a 02:00 backup depend on working DNS and internet egress at 02:00 — and when cluster DNS got flaky, the job died before its first echo, with the only evidence redirected to /dev/null.

The fix was architectural, not a retry loop: an image with rsync and ssh baked in (`instrumentisto/rsync-ssh`), no network dependency at run time. Trade-off: one more upstream image to track, in exchange for a backup that only depends on the thing it actually backs up.

The honest part of this story: the first successful run after the fix transferred 42 GB. That was the *initial* full sync. In 28 days of "running nightly", the job had never completed once — and nothing told me. A backup that fails silently is not a backup. It is a feeling.

So the second fix was alerting: `KubeJobFailed` added to the Alertmanager email whitelist. Cheap, boring, and it converts this entire class of failure from "discovered after a month" to "email at 02:01".

## The Second Question: Who Is Eating the Core

Fixing DNS flakiness meant explaining it. The journal on the affected server was full of kubelet noise, CoreDNS had restart counts, and `kubectl top` looked normal-ish. But `pidstat` did not:

<pre><code>Average:  UID  PID      %usr  %system  %CPU   Command
          0    3566488  102.3  2.0     104.3  k3s-server
</code></pre>

One flat core. Prometheus history made it stranger: ~0.15 cores for days, then a step to ~1.05 cores — and it stays there, perfectly flat, through load changes and through *k3s restarts*. A load problem fluctuates. A stuck spin does not.

That restart detail broke my first mental model. I assumed some one-off event had wedged a goroutine, and a restart would clear it. The metrics said no: the freshly restarted process spun from minute one. Whatever ignited it — and the ignition lined up exactly with the *other* etcd server being down — the condition re-established itself every time.

## Profiling a Binary You Cannot Read

`perf record` on the k3s process gave me a profile full of raw addresses. The k3s binary is stripped — no ELF symbol table, nothing for perf to name.

But Go binaries cannot be fully stripped. The runtime needs `gopclntab` — the function table used for panics — and `go tool addr2line` happily reads it. So the workflow becomes: perf for the hot addresses, addr2line for the names.

Almost works. The first resolution pass produced garbage — runtime tracer internals that sent me down a flight-recorder rabbit hole for a while. The reveal: I was resolving against `/usr/local/bin/k3s`, but k3s *self-extracts* at startup and the running code lives in `/var/lib/rancher/k3s/data/&lt;hash&gt;/bin/k3s`. Same name, different build layout. `/proc/&lt;pid&gt;/exe` settles the question in one line; I asked it an hour too late.

## The Keyhole

The clean tool was hiding in plain sight. k3s runs everything — apiserver, etcd, scheduler, kubelet — in one process. And the kubelet ships a debug endpoint that the Kubernetes API will proxy for you:

<pre><code>kubectl get --raw /api/v1/nodes/k3frigate/proxy/debug/pprof/profile?seconds=20
</code></pre>

On a normal cluster that profiles the kubelet. On k3s, the kubelet's debug port is not a kubelet endpoint — it is a keyhole into the entire control plane. Fully symbolized CPU profile, no restarts, no flags, evidence intact.

The profile was unambiguous:

<pre><code>1.83s  8.68%  runtime.selectgo (66.62% cum)
       99.79% of callers:
       go.etcd.io/etcd/server/v3/storage/backend.(*backend).run
</code></pre>

etcd's batch-commit loop. It should tick every 100 ms and cost nothing. The goroutine dump showed why it did not: **two** `backend.run` goroutines. One parked in select, healthy. One `[runnable]`, created by `newBackend` on *goroutine 1* — the bootstrap path — spinning on `time.NewTimer(0)`:

<pre><code>t := time.NewTimer(b.batchInterval)   // batchInterval == 0 → fires immediately
for {
    select {
    case <-t.C:
    case <-b.stopc:
        ...
    }
    ...
    t.Reset(b.batchInterval)          // 0 again → fires immediately, forever
}
</code></pre>

A leaked backend from bootstrap, never closed, with a zero batch interval and no guard. One goroutine, one core, forever.

## Upstream Already Knew

Searching with the right nouns — `backend.run`, busy loop, k3s — found it immediately: closing the MVCC store does not close the backend passed into it, the leaked backend "busy-waits in the select loop chewing up CPU". Fixed in k3s v1.35.1+k3s1 and backported across release branches. We were on v1.35.0+k3s3, three patch releases behind the fix.

etcd snapshot first, then the standard installer with `INSTALL_K3S_VERSION` pinned, one server at a time. CPU went from 104% to ~10% on both servers. The agents kept running through the whole thing.

And here the two stories close into one. The leaked goroutine had been degrading the first server for days — slow kubelet, struggling CoreDNS, the DNS flakiness that killed an `apk add` at 02:00 in a backup job that could not tell anyone. The backup did not fail because of rsync. It failed because a goroutine three layers down was spinning on a zero-length timer.

## Design Decisions

- Baked rsync into the backup image instead of installing at runtime — a backup job should depend on nothing that is not the data itself
- Whitelisted `KubeJobFailed` for email instead of un-blackholing all warnings — alert on what is actionable, keep the rest quiet
- Profiled in production via the kubelet pprof proxy instead of restarting with debug flags — keeps the evidence alive, and on k3s it covers the whole control plane
- Fixed by version upgrade, not workaround — a systemd CPU quota would have "contained" the spin and hidden the bug for another year

## Things That Went Wrong

**The logs were empty by design.** The failing line's output went to /dev/null for tidiness. Tidiness cost 28 days of undetected failure. Discard output only after the script has proven it can speak.

**`kubectl top` averaged the problem away.** Trailing windows showed 13% on a node that pidstat showed at 104% in real time. Averages are for capacity planning, not for diagnosis.

**I resolved symbols against the wrong binary.** Same k3s version, same filename — but the self-extracted copy under `/var/lib/rancher/k3s/data/` is what actually runs. Half a day of tracer red herrings from this kind of mismatch.

**I assumed restart clears a stuck goroutine.** Prometheus showed the spin surviving k3s restarts cleanly. The assumption was not just wrong — it nearly made me destroy the evidence by restarting with debug flags.

**Two etcd members, so every server restart was a quorum hold-down.** With the third member long retired, each upgrade restart froze the API briefly. Survivable in a homelab; the reminder that two-member etcd is availability theater is free.

## Impact

- k3s-server CPU: ~104% → ~10% of a core, on both control-plane servers
- Busiest node overall: 67% → ~35% CPU
- Backup success rate: 0 completed runs in 28 days → verified nightly mirror (42 GB initial, ~250 MB/day incremental)
- Failure detection: never → email within a minute via KubeJobFailed
- Diagnosis artifacts: zero restarts needed before the fix was identified

## The Payoff

The artifact is small: one image swap, one alert rule, one version bump. The understanding is the part I keep — that a stripped Go binary in production is still a fully observable system if you know where the keyholes are: `gopclntab` for names, the kubelet proxy for profiles, the goroutine dump for the created-by line that points at the leak.

Logs tell you what the code chose to say. The profiler tells you what it actually did.

## Cluster Context

| Node | Role | Hardware |
|---|---|---|
| k3master (.52) | control-plane, etcd | x86 mini PC, backup target (1.8 TB) |
| k3frigate (.56) | control-plane, etcd, NVR | x86, GTX 1050 Ti, 16 GB RAM, Frigate + 7 cameras |
| one61 / one62 / one6t | workers | OnePlus 6/6T phones, postmarketOS, USB-C network |

---

Manifests for the backup job and alert routing live in the [homelab-config](https://github.com/ivemcfire/homelab-config) repo.

<!-- TODO: previous-posts footer line · update index.html card · two screenshots: substrate proof (pidstat/pprof) + workload proof (kubectl get nodes after upgrade) -->
