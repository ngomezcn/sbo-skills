# Service Layer Skills For SAP Business One

A Claude Code plugin to work with a real SAP Business One Service Layer: read data, change it safely and look up how the Service Layer behaves, without ever handing your user and password to the AI.

Working against a live ERP is hard. Pasting passwords into a chat, letting an agent guess entity names, or having it fire a `POST` at production on its own are all ways to have a very bad day. These skills are designed to be small, explicit and safe by default: every write is shown to you before it is sent.

## Installation (30-second setup)

This is a **Claude Code** plugin. It needs Claude Code and Node 20 or later; other agents are not supported for now.

### 1. Install the plugin

In Claude Code:

```
/plugin marketplace add ngomezcn/sbo-skills
/plugin install service-layer@sbo-skills
```

It installs the three skills (`setup`, `use`, `docs`) as one bundle: they are meant to be used together, so there is nothing to pick.

### 2. Run the Setup

Open Claude Code in your repo and say *"set up Service Layer"*. It is a conversation:

1. Claude asks for the B1 version, the OData version (`v1` is OData V3, `v2` is OData V4; it proposes `v2` from FP 2405), which environments you want (`dev`, `uat`, `prod`; at least one) and, for each, the Service Layer URL and the company database. It shows you a summary and waits for your approval.
2. Claude creates `.sbo-skills/service-layer/` with the configuration and one `credentials.json` per environment, with the URL and company filled in and `userName` and `password` **empty**. It also adds `.sbo-skills/` to your `.gitignore`.
3. **You** open each `credentials.json` in your editor, type the user and the password and save. Tell Claude when you are done.
4. Claude tests every login and builds an index of the entities each environment exposes. If a login fails, it tells you the Service Layer error; fix the file and ask it to verify again.

Run the Setup again at any time to start from scratch.

### 3. Bam - you're ready to go.

Just talk to Claude. The skills trigger from what you ask.

## Why These Skills Exist

I built these to fix the common failure modes of an agent talking to a live Service Layer.

### #1: The Agent Sees My Credentials

**The Problem**: To log in, the agent needs a URL, a company, a user and a password. The easy path is to paste them into the chat. From then on they live in the conversation, in logs, and in whatever the model provider keeps.

**The Fix** is the **`setup` skill**: Claude asks only what is not secret (versions, environments, URL, company) and you type the user and the password yourself into a file in your repo. The skill's rules forbid Claude to ask for them, to read that file or to repeat them if you paste one by mistake. The tool logs in by itself, so `Login` and `Logout` are not even requestable.

> [!NOTE]
> The password stays in plain text in `.sbo-skills/service-layer/<env>/credentials.json`, ignored by git. Anyone who can read your repo folder can read it. Keeping it out of the conversation relies on the skill's rules: nothing technical stops Claude's tools from opening that file.

### #2: The Agent Guesses Entities And Fields

**The Problem**: The agent invents `U_` fields that do not exist, misnames a custom table, or builds a body that the Service Layer rejects. User fields differ between companies and between `dev` and `prod`, so no model knows them.

**The Fix** is the **`use` skill**, which gives the agent two maps built from *your* environment:

- **Object context**: for an entity, its standard and user fields (`U_*`), types, whether they can be empty, and the valid values. Claude reads it before building a body or a `$select`.
- **Entity index**: which entities the environment exposes, standard and user-defined. Claude tries the name it knows first, and reads the index only when it is not sure or the Service Layer says the entity does not exist.

Both refresh by themselves after a week. If you created a user table a minute ago, ask for `entities --refresh`.

### #3: The Agent Changes Data Without Asking

**The Problem**: Giving an agent write access to an ERP is only safe if you see what it is about to do, exactly.

**The Fix** is a write flow with a **dry run by default**. Everything that is not a GET is a dry run until you approve it:

1. Claude runs the call **without** `--execute`. Nothing is sent. It shows you the exact method, full URL, headers, body and environment.
2. You approve in words, or ask for changes (and get a new dry run).
3. Claude repeats the same call with `--execute` and reports the status.

On `prod`, every write also needs `--allow-prod`, added only after you approve that production write explicitly. Claude never proposes to skip the approval.

### #4: The Agent Doesn't Know How The Service Layer Behaves

