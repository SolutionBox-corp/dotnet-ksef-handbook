# What KSeF is and who it affects

**KSeF** (Krajowy System e-Faktur) is Poland's national e-invoicing system. Structured invoices (XML, the FA schema) are submitted to a central government platform, which validates them and returns a confirmation (UPO). From the mandate dates, a paper or PDF invoice is no longer a legally issued invoice for in-scope transactions — the version in KSeF is.

## The dates

- **1 February 2026** — mandatory for large taxpayers (2024 gross sales over PLN 200m). From this date all taxpayers must at least be able to *receive* via KSeF.
- **1 April 2026** — mandatory issuing for the remaining established VAT payers (excluding the smallest micro-segment).
- **1 January 2027** — covers the smallest taxpayers; KSeF error penalties begin.

Always check the current state of the rules — dates in this space have moved before.

## Who it affects (and why it's a .NET problem)

Any company issuing B2B invoices in Poland, and the software that does it for them: ERP systems, accounting platforms, invoicing products, and Dynamics 365 Business Central deployments. If you build or maintain any of those on .NET, KSeF lands on your desk.

It is not "just another REST endpoint". KSeF is **asynchronous, rate-limited, occasionally unavailable, and stateful** — you authenticate into a session, submit, then poll for a result and a UPO. That combination is exactly what breaks naive integrations, which is why the rest of this handbook is about doing it durably.

For Czech companies invoicing into Poland, the same applies — and the same engineering transfers later to EU **ViDA** and **PEPPOL**.

---

Full write-up: <https://www.solutionbox.cz/blog/ksef-co-to-je> · Maintained by [SolutionBox](https://www.solutionbox.cz)
