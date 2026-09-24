---
name: chathub-connector-tools
description: "Discover and execute ChatHub connector-platform tools correctly (mail, calendar, IM, drives, knowledge bases): bridge call syntax, narrow-keyword discovery, error recovery, and rules against false negatives and brittle memory. Use when a task targets an external system reached through ChatHub connectors (WPS365, Microsoft 365, etc.)."
version: 1.0.0
author: ChatHub Platform
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [Chathub, Connectors, Tools, Mail, Calendar, Integration]
---

# ChatHub Connector Tools

The user's external systems (mail, calendar, IM, drives, knowledge bases) are exposed through the **ChatHub connector platform**. Two meta-tools bridge them:

- `list_chathub_tools(query?)` — catalogue of connector tools the user has enabled
- `exec_chathub_tools(name, args)` — execute one tool by its **full prefixed name** (e.g. `wps365/mail_mailbox_list`)

## 1. Canonical call flow

1. **Discover**: `list_chathub_tools` with a short keyword query.
2. **Execute**: `exec_chathub_tools` with the exact name from the catalogue and a JSON args object.
3. **Report**: summarize the result; on failure quote the exact error and the next action (e.g. re-authorization).

Call these through the **tool bridge** like any other plugin tool; never invent tool names and never call a connector tool name directly:

| Mistake | Error you will see | Fix |
|---|---|---|
| Calling `list_chathub_tools` without the bridge | `Tool 'list_chathub_tools' does not exist` | Re-issue the call through the bridge (`tool_call` with `name` + `arguments`) |
| Calling `wps365/...` (a connector tool) directly | `... is not a deferrable tool` | Connector tools run **only** via `exec_chathub_tools` |
| `exec_chathub_tools` with flattened args | `missing required argument(s): name, args` | Pass `{"name": "...", "args": "{...}"}` — args is an object or JSON string, not key-by-key parameters |
| 401 / 403 from upstream | `unauthorized` / `...permission...` | Explain which permission/consent is missing and how to renew it; do not retry blindly |

## 2. Discovery strategy (avoid the two classic failures)

- **Use short, tool-like keywords, not sentences.** Good: `wps365 mail`, `ms365 send-mail`, `list-mail-messages`, `calendar 日程`. Bad: `查找我昨天收到但是没有读的邮件` as one query, or re-sending the same broad query a third time.
- **Truncated ≠ missing.** Wide queries (or no query) return a compact overview, and very large matches are explicitly marked `catalog truncated: total N / matched K / showing M`. If your target is not visible: **narrow the query** (add the connector key or a longer tool-name fragment). Never conclude "there is no tool for X" from a truncated or empty result.
- **Zero matches** returns closest-name suggestions — use them (they are real names); do not fall back to guessing.
- **Attempt budget: 2.** After two failed discovery attempts, change strategy: query by connector key alone, or ask the user which connector/service they mean. Repeated identical queries are never useful.

## 3. Execution discipline

- Found the right tool → **execute it with `exec_chathub_tools`**. Do not stop at discovery and ask the user "which tool?" when the task already identifies one.
- Do **not** replace connector tools with browser automation, raw HTTP calls, or ad-hoc scripts (e.g. puppeteer-opening a mail web page). Only if **no** connector covers the task, say so and ask whether an alternative is acceptable.
- `args` must match the tool's input schema. If the execution error mentions a schema/argument problem, re-check with `list_chathub_tools(query="<exact tool name>", detail=true)` before retrying.
- Quote exact upstream errors when reporting failures (permission, not-found, rate-limit) — do not paraphrase them into "it didn't work".

## 4. Memory & evidence discipline

- **Do not enshrine availability rules.** Never write "X has no tool", "only use WPS365", "never fall back to Microsoft 365" into long-term memory. Catalogue availability changes (subscriptions, credentials, platform credentials) and must be re-checked each turn.
- Do not let an earlier instruction ("strictly use WPS365") override evidence: if the tool is unavailable, state the fact and ask the user how to proceed — do not silently refuse a viable alternative, and do not invent one.
- Never claim a tool exists unless the catalogue in *this* turn showed it.

## 5. Worked patterns

**Good (narrow → execute):**
```
list_chathub_tools({query: "mail email 邮件"})     → wps365/mail_mailbox_list visible
exec_chathub_tools({name: "wps365/mail_mailbox_list", args: {}})
→ 403 kso.mailbox.read … → report: mailbox permission not granted; suggest renewing WPS365 authorization
```

**Bad (broad → false negative):**
```
list_chathub_tools({query: "mail"})            → truncated before wps365/*
list_chathub_tools({query: "mail"})            → same again
"WPS 365 has no mail tools"                    ← WRONG (false negative; tools exist)
```
Correct recovery: `list_chathub_tools({query: "wps365 mail"})` → tools visible → execute.

## Checklist before answering a connector task

1. Did I discover with a **narrow** query (connector key or tool-name fragment)?
2. If the target was not visible: did I narrow **before** concluding anything?
3. Did I **execute** through `exec_chathub_tools` (not stop at listing, not use the browser)?
4. Did I quote exact errors and state the concrete next action?
5. Did I avoid writing availability rules or "never use X" into memory?
