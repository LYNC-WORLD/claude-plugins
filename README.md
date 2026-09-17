# LYNC Claude plugins

Claude skills and plugins for building on Canton Network with LYNC developer tooling.

## Install

Claude Code:

```
claude plugin marketplace add LYNC-WORLD/claude-plugins
claude plugin install canton-deploy@lync
```

Cowork (Claude desktop app): Customize → Plugins → Add marketplace → `https://github.com/LYNC-WORLD/claude-plugins` → Install **canton-deploy**.

Plain skill, no plugin: copy `canton-deploy/skills/canton-deploy/` into `~/.claude/skills/` (or your project's `.claude/skills/`).

## Plugins

| Plugin | What it does | Docs |
|---|---|---|
| `canton-deploy` | Deploy Daml packages to a Canton participant — LocalNet sandbox, DevNet, TestNet, MainNet — with `dpm canton-deploy`: build, upload, vet, allocate parties, create users, run Daml Scripts, inspect the ledger. | [docs.lync.world](https://docs.lync.world/docs/CANTON/deploy/canton-deploy) · [source](https://github.com/LYNC-WORLD/canton-deploy) |

## Layout

```
.claude-plugin/marketplace.json     # the "lync" marketplace
canton-deploy/
  .claude-plugin/plugin.json
  skills/canton-deploy/SKILL.md
  skills/canton-deploy/references/
```

## License

Apache-2.0
