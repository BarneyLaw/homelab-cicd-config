# obsync-worker

The obsync worker as a CronJob in the `obsync` namespace. Once a day at 18:17
Singapore time it pulls every active Canvas course's files into the
garage-obsync store. Source, image and CI:
[BarneyLaw/obsync-man-worker](https://github.com/BarneyLaw/obsync-man-worker).

CI owns one line of this directory: `newTag` in `kustomization.yaml`, which
every release on obsync-man-worker's `main` sets to the tested commit. The
rest was copied in once, on the first deploy, and is edited here from then on.

Depends on `apps/garage-obsync`, which owns the `obsync` namespace and must
have a bucket and key before the first run.

---

## 1. Seal the secret (before the first sync)

`kustomization.yaml` references `sealed-secret.yaml`, which is not committed
until you generate it. Until then `kustomize build` fails and the Application
shows `Unknown`, which is the intended failure mode.

The Garage key is the one created in `apps/garage-obsync/README.md` step 4
(`garage key info obsync-app --show-secret`). The Canvas token comes from
Canvas, **Account → Settings → New access token**. Write it to a file rather
than the command line, so it never lands in shell history.

```bash
kubectl create secret generic obsync-worker \
  --namespace obsync \
  --from-file=canvas-token=./canvas-token.txt \
  --from-literal=garage-access-key='GK...' \
  --from-literal=garage-secret-key='...' \
  --dry-run=client -o yaml \
| kubeseal --cert sealed-secrets.pem --format yaml \
  > apps/obsync-worker/sealed-secret.yaml

shred -u canvas-token.txt
git add apps/obsync-worker/sealed-secret.yaml && git commit && git push
```

SealedSecrets are encrypted to namespace and name: `obsync` / `obsync-worker`
must match exactly.

## 2. Apply the Application

```bash
kubectl apply -f argocd/obsync-worker.yaml
kubectl -n obsync get cronjob obsync-worker
# SCHEDULE      TIMEZONE         SUSPEND
# 17 18 * * *   Asia/Singapore   False
```

Then run it once by hand (below) instead of waiting for 18:17.

---

## Run it now

```bash
kubectl -n obsync create job --from=cronjob/obsync-worker obsync-manual-$(date +%s)
kubectl -n obsync logs -f job/obsync-manual-<timestamp>
```

Or in Argo CD: **obsync-worker → the CronJob → ⋮ → Create Job**.

This is safe next to the schedule. Kubernetes' `concurrencyPolicy: Forbid`
does not cover Jobs made this way, but the worker takes a lease in the store
and the second of two overlapping runs exits 3. On Garage that lease is
best-effort (Garage ignores `If-None-Match`), so avoid starting one in the
minutes around 18:17.

### One course, or part of one

`--from=cronjob` copies the CronJob's arguments exactly. For a scoped pull,
render the Job and override them:

```bash
kubectl -n obsync create job --from=cronjob/obsync-worker obsync-cs3103 \
  --dry-run=client -o json \
| jq '.spec.template.spec.containers[0].args = [
    "pull", "-course", "CS3103", "-path", "Lecture Notes",
    "-rules=/etc/obsync/rules.json", "-tmp-dir=/tmp", "-lock-ttl=3h"]' \
| kubectl apply -f -
```

Add `"-dry-run"` to log every decision without downloading or writing.

### Inspect the store

The image also carries the `obsync` CLI:

```bash
kubectl -n obsync create job --from=cronjob/obsync-worker obsync-ls \
  --dry-run=client -o json \
| jq '.spec.template.spec.containers[0].command = ["/usr/local/bin/obsync"]
    | .spec.template.spec.containers[0].args = ["ls"]' \
| kubectl apply -f - \
&& kubectl -n obsync wait --for=condition=complete job/obsync-ls \
&& kubectl -n obsync logs job/obsync-ls \
&& kubectl -n obsync delete job obsync-ls
```

Swap `ls` for `preview <course-id>`, `log <course-id>`, or `gc` (a dry run;
`gc -apply` deletes, and takes the store lease).

---

## Change it

| What | Where | Takes effect |
|---|---|---|
| Schedule | `schedule` in `cronjob.yaml`, Asia/Singapore time | next sync |
| Worker rules | `rules.json`: keep these to hard caps, aggressive filtering belongs in the plugin | next run |
| Pause | `suspend: true` in `cronjob.yaml`. `kubectl patch` is reverted by selfHeal | next sync |
| Canvas token | reseal step 1 | next run |

## Alerts

`prometheusrule.yaml`, from kube-state-metrics (no Pushgateway here):

| Alert | Fires when |
|---|---|
| `ObsyncWorkerStale` | no successful run in 26h |
| `ObsyncWorkerLastRunFailed` | the latest scheduled run has not succeeded 3h after it was scheduled |
| `ObsyncWorkerMissing` | the CronJob is absent (Application not synced, secret missing) |
| `ObsyncWorkerSuspended` | suspended for over a day |

## Operational notes

- **Exit codes:** 0 ok, 1 failed, 2 configuration, 3 lease held by another
  run, 4 Canvas rejected the token. A 4 does not fix itself: NUS tokens expire.
- **Logs** are JSON, one event per line; every writing run also leaves
  `runs/<run_id>.json` in the bucket.
- **A failed run changes nothing consumers see.** `latest` is published last,
  so the previous manifest stays live.
- **Courses with a hidden Files tab** are reported as `forbidden` and do not
  fail a scheduled run.
- **No NetworkPolicy yet.** The worker needs DNS, Garage on 3900, and HTTPS to
  `canvas.nus.edu.sg` (whose file downloads redirect to other hosts, so egress
  cannot be pinned to one name).
