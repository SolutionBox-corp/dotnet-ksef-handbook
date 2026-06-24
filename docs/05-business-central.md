# 05 — Business Central + KSeF Integration

Connecting Microsoft Dynamics 365 Business Central to KSeF is not configuration — it is a custom integration layer. BC has a mature invoicing module with its own state machine, number series, and event model. KSeF is asynchronous, rate-limited, and occasionally unavailable. The two systems do not share vocabulary, so every step between them requires explicit mapping, durability, and reconciliation.

This chapter covers what the integration must do, where it commonly breaks, and how to build it so it does not.

---

## Integration Architecture Overview

```
BC (Posted Invoice)
      │
      ▼
[Mapping layer]   ← BC schema → KSeF FA(2) XML
      │
      ▼
[Submit queue]    ← persisted before send (Hangfire / EF)
      │
      ▼
KSeF API          ← idempotency key per invoice
      │
      ▼
[Status poller]   ← exponential backoff, respects rate limits
      │
      ▼
KSeF (UPO)        ← Urzędowe Poświadczenie Odbioru
      │
      ▼
BC (UPO written back, invoice state updated)
      │
      ▼
[Reconciliation job]  ← daily delta: BC sent vs UPO confirmed
```

---

## 1. Mapping BC Invoices to KSeF FA(2) Format

Business Central stores invoice data across multiple tables: `Sales Invoice Header`, `Sales Invoice Line`, `Customer`, `Company Information`. KSeF FA(2) XML expects a single document with specific field positions, mandatory attributes, and enumerated values.

### Counterparty identifiers

BC may store NIP/VAT numbers in `VAT Registration No.` on the customer card, or as a secondary identifier. KSeF requires the NIP in a specific format (digits only, no prefix) at `fa:Podmiot2/fa:DaneIdentyfikacyjne/fa:NIP`. Validate and strip whitespace and country prefix before mapping.

```csharp
string NormalizeNip(string raw) =>
    Regex.Replace(raw ?? "", @"[^0-9]", "");
```

### Number series

BC assigns its own invoice numbers from a configured number series (`No. Series`). KSeF assigns its own `KSeF Number` after acceptance. Both must be stored and both must be queryable. Design a dedicated `KSeF Invoice Entry` table (or custom fields on `Sales Invoice Header`) that holds:

| Field | Source |
|---|---|
| `BC Invoice No.` | BC number series |
| `KSeF Reference No.` | returned on submit, before UPO |
| `KSeF Invoice No.` | assigned by KSeF, returned in UPO |
| `Submit Status` | `Pending` / `Submitted` / `Accepted` / `Rejected` / `Error` |
| `UPO Received At` | datetime, UTC |
| `Idempotency Key` | deterministic, derived from BC Invoice No. |

Never use KSeF's number as the primary invoice identifier inside BC. BC number series remain authoritative for accounting.

### Invoice types

| BC Document Type | KSeF `RodzajFaktury` |
|---|---|
| Invoice | `VAT` |
| Credit Memo | `KOR` |
| Prepayment Invoice | `ZAL` |
| Prepayment Credit Memo | `KOREKTA_ZAL` |

Each type has different mandatory fields. Credit memos require a reference to the original invoice (`fa:DaneFaktury/fa:NrFaKorygowanej`), including invoices that predate the KSeF integration.

### VAT rates and rounding

BC calculates VAT with its own rounding rules. KSeF requires amounts to exactly two decimal places and will reject documents where line-level and header-level sums do not reconcile. Compute `NetoValue`, `KwotaVAT`, and `BruttoValue` from BC values; do not re-round independently.

---

## 2. Durable Submit with Idempotency Key

Do not call the KSeF API directly from a BC event handler or a synchronous API endpoint. If the call times out or the process restarts mid-flight, you either lose the invoice or submit it twice.

### Persist before send

