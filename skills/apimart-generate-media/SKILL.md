---
name: apimart-generate-media
description: Use APIMart MCP tools to discover models, fetch live input schemas, submit billed image or video generations, and check asynchronous tasks safely. Use when a user asks an agent to create images or videos with APIMart, choose or inspect an APIMart model, determine a model's supported parameters, or resume an APIMart generation task.
---

# Generate Media with APIMart

Use the APIMart MCP as the execution layer. Treat its live model schema as the
source of truth instead of relying on remembered model parameters.

## Preconditions

1. Require the MCP tools `list_models`, `get_model_schema`,
   `generate_image`, `generate_video`, and `get_task`.
2. If the tools are unavailable, stop and ask the user to configure
   `apimart-mcp`. Never request or copy an API key into chat, prompts, files,
   tool inputs, or generated artifacts.
3. Treat generation calls as billable. Proceed when the user explicitly asks
   to generate media. If the user is only comparing options or their intent is
   unclear, stop before calling a generation tool.

## Workflow

### 1. Establish the request

Identify whether the user wants an image or video and collect the essential
creative inputs. Reuse any exact model ID the user supplied. Do not rewrite,
parse, or normalize model IDs or task IDs, even when they resemble JSON.

### 2. Discover a model

When the user did not specify a model:

1. Call `list_models`, using `query` to narrow candidates when the request
   names a model family.
2. Follow `next_cursor` only when more candidates are needed.
3. If several candidates differ materially in capability, speed, or cost and
   the user's preference is unknown, present a short choice instead of
   selecting arbitrarily.

### 3. Fetch the live schema

Call `get_model_schema` with the exact model ID before every generation.
Specify `operation` only when inference is ambiguous.

Build the generation `input` directly from `input_schema`:

- Satisfy `required`, `anyOf`, `oneOf`, types, ranges, formats, and enums.
- Put the model ID only in the generation tool's top-level `model` field.
- Never put `model` inside `input`.
- Omit optional fields the user did not request.
- Never send `response_format` unless the live schema explicitly defines it.
- Prefer reachable media URLs over large base64 payloads.

Read [references/tool-contracts.md](references/tool-contracts.md) when exact
tool arguments, result states, or recovery behavior are needed.

### 4. Submit exactly once

Choose the tool that matches `operation`:

- `image_generation` -> `generate_image`
- `video_generation` -> `generate_video`

Create and retain a unique `idempotency_key` before every billed submission.
Always pass it in the first generation call so a lost MCP response can be
recovered safely. A retry must reuse the same key, model, and input. Never
retry an uncertain submission with a new key.

### 5. Handle the result

- If `kind` is `result`, return the completed media result.
- If `kind` is `task`, preserve `task_id` exactly.
- Call `get_task` only while `should_poll` is `true`.
- Wait at least `next_poll_after_seconds` between checks; never busy-loop.
- Before polling, set a bounded budget: by default, stop after 10 minutes or
  120 checks, whichever comes first. When the budget is exhausted, return the
  exact `task_id` so the user can resume later.
- Stop when `terminal` is `true`. Treat `status: failed` as a normal terminal
  task result and explain the reported failure without hiding it.

## Error Handling

- On schema validation failure, fetch the live schema again and correct only
  the rejected parameters.
- On authentication failure, ask the user to repair their MCP configuration;
  do not ask them to paste a key.
- On a channel or provider failure, report it and avoid repeated submissions.
  Do not switch models or change the creative request without user consent.
- When a retryable error includes `retry_after_seconds`, wait at least that
  long before retrying.
- On an indeterminate outcome, preserve the original idempotency key and do
  not create a second logical request.

## Response Style

Report the selected model, whether the result is synchronous or asynchronous,
the terminal status, and any returned media URLs. Keep internal envelopes,
request IDs, and retry details out of the response unless they help diagnose
an error or recover a task.
