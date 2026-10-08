---
name: service-layer
description: SAP Business One Service Layer, to ask about it and to work with it. Answers how Service Layer behaves from a reference that follows the developer's SAP Business One version (login and sessions, OData, query options, $batch, UDF/UDT/UDO, attachments, SQL queries, ETag, b1s.conf, webhooks, server-side JavaScript scripts, limitations, DI API comparison), and operates against a real configured environment (dev, uat, prod) - read entities, count, read the fields of an entity, create, update, delete, call actions, send $batch, run SQL queries, upload and download files; writes are a dry run until approved. Use for any question or task that involves Service Layer, even when the developer does not name it (/b1s/, B1SESSION, OData on SAP Business One). Needs the Setup (service-layer-setup).
---

# Service Layer

One skill for everything about Service Layer. Never answer a Service Layer question from memory: it has details an educated guess gets wrong (its scripts are JavaScript, for example). The reference below is the source.

## 1. Check the Setup

Read `.sbo-skills/service-layer/config.md` (it holds no secrets). If it does not exist, **stop**: tell the developer to run `/service-layer-setup`, and do not answer without it.

From its front matter take:

- `versionB1` (for example `FP 2608`): the version of SAP Business One. The reference says when a section does not exist in an older one.
- `versionOData` (`v1` is OData V3, `v2` is OData V4): the version the tool calls. Keep examples in line with it.
- `language` (`en` English, `es` Spanish; English if the line is missing): the language you talk to the developer in. Code, entity names, commands and quotes from the reference stay as they are.

## 2. Ask or do

- **To know how something works** (what it is, how it is configured, which header or verb, what an error means): read [docs/index.md](docs/index.md) and follow it to the single page that answers. If the reference does not cover it, say so; do not fill the gap from memory.
- **To touch a real system** (read data, change data, find which entities or fields exist): read [use.md](use.md) and follow it. If you are not sure how a call must be built, read the reference first.
- **Both** (for example "create an order"): the reference for how, then `use.md` to run it.