```
1. BC raises "Posted Invoice" event
2. Integration writes KSeF Invoice Entry with Status = Pending
3. Hangfire job picks up the entry
4. Job serializes FA(2) XML
5. Job calls KSeF API → gets Reference No.
6. Job updates entry: Status = Submitted, ReferenceNo = <value>
```

The queue entry must exist before the HTTP call. If the process dies between steps 4 and 5, the job retries — the entry is still `Pending`.

### Idempotency key

Derive the key deterministically from the BC invoice number:

```csharp
string IdempotencyKey(string bcInvoiceNo) =>
    Convert.ToHexString(
        SHA256.HashData(Encoding.UTF8.GetBytes($"bc:{bcInvoiceNo}"))
    )[..32];
```

Pass this key in the KSeF submission headers. KSeF will reject a second submission with the same key with a distinct error code (`409` or equivalent), which you handle as "already submitted — fetch reference" rather than a hard failure.

### Retry policy (Hangfire example)

```csharp
[AutomaticRetry(Attempts = 10, DelaysInSeconds =
    new[] { 30, 60, 120, 300, 600, 1200, 1800, 3600, 7200, 14400 })]
public async Task SubmitInvoiceToKSeF(string kSeFEntryId, ...)
{
    // idempotency check: if already Submitted, skip and return
    // serialize XML
    // call API
    // update entry
}
```

Do not retry on validation errors (`400` with a schema violation) — those require human intervention, not automatic retry.

---

## 3. Status Polling and UPO Back into BC

After submission you hold a `ReferenceNo`. KSeF processes the document asynchronously. You must poll until status is `Accepted` or `Rejected`.

### Polling job

```csharp
// Runs every N minutes via Hangfire recurring job
public async Task PollKSeFStatus(CancellationToken ct)
{
    var pending = await db.KSeFEntries
        .Where(e => e.Status == KSeFStatus.Submitted)
        .ToListAsync(ct);

    foreach (var entry in pending)
    {
        var result = await kSeFClient.GetStatusAsync(entry.ReferenceNo, ct);

        if (result.Status == "200")  // KSeF accepted
        {
            entry.Status = KSeFStatus.Accepted;
            entry.KSeFInvoiceNo = result.InvoiceNumber;
            entry.UpoReceivedAt = DateTime.UtcNow;
            // write UPO document to BC attachment / blob
        }
        else if (result.Status == "ERROR")
        {
            entry.Status = KSeFStatus.Rejected;
            entry.ErrorDescription = result.ExceptionDescription;
        }
        // else still processing — leave as Submitted
    }

    await db.SaveChangesAsync(ct);
}
```

Rate limits apply to polling as well as submission. Use a configurable delay between individual status checks. Do not poll all pending invoices in a tight loop.

### Backoff on errors

If the poll call itself returns a server error (5xx) or times out, apply exponential backoff before the next attempt. Track `LastPolledAt` and `PollAttempts` on the entry.

### Writing UPO back into BC

UPO (Urzędowe Poświadczenie Odbioru) is the legally binding receipt. Store it where accountants can access it without leaving BC:

- As a `Document Attachment` on the `Sales Invoice Header` (binary blob)
- Or in a dedicated linked table with a download link

Update the BC invoice custom fields (`KSeF No.`, `KSeF Acceptance Date`) at the same time. Accountants must be able to filter invoices by `Submit Status` and see UPO directly in the invoice card.

---

## 4. Reconciliation

Run a reconciliation job once per day (or more frequently during high-volume periods).

```
For every Sales Invoice in BC with posting date in range:
  1. Does a KSeF Invoice Entry exist?       → if not: alert "missing entry"
  2. Is status Accepted with UPO received?  → OK
  3. Is status Submitted for > T hours?     → alert "stuck"
  4. Is status Rejected?                    → alert "rejected, needs resubmit"
  5. Is status Error?                       → alert "submission failed"
```

The reconciliation job is the safety net for silent failures: network drops after submission before the response was written, process crashes, BC environment issues. Without it, stuck invoices accumulate until the auditor finds them.

