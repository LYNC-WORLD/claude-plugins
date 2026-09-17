---
name: canton-deploy
description: Deploy Daml packages (DARs) to a Canton Network participant or validator — LocalNet sandbox, DevNet, TestNet or MainNet — using the canton-deploy DPM component (`dpm canton-deploy`). Use this whenever someone wants to deploy, upload, publish or push a Daml contract/package/DAR to Canton, set up a Daml project's deployment, allocate parties or create users on a participant, run a Daml Script against a live node, vet packages, or inspect what is on a Canton ledger — even if they only say "get my Daml code onto the validator", mention `daml.yaml`, `dpm`, Splice, or a Canton validator, or ask how to script or automate Canton deployments in CI.
compatibility: Requires DPM 1.0.20+ (Digital Asset Package Manager) and Node.js 18+ on the machine running the commands, plus network access to the target participant's Admin, Ledger and JSON APIs.
---

# Deploying Daml packages to Canton with canton-deploy

canton-deploy is a DPM component that turns the manual Canton deployment routine — `dpm build`, find the DAR, upload it over the Admin API, vet it, allocate parties, create users, grant rights, check it landed — into one command that runs from the Daml project root. It is invoked as `dpm canton-deploy`. Source and issues: https://github.com/LYNC-WORLD/canton-deploy. Docs: https://docs.lync.world/docs/CANTON/deploy/canton-deploy.

Prefer it over hand-written Admin API calls, console scripts, or `curl` against the JSON API when the user wants to get a package onto a participant. It handles the parts people get wrong: nginx-fronted validators that route by `:authority`, JWT resolution, synchronizer ids, idempotent party/user onboarding.

## The shape of a deployment

1. Add the component to the project's `daml.yaml` and install it.
2. Make sure a participant is reachable (for local work, start `dpm sandbox`).
3. `dpm canton-deploy init` writes `canton-deploy.config.js` with one profile per network.
4. `dpm canton-deploy status --network <name>` proves Admin, Ledger and JSON APIs are reachable and authenticated.
5. `dpm canton-deploy deploy --network <name>` builds, uploads, vets, and onboards parties and users. Add `--script Module:fn` to seed the ledger.
6. Inspect with `dars`, `packages`, `parties`, `users`, `contracts`.

Walk the user through these in order, and run `status` before `deploy` — almost every failure is connectivity or auth, and `status` isolates it in seconds.

## 1. Install

From the Daml project root (the directory with `daml.yaml`, or the `multi-package.yaml` root):

```yaml
components:
  - damlc:3.5.2
  - daml-script:3.5.2
  - oci://ghcr.io/lync-world/canton-deploy:0.1.1
```

```bash
dpm install package
dpm canton-deploy --help
```

Things to know so you don't send the user down a dead end:

- Use the full `oci://ghcr.io/lync-world/canton-deploy:<tag>` reference. A bare `canton-deploy:0.1.1` makes DPM look in Digital Asset's registry, where the component is not (yet) published, and fails with `not found`.
- `dpm add component ghcr.io/lync-world/canton-deploy:0.1.1` is the one-step alternative to editing `daml.yaml`; it needs the full registry reference too.
- DPM does not allow `sdk-version` and `components:` in the same `daml.yaml`; pin `damlc` and `daml-script` in `components:` instead. Component versions (e.g. `3.5.2`) are not the same as the SDK release number — `ls ~/.dpm/cache/components/damlc/` shows the exact strings available.
- On first install DPM rewrites the line with an `@sha256:…` digest. That is expected; leave it.
- While any listed component is uninstalled, every `dpm` command in that directory fails, including `dpm --help`. Fix the `components:` line or run `dpm install package` before anything else.
- Check the latest tag at https://github.com/orgs/LYNC-WORLD/packages/container/package/canton-deploy if `0.1.1` looks old.

## 2. A participant to deploy to

For local development, `dpm sandbox` gives a full Canton participant in one process. It comes from the `canton-open-source` component, so add it alongside the others (for example `- canton-open-source:3.5.17`), run `dpm install package`, then in its own terminal:

```bash
dpm sandbox --ledger-api-port 5001 --admin-api-port 5002 --json-api-port 7575
```

It takes about 30 seconds and prints `Canton sandbox is ready.` Those ports are what the `localnet` profile expects by default.

For DevNet/TestNet/MainNet the user brings their own validator. Ask for: host (often a Tailscale or VPN address), which ports expose the Admin, Ledger and JSON APIs, whether a reverse proxy (nginx is common on Splice validators) fronts them and routes by hostname, and where the JWT comes from. `status` will tell you which of those is wrong.

## 3. Configure

`dpm canton-deploy init` asks whether to add a DevNet profile alongside LocalNet and whether to accept the defaults for each; answering No lets the user type their own values. The result is `canton-deploy.config.js` (`.cjs` if the nearest `package.json` has `"type": "module"`). Precedence is CLI flags → environment variables → config file.

A minimal LocalNet profile:

