# LoadBalancer guitar-backend pending (issue #11)

Real cause: `can't change sharing key for "default/guitar-backend", address also in use by default/guitar-frontend`. Both want `192.168.100.202`. Pool annotation already corrected to `homelab-pool`.

Pick fix:

## A. ClusterIP (recommended if guitar-frontend proxies to backend)

```bash
kubectl -n default patch svc guitar-backend -p '{"spec":{"type":"ClusterIP"}}'
```

## B. Different IP (.203)

```bash
kubectl -n default patch svc guitar-backend --type=merge -p '{"spec":{"loadBalancerIP":"192.168.100.203"}}'
```

## C. Shared IP via annotation (port-sharing only, both must pick distinct ports)

```bash
kubectl -n default annotate svc guitar-frontend metallb.universe.tf/allow-shared-ip=guitar-shared
kubectl -n default annotate svc guitar-backend metallb.universe.tf/allow-shared-ip=guitar-shared
```

Confirm pick + apply. Verify with `kubectl -n default get svc guitar-backend`.
