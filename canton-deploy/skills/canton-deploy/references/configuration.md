# canton-deploy configuration and authentication

Settings live in `canton-deploy.config.js` at the project root (or a parent directory). Select a named network with `--network`. Run `init` or copy [`canton-deploy.config.example.js`](./canton-deploy.config.example.js).

If the nearest `package.json` has `"type": "module"`, name the file `canton-deploy.config.cjs` instead.

```js
module.exports = {
  defaultNetwork: 'localnet',
  networks: {
    localnet: {
      host: 'localhost',
      adminPort: 5002,
      ledgerPort: 5001,
      httpPort: 7575,
      vetOnUpload: true,
      excludePackages: ['./tests', 'my-app-tests'],
      additionalDars: [],
      parties: ['Alice', 'Bob'],
      users: [{
        userId: 'ledger-api-user',
        parties: ['Alice', 'Bob'],
        rights: ['CanActAs', 'CanReadAs'],
      }],
    },
    devnet: {
      host: 'validator.example.com',
      adminPort: 5002,
      ledgerPort: 5011,
      httpPort: 8080,
      token: process.env.DEVNET_JWT_TOKEN,
      vetOnUpload: true,
      parties: ['Operator'],
      users: [{
        userId: 'app-operator',
        parties: ['Operator'],
        rights: ['CanActAs', 'CanReadAs'],
      }],
    },
  },
};
```

CLI flags override environment variables, which override the config file.

| Key | What it controls |
| --- | --- |
| `host` | Validator host or IP |
| `adminPort` | Admin gRPC port (default `5002`) |
| `ledgerPort` | Ledger gRPC port (default `5001`) |
| `httpPort` | HTTP JSON API port (default `7575`) |
| `grpcAuthority` | gRPC `:authority` for the Ledger API when a proxy routes on name |
| `adminGrpcAuthority` | Same for the Admin API |
| `httpHost` | HTTP `Host` header for the JSON API |
| `httpUseTls` | Use `https` for JSON API calls |
| `token` / `tokenFile` / `tokenCommand` | JWT source (see [Authentication](#authentication)) |
| `tls` / `tlsCertFile` | TLS for gRPC; optional CA file |
| `synchronizerId` | Logical synchronizer id (`namespace::fingerprint`). Required when the participant has more than one synchronizer. Use the logical id from `status`, not a trailing `::NN-N` suffix. |
| `vetOnUpload` | Vet during upload. Defaults **on** for `localnet`, **off** otherwise. Override with `--vet` / `--no-vet`. |
| `additionalDars` | Extra DAR paths uploaded before project DARs |
| `includePackages` / `excludePackages` | Filter packages from `daml.yaml` / `multi-package.yaml` |
| `parties` | Display names allocated on `deploy` (skipped if they already exist) |
| `users` | Users created on `deploy` with `CanActAs` / `CanReadAs` for their parties |
| `scriptUserId` | `--user-id` passed to `dpm script` (otherwise JWT `sub`) |

Vendored DARs go in `additionalDars` or `--dar`. `data-dependencies` are not uploaded on their own.

DAR upload uses the **Admin API**, so `adminPort` must be reachable.

## Authentication

The JWT is resolved in this order:

1. `--token`
2. `CANTON_DEPLOY_TOKEN`
3. config `token`
4. config `tokenCommand` (a shell command whose stdout is the token)
5. config `tokenFile`
6. LocalNet development HMAC (network name `localnet` only)

```bash
dpm canton-deploy token --decode --network localnet
dpm canton-deploy token --show --network devnet
```

## Environment variables

| Variable | Purpose |
| --- | --- |
| `CANTON_DEPLOY_NETWORK` | Default `--network` |
| `CANTON_DEPLOY_HOST` | Override `host` |
| `CANTON_DEPLOY_ADMIN_PORT` / `LEDGER_PORT` / `HTTP_PORT` | Ports |
| `CANTON_DEPLOY_TOKEN` | JWT |
| `CANTON_DEPLOY_HTTP_HOST` / `GRPC_AUTHORITY` / `ADMIN_GRPC_AUTHORITY` | Proxy name overrides |
| `CANTON_DEPLOY_HTTP_USE_TLS` | `true` / `false` |
| `CANTON_DEPLOY_CONFIG` | Path to the config file |
| `CANTON_DEPLOY_SCRIPT_USER_ID` | User id for `dpm script` |
| `CANTON_DEPLOY_GRPC_DEADLINE_MS` | Per-RPC deadline (default `60000`) |
| `CANTON_DEPLOY_GRPC_CONNECT_MS` | Channel ready wait (default `10000`) |
