---
name: setup
description: Set up or reset the connection to SAP Business One Service Layer for this repo - B1 version, OData version and the dev, uat or prod environments, with a login test. Use when the developer asks to configure, reconfigure or reset Service Layer, or when a Service Layer command fails with SETUP_MISSING or ENVIRONMENT_NOT_CONFIGURED. Conversational; the developer types the user and the password into a file themselves, this skill never handles them.
---

# Service Layer: setup

A conversation, then two commands. You ask everything that is not secret in the chat; the **user name and the password** the developer types into a file in the repo. **You never see them**: do not ask for them, do not read the file.

## 1. Look at the repo

Before asking, check what already exists, so you can propose instead of interrogate:

- `.sbo-skills/service-layer/` present: run `node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --status` (answers `versionB1`, `versionOData`, `environments`, never credentials). Tell the developer what is configured and that going on **erases it**, with its data and object contexts. Ask before continuing.
- Nothing there: a fresh setup.

## 2. Ask, one section at a time

Propose a default when there is one; let the developer correct it.

1. **B1 version**, for example `FP 2608`. One outside the tested list is accepted with a warning.
2. **OData version**: `v2` (OData V4) from FP 2405, `v1` (OData V3) before. Propose the one that fits the B1 version; the developer decides, also one the B1 version may not support.
3. **Environments**: which of `dev`, `uat`, `prod` they want. At least one.
4. For each: the **Service Layer URL** (for example `https://host:50000`) and the **company database**. Both are fine in the chat.

## 3. Confirm, then initialise

Show a short summary (B1, OData, and URL + company per environment) and ask for approval. Then run, one `--<env>-url`/`--<env>-company` pair per environment:

```
node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --init --b1 "FP 2608" --odata v2 --dev-url https://host:50000 --dev-company DB
```

It starts from scratch, writes `.sbo-skills/service-layer/config.md`, adds `.sbo-skills/` to `.gitignore`, and creates one `credentials.json` per environment with the URL and company filled in and `userName` and `password` **empty**. Its answer lists those files under `fillIn`.

## 4. The developer fills in the secrets

Tell the developer to open each file listed in `fillIn` in their editor and write `userName` and `password`, then save. Give the paths relative to the repo root. Wait until they say they are done. Do not open the files.

If the answer has a warning about `.gitignore`, ask them to add the line `.sbo-skills/` themselves: the folder holds passwords in plain text.

## 5. Verify

Run `node "${CLAUDE_PLUGIN_ROOT}/dist/setup.mjs" --verify`. It tests the login of every environment, discards the test session and makes the entity index. The answer has no credentials:

- `verified`: logins that worked. `indexed`: environments with their index made (a failure there is a warning; the Uso makes it on its next command).
- `pending`: environments whose file still has empty fields (it names the fields, not values). Ask the developer to finish them.
- `failed`: environments whose login failed, with the Service Layer `code` and `message` (for example `-304` wrong user or password, `-306` unknown company, `SL_UNREACHABLE` bad URL). Say what it means. A wrong URL or company you can fix by running `--init` again with the right values (it starts over, so the developer fills the files in again); a wrong user or password the developer fixes in the file. Then `--verify` again.

Done when `ok` is true. Tell the developer which environments are ready.

## Rules

- Never ask for, accept or repeat a user name or a password in the chat. If the developer pastes one, do not repeat or store it; tell them to type it into the file instead and consider it exposed.
- Never read, print or edit `.sbo-skills/service-layer/*/credentials.json`, and never write `config.md` or `credentials.json` by hand.
- Certificates are never validated (self-signed ones are the norm). The developer should work from a network they trust.

## Limits of this design

- The password stays in plain text in `.sbo-skills/service-layer/<environment>/credentials.json` (ignored by git). Anyone who can read the repo folder can read it.
- URL and company pass through the conversation; only the user and the password stay out of it.
- Keeping the secrets out of the AI context depends on this skill's rules: the files are readable by your tools, nothing technical prevents it.