Write reconciliation results to a dedicated log table with timestamp and action taken. Alert (email, Teams webhook, or BC notification) on any anomaly.

---

## 5. Test vs Production Environments

KSeF exposes two base URLs:

| Environment | Base URL |
|---|---|
| Test | `https://ksef-test.mf.gov.pl/api` |
| Production | `https://ksef.mf.gov.pl/api` |

Configuration must select the environment without code changes:

```json
{
  "KSeF": {
    "BaseUrl": "https://ksef-test.mf.gov.pl/api",
    "AuthToken": "<token>",
    "IdempotencyKeyPrefix": "bc"
  }
}
```

The test environment has different rate limits, sometimes different error codes, and does not generate legally valid UPOs. Build your error handling against both environments from the start — do not assume test behavior matches production.

Never hardcode the base URL or switch it with a `#if DEBUG` flag. The integration must be deployable to production against test KSeF (for UAT) and to production against production KSeF.

---

## Where It Commonly Breaks

### Field mapping edge cases

- NIP with spaces or country prefix (`PL1234567890`) — KSeF returns `400` with a schema error; the XML is valid but the value fails KSeF-specific validation
- Missing `BankAccount` on the seller side — FA(2) v2 requires it for certain invoice types
- `UnitOfMeasure` codes — BC uses its own UoM codes; KSeF expects GUS-standard codes (e.g., `szt`, `kg`, `m`)

### Number series collisions

If BC has multiple invoice number series (e.g., per warehouse, per salesperson), the idempotency key scheme must cover all of them without collisions. Include the series code in the key prefix.

### Invoice states diverging between BC and KSeF

If a BC invoice is cancelled after it has been submitted to KSeF but before UPO arrives, you must issue a corrective invoice (KOR) in KSeF — you cannot cancel a submitted document by simply deleting it in BC. The state machine must prevent BC cancellation of a submitted-but-not-yet-accepted invoice without a corresponding KSeF corrective action.

### Retry creating duplicates

A retry without idempotency key check will submit the same invoice twice. KSeF may accept both if they have different reference numbers. Deduplication then requires a manual KSeF portal intervention. Always check `Status == Pending` before submitting and use the idempotency key.

### Credit note references to pre-integration invoices

Credit notes for invoices issued before the integration went live cannot reference a KSeF number (there is none). Handle this case explicitly: either submit the corrective invoice without the reference (if KSeF rules allow for the period) or document the manual exception path.

### Accountant-facing status vs technical state

Technical states (`Submitted`, `Polling`, `Retry3`) mean nothing to an accountant. Provide a dedicated BC page or FactBox with human-readable status:

| Technical | Accountant sees |
|---|---|
| `Pending` | Queued for sending |
| `Submitted` | Sent — awaiting confirmation |
| `Accepted` | Confirmed by KSeF |
| `Rejected` | Rejected — action required |
| `Error` | System error — contact IT |

---

## Production Reference

Systems built with this architecture have processed over 40,000 documents in production with 100% UPO delivery and no lost or duplicate invoices. The key factors: persistent queue before any HTTP call, deterministic idempotency keys, and daily reconciliation that catches what the happy path misses.

---

## Further Reading

- Full walkthrough with BC-specific implementation notes: [KSeF and Dynamics 365 Business Central: how to connect them](https://www.solutionbox.cz/blog/ksef-business-central-napojeni)
- [01-ksef-api-basics.md](01-ksef-api-basics.md) — session auth, token lifecycle, base URL selection
- [02-fa2-xml-schema.md](02-fa2-xml-schema.md) — FA(2) schema reference and validation
- [03-durable-submit.md](03-durable-submit.md) — submit pipeline, idempotency, Hangfire setup
- [04-status-polling.md](04-status-polling.md) — UPO polling, backoff, alert thresholds

---

Maintained by SolutionBox — https://www.solutionbox.cz
