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

**Why a LoadBalancer and not a Traefik route.** Postgres does not open with a
TLS ClientHello; it sends a plaintext `SSLRequest` first and upgrades in band.
Traefik has no SNI to match on, and `HostSNI(*)` in a TCP router only applies to
non-TLS traffic, so hostname-based routing of Postgres through Traefik is not
possible. The name has to resolve straight to an L4 listener.

**Why source-IP rules are absent from pg_hba.** K3s ServiceLB masquerades
off-cluster connections to the ingress node's flannel address. A connection made
from a Tailscale peer to `192.168.1.250:5432` reaches Postgres from `10.42.0.0`,
identical to in-cluster traffic. Rules like `host penny penny 192.168.1.0/24`
therefore never match and give false assurance; the old config had exactly that
rule. Network scoping is Tailscale's job. pg_hba's job is
`hostnossl all all all reject`, which stops a client from handing its password
over in the clear.

## Required DNS record

Not managed by this repo. In Cloudflare, zone `packetcraft.dev`:

| Type | Name                    | Content       | Proxy    |
| ---- | ----------------------- | ------------- | -------- |
| A    | `penny-dev.lab`         | `192.168.1.250` | DNS only |

Proxying must stay off: Cloudflare's proxy does not carry raw TCP on 5432, and
the target is a private address reachable only over the tailnet.

`192.168.1.250` is `deus`, the only node labelled
`svccontroller.k3s.cattle.io/lbpool=deus`, which is what `lb-service.yaml` pins
the listener to. If that record is ever repointed, move the label with it.
