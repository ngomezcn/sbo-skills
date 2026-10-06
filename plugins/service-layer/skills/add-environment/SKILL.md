---
name: add-environment
description: Add one more environment (dev, uat or prod) to an existing Service Layer setup without losing what is already configured, then test its login. Use when the developer asks to add, create or connect another environment, or runs /service-layer:add-environment. Needs the setup done first. The developer types the connection data into a file themselves, this skill never handles it.
---

# Service Layer: add an environment

Adds one environment to a setup that already exists. What is configured stays as it is. **The connection data (Service Layer URL, company database, user name and password) is sensitive: the developer types it into a file in the repo.** You never see it: do not ask for it, do not read the file.

## 1. Which environment

If the developer did not say which one, run `node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --status` and ask which of `dev`, `uat` and `prod` they want to add, one question only. Do not offer the ones `--status` lists as configured.

## 2. Add it

```
node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --add-env uat
```

- `SETUP_MISSING`: there is no setup yet. Tell the developer to run `/service-layer:setup` first.
- `ENVIRONMENT_EXISTS`: it already exists. Tell them to edit its `credentials.json`, or to run `/service-layer:setup` to start over (that erases everything).

The answer lists the new file under `fillIn`, with all four fields empty.

## 3. The developer fills in the connection data

Tell the developer to open that file, relative to the repo root, and write the four fields: Service Layer URL (for example `https://host:50000`), company database, user name and password. Then save. Include this notice:

> **Privacy:** this data never leaves your machine. Only the Service Layer tool reads it, to connect to your server. The AI agent cannot see it and does not read this file.

Tell them to say when they are done. Do not open the file.

## 4. Verify

Run `node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --verify`. It tests every environment and makes the entity index. Read the answer as the setup skill does: `verified`, `indexed`, `pending` (fields still empty, named without values), `failed` (Service Layer `code` and `message`: `-304` wrong user or password, `-306` unknown company, `SL_UNREACHABLE` bad URL). On `pending` or `failed`, ask the developer to correct the file and run `--verify` again.

Done when `ok` is true. Tell the developer the new environment is ready.

## Rules

- Never ask for, accept or repeat the URL, the company database, a user name or a password in the chat. If the developer pastes one, do not repeat or store it; tell them to type it into the file instead and consider the password exposed.
- Never read, print or edit `.sbo-skills/service-layer/*/credentials.json`, and never write it by hand.
