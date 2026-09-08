# OpenBao

Argo CD reconciles this directory into the `openbao` namespace. One replica,
integrated raft storage on a 10Gi Longhorn volume, reachable on the tailnet at
`https://bao.lab.packetcraft.dev` through Traefik's default wildcard certificate.

This is the secrets manager the cluster is moving to. Today most secrets in this
repo are SealedSecrets — ciphertext committed to git, decryptable only by the
controller's private key. That works, but it means the encrypted material is
public, rotation means re-sealing and committing, and there is no audit log of
who read what. OpenBao replaces the model: nothing secret is committed at all,
apps authenticate with the ServiceAccount token Kubernetes already gave them,
and every read is leased and logged.

**The existing SealedSecrets are not going anywhere yet.** Nothing in this
directory touches them, and the migration is per-secret and manual — see
"Migrating off SealedSecrets" below. Do not delete `sealed-secrets-pub.pem` or
the `sealed-secrets` Application.

## Before the first sync

**DNS.** Not managed by this repo. In Cloudflare, zone `packetcraft.dev`:

| Type | Name      | Content         | Proxy    |
| ---- | --------- | --------------- | -------- |
| A    | `bao.lab` | `192.168.1.250` | DNS only |

`192.168.1.250` is Traefik's LoadBalancer address, the same one every other
`*.lab` record points at. Proxying must stay off: the target is a private
address reachable only over the tailnet, which is the point.

**Then:**

```sh
kubectl apply -f argocd/openbao.yaml
```

The `argocd/` directory is not watched by an app-of-apps, so this is manual, the
same as every other Application here.

## Bring-up

After the first sync the pod is running, Ready, and **uninitialised**. Ready
means reachable, not usable — see the probe comments in `statefulset.yaml` for
why it has to work that way.

### 1. Initialise

```sh
kubectl exec -n openbao openbao-0 -- bao operator init -key-shares=3 -key-threshold=2
```

This prints three unseal key shares and an initial root token, **once**. They
are not recoverable. If you lose them the raft volume is a 10Gi encrypted brick
and the only path forward is deleting the PVC and starting over.

Three shares with a threshold of two is the sensible homelab shape: any two
unseal it, so losing one share is survivable, and no single stored copy is
sufficient on its own. Put them somewhere that is not this cluster and not this
repo — a password manager, a printed copy, or split across both. Somewhere you
can reach when the cluster is down, because that is exactly when you will need
them.

### 2. Unseal

One command, run twice, with a **different** key share each time.

```sh
kubectl exec -it -n openbao openbao-0 -- bao operator unseal
```

It prompts `Unseal Key (will be hidden):`. After the first share:

```
Sealed             true
Total Shares       3
Threshold          2
Unseal Progress    1/2
```

Run the same command again with a second share and `Sealed` flips to `false`.
Confirm:

```sh
kubectl exec -n openbao openbao-0 -- bao status
```

No `-address` flag is needed anywhere, because `BAO_ADDR` is set to the pod's
own listener in `statefulset.yaml`.

**Unseal through `kubectl exec`, not through the UI or the CLI over the
tailnet.** Both of those work — `https://bao.lab.packetcraft.dev/ui` has an
unseal form — but they hand the key share to Traefik over TLS and then forward
it to the pod in cleartext, which is exactly the `tls_disable` tradeoff
described under "Upgrading to in-cluster TLS". `kubectl exec` goes laptop → API
server → kubelet → container stdin, TLS the whole way, and the share never
touches the pod network. Reconsider once in-cluster TLS is done.

The `-it` is load-bearing: without a TTY the prompt never appears. And do not
pass the share as an argument (`bao operator unseal <key>`) — it lands in your
shell history and is visible in `ps` inside the pod.

**Failure modes, all of which look confusing at an awkward hour:**

- *Progress stuck at 1/2.* The shares have to be **different**. A repeated share
  is rejected as a duplicate and progress does not advance.