**The Problem**: Pagination, ETag and `If-Match`, `$batch` change sets, UDOs, webhooks: the answers are in a 600-page guide, and the model half-remembers a different version.

**The Fix** is the **`docs` skill**: the Service Layer reference (guide v1.29), plus an OData protocol section and facts verified against a live Service Layer. It reads the B1 version you declared in the Setup and checks which sections exist in your version before answering.

## Usage Examples

### Reading data

> *"How many business partners are there in dev?"*
> *"Show me the first 5 open sales orders of customer C20000 in uat."*
> *"Read item A1 and tell me which user fields it has."*

Behind this, Claude runs:

```
use.mjs count BusinessPartners --entorno dev
use.mjs page Orders --filter "CardCode eq 'C20000' and DocumentStatus eq 'bost_Open'" --select DocEntry,DocNum,DocTotal --top 5 --entorno uat
use.mjs get Items "'A1'" --entorno dev
use.mjs traverse Items --filter "Valid eq 'tYES'" --max-rows 500 --entorno dev
```

**Records never enter the conversation.** They are written to a **Volcado**: a local folder with one file per record and an `_index.json`. Claude gets only the path and a summary and reads the files it needs, so a big answer cannot overflow its context. With several environments configured, add `--entorno dev|uat|prod`.

### Changing data

> *"Close order 5 in dev."*

```
use.mjs request POST "Orders(5)/Close" --entorno dev             # dry run: shows the request, sends nothing
use.mjs request POST "Orders(5)/Close" --entorno dev --execute   # after you approve
```

More `request` examples:

```
# Update a field, replacing the lines of the document
use.mjs request PATCH "Orders(5)" --header "B1S-ReplaceCollectionsOnPatch: true" --body-file patch.json --entorno dev

# Create an entity from a UTF-8 JSON file (recommended on Windows)
use.mjs request POST BusinessPartners --body-file bp.json --entorno dev

# Run a stored SQL query: a POST that only reads is declared with --read and runs directly
use.mjs request POST "SQLQueries('open_orders')/List" --read --entorno uat

# Any header goes through
use.mjs request GET "Items?\$top=3" --header "B1S-CaseInsensitive: true"
```

`<path>` is what follows the service root (`BusinessPartners('C1')`, `Orders(5)/Cancel`, `$batch`). Do not write `/b1s/v2`: the saved OData version is added. Quote the path in the shell. The tool sends exactly what Claude gives it, with no `If-Match`, ETag or `Prefer` of its own.

### `$batch`

> *"In one batch: read item i001, then create an order and update its comment, all or nothing."*

Claude writes the batch as a JSON file; the tool builds the multipart body and parses the answer:

```json
{ "requests": [
  { "method": "GET", "path": "Items('i001')", "contentId": "read" },
  { "changeset": [
    { "method": "POST", "path": "Orders", "contentId": "1", "body": { "CardCode": "C1", "DocumentLines": [] } },
    { "method": "PATCH", "path": "$1", "contentId": "2", "body": { "Comments": "x" } }
  ] }
] }
```

```
use.mjs request POST '$batch' --body-file batch.json --entorno dev             # dry run lists every sub-request
use.mjs request POST '$batch' --body-file batch.json --entorno dev --execute
```

A `changeset` is atomic and holds no GET; `$1` refers to the request with `contentId` 1 of the same changeset. The outer HTTP status is 200 even when a sub-request fails: Claude reads `errores` and `subrespuestas`.

### Files and images

```
use.mjs request POST Attachments2 --file report.pdf --file other.png --entorno dev
use.mjs request PATCH "ItemImages('A1')" --file photo.jpg --entorno dev
use.mjs request GET "Attachments2(3)/\$value?filename='line2.png'" --entorno dev
```

Files must be under 50 MB. Downloads are saved in the Volcado and Claude tells you where.

> [!WARNING]
> Attachment and image upload/download are covered by unit tests only. The test Service Layer had no usable attachment or picture folder, so they were not run end to end.

### Asking how it works

> *"How do I paginate a GET?"*
> *"What does a 412 on PATCH mean and how do I use If-Match?"*
> *"Can I roll back a $batch?"*
> *"How do I create a webhook subscription?"*

