# Review follow-up — 27 October 2026

The following responses were independently checked against the code/design. HLD remains a design exercise, not a deployed implementation. Old GitHub comments may remain visible after these changes.

- [🟡 Medium · Accepted submissions lose their code](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231727): Staged-object row locks coordinate acceptance and cleanup; a grace period alone is no longer relied on.

- [🟡 Medium · On-time submissions are rejected](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231734): All deadline checks use the same trusted complete-request receipt timestamp.

- [🟡 Medium · Concurrent requests exceed intake cap](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231739): Admission capacity is reserved atomically in the acceptance transaction.

- [🟡 Medium · Concurrent key reuse skips conflict check](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231745): The uniqueness-conflict path compares request digests and returns 409 on mismatch.

- [🟡 Medium · Older rejudge result becomes current](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231753): Worker claims and writes check desired judge version as well as the fencing token.

- [🟡 Medium · Rejudges leave results ambiguous](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231760): Status and score aggregation select only the desired judge version.

- [🟠 High · Question versions cannot coexist](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231767): Questions now use a composite (question_id, version) primary key.

- [🟠 High · Students cannot join assessments](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231775): Join explicitly does not require prior enrollment.

- [🟠 High · Early joining consumes exam time](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231782): Early joining uses max(joined_at, start_at) before adding duration.

- [🟠 High · Republishing changes grading mid-exam](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231794): Publishing freezes the assessment manifest; submissions specify its question version.

- [🟡 Medium · Expired keys lose submission receipts](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231800): Receipts have an advertised reconciliation window and remain for the assessment record lifetime, minimum 90 days.

- [🟡 Medium · Cursor pages can omit or repeat rows](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231804): Pagination specifies immutable snapshots, total ordering, scoped cursors and expiry.

- [🟡 Medium · Outbox polling degrades with history](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231806): Outbox polling uses a partial unpublished index and bounded published-event retention.

- [🟡 Medium · Admins cannot enroll assigned students](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231816): Admin provisioning endpoint added; provisioning does not start the student's timer.

- [🟡 Medium · Students cannot retrieve the leaderboard](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231822): Student leaderboard and admin result-publication endpoints added with privacy and release rules.

- [🟡 Medium · Admins cannot create assessments](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231829): Creation assigns the admin as owner; ownership checks apply to existing resources.

- [⚪ Low · Week-one link misses the changes](https://github.com/Vishwateja-123/HLD/pull/1#discussion_r4384231834): Companion link points to the existing review branch.


