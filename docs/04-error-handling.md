# 04 — Error Handling

KSeF is asynchronous, rate-limited, stateful, and operated by a government agency. Those four
properties together produce failure modes that a normal REST API does not have. This chapter
documents the error categories encountered in production .NET integrations and the handling
pattern for each one.

---

## 1. Session Expiry

### Problem

A KSeF session token has a limited lifetime. Applications that cache the token in memory and
reuse it across requests will hit an authorization error after the session expires. A naive retry
that reuses the same token will loop on the same failure indefinitely.

### Handling pattern

- Store the session token and its expiry timestamp in a shared, durable store (database or
  distributed cache) — not in-memory. All application instances must share the same session state.
- Before every KSeF call, check whether the token is still valid with an adequate safety margin
  (e.g. expire it 60 seconds early). Refresh proactively, not in response to a 401.
- Wrap the session lifecycle in a single component — your anti-corruption layer (see section 8).
  Callers never touch the raw token; the layer handles acquisition and renewal transparently.

---

## 2. Rate Limiting (HTTP 429)

### Problem

KSeF enforces a request-rate limit per time window. Bulk invoice submission at period close hits
this limit reliably. Without handling, a batch stops mid-flight with no record of which invoices
were sent.

### Handling pattern

- Use a **shared** token-bucket or sliding-window rate limiter. An in-memory limiter is
  insufficient when more than one application instance is running — they will each consume the
  full quota independently.
- A Redis-backed counter (or Polly's `RateLimiterPolicy` with a shared provider) ensures the
  combined request rate across all instances stays within the KSeF limit.
- On receipt of a 429 response, honour the retry-after interval returned by KSeF. Put the
  request back into the outbox queue; do not surface it as an application error.

---

## 3. Retry Without Idempotency

### Problem

A transient error, a timeout, or an application restart causes the integration to resend an
invoice. KSeF has no built-in idempotency at the submission endpoint — it accepts the duplicate
as a new document. The result is one invoice in the ERP and two in the government system.
Detecting and reversing this after the fact requires manual correction with the tax authority.

### Handling pattern

- Assign each invoice a **deterministic idempotency key** before the first submission. Derive the
  key from stable invoice data — document number, issue date, seller NIP — rather than a random
  UUID generated at send time.
- Record the key in the database together with the submission status. Before any retry, check
  whether the key has already been submitted successfully.
- KSeF does not enforce idempotency on your behalf — this logic lives entirely in the integration
  layer.
- If an invoice is **rejected** (see section 4), it must be corrected before resubmission. A
  corrected invoice is a different document and must receive a new key.

---

## 4. Validation Errors

### Problem

KSeF validates the invoice against the FA(2) logical invoice schema and its own business rules.
Rejection is asynchronous — the submission call succeeds, and only a subsequent polling call
reveals that the invoice was rejected. This means the invoice can sit in a "sent" state while
actually rejected, invisible to anyone who is not actively polling.

Common causes: incorrect NIP (tax identifier) format, missing or malformed timestamps,
inconsistent VAT rate calculations, structural errors in invoice line items.

### Handling pattern

- Validate the serialised XML against the FA(2) XSD schema **before** sending it to KSeF. Catch
  schema violations locally; do not rely on the government system as your only validator.
- During UPO polling (section 5), treat a rejection response as a distinct outcome — not an
  exception to log and ignore. Write the rejection reason to the audit log, update the invoice
  status, and trigger an alert so the invoice can be corrected.

---

## 5. UPO Polling

### Problem

Submitting an invoice is not the end of the process. KSeF processing is asynchronous. The
Urzędowe Poświadczenie Odbioru (UPO) — the official receipt that constitutes proof of acceptance
— is available only after polling. Integrations that treat a successful submission as done skip
this step entirely. An invoice without UPO is not a legally accepted invoice.

### Handling pattern

- After each submission, write the invoice to the database with status `Pending`.
- A dedicated background job polls KSeF periodically for each pending invoice.
- On receipt of UPO, store the document and update the status to `Accepted`. On rejection, store
  the rejection reason and update to `Rejected`.
- UPO must be retained — it is the audit proof.

---

## 6. Timeouts and Outages

### Problem

KSeF has unplanned outages. A synchronous call inside an HTTP request handler means that when
KSeF is unavailable the user-facing request times out, the invoice is lost from the queue, and
there is no mechanism for recovery.

### Handling pattern

- Every KSeF call must happen **outside the HTTP request**. The invoice is written to a durable
  outbox table first; a background worker processes it asynchronously.
- Apply Polly retry with exponential backoff and jitter. Jitter prevents a fleet of workers from
  simultaneously hammering KSeF the moment it returns after an outage.
- Respect rate limits during catch-up (section 2).

```csharp
// Polly retry policy — exponential backoff with jitter
var retryPolicy = Policy
    .Handle<KSeFApiException>(ex => ex.IsTransient)
    .WaitAndRetryAsync(
        retryCount: 5,
        sleepDurationProvider: attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 500)));
```

`IsTransient` should return `true` for network errors, 5xx responses, and 429 responses; `false`
for validation rejections and authentication errors that require intervention.

---

## 7. Missing Reconciliation

### Problem

Retry handles individual transient failures. It does not answer the question: "Is the overall
state of submitted invoices consistent?" Invoices can end up stuck in `Pending` beyond any
reasonable processing window, orphaned (sent to KSeF but missing from the local database), or in
a terminal failure state with no alert raised. Without a reconciliation job this situation is
discovered at an audit, not in the monitoring dashboard.

### Handling pattern

- Run a scheduled reconciliation job (daily minimum, hourly if volume warrants it).
- The job checks for:
  - Invoices in `Pending` status older than an expected maximum processing window.
  - Invoices with a KSeF reference number that cannot be matched to a local record (orphans).
  - Terminal failures that did not produce an alert.
- Any anomaly triggers an alert and writes a structured event to the audit log.
- Reconciliation is not a replacement for retry — it is a separate safety net that covers
  whole-batch consistency, not individual request recovery.

---

## 8. Anti-Corruption Layer

### Problem

The KSeF .NET SDK has its own conventions — proprietary error strings, numeric status codes, a
session model that has changed between SDK versions. Scattering direct SDK calls across the
codebase means every SDK update or breaking change requires a project-wide search-and-replace.
SDK quirks also leak into business logic that should not need to know about them.

### Handling pattern

- Wrap the SDK behind a single interface in your infrastructure layer. This anti-corruption layer
  (ACL) translates KSeF concepts — session, submission result, UPO, rejection — into your own
  domain types.
- Business logic depends on your interface, not the SDK. SDK changes are contained to the ACL.
- The ACL is also the right place to encapsulate session lifecycle management (section 1), the
  retry policy (section 6), and the rate limiter (section 2).

---

## Putting It Together

All eight categories resolve to the same underlying architecture:

1. Invoice is written to the DB with a deterministic idempotency key.
2. Background worker picks it up and submits it through the ACL.
3. Retry with backoff and rate limiting are inside the ACL; callers do not see them.
4. UPO polling runs as a separate background job; it updates invoice status in the DB.
5. Reconciliation job checks overall batch consistency on a schedule.
6. Every status transition produces a structured event in the audit log.
7. Terminal failures and stuck records trigger an alert.

This is not over-engineering. It is the minimum required to operate a system that is
asynchronous, rate-limited, and periodically unavailable.

---

Further reading: [KSeF API in .NET — the most common errors and how to handle them](https://www.solutionbox.cz/blog/ksef-api-chyby-dotnet)

Maintained by SolutionBox — https://www.solutionbox.cz