If you ask before running the Setup, the `docs` skill stops and asks you to run it first: it will not answer from the reference without knowing your B1 version.

## Reference

### Skills

- **[setup](./skills/setup/SKILL.md)**: Configure the connection to Service Layer: B1 version, OData version and `dev`, `uat` and `prod`, with a login test. A conversation; you type the user and the password into a file yourself and Claude never handles them.
- **[use](./skills/use/SKILL.md)**: Operate against a configured environment: read entities, read the object context of an entity, find which entities exist, and make any other call with `request`, with the dry-run-first write flow.
- **[docs](./skills/docs/SKILL.md)**: Service Layer reference (guide v1.29) for all B1 versions. Needs the Setup.

### Commands of `use`

Run as `node "${CLAUDE_PLUGIN_ROOT}/dist/use.mjs" <command> ...` from your repo root. Every answer is short JSON: `ok`, `status`, `resumen` and, on failure, `error`.

| Command | Does |
|---|---|
| `get <EntitySet> <key>` | One record. A string key made only of digits is quoted: `'123'`. |
| `page <EntitySet> [--top N] [--skip N]` | One page (20 rows by default). |
| `traverse <EntitySet> [--max-rows N]` | Follows `nextLink` up to the cap (1000 by default). |
| `count <EntitySet>` | Only the number of rows. |
| `context <EntitySet> [--refresh]` | The object context of an entity. |
| `entities [--refresh]` | Where the entity index is. |
| `request <METHOD> <path> [...]` | Any call: `--header`, `--body`, `--body-file`, `--file`, `--stream-file`, `--read`, `--execute`, `--allow-prod`. |
| `clean <id or path>` | Deletes a Volcado. |

`get`, `page` and `traverse` accept `--filter`, `--select`, `--orderby` and `--expand`, passed to the Service Layer as written.

### Permissions (recommended)

Reads are safe to allow so that Claude Code does not ask each time, for example in `.claude/settings.json`:

```json
{ "permissions": { "allow": [
  "Bash(node *service-layer*/dist/use.mjs get *)",
  "Bash(node *service-layer*/dist/use.mjs page *)",
  "Bash(node *service-layer*/dist/use.mjs traverse *)",
  "Bash(node *service-layer*/dist/use.mjs count *)",
  "Bash(node *service-layer*/dist/use.mjs request GET *)",
  "Bash(node *service-layer*/dist/use.mjs context *)",
  "Bash(node *service-layer*/dist/use.mjs clean *)",
  "Bash(node *service-layer*/dist/setup.mjs --status)"
] } }
```

Do **not** allow `request` for other methods (nor `request POST ... --read`): the permission prompt they raise is the second guard of the write flow, and it cannot tell a dry run from `--execute`, so each of them is asked. Adjust the path pattern to where the plugin is installed.

### Troubleshooting

| You see | Do |
|---|---|
| `SETUP_MISSING`, `ENVIRONMENT_NOT_CONFIGURED` | Run the Setup (again), and check that `userName` and `password` are filled in the environment's `credentials.json`. |
| `ENTITY_NOT_FOUND` | The name is wrong or the entity is not exposed. Claude reads the entity index and retries with the right name. |
| `PROD_WRITE_NOT_ALLOWED` | A write on `prod` without `--allow-prod`. Approve the production write explicitly if you mean it. |
| `HEADER_RESERVED` | `Cookie`, `Host` and `Content-Length` belong to the tool. |
| `FILE_TOO_LARGE`, `FILE_FORBIDDEN` | File over 50 MB, or one of the Setup's own files. |
| `Property 'X' of 'Y' is invalid` | The object context may be out of date; ask for `context <Entity> --refresh`. |
| Any Service Layer error with a status | Reported literally: its code and message are the Service Layer's own. |

### Good to know

- The certificate of the Service Layer is never validated (they are usually self-signed): work from a network you trust.
- Local data lives in `.sbo-skills/service-layer/` of your repo: `config.md`, and per environment the credentials, the session, the object contexts, the entity index and the Volcados. Volcados are removed by `clean` and by the tool after 24 hours.
- `dist/` and `skills/` are generated from the factory repository; do not edit them here. Only this `README.md` and `.claude-plugin/plugin.json` are written by hand.
