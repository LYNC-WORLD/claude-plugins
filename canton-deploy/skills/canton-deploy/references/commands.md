# canton-deploy command reference

Shared flags: `--network`, `--host`, `--admin-port`, `--ledger-port`, `--http-port`, `--grpc-authority`, `--http-host`, `--token`, `--log-file`.

Exit code `0` is success. `--log-file` appends output for CI.

| Command | What it does |
| --- | --- |
| `deploy` | Build (unless `--skip-build`), upload DARs, optionally vet, onboard parties/users, optional `--script` |
| `vet` | Vet the same DAR set `deploy` would upload |
| `vet-dar <mainPackageId>` | Vet one already-uploaded DAR by main package id |
| `dars` | List uploaded DARs (Admin API) |
| `packages` | List known packages (Ledger API) |
| `status` | Check Admin, Ledger, and JSON API connectivity |
| `parties` | List known parties |
| `allocate-party <name>` | Allocate a party by display name if it does not already exist |
| `users` | List participant users |
| `create-user --user-id` | Create a user from config `users[]` and grant rights |
| `run <Module:fn>` | Run a Daml Script (`dpm script`) |
| `contracts` | List active contracts (JSON API) |
| `token` | Show or decode the resolved JWT |
| `init` | Write `canton-deploy.config.js` |

### deploy

```bash
dpm canton-deploy deploy --network localnet
dpm canton-deploy deploy --network localnet --dar ./vendor/token-standard.dar
dpm canton-deploy deploy --network localnet --script My.Module:setup
dpm canton-deploy deploy --network devnet --skip-build --dry-run
dpm canton-deploy deploy --network localnet --no-vet
```

| Flag | Meaning |
| --- | --- |
| `--dar <path>` | Extra DAR (repeatable) |
| `--skip-build` | Use existing `.daml/dist` or `.dpm/dist` artifacts |
| `--vet` / `--no-vet` | Override `vetOnUpload` |
| `--dry-run` | Print the DAR set; do not upload |
| `--script <Module:fn>` | Run a Daml Script after upload |

Upload does not create contracts. Use `--script` or `run` to seed the ledger.

A party name has one owner: either the config `parties` list or your script, not both. `deploy` allocates config parties before running `--script`, and Canton rejects a second `allocatePartyByHint` with the same hint. If a name is in `parties`, have the script look the party up instead of allocating it (filter `listKnownParties` by the `Alice::` prefix, or pass party ids in with `--input-file`); if the script allocates it, leave it out of `parties`.

### vet / vet-dar

```bash
dpm canton-deploy vet --network localnet
dpm canton-deploy vet --network localnet --skip-build
dpm canton-deploy vet-dar <mainPackageId> --network localnet
```

By default vetting waits until it is observed on the synchronizer. `--no-sync` returns as soon as the request is accepted.

### dars / packages

```bash
dpm canton-deploy dars --network localnet
dpm canton-deploy packages --network localnet
```

### status

```bash
dpm canton-deploy status --network localnet
```

### parties / allocate-party

```bash
dpm canton-deploy parties --network localnet
dpm canton-deploy parties --network localnet --local
dpm canton-deploy allocate-party Alice --network localnet
```

| Flag | Meaning |
| --- | --- |
| `--local` | Only parties hosted on this participant |
| `--filter-party <prefix>` | Prefix filter |
| `--party <ids>` | Comma-separated party ids to look up |
| `--limit <n>` / `--page-token` | Pagination |

Config `parties` are display names (for example `Alice`). The name is used as the party id hint exactly as written, so `Alice` becomes `Alice::1220…` — the same party a Daml Script gets from `allocatePartyByHint "Alice"`. An existing party with that hint is reused.

### users / create-user

```bash
dpm canton-deploy users --network localnet
dpm canton-deploy create-user --network localnet --user-id ledger-api-user
```

`--user-id` must match a `users[]` entry on that network.

### run

```bash
dpm canton-deploy run My.Module:setup --network localnet
dpm canton-deploy run Setup:seed --network devnet --dar .daml/dist/my-app-0.1.0.dar --input-file seed.json
```

Parties listed in the config are already allocated when the script runs; look them up rather than calling `allocatePartyByHint` for the same name (see [deploy](#deploy)).

`dpm script` uses `--ledger-host` for both TCP and gRPC `:authority`. If `grpcAuthority` differs from `host`, that name must resolve and accept connections on `ledgerPort`.

### contracts

```bash
dpm canton-deploy contracts --network localnet
dpm canton-deploy contracts --network localnet --party 'Alice::1220...'
dpm canton-deploy contracts --network localnet --party 'Alice::1220...' --template '#my-package:Module:Template'
```

`--template` is `#<package-name>:Module:Template` from `daml.yaml`, not the hex package id from `deploy`. The JWT user needs `CanReadAs` for `--party`.

### token

```bash
dpm canton-deploy token --network localnet
dpm canton-deploy token --decode --network localnet
dpm canton-deploy token --show --network devnet
```

### init

```bash
dpm canton-deploy init
```

Writes `canton-deploy.config.js` with a LocalNet profile and an optional DevNet profile. It asks whether to add DevNet and whether to accept the defaults for each profile; answer No to enter host, ports, TLS, and token source by hand.
