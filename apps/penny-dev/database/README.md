# penny-dev database

CloudNativePG cluster `penny-dev-db`, one instance, pinned to `nebula`, Longhorn
backed. Database `penny`, owner `penny`. **No backups are configured** — this is
a dev database and its contents are disposable.

In-cluster workloads should keep using
`penny-dev-db-rw.penny-dev.svc.cluster.local:5432`. Everything below is about
reaching it from a laptop.

## How a developer connects

```
postgresql://penny@penny-dev.lab.packetcraft.dev:5432/penny?sslmode=verify-full&sslrootcert=penny-dev-ca.crt
```

Three things have to be true.

**1. You are on the tailnet.** `penny-dev.lab.packetcraft.dev` is a public DNS
record pointing at `192.168.1.250`, a private address. `control-node` (deus)
advertises `192.168.1.0/24` as a Tailscale subnet route, so that address is only
routable to tailnet members. Off the tailnet the name resolves and then hangs.

On macOS and Windows subnet routes are accepted by default. On Linux they are
not — you need `sudo tailscale up --accept-routes`.

**2. You have the CA certificate.** The server certificate is issued by the
cluster's own CNPG CA, not by a public authority, so your client has to be told
to trust it. Fetch it once (needs kubectl access; otherwise ask for the file):

```bash
kubectl get secret -n penny-dev penny-dev-db-ca \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > penny-dev-ca.crt
```

It is a CA public certificate — safe to pass around, no secret in it.

**This file expires and has to be re-fetched.** CNPG issues its CA with a
90-day lifetime and rotates it automatically shortly before expiry; the current
one is good until 2026-11-05. When it rotates, everyone holding an old copy
starts getting certificate-verify failures until they run the command again.
Check the expiry of your copy with:

```bash
openssl x509 -in penny-dev-ca.crt -noout -subject -dates
```

If that quarterly chore is not worth it, the alternative is to issue the server
certificate from the existing `letsencrypt-cloudflare` ClusterIssuer instead and
point `spec.certificates.serverTLSSecret` at it — then `sslmode=verify-full`
works off system trust with no CA file anywhere, at the cost of a cert-manager
dependency on the database's TLS.

If you would rather not carry the file, `sslmode=require` still encrypts the
connection and needs nothing extra; it just does not verify who you are talking
to. `sslmode=disable` is refused by the server outright.

**3. You have the password.** There is one shared account, `penny`, which owns
the database:

```bash
kubectl get secret -n penny-dev penny-dev-db-app \
  -o jsonpath='{.data.password}' | base64 -d
```

If you want per-person accounts later, add them under `spec.managed.roles` in
`cluster.yaml` with `passwordSecret` pointing at a SealedSecret, and give each
`inRoles: [penny]` so they inherit the owner's rights.

### psql

```bash
PGPASSWORD='...' psql \
  "host=penny-dev.lab.packetcraft.dev port=5432 dbname=penny user=penny \
   sslmode=verify-full sslrootcert=penny-dev-ca.crt"
```

### JDBC / DBeaver

```
jdbc:postgresql://penny-dev.lab.packetcraft.dev:5432/penny?ssl=true&sslmode=verify-full&sslrootcert=/path/to/penny-dev-ca.crt
```

### node-postgres

```js
new Pool({
  host: "penny-dev.lab.packetcraft.dev",
  port: 5432,
  database: "penny",
  user: "penny",
  password: process.env.PGPASSWORD,
  ssl: { ca: fs.readFileSync("penny-dev-ca.crt") },
});
```

## Why it is wired this way

**How it is exposed.** Through Traefik, like every other `*.lab` host, via an
`IngressRouteTCP` on a dedicated `postgres` entrypoint. It does not have its own
LoadBalancer.

It has to be a *TCP* route on its own port rather than a Host rule, because
Postgres does not open with a TLS ClientHello — it sends a plaintext
`SSLRequest` and upgrades in band. Traefik therefore never sees SNI and the
route matches `HostSNI(`*`)`. Two consequences worth knowing:

- The hostname is not enforced. Any name resolving to the cluster on 5432
  reaches this database; `penny-dev.lab.packetcraft.dev` is the name you should
  use, but it is the *port* that selects the backend.
- A second database cannot share the entrypoint. It needs its own port and its
  own entry under `ports:` in `apps/traefik/helmchartconfig.yaml`.

Traefik passes the bytes through untouched — there is no `tls:` block on the
route — so Postgres terminates TLS itself and `sslmode=verify-full` against the
CNPG CA works end to end.

**Why source-IP rules are absent from pg_hba.** Connections are masqueraded on
the way in, so by the time a packet reaches Postgres its source sits inside
`10.42.0.0/16` — LAN, Tailscale and in-cluster traffic are indistinguishable at
that layer. Rules like `host penny penny 192.168.1.0/24` never match and give
false assurance; the old config had exactly that rule. Network scoping is
Tailscale's job. pg_hba's job is `hostnossl all all all reject`, which stops a
client from handing its password over in the clear.

## Required DNS record

Not managed by this repo. In Cloudflare, zone `packetcraft.dev`:

| Type | Name                    | Content       | Proxy    |
| ---- | ----------------------- | ------------- | -------- |
| A    | `penny-dev.lab`         | `192.168.1.250` | DNS only |

Proxying must stay off: Cloudflare's proxy does not carry raw TCP on 5432, and
the target is a private address reachable only over the tailnet.

`192.168.1.250` is `deus`. This is Traefik's own LoadBalancer address, the same
one every other `*.lab` record points at, so nothing here is specific to the
database. Traefik answers on 5432 from any of its node IPs, so if deus is down
you can point a client at another one by hand.