- *Progress reset itself.* It is in-memory and per-pod, so a pod restart
  midway puts you back at 0/2. `bao operator unseal -reset` clears it
  deliberately.
- *`failed to unseal: invalid key` on the second share.* Shamir only
  reconstructs at the threshold, so a wrong share is not detected when it is
  entered, only when the set fails to combine — and it cannot tell you which one
  was wrong. Progress resets; try a different pair.
- *Nothing happened to the other pods.* Only relevant at three replicas: raft
  replicates the data, not the in-memory master key. Each pod is unsealed
  separately, two shares each.

### 3. Log in and stop using the root token

```sh
export BAO_ADDR=https://bao.lab.packetcraft.dev
bao login <initial-root-token>
```

The root token should be revoked once the Kubernetes auth method is configured
and you have confirmed a real login works. Keep one recovery path: `bao operator
generate-root` mints a new one from the unseal shares, so revoking root is not a
one-way door as long as you still have the shares.

## Day-to-day: every restart seals it

This is the standing operational cost and it is worth stating plainly. A node
reboot, an eviction, an image bump, a `kubectl delete pod` — any of these bring
OpenBao back **sealed**. Sealed means every client reading a secret gets a 503
until a human runs `bao operator unseal` twice — the procedure and its failure
modes are under "2. Unseal" above, and it is the same every time. Nothing in
this deployment automates it.

Two consequences to design around:

- **Do not put a secret that OpenBao itself needs to boot behind OpenBao.** The
  circular dependency shows up at the worst time.
- **Apps that read secrets only at startup will not recover on their own.** If
  OpenBao is sealed while an app restarts, that app stays broken after the
  unseal until it is restarted too.

`updateStrategy: OnDelete` exists because of this. A `Synced` and `Healthy`
Application does **not** mean the running pod matches git; spec changes land in
the API and wait. To actually apply one:

```sh
kubectl delete pod openbao-0 -n openbao    # then unseal, twice
```

The alternative is auto-unseal, which is deliberately not configured here. Every
option trades the problem for a different one: Transit unseal needs a second
OpenBao that has the same problem one level down, and a cloud KMS puts the seal
key at a provider and needs the internet egress that `networkpolicy.yaml`
currently refuses. Manual unseal is the honest choice for a homelab that reboots
rarely; revisit it if it stops being rare.

## Letting apps authenticate

Configure the Kubernetes auth method once. It works because
`serviceaccount.yaml` binds this pod's ServiceAccount to
`system:auth-delegator`, which lets OpenBao call TokenReview.

```sh
bao auth enable kubernetes

# token_reviewer_jwt and kubernetes_ca_cert are omitted on purpose: left unset,
# OpenBao uses its own projected ServiceAccount token and the cluster CA from
# inside the pod. Pasting a long-lived token here is the usual way this config
# rots.
bao write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc.cluster.local
```

Then, per app — a policy naming exactly what it may read, and a role binding
that policy to one ServiceAccount in one namespace:

```sh
bao policy write portfolio-backend - <<'EOF'
path "secret/data/portfolio/backend" {
  capabilities = ["read"]
}
EOF

bao write auth/kubernetes/role/portfolio-backend \
  bound_service_account_names=portfolio-backend \
  bound_service_account_namespaces=portfolio \
  policies=portfolio-backend \
  ttl=1h
```

Keep `bound_service_account_namespaces` to a single namespace. A role bound to
`*` is reachable by any pod in the cluster that can create a ServiceAccount,
which is most of them.

## Migrating off SealedSecrets

Getting a secret from OpenBao into a pod needs a delivery mechanism, and this
deployment does not include one. The two realistic options:

- **External Secrets Operator** — a `ClusterSecretStore` pointed at
  `http://openbao.openbao.svc.cluster.local:8200` with Kubernetes auth, plus an
  `ExternalSecret` per secret. ESO writes a normal `Secret` and refreshes it, so
  nothing about how existing Deployments consume secrets has to change. This is
  the path of least disruption for this repo and the one to start with.
