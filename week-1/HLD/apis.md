# API contracts

Base: `/api/v1`; JSON over HTTPS. UTC ISO-8601 timestamps. IDs are opaque UUIDs. Login is unauthenticated and rate limited. Joining requires a valid student bearer token, eligibility and any configured passkey, but no prior enrollment. Other assessment student endpoints require enrollment. Creating an assessment requires an authorized admin role and assigns the creator as owner; operations on an existing assessment require its ownership. These are design contracts, not implemented endpoints.

| Method and path | Request / authorization | Success |
| --- | --- | --- |
| `POST /auth/login` | email/password; rate limited | 200, short-lived access token, role and expiry |
| `POST /assessments/{id}/join` | authenticated eligible student; passkey if configured; no prior enrollment | 200, enrollment and server-computed deadline |
| `GET /assessments/{id}/questions` | enrolled student during permitted window | 200, public prompts, constraints, language limits and samples only |
| `POST /submissions` | enrolled student + `Idempotency-Key`; exact published question version | 202 after durable commit, submission ID/state |
| `GET /submissions/{id}` | owner or authorized admin | 200, state, verdict and score; no hidden input/output |
| `GET /assessments/{id}/submissions?cursor=...&limit=50` | student sees own history | 200, items and opaque next cursor; limit capped at 100 |
| `POST /admin/assessments` | authorized admin; creator becomes owner | 201, assessment ID |
| `POST /admin/assessments/{id}/enrollments` | owning admin; eligible student IDs | 200, provisioned enrollments; repeat is idempotent |
| `POST /admin/assessments/{id}/questions` | owning admin | 201, question ID and immutable version |
| `POST /admin/assessments/{id}/publish` | owning admin; validates tests/time window | 200, published version |
| `GET /admin/assessments/{id}/results?cursor=...` | owning admin | 200, paginated scores |
| `POST /admin/submissions/{id}/rejudge` | owning admin; reason required | 202, new judge version; audited |

## Enrollment, publication and pagination

Admin provisioning creates eligibility/enrollment with `joined_at` and `personal_deadline` unset. The first authenticated student join sets these once; repeat joins return the same values without restarting the clock. Self-join may create enrollment when the assessment permits it. Joining after `end_at` is rejected. Early join is allowed, but questions and submissions remain unavailable before `start_at`. The deadline is `min(max(joined_at, start_at) + duration, end_at)`.

Question/test versions are immutable. Publishing an assessment fixes its question-version manifest; changes require a new assessment, so a live exam cannot change beneath a student. Question responses include `question_id` and `question_version`; submission intake validates the exact manifest pair. Rejudge changes only the judge implementation version, not the prompt/test manifest.

`GET /assessments/{id}/leaderboard?cursor=...&limit=50` requires enrollment. It is available only after the assessment ends and the owner explicitly releases a finalized results snapshot; otherwise return 409 `RESULTS_NOT_RELEASED`. Return rank, assessment-scoped display alias and total score, with no email, code or hidden-test data. `POST /admin/assessments/{id}/results/publish` lets the owner release a new immutable snapshot only when all desired judge versions are terminal. A rejudge invalidates the old publication for new readers until a replacement is released.

History pages use ascending `(received_at, submission_id)` keyset ordering. The first page materializes a bounded snapshot of the authorized matching rows; subsequent pages read that immutable snapshot. Admin results and leaderboards use an immutable result snapshot ordered by `(total_score DESC, user_id ASC)`. Signed cursors bind snapshot ID, ordering position, assessment, viewer and expiry; every page reauthorizes access. Page size is capped at 100. Expired or invalid cursors return 400 and require restarting; new commits and rejudges cannot shift rows within a snapshot.

## Submission example

`POST /api/v1/submissions`, `Idempotency-Key: 73fa1c09-7d64-4a3c-bbfb-d1a8db060451`

```json
{
  "assessment_id": "asm-demo",
  "question_id": "question-demo",
  "question_version": 1,
  "language": "PYTHON",
  "code": "print(sum(map(int, input().split())))"
}
```

After durable commit:

```json
{"submission_id":"sub-demo","state":"QUEUED","received_at":"2026-10-26T09:00:00Z"}
```

Both initial lookup and a concurrent uniqueness-conflict re-read compare canonical request digests. The same key/body returns the same submission (200 on replay); changed body with that key returns 409. Code length is bounded (64 KiB design limit); toolchain versions are server-selected and recorded, never arbitrary client-supplied container images. Student clients cannot choose hidden tests, time limits, grades or another student's ID.

## Result example

```json
{"submission_id":"sub-demo","state":"COMPLETED","verdict":"ACCEPTED","score":100,"passed_tests":10,"total_tests":10}
```

States: `QUEUED -> RUNNING -> COMPLETED`; infrastructure retries return to `QUEUED`, and exhausted infrastructure retries end in `FAILED`. `COMPLETED` verdicts include accepted, wrong answer, compilation error, time limit and memory limit. `FAILED` carries a system error rather than misreporting the student's answer as wrong.

Status and score reads use only the result matching `desired_judge_version`; an older version is audit history and cannot contribute marks. Rejudge atomically increments that version and the fencing token, clears the lease, sets `QUEUED`, invalidates published results, reuses the existing reservation for pending work or atomically reserves capacity for a terminal submission, and writes an outbox event. Duplicate rejudge requests use a scoped idempotency key. Claims and completion writes compare both judge version and attempt token. While rejudging, show pending, not a stale verdict; recompute scores from current results only.

## Admin creation examples

```json
{"title":"Week 1 assessment","start_at":"2026-11-01T09:00:00Z","end_at":"2026-11-01T11:00:00Z","duration_minutes":120}
```

Question creation accepts prompt, points, allowed languages, per-language limits and private uploaded test-object references. Verify all referenced objects belong to this assessment. Publishing freezes the assessment question/test manifest. The student's deadline is `min(max(joined_at, start_at) + duration, end_at)`. Intake captures trusted receipt time when the complete bounded request arrives; all later deadline checks use that same timestamp, never processing time or the browser clock.

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

Receipt mappings remain for the assessment record lifetime (minimum 90 days after end). The API advertises `receipt_expires_at`; reconciliation is supported until that date. After archival, the closed assessment rejects new submissions, so expired keys cannot create duplicate attempts.
