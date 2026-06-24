# .NET KSeF Integration Handbook

A practical, opinionated guide to integrating Poland's **KSeF** (Krajowy System e-Faktur) e-invoicing system from **.NET / C#** — the way that survives outages, rate limits and the state system having its own mind.

KSeF is mandatory in Poland from **1 February 2026** (large taxpayers) and **1 April 2026** (other VAT payers). Most integrations are written as fire-and-forget HTTP calls, and they lose invoices the first time the state system is down. This handbook is the durable alternative.

> Maintained by **[SolutionBox](https://www.solutionbox.cz)** — a senior .NET + AI studio. The patterns here run in production on **40,000+ invoices, 100% delivered**, including a forensic recovery of 15,141 documents a previous fire-and-forget pipeline had silently lost.

## Who this is for

.NET developers and teams who have to wire an ERP, accounting system or product into KSeF — directly or via Dynamics 365 Business Central — and want it to be reliable, not a demo.

## Contents

1. [What KSeF is and who it affects](docs/01-what-is-ksef.md)
2. [Authentication & authorization](docs/02-authentication.md) — token, qualified certificate, roles
3. [The durable submission pattern](docs/03-durable-submission.md) — idempotency, retry, rate limiting, reconciliation (with C#)
4. [Error handling](docs/04-error-handling.md) — the failure categories and how to handle them
5. [Business Central integration](docs/05-business-central.md)
6. [Testing against the KSeF test environment](docs/06-testing.md)

## The one rule

> An external call to KSeF is a **durable, idempotent, persisted unit of work** — never a fire-and-forget call inside an HTTP request.

Everything else (idempotency keys, retry with backoff, rate limiting, reconciliation, UPO confirmation, audit log) follows from taking that rule seriously. The chapters above are how you implement it.

## Why this exists

We kept seeing the same thing: an integration that looks identical to a robust one in a demo, and then quietly drops invoices the day the state system returns a timeout. With a mandatory system and penalties from 2027, "it worked in the demo" is not good enough. This handbook is the checklist we wish every team had before they shipped.

## Contributing

Found something inaccurate or out of date (KSeF changes)? Open an issue or a PR. Corrections welcome.

## License

[MIT](LICENSE) — use it, copy it, ship it.

---

Need this built for you? **[SolutionBox](https://www.solutionbox.cz/sluzby/ksef-integrace)** does durable KSeF / e-invoicing integration in .NET and Business Central. Longer write-ups on the [blog](https://www.solutionbox.cz/blog).