- **The OpenBao agent injector** — a sidecar renders secrets to a file in the
  pod. Better isolation, no Secret object at rest, but every consuming
  Deployment needs annotations and most need to learn to read a file.

Either way, migrate one secret at a time and delete its SealedSecret only after
the app is confirmed running on the new path. There is no need to hurry: the two
systems coexist without conflict.

## Backups

Longhorn replicates the volume, which protects against a disk or a node. It does
not protect against the failure that actually happens here — a corrupted raft
store, or a bad `bao delete` — because it faithfully replicates both.

Take a real snapshot before anything structural:

```sh
kubectl exec -n openbao openbao-0 -- bao operator raft snapshot save /tmp/bao.snap
kubectl cp openbao/openbao-0:/tmp/bao.snap ./bao-$(date +%F).snap
```

The snapshot is encrypted with the same seal keys, so restoring it needs the
unseal shares. **A snapshot without the shares is worthless**, and shares
without a snapshot are equally so. They are two halves of one backup and should
not be lost together — or stored together.

## Growing to three nodes

One replica is the deliberate starting point. Longhorn already replicates the
volume, so a second copy of the data is not what a three-node raft buys — it
buys surviving a node loss without a manual unseal-and-wait, at the cost of
three volumes to unseal individually and a quorum that can get stuck.

When it is worth it:

1. Uncomment the `retry_join` stanzas in `configmap.yaml`.
2. Add `service_registration "kubernetes" {}` to the config, plus a Role granting
   `get`/`patch` on pods, so the front-door Service can exclude sealed pods. At
   one replica this is actively harmful — see the note at the bottom of
   `configmap.yaml`.
3. Set `replicas: 3` in `statefulset.yaml`.
4. Unseal each new pod individually. They join as raft peers but come up sealed,
   the same as pod 0.
5. Enable the `OpenBaoNoLeader` rule in `prometheusrule.yaml` once the metric
   name is confirmed.

Also worth adding at that point: a PodDisruptionBudget with `minAvailable: 2`.
Deliberately absent at one replica, where `minAvailable: 1` would block every
node drain outright.

## Upgrading to in-cluster TLS

The listener runs with `tls_disable = 1` and TLS terminates at Traefik, matching
every other `*.lab` host in this repo. For a secrets manager that is a real cost,
not a formality: unseal shares, root tokens and every secret read cross the
cluster network in cleartext between Traefik and this pod.

It is accepted for bring-up because getting it wrong locks you out of your own
vault, and the failure is easier to fix before there is anything in it. To close
it later:

1. A cert-manager `Certificate` for `openbao.openbao.svc.cluster.local` and
   `bao.lab.packetcraft.dev`, issued by an issuer the cluster trusts.
2. Mount the resulting Secret and point `tls_cert_file` / `tls_key_file` at it,
   dropping `tls_disable`.
3. A Traefik `ServersTransport` on the IngressRoute that trusts the issuing CA —
   `insecureSkipVerify: true` here would re-open most of what step 1 bought.
4. Change `BAO_ADDR` and `BAO_API_ADDR` to `https://`, and the `retry_join`
   `leader_api_addr` values with them.

Do it as its own change, with a fresh raft snapshot taken first.

## Monitoring

`OpenBaoSealedOrDown` fires on `up == 0`, which covers both states, because a
sealed OpenBao returns 503 on the metrics endpoint and the target drops. Telling
them apart:

```sh
kubectl exec -n openbao openbao-0 -- bao status
```

The richer alerts in `prometheusrule.yaml` are commented out pending
verification of the metric names — OpenBao is midway through renaming Vault's
`vault_` telemetry prefix, so confirm against a live pod before enabling them:

```sh
kubectl exec -n openbao openbao-0 -- \
  wget -qO- 'http://127.0.0.1:8200/v1/sys/metrics?format=prometheus' | grep -i unsealed
```
