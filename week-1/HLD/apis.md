# API contracts

Base: `/api/v1`; JSON over HTTPS. UTC ISO-8601 timestamps. IDs are opaque UUIDs. Student access requires a valid bearer token and assessment enrollment; administrators also require ownership of the assessment. These are design contracts, not implemented endpoints.

| Method and path | Request / authorization | Success |
| --- | --- | --- |
| `POST /auth/login` | email/password; rate limited | 200, short-lived access token, role and expiry |
| `POST /assessments/{id}/join` | student; optional assessment passkey | 200, enrollment and server-computed deadline |
| `GET /assessments/{id}/questions` | enrolled student during permitted window | 200, public prompts, constraints, language limits and samples only |
| `POST /submissions` | student + `Idempotency-Key` | 202 after durable commit, submission ID/state |
| `GET /submissions/{id}` | owner or authorized admin | 200, state, verdict and score; no hidden input/output |
| `GET /assessments/{id}/submissions?cursor=...&limit=50` | student sees own history | 200, items and opaque next cursor; limit capped at 100 |
| `POST /admin/assessments` | admin | 201, assessment ID |
| `POST /admin/assessments/{id}/questions` | owning admin | 201, question ID and immutable version |
| `POST /admin/assessments/{id}/publish` | owning admin; validates tests/time window | 200, published version |
| `GET /admin/assessments/{id}/results?cursor=...` | owning admin | 200, paginated scores |
| `POST /admin/submissions/{id}/rejudge` | owning admin; reason required | 202, new judge version; audited |

## Submission example

`POST /api/v1/submissions`, `Idempotency-Key: 73fa1c09-7d64-4a3c-bbfb-d1a8db060451`

```json
{
  "assessment_id": "asm-demo",
  "question_id": "question-demo",
  "language": "PYTHON",
  "code": "print(sum(map(int, input().split())))"
}
```

After durable commit:

```json
{"submission_id":"sub-demo","state":"QUEUED","received_at":"2026-10-26T09:00:00Z"}
```

The same key/body returns the same submission (200 on replay); changed body with that key returns 409. Code length is bounded (64 KiB design limit); toolchain versions are server-selected and recorded, never arbitrary client-supplied container images. Student clients cannot choose hidden tests, time limits, grades or another student's ID.

## Result example

```json
{"submission_id":"sub-demo","state":"COMPLETED","verdict":"ACCEPTED","score":100,"passed_tests":10,"total_tests":10}
```

States: `QUEUED -> RUNNING -> COMPLETED`; infrastructure retries return to `QUEUED`, and exhausted infrastructure retries end in `FAILED`. `COMPLETED` verdicts include accepted, wrong answer, compilation error, time limit and memory limit. `FAILED` carries a system error rather than misreporting the student's answer as wrong.

## Admin creation examples

```json
{"title":"Week 1 assessment","start_at":"2026-11-01T09:00:00Z","end_at":"2026-11-01T11:00:00Z","duration_minutes":120}
```

Question creation accepts prompt, points, allowed languages, per-language limits and private uploaded test-object references. Verify all referenced objects belong to this assessment. Publishing freezes the question/test version for existing submissions. The student's deadline is `min(joined_at + duration, end_at)`; intake uses trusted server receipt time, not the browser clock.

## Errors and retries

Errors use `{"error":{"code":"ASSESSMENT_CLOSED","message":"The submission window has ended","request_id":"req-demo"}}`.

- 400: malformed fields or invalid assessment time range.
- 401: absent/expired credentials.
- 403: role/enrollment does not permit the action.
- 404: missing resource or concealed resource belonging to another student.
- 409: idempotency key conflict or invalid state transition.
- 413: code/request too large.
- 422: unsupported language or closed submission window.
- 429: rate limited; obey `Retry-After`.
- 503: not accepted because a required dependency is unavailable; retry with the same key and bounded backoff.

An authorized replay of an already accepted key and identical body returns the original receipt even after the assessment deadline. Deadline and admission checks apply only to new acceptances; a changed body with the same key returns 409.

Clients must retain their submission idempotency key until acceptance is resolved. A timeout is not proof that acceptance failed. Never automatically retry with a new key after a timeout.
