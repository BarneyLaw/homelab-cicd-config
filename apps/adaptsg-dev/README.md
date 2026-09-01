# AdaptSG development environment

Argo CD reconciles this directory into the `adaptsg-dev` namespace. The caregiver UI is
available on the LAN at `https://sim-next.lab.packetcraft.dev` using Traefik's default wildcard
TLS certificate.

The single Streamlit replica supports independent browser/WebSocket sessions. It does not
provide separate filesystems: Kubernetes users with `pods/exec` permission share the Longhorn
workspace. The debugger listens only on pod loopback; no Service or Ingress exposes it.

Run the full app gate:

```sh
kubectl exec -n adaptsg-dev deploy/adaptsg-dev -c app -- \
  /bin/sh -lc 'cd /workspace/source && ./scripts/check.sh'
```

Use network and process diagnostics:

```sh
kubectl exec -it -n adaptsg-dev deploy/adaptsg-dev -c diagnostics -- /bin/sh
```

Attach a Python debugger:

```sh
kubectl port-forward -n adaptsg-dev deploy/adaptsg-dev 5678:5678
```

The PVC is protected from Argo prune/delete, but it is not a backup. The init container refuses
to overwrite tracked local changes. Commit or copy irreplaceable edits before deleting the
claim or restarting after an unfinished session.
