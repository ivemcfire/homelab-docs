# LinkedIn — k3s leaked goroutine post

My nightly backup failed for 28 days straight. Zero log lines.
Same week, both k3s servers were burning a full CPU core each — flat, around the clock, surviving restarts.
Turned out: same bug, three layers down.

The backup job installed rsync at runtime with the output sent to /dev/null. When cluster DNS got flaky at 02:00, the job died on its first line with nothing to say. The DNS flakiness itself came from somewhere much deeper — a leaked etcd backend goroutine inside k3s, spinning on a zero-length timer.

What the debugging actually took:

→ empty logs, exit code 2, dead in 10 seconds — failure was designed to be invisible
→ kubectl top said 13%, pidstat said 104% — averages hide spins
→ perf on a stripped Go binary — raw addresses only
→ go tool addr2line reads gopclntab, but only against the self-extracted binary in /var/lib/rancher/k3s/data, not /usr/local/bin/k3s
→ the keyhole: kubectl get --raw /api/v1/nodes/X/proxy/debug/pprof — on k3s this profiles the whole control plane, no restart, no flags
→ goroutine dump: two backend.run loops where there should be one, the leaked one created at bootstrap and never closed

Upstream already knew — fixed in k3s v1.35.1, we were three patches behind.

Outcome:
- k3s-server CPU: ~104% → ~10% on both servers
- backup: 0 successful runs in 28 days → verified nightly mirror, alert on failure within a minute
- zero restarts needed before root cause was identified

The fix was one image swap, one alert rule, one version bump. The understanding is the part I keep.

Logs tell you what the code chose to say. The profiler tells you what it actually did.

https://ivemcfire.github.io/posts/k3s-leaked-goroutine.html
https://github.com/ivemcfire/homelab-config

<!-- char count ~1750 incl URLs; hook = first 3 lines, lands before "see more" -->