```js
localnet: {
  host: 'localhost', adminPort: 5002, ledgerPort: 5001, httpPort: 7575,
  vetOnUpload: true,
  parties: ['Alice', 'Bob'],
  users: [{ userId: 'ledger-api-user', parties: ['Alice', 'Bob'], rights: ['CanActAs', 'CanReadAs'] }],
}
```

A validator behind a name-routing proxy on a single port looks like this — all three APIs on the same port, distinguished by the name each request carries:

```js
devnet: {
  host: 'validator.example.com',
  ledgerPort: 81, grpcAuthority: 'grpc-ledger-api.localhost',
  adminPort: 81,  adminGrpcAuthority: 'grpc-admin-api.localhost',
  httpPort: 81,   httpHost: 'json-ledger-api.localhost', httpUseTls: false,
  tls: false,
  tokenFile: './.tokens/devnet.jwt',
  synchronizerId: 'global-domain::1220…',   // logical id from `status`, without the trailing ::NN-N
  parties: ['Operator'],
  users: [{ userId: '<jwt sub>', parties: ['Operator'], rights: ['CanActAs', 'CanReadAs'] }],
}
```

Authentication: on a network named `localnet` the tool mints a development HMAC JWT (Splice's `unsafe` secret, user `ledger-api-user`) so nothing needs configuring. Every other network needs a real JWT, resolved in this order: `--token` → `CANTON_DEPLOY_TOKEN` → config `token` → `tokenCommand` (shell command whose stdout is the token) → `tokenFile`. Keep token files out of git. `dpm canton-deploy token --decode --network <name>` shows `sub`, `aud` and expiry of whatever will be sent — use it first when auth fails.

Full key table and env vars: read `references/configuration.md`.

## 4. Deploy

```bash
dpm canton-deploy status --network localnet
dpm canton-deploy deploy --network localnet
dpm canton-deploy deploy --network localnet --script Main:setup   # also seed the ledger
dpm canton-deploy contracts --network localnet
```

`deploy` builds (skip with `--skip-build` when CI already ran `dpm build`), uploads the DAR set over the Admin API, vets it (default on for `localnet`, off elsewhere; `--vet`/`--no-vet` override), allocates config `parties` and creates config `users` with their rights. Re-running is safe: existing parties and users are reported as existing, not re-created. `--dry-run` prints the DAR set without uploading; `--dar path` adds vendored DARs.

One rule that bites people: **a party name has one owner**. `deploy` allocates config `parties` before running `--script`, and Canton rejects a second `allocatePartyByHint` with the same hint. So either the config allocates the party and the script looks it up (`listKnownParties`, filter by the `Alice::` prefix, or pass ids via `--input-file`), or the script allocates it and the name stays out of `parties`. Never both. The party id keeps the name's case: `Alice` → `Alice::1220…`.

Uploading a DAR creates no contracts. If `contracts` is empty after `deploy`, that is expected until a script runs.

## 5. Inspect and operate

| Need | Command |
|---|---|
| What's uploaded | `dars` (Admin API), `packages` (Ledger API) |
| Vet after the fact | `vet`, `vet-dar <mainPackageId>` |
| Parties | `parties`, `parties --filter-party <prefix>`, `allocate-party <Name>` (idempotent) |
| Users | `users`, `create-user --user-id <id>` (from config `users[]`) |
| Run a Daml Script | `run Module:fn --network <name> [--dar …] [--input-file …]` |
| Active contracts | `contracts [--party 'Alice::1220…'] [--template '#pkg-name:Module:Template']` |
| What JWT is used | `token --decode` |

`--template` takes the package **name** from `daml.yaml` (`#my-package:Module:Template`), not the hex package id. On networks with thousands of parties prefer `--filter-party` over `--local`, which scans client-side.

Every command takes `--network`, `--host`, `--admin-port`, `--ledger-port`, `--http-port`, `--grpc-authority`, `--http-host`, `--token`, `--log-file`; exit code 0 is success, so it drops into CI directly. Full per-command flags: `references/commands.md`.

## When something fails

`status` first, then `token --decode`. The specific messages and fixes — registry `not found`, "component is currently not installed", `dpm sandbox` unknown, `PROTO_DESERIALIZATION_FAILURE` (synchronizer id form), `PACKAGE_NAMES_NOT_FOUND` (template id form), `Party already exists` (the one-owner rule), unreachable `grpcAuthority` — are in `references/troubleshooting.md`. Read it before improvising a fix.

## Don't

- Don't run party-allocating scripts (`run`, `deploy --script`) against DevNet/TestNet/MainNet just to "test" — those parties are permanent on a shared network. Test scripts on `dpm sandbox`.
- Don't put a JWT in `canton-deploy.config.js` or commit `.tokens/`; use `tokenFile`, `tokenCommand` or an env var.
- Don't fall back to hand-rolled gRPC/JSON API calls when `status` fails; fix the profile (ports, `*Authority`, `httpHost`, TLS) — that is the actual problem.
