# canton-deploy troubleshooting

**`dpm install package` says `…/components/canton-deploy:0.1.1: not found`** — a bare `canton-deploy:0.1.1` resolves against Digital Asset's registry (`europe-docker.pkg.dev/da-images`). Use `oci://ghcr.io/lync-world/canton-deploy:0.1.1` until the component is published there.

**Every `dpm` command fails with `component "…" is currently not installed`, even `dpm --help`** — while any component listed in `daml.yaml` is not installed, DPM refuses all commands in that directory. Run `dpm install package`, or fix/remove the offending line.

**`dpm canton-deploy` not found** — the component is not installed for this project. Add it under `components:` and run `dpm install package`.

**`dpm sandbox` is an unknown command** — it comes from the `canton-open-source` component. Add `- canton-open-source:<version>` under `components:` and run `dpm install package`.

**`status` cannot reach Admin API** — check `adminPort` (and `adminGrpcAuthority` if a proxy routes Admin gRPC by name). Upload uses the Admin API.

**Token expired / unauthenticated** — `dpm canton-deploy token --decode`, then refresh `token`, `tokenFile`, or `tokenCommand`.

**`PROTO_DESERIALIZATION_FAILURE` on upload** — `synchronizerId` must be logical (`namespace::fingerprint`). Drop a trailing `::NN-N` from `status`, or omit the field on a single-synchronizer participant.

**`run` cannot reach `grpcAuthority`** — that name must resolve to the validator on `ledgerPort`, or set `host` and `grpcAuthority` to the same reachable name.

**`contracts` returns `PACKAGE_NAMES_NOT_FOUND`** — use `#<package-name>:Module:Template` from `daml.yaml`, not a hex package id.

**`contracts` is empty after `deploy`** — run a script (`deploy --script` or `run`).

**Script fails with `Party already exists` on `allocatePartyByHint`** — the name is also in config `parties`, so `deploy` allocated it first. Look the party up in the script, or remove it from `parties`.

**JSON API unreachable, Admin and Ledger OK** — upload can still succeed. Fix `httpPort` / `httpHost` / `httpUseTls` for `contracts` and a full `status`.
