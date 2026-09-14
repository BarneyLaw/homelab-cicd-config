# garage-obsync

A 3-node Garage cluster in the `obsync` namespace, serving as the S3 store for
obsync phase 1. Independent of the Garage on nebula that backs CNPG/etcd
backups — separate cluster, separate RPC secret, separate failure domain.

The manifests get you running pods. Forming the cluster takes the steps below,
in order; only steps 1 and 5 end in a commit.

---

## 1. Seal the secrets (before the first sync)

`kustomization.yaml` references `sealed-secret.yaml`, which is not committed
until you generate it. Until then `kustomize build` fails and the Application
shows `Unknown` — that is the intended failure mode.

```bash
# rpc-secret must be exactly 32 bytes, hex-encoded (64 chars). Garage refuses
# anything else at startup.
kubectl create secret generic garage-secrets \
  --namespace obsync \
  --from-literal=rpc-secret="$(openssl rand -hex 32)" \
  --from-literal=admin-token="$(openssl rand -base64 32)" \
  --dry-run=client -o yaml \
| kubeseal --cert sealed-secrets.pem --format yaml \
  > apps/garage-obsync/sealed-secret.yaml

git add apps/garage-obsync/sealed-secret.yaml && git commit && git push
```

SealedSecrets are encrypted to namespace+name. `obsync` / `garage-secrets` must
match exactly or decryption fails silently at the controller.

## 2. Apply the Application

```bash
kubectl apply -f argocd/garage-obsync.yaml
kubectl -n obsync get pods -w
```

Expect three pods `Running` but `0/1 Ready`. That is correct at this point:
readiness is `/health`, and `/health` is 503 until a layout exists.

## 3. Connect the nodes (one time)

Nothing discovers peers here: each pod starts knowing only itself, and
`garage status` on garage-0 lists a single node. Connect them by hand so the
layout can be assigned. Step 5 makes the connection permanent.

```bash
for i in 0 1 2; do
  echo "garage-$i $(kubectl -n obsync exec garage-$i -- /garage node id -q)"
done

kubectl -n obsync exec garage-0 -- /garage node connect \
  <id-1>@garage-1.garage-rpc.obsync.svc.cluster.local:3901
kubectl -n obsync exec garage-0 -- /garage node connect \
  <id-2>@garage-2.garage-rpc.obsync.svc.cluster.local:3901

kubectl -n obsync exec garage-0 -- /garage status   # HEALTHY NODES lists all three
```

Keep the three IDs for step 5.

## 4. Assign the cluster layout (one time, per node)

Garage's layout is cluster metadata, not Kubernetes state. ArgoCD has no
visibility into it and cannot recreate it.

```bash
kubectl -n obsync get pods -o wide   # which node each garage-N landed on

# One zone per node, named after the node the pod runs on: Garage places the 3
# replicas in distinct zones. Capacity should be the real free space on that
# node's disk, not the PVC size: local-path does not enforce the 100Gi request.
kubectl -n obsync exec garage-0 -- /garage layout assign -z <node-of-garage-0> -c 90G <id-0>
kubectl -n obsync exec garage-0 -- /garage layout assign -z <node-of-garage-1> -c 90G <id-1>
kubectl -n obsync exec garage-0 -- /garage layout assign -z <node-of-garage-2> -c 90G <id-2>

kubectl -n obsync exec garage-0 -- /garage layout show
kubectl -n obsync exec garage-0 -- /garage layout apply --version 1
```

Within a poll cycle all three pods go `1/1 Ready`, and `garage status` shows
each node with its zone and capacity.

## 5. Commit the peers

The connections from step 3 live only in each node's peer cache. That survives
a restart, but not every pod coming back on a new IP at once, after which no
node knows where the others are. Uncomment `bootstrap_peers` in `garage.toml`,
fill in the three IDs, and commit.

```bash
git add apps/garage-obsync/garage.toml && git commit && git push
```

The config hash changes, so Argo CD rolls the pods one at a time. This step has
to come after the layout: a rolling update waits for each pod to be Ready, and
before step 4 none can be.

## 6. Bucket and access key for obsync

```bash
kubectl -n obsync exec garage-0 -- /garage bucket create obsync
kubectl -n obsync exec garage-0 -- /garage key create obsync-app
kubectl -n obsync exec garage-0 -- /garage bucket allow --read --write obsync --key obsync-app
```

The worker's SealedSecret takes this key: see `apps/obsync-worker/README.md`
step 1. `garage key info obsync-app --show-secret` prints both halves. Client
config:

| Setting | Value |
|---|---|
| Endpoint | `http://garage.obsync.svc.cluster.local:3900` |
| Region | `garage` |
| Addressing | **path-style** (`UsePathStyle: true`) |
| TLS | none — in-cluster only |

Path-style is not optional: vhost-style needs `<bucket>.s3.garage.internal` to
resolve, and nothing serves that zone.

---

## Operational notes

- **Quorum is 2 of 3.** kured's `concurrency: 1` already guarantees one node at
  a time, which this survives. Two nodes down means reads and writes fail.
- **local-path pins pods to nodes.** A PVC binds to the node it was first
  scheduled on; that pod can never move. Losing a node means losing that
  replica until the node returns — Garage rebuilds from the other two, it does
  not self-migrate.
- **Resizing `data` later** requires
  `kubectl delete statefulset garage -n obsync --cascade=orphan`, then editing
  the template and re-syncing. PVCs and data survive; volumeClaimTemplates are
  immutable in place and a naive sync fails with a forbidden-field error.
- **Config edits roll the pods.** `garage.toml` is a `configMapGenerator`, so a
  commit produces a new ConfigMap name and a rolling restart. That is deliberate
  (subPath mounts never pick up in-place ConfigMap changes).
- **No IngressRoute.** The S3 and admin ports are cluster-internal. If obsync
  clients outside the cluster ever need the endpoint, that is a Traefik
  IngressRoute plus a decision about exposing S3 auth over the tunnel.
- **No NetworkPolicy yet.** Worth adding once the obsync backend exists, so only
  it can reach port 3900 and nothing can reach 3903.