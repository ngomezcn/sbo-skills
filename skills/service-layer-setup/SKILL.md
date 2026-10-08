---
name: service-layer-setup
description: Set up or reset the connection to SAP Business One Service Layer for this repo - language of the skills (English or Spanish), SAP Business One version, OData version and the dev, uat or prod environments, with a login test. Use when the developer asks to configure, reconfigure or reset Service Layer, or when a Service Layer command fails with SETUP_MISSING or ENVIRONMENT_NOT_CONFIGURED. Conversational; the developer types the connection data (URL, company database, user, password) into a file themselves, this skill never handles them.
---

# Service Layer: setup

Four questions, then two commands, in six visible steps. You ask only about language, versions and environments. **The connection data (Service Layer URL, company database, user name and password) is sensitive: the developer types it into a file in the repo.** You never see it: do not ask for it, do not read the file.

## 1. Look at the repo

Before asking, check what already exists:

- `.sbo-skills/service-layer/` present: run `node "${CLAUDE_SKILL_DIR}/scripts/setup.mjs" --status` (answers `versionB1`, `versionOData`, `language`, `environments`, never credentials). Tell the developer what is configured and that going on **erases it**, with its data and object contexts. Ask before continuing.
- Nothing there: a fresh setup.

## 2. Ask, one question per message

Ask one question, wait for the answer, then ask the next. Never put two questions in one message. Say "SAP Business One", never "B1", when you talk to the developer.

**Make the progress visible.** Every message of the setup starts with a short title that says the step, so the developer always knows where they are. The steps are:

1. Language · 2. SAP Business One version · 3. OData version · 4. Environments · 5. Connection data · 6. Verification

Format (translate the labels into the chosen language):

```
━━━ Step 2 of 6 · SAP Business One version ━━━
```

No progress bars and no emojis anywhere during the setup. Confirm each answer in one plain line (for example "Language: English"). The only emojis allowed are the green ticks `✅` in the final summary (section 5).

**Offer choices as lists, never as open questions.** Use the `AskUserQuestion` tool when it is available (single choice, the recommended option first with "(Recommended)"); otherwise write a numbered list and tell them to answer with the number. Never make them type "ok" to accept a default: the default is just the first option of the list.

1. **Language.** This comes first and is part of the setup. The language is not known yet, so this one screen is bilingual: title `Step 1 of 6 · Language / Idioma`, then the list `1. English`, `2. Español`. Only `en` and `es` are supported. From their answer on, talk to the developer in that language, in this skill and in the others of this plugin (the setup saves it as `language`). Everything below is written in English; translate it, keep the commands, file names and field names as they are.
2. **SAP Business One version.** Show the whole tested list as a numbered list, not only an example: `FP 2208`, `FP 2305`, `SP 2308`, `SP 2311`, `SP 2402`, `FP 2405`, `SP 2408`, `SP 2411`, `FP 2502`, `SP 2505`, `FP 2508`, `SP 2511`, `FP 2602`, `SP 2605`, `FP 2608`. Add a last option "Other" for one outside the list. Say that it is accepted, but that other versions may fail in the responses even though it should work. (Too long for `AskUserQuestion`: use the numbered list.)
3. **OData version.** A list of two options, shown with the Service Layer path so they know what we mean: `b1s/v2` (OData V4) marked "(Recommended, default)" and first, and `b1s/v1` (OData V3). The value passed to `--odata` stays `v2` or `v1`. If their version is older than `FP 2405`, make `v1` the recommended one instead and say why in one line.
4. **Environments.** Recommend starting with `dev` only. Tell them they can add another environment whenever they want with `/service-layer-add-environment`, which keeps what is already configured. Then a list to pick from: `dev` (Recommended), `uat`, `prod`; they may pick several, at least one (`AskUserQuestion` with multi-select).

## 3. Initialise

Run, with the environments as a comma-separated list (no summary or approval step; nothing here is secret or costly to redo):

```
node "${CLAUDE_SKILL_DIR}/scripts/setup.mjs" --init --b1 "FP 2608" --odata v2 --language es --envs dev,uat
```

It starts from scratch, writes `.sbo-skills/service-layer/config.md`, adds `.sbo-skills/` to `.gitignore`, and creates one `credentials.json` per environment with example values: `url` `https://localhost:50000/`, `companyDB` `SBODemoES`, `userName` `manager` and `password` `your-password-here`. Its answer lists those files under `fillIn`.

## 4. The developer fills in the connection data

Heading: `Step 5 of 6 · Connection data`. Tell the developer to open each file listed in `fillIn`, relative to the repo root, and replace the example values with their own, as a checklist with the file path in bold:

- `url`: the Service Layer URL
- `companyDB`: the company database
- `userName`: the user name
- `password`: the password. It must be changed, otherwise the environment stays pending.

Then save. Include this notice:

> **Privacy:** this data never leaves your machine. Only the Service Layer tool reads it, to connect to your server. The AI agent cannot see it and does not read these files.

Tell them to say when they are done. Do not open the files.

If the answer has a warning about `.gitignore`, ask them to add the line `.sbo-skills/` themselves: the folder holds passwords in plain text.

## 5. Verify

Heading: `Step 6 of 6 · Verification`. Run `node "${CLAUDE_SKILL_DIR}/scripts/setup.mjs" --verify`. It tests the login of every environment, discards the test session and makes the entity index. The answer has no credentials:

- `verified`: logins that worked. `indexed`: environments with their index made (a failure there is a warning; the Uso makes it on its next command).
- `pending`: environments whose file still has an empty field or the example password (it names the fields, not values). Ask the developer to finish them.
- `failed`: environments whose login failed, with the Service Layer `code` and `message` (for example `-304` wrong user or password, `-306` unknown company, `SL_UNREACHABLE` bad URL). Say what it means and ask the developer to correct the file. Then `--verify` again.

Done when `ok` is true. Close with a short summary block ("Setup complete"): language, SAP Business One version, OData version and the ready environments, each with a green `✅` (the only place emojis are used), and a pointer to `/service-layer-add-environment`.

## Rules

- To add an environment to an existing setup, send the developer to `/service-layer-add-environment`. Never run `--init` for that: it erases everything.
- Never ask for, accept or repeat the URL, the company database, a user name or a password in the chat. If the developer pastes one, do not repeat or store it; tell them to type it into the file instead and consider the password exposed.
- Never read, print or edit `.sbo-skills/service-layer/*/credentials.json`, and never write `config.md` or `credentials.json` by hand.
- Certificates are never validated (self-signed ones are the norm). The developer should work from a network they trust.

## Limits of this design

- The password stays in plain text in `.sbo-skills/service-layer/<environment>/credentials.json` (ignored by git). Anyone who can read the repo folder can read it.
- Keeping the connection data out of the AI context depends on this skill's rules: the files are readable by your tools, nothing technical prevents it.
