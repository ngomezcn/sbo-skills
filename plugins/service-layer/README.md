# service-layer

Claude Code plugin to work with a real SAP Business One Service Layer. It is installed whole and has three parts:

| Skill | What it does |
|---|---|
| `setup` | The developer runs a script in their own terminal: B1 version, OData version (`v1` or `v2`) and the credentials of `dev`, `uat` and `prod` (as many as wanted), with a login test. The credentials never enter the AI context. |
| `use` | Reads (by key, a page, a full traversal, a count), builds the object context of an entity, and writes (POST, PATCH, DELETE). Writes are dry runs until the same call is repeated with `--execute`; `prod` also needs `--allow-prod`. |
| `docs` | Service Layer reference valid for all B1 versions. Needs the Setup (reads the B1 version from `config.md`). |

Requires Node 20 or later. Local data lives in `.sbo-skills/service-layer/` of the developer's repo (the Setup adds it to `.gitignore`): `config.md`, and per environment `credentials.json` (plain text), the session, the object contexts and the Volcados.

The certificate of the Service Layer is never validated (they are usually self-signed): work from a network you trust.

## Permissions (recommended)

Reads are safe to allow so that Claude Code does not ask each time, for example in `.claude/settings.json`:

```json
{ "permissions": { "allow": [
  "Bash(node *service-layer*/dist/use.mjs get *)",
  "Bash(node *service-layer*/dist/use.mjs page *)",
  "Bash(node *service-layer*/dist/use.mjs traverse *)",
  "Bash(node *service-layer*/dist/use.mjs count *)",
  "Bash(node *service-layer*/dist/use.mjs context *)",
  "Bash(node *service-layer*/dist/use.mjs clean *)",
  "Bash(node *service-layer*/dist/setup.mjs --status)"
] } }
```

Do **not** allow `post`, `patch` or `delete`: the permission prompt they raise is the second guard of the write flow (ADR 0008), and it cannot tell a dry run from `--execute`, so each of them is asked. Adjust the path pattern to where the plugin is installed.

`dist/` and `skills/` are generated from the factory repository; do not edit them here.
