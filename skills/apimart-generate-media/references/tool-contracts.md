# APIMart MCP Tool Contracts

Use this reference only for exact tool shapes and recovery rules. The live
`get_model_schema` response remains authoritative for model-specific inputs.

## `list_models`

Read-only model discovery:

```json
{
  "query": "optional substring",
  "limit": 50,
  "cursor": "opaque cursor from a previous response"
}
```

- `limit` must be from 1 through 200.
- Copy every returned model `id` exactly.
- Use `next_cursor` unchanged for another page.

## `get_model_schema`

Read-only live schema lookup:

```json
{
  "model": "exact model ID",
  "operation": "image_generation or video_generation"
}
```

`operation` is optional unless the model is ambiguous. Inspect:

- `operation` to select the generation tool.
- `input_schema` to construct and validate `input`.
- `endpoint`, `schema_version`, `idempotency`, and `response_contract` when
  diagnosing integration behavior.

Do not put `model` inside a generation tool's `input` object.

## `generate_image` and `generate_video`

Billable submission:

```json
{
  "model": "exact model ID",
  "input": {
    "prompt": "model-specific input from the live schema"
  },
  "idempotency_key": "stable visible-ASCII key chosen before the first call"
}
```

- `input` contains every generation parameter except `model`.
- Always supply `idempotency_key`; omitting it forfeits disconnect-safe
  recovery because a server-generated key may not reach the client.
- `idempotency_key` is 1-191 visible ASCII characters without spaces.
- A retry must use the same key, model, and input.
- `kind: result` is synchronous completion.
- `kind: task` includes `task_id`, `should_poll`, and a suggested interval.
- Preserve the returned `idempotency_key` for recovery.

## `get_task`

One read-only status lookup:

```json
{
  "task_id": "exact task ID",
  "language": "zh"
}
```

`language` is optional and may be `zh`, `en`, `ko`, or `ja`.

- `should_poll: true` means another later check is allowed.
- Respect `next_poll_after_seconds`.
- Use a finite polling budget (default: 10 minutes or 120 checks). When it is
  exhausted, stop and return the exact `task_id` for later resumption.
- `terminal: true` means stop polling.
- Both `completed` and `failed` are terminal statuses.

## Recovery Matrix

| Condition | Action |
| --- | --- |
| Local schema validation error | Refresh the schema and correct rejected fields |
| Definite API rejection | Report the error; retry only when it is marked retryable, and wait at least `retry_after_seconds` when present |
| Unknown submission outcome | Reuse the same idempotency key, model, and input |
| Terminal failed task | Stop polling and report the task failure |
| Missing MCP authentication | Repair client configuration; never paste credentials into chat |
