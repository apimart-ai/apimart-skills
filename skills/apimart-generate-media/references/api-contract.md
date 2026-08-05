# APIMart Media API Contract

## Contents

1. Configuration
2. Local client commands
3. API endpoints
4. Media upload rules
5. Generation and task semantics
6. Error handling

## Configuration

The bundled client reads:

| Variable | Required | Meaning |
| --- | --- | --- |
| `APIMART_API_KEY` | Yes | Preferred APIMart API key |
| `API_KEY` | Fallback | Legacy key name |
| `APIMART_BASE_URL` | No | API origin; defaults to `https://api.apimart.ai` |
| `APIMART_REQUEST_TIMEOUT_MS` | No | Read request timeout; defaults to `15000` |
| `APIMART_SUBMIT_TIMEOUT_MS` | No | Submit timeout; defaults to `45000` |
| `APIMART_UPLOAD_TIMEOUT_MS` | No | Image upload timeout; defaults to `120000` |
| `APIMART_MAX_RESPONSE_BYTES` | No | Maximum JSON response size; defaults to `33554432` |

The client sends:

```http
Authorization: Bearer <APIMart API key>
Accept: application/json
```

`APIMART_BASE_URL` must be an APIMart API origin. Keep the default for normal
production use; change it only when APIMart supplies another API origin.

## Local Client Commands

Assume:

```text
CLIENT=<skill-directory>/scripts/apimart-media.mjs
```

Generate a new idempotency key:

```bash
node "$CLIENT" key
```

List models:

```bash
node "$CLIENT" models [--query <substring>] [--limit <1-200>] [--offset <n>]
```

Read the exact model documentation:

```bash
node "$CLIENT" docs --model <exact-model-id>
```

Get the compatibility operation and input schema:

```bash
node "$CLIENT" schema \
  --model <exact-model-id> \
  [--operation image_generation|video_generation]
```

Upload a local reference image and receive an HTTP(S) URL:

```bash
node "$CLIENT" upload-image --file <local-image-path>
```

The client does not pre-reject an image by byte size. It streams JPEG, PNG,
GIF, or WebP content to the upload endpoint and surfaces that endpoint's size
decision. The upload is not a generation request and does not take an
idempotency key.

Submit a billable image or video generation:

```bash
node "$CLIENT" generate-image \
  --model <exact-model-id> \
  --input-json '<JSON object>' \
  --idempotency-key <saved-key>
```

```bash
node "$CLIENT" generate-video \
  --model <exact-model-id> \
  --input-file <JSON file> \
  --idempotency-key <saved-key>
```

Exactly one of `--input-json` and `--input-file` is required. Use
`--input-file -` to read a JSON object from standard input.

Query a task once:

```bash
node "$CLIENT" task \
  --task-id <exact-task-id> \
  [--language zh|en|ko|ja]
```

All successful commands emit one JSON document to standard output. Errors emit
one structured JSON error to standard error and exit nonzero.

## API Endpoints

| Operation | Method | Path |
| --- | --- | --- |
| List models | GET | `/v1/models` |
| Read model documentation | GET | `/v1/model-docs?model=...` |
| Read model schema | GET | `/v1/model-schema?model=...&operation=...` |
| Upload image | POST | `/v1/uploads/images` |
| Generate image | POST | `/v1/images/generations` |
| Generate video | POST | `/v1/videos/generations` |
| Query task | GET | `/v1/tasks/{task_id}` |

Generation requests also send:

```http
Idempotency-Key: <saved-key>
X-APIMart-Response-Version: 2026-07-27
Content-Type: application/json
```

The request body is the documentation-derived input plus a top-level `model`.
The compatibility schema is fetched automatically to confirm the requested
image or video operation; the APIMart API performs exact model-input
validation. Model IDs and task IDs are opaque strings and must be copied
exactly.

## Media Upload Rules

`POST /v1/uploads/images` uses `multipart/form-data` with one `file` field and
the same Bearer API key as the other endpoints. It accepts content-detected
JPEG, PNG, GIF, or WebP bytes and returns:

```json
{
  "url": "https://...",
  "filename": "reference.png",
  "content_type": "image/png",
  "bytes": 12345,
  "created_at": 1785571200
}
```

Use `url` in the model-specific image field. Do not pass the local path,
base64, or data URI to a generation request. The returned upload URL is valid
for 72 hours after creation; use it or download it before it expires. Uploads
are not automatically retried because the endpoint has no upload idempotency
key; an uncertain retry could create a duplicate object.

Do not enforce a separate image byte limit in the local client. Submit the
upload once and surface the API response, including `413` when the upload
service rejects the request. Generic client, gateway, and HTTP transport limits
remain operational concerns rather than APIMart image-size rules.

There are no APIMart audio or video upload endpoints in this workflow. Every
audio/video media value used by a generation must be a public HTTP(S) URL.
Local paths, attachments, `file://`, raw/base64 bytes, and audio/video data URIs
are rejected locally before schema lookup or billable submission.

## Generation and Task Semantics

The local client normalizes a generation response with:

- `kind`: `task` for asynchronous work or `result` for a synchronous result.
- `idempotency_key`: the caller-provided stable key.
- `task_id`: the opaque task identifier when `kind` is `task`.
- `status`: commonly `pending`, `processing`, `completed`, or `failed`.
- `terminal`: true for `completed` or `failed`.
- `should_poll`: true only for a nonterminal asynchronous task.
- `next_poll_after_seconds`: suggested delay, currently 2 seconds.
- `replayed`: true when the server reports an idempotent replay.
- `response_version`: negotiated response contract version.
- `media_url_ttl_hours`: media URL validity, currently 72 hours.

The `task` command returns the same polling fields. Query only while
`should_poll` is true. Stop after 10 minutes or 120 checks by default,
whichever comes first, and return the task ID for later resumption.
Media URLs returned by synchronous results or completed tasks are valid for 72
hours. User-facing responses must state 72 hours explicitly, never 24 hours or
an unknown duration.

## Error Handling

Known API errors preserve `message`, `type`, `code`, `param`, HTTP status,
retryability, and any `Retry-After` delay.

For a network failure or timeout during POST, the client reports:

```json
{
  "error": {
    "code": "request_outcome_unknown",
    "indeterminate": true,
    "idempotency_key": "<original-key>"
  }
}
```

The server may already have accepted that request. Retry only the identical
request with the original key. Do not generate a replacement key.

For an ordinary 4xx validation response, `indeterminate` is false; refresh the
model documentation and compatibility schema, correct the input, and use a new
key only when creating a changed logical request.
