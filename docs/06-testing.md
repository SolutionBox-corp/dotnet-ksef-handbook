# Testing against the KSeF test environment

KSeF exposes a **test environment** separate from production. Never develop or run integration tests against the live system — you risk submitting real documents and burning rate limits.

## Principles

- **Separate configuration per environment.** Base URL, credentials and certificates are environment-scoped. Make it impossible to accidentally point test runs at production (fail fast if the environment isn't explicitly set).
- **Test the boundary, not just the happy path.** The valuable tests are the ones that prove behaviour under failure: a timeout mid-submit, a 429, a session that expired, a duplicate retry. Assert that none of these produce a duplicate document or a lost one.
- **Idempotency tests are mandatory.** Submit the same logical invoice twice with the same idempotency key and assert exactly one document reaches KSeF. This is the test that catches the bug that loses or duplicates invoices in production.
- **Reconciliation is testable too.** Seed a state where a submit "succeeded" locally but has no UPO, run the reconciliation job, and assert it detects and resolves the orphan.
- **Verify what crosses the wire.** It is not enough that a call doesn't throw — assert that the request actually carried what you intended (correct payload, headers, session). Abstractions can silently drop fields; only a boundary test catches it.

## Practical setup

- A dedicated test taxpayer / certificate for the KSeF test environment.
- A test double (fake/anti-corruption layer) for fast unit tests, plus a smaller suite that hits the real KSeF test endpoint for integration confidence.
- CI runs the fast suite on every change; the real-endpoint suite on a schedule or before release.

---

Maintained by [SolutionBox](https://www.solutionbox.cz) — durable KSeF integration in .NET and Business Central.
