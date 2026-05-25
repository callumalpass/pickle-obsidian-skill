---
name: pickle-obsidian
description: Use when an agent needs to create, inspect, wait for, or consume Pickle Inbox requests stored as mdbase-compatible Markdown files in an Obsidian vault or repo-local Pickle collection.
---

# Pickle Inbox

Pickle Inbox is a local-first, file-based inbox for typed agent requests and
structured human responses. It is represented as an mdbase-compatible Markdown
collection, so the backend is plain files:

- requests are files of type `pickle_request`
- responses are files whose type is named by the request's `response_type`
- response files link back to the request with a `request` link field
- attachments live under the Pickle collection folder, usually `attachments/`
- the Obsidian plugin reads/writes through the vault API, but agents should
  preserve the mdbase file contract

Default collection layout:

```text
_pickle/
  mdbase.yaml
  Pickle Requests.base
  _types/
    pickle_request.md
    pickle_response_approval.md
    pickle_response_ack.md
  requests/
  responses/
  attachments/
```

## Creating A Request

1. Find the collection folder, defaulting to `_pickle`.
2. Decide the expected response shape and ensure its type file exists under
   `_pickle/_types/`.
3. Create one request file under `_pickle/requests/`.
4. Put supporting files under `_pickle/attachments/<request-id>/` and reference them by collection-relative paths.
5. Validate with `(cd _pickle && mdbase validate)` when the CLI is available.

Use this default approval response type for ordinary approve/reject/revise
requests:

```yaml
response_type: pickle_response_approval
```

Request frontmatter pattern:

```yaml
---
type: pickle_request
title: "Approve repo hygiene cleanup?"
source: "agent"
kind: approval
message: "Approve repo hygiene cleanup?"
priority: normal
response_type: pickle_response_approval
created_at: "2026-05-24T00:00:00.000Z"
dedupe_key: "repo-hygiene-2026-05-24"
attachment_paths:
  - "attachments/repo-hygiene/context.md"
context:
  cwd: "/home/calluma/projects/example"
  task: "repo hygiene digest"
---
Concise question and enough context for a human decision.
```

Write request values explicitly. Do not rely on defaults being present in the
built-in Pickle type files. In particular, write `kind`, `message`,
`priority`, and `response_type` when creating a request.

Do not write `status: answered`. Request state is derived from linked response
files. `status` is retained only for explicit `cancelled` requests and legacy
interop.

## Writing Expected Response Types

The request's `response_type` names the mdbase type the human must fill out.
The value should match the `name` field in `_pickle/_types/<response_type>.md`;
do not include `.md` in the request.

For ordinary approvals, use the built-in type:

```yaml
response_type: pickle_response_approval
```

For message/notice requests that only need acknowledgement, use:

```yaml
response_type: pickle_response_ack
```

For custom workflows, create a response type before creating requests that
refer to it. A response type is a Markdown file with YAML frontmatter:

```yaml
---
name: pickle_response_rollout
description: Approve, reject, or revise a rollout plan.
display_name_key: decision
fields:
  request:
    type: link
    target: pickle_request
    validate_exists: true
    required: true
  decision:
    type: enum
    values: [approve, reject, revise]
    required: true
  risk_accepted:
    type: boolean
    required: true
  rollout_steps:
    type: list
    items:
      type: string
    min_items: 2
  review:
    type: object
    fields:
      summary:
        type: string
        required: true
      severity:
        type: enum
        values: [low, medium, high]
  comment:
    type: string
  responded_at:
    type: datetime
    generated: now
  responder:
    type: string
  attachment_paths:
    type: list
    items:
      type: string
---
```

Guidelines for response types:

- Include a required `request` link field targeting `pickle_request`; this is
  how automation finds the response for a request.
- Use `decision` for the primary answer when there is one. Common values are
  `approve`, `reject`, and `revise`.
- Mark fields `required: true` when downstream automation cannot safely act
  without them.
- Prefer explicit required fields over defaults. Only define `default` if the
  workflow owner deliberately wants an omitted response field to have meaning.
- Use `description` to tell the human what the response is for; the modal shows
  this near the top of the form.
- Use `display_name_key` for the field that should summarize response rows.
- Keep large context out of response fields. Use request attachments for logs,
  screenshots, diffs, traces, or generated plans.

The Obsidian modal supports these field shapes:

- `enum`: dropdown with `values`
- `boolean`: toggle
- `string`: text input, or textarea for longer/comment fields
- `number` and `integer`: numeric scalar values
- `datetime`: datetime string
- `list`: repeatable items using `items`
- `object`: nested fields using `fields`
- `link`: mdbase/Obsidian link values

The plugin writes or maintains these response fields itself when the response is
created through Obsidian, so do not ask the human to fill them manually:

- `type`
- `request`
- `responded_at`
- `responder`
- `attachment_paths`

Generated fields such as `id` may appear in schemas for external tooling, but
the response form should not depend on the human entering them.

## Consuming A Response

Do not infer approval from the request file alone. A request is answered when a
response file links back to it:

```yaml
---
type: pickle_response_approval
request: "[[requests/approve-repo-hygiene]]"
decision: approve
comment: "Looks good."
responded_at: "2026-05-24T00:05:00.000Z"
responder: human
---
```

After finding a response:

1. Validate that the response file's type matches the original request's `response_type`.
2. Validate required fields from the response type file, including nested object
   and list constraints when present.
3. Confirm the response's `request` link targets the request file you created.
4. Copy the decision back into the upstream workflow state before acting.
5. Leave the request/response files intact as the audit trail.

## Guardrails

- Ask one clear question per request.
- Prefer links and attachment paths over copying large logs into the request body.
- Use `dedupe_key` for repeated automation checks.
- Keep private context in the local collection; expose it only over trusted sync or private networking.
- Prefer the Rust `pickle` CLI for automation. It writes the same mdbase files
  and supports named collections with `--collection`.
- Do not change an existing response type casually once requests are already
  waiting on it; add a new response type name for incompatible workflow changes.
