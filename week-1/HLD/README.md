# Week 1 — Coding Assessment Platform HLD

**Prepared for:** Paidipelly Vishwateja, SRU internship.

Design deliverable for a platform where students take timed programming assessments and isolated workers evaluate submissions. This is an architecture exercise, not a deployed application or a performance benchmark.

1. [Architecture, diagram and tradeoffs](architecture.md)
2. [API contracts](apis.md)
3. [Requirements and design-review checklist](review-checklist.md)
4. [Other week-1 tasks](https://github.com/Vishwateja-123/DSATraining/tree/organize/week-1-review/week-1)

The scope and folder organization follow [Varun Suddala's week-1 reference](https://github.com/VarunSuddala/QuantumNique-weekly-tasks/tree/a38d74d391bd5f4d013be8c32277799f13e7e28e/week-1/HLD). The design here was newly prepared with AI assistance and includes explicit reliability, capacity and security assumptions. The original assignment rubric was not supplied; this covers the reference project rather than claiming instructor acceptance.

## Five-minute walkthrough

- Explain why submission acceptance is separate from execution.
- Follow one submission from the gateway through the database/outbox to a sandbox and result.
- Explain how a worker crash is retried without counting a score twice.
- Show why hidden tests and worker credentials never reach student code.
- Explain the capacity calculation and which assumptions would need load testing.

Earlier Rapido, Amazon and Netflix diagrams remain preserved under [training/HLD](../../training/HLD/2026-09-17-rapido-amazon-netflix.pdf).
