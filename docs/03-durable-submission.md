# 03 — Durable Submission Pattern

KSeF is asynchronous, rate-limited, and subject to outages. A synchronous "call the API in a controller" approach will lose invoices — silently, at exactly the wrong moment. This section describes the architecture that prevents that.

The core rule: **every KSeF call is a durable, idempotent, persisted unit of work**. Never fire-and-forget. Never inside an HTTP request.

---

## Why Fire-and-Forget Fails

When you POST an invoice synchronously from a web request:

- A KSeF timeout → exception → invoice disappears
- An application restart mid-submission → invoice disappears
- A retry without idempotence → invoice appears twice in the government system
- A rate-limit 429 → exception → invoice disappears

Each of these is a real failure mode. KSeF returns transient errors, has maintenance windows, and imposes per-client quotas. The architecture below handles all of them.

---

## The Seven Pillars

### 1. Persist Before Sending

The invoice is written to the database as a task before any external call is made. If the application crashes after persistence but before the HTTP call, the background worker will pick it up on restart. Nothing is lost.

```csharp
// Invoice is enqueued, never sent directly from the web layer
public async Task EnqueueInvoiceAsync(Invoice invoice, CancellationToken ct)
{
    var key = IdempotencyKey.For(invoice); // deterministic — see below

    var existing = await _db.KSeFSubmissions
        .FirstOrDefaultAsync(s => s.IdempotencyKey == key, ct);

    if (existing is not null)
        return; // already enqueued or submitted — safe to ignore

    _db.KSeFSubmissions.Add(new KSeFSubmission
    {
        IdempotencyKey = key,
        InvoiceId      = invoice.Id,
        Status         = SubmissionStatus.Pending,
        CreatedAt      = DateTime.UtcNow,
    });

    await _db.SaveChangesAsync(ct);
}
```

A background worker (`IHostedService` or Hangfire job) polls the `Pending` queue and performs the actual HTTP call.

---

### 2. Deterministic Idempotency Key

The key must be derived from invoice content, not from a random UUID. A random key defeats idempotency: if the key is regenerated on retry, you submit the invoice twice.

```csharp
public static class IdempotencyKey
{
    // Derive from stable invoice identity — number, date, issuer NIP.
    // SHA-256 gives a fixed-length, safe-to-store string.
    public static string For(Invoice invoice)
    {
        var raw = $"{invoice.Number}|{invoice.IssueDate:yyyy-MM-dd}|{invoice.IssuerNip}";
        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(raw));
        return Convert.ToHexString(hash).ToLowerInvariant();
    }
}
```

Before every submission the worker checks whether an entry with this key already exists in the submissions table. If it does — and its status is `Submitted` or `Accepted` — the call is skipped. If status is `Pending` or `Failed`, it proceeds.

A corrected invoice (content changed) automatically gets a different key, so resubmission after a rejection is safe.

---

### 3. Retry with Exponential Backoff and Jitter

Transient errors are the norm. A hand-rolled `for` loop without jitter will flood a recovering API with a thundering herd from all running instances simultaneously.

Use Polly:

```csharp
public static AsyncRetryPolicy<HttpResponseMessage> BuildKSeFRetryPolicy()
{
    return Policy<HttpResponseMessage>
        .Handle<HttpRequestException>()
        .OrResult(r => r.StatusCode is
            HttpStatusCode.TooManyRequests or
            HttpStatusCode.ServiceUnavailable or
            HttpStatusCode.GatewayTimeout)
        .WaitAndRetryAsync(
            retryCount: 5,
            sleepDurationProvider: (attempt, result, _) =>
            {
                // Respect Retry-After header when KSeF sends one (429)
                if (result.Result?.Headers.RetryAfter?.Delta is { } retryAfter)
                    return retryAfter + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 200));

                // Otherwise: exponential backoff — 2s, 4s, 8s, 16s, 32s
                return TimeSpan.FromSeconds(Math.Pow(2, attempt))
                     + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 500));
            },
            onRetryAsync: async (result, delay, attempt, _) =>
            {
                _logger.LogWarning(
                    "KSeF retry {Attempt}/5 in {Delay}s — {Reason}",
                    attempt, delay.TotalSeconds,
                    result.Exception?.Message ?? result.Result?.StatusCode.ToString());

                await Task.CompletedTask;
            });
}
```

Register the policy via `Polly.Extensions` on `IHttpClientBuilder`:

```csharp
services.AddHttpClient<IKSeFClient, KSeFClient>()
        .AddPolicyHandler(KSeFPolicies.BuildKSeFRetryPolicy());
```

---

### 4. Shared Rate Limiting Across Instances

An in-memory token bucket is sufficient for a single instance. The moment you add a second instance for availability or scale, both processes consume quota independently and together easily exceed the KSeF limit.

Use a Redis-backed token bucket (e.g., `RedisRateLimiter` from `StackExchange.Redis` with a Lua script, or a library such as `RateLimiter` from `Polly.RateLimiting`):

```csharp
// Pseudo-code: acquire a token before every outbound KSeF request.
// The bucket is stored in Redis — all instances share the same counter.
public async Task<bool> TryAcquireAsync(CancellationToken ct)
{
    // Lua script atomically decrements a Redis counter and
    // sets a TTL matching the KSeF quota window (e.g., 1 second).
    var allowed = await _redis.ScriptEvaluateAsync(_tokenBucketScript,
        keys: new RedisKey[] { "ksef:rate:tokens" },
        values: new RedisValue[] { _capacityPerSecond, DateTimeOffset.UtcNow.ToUnixTimeMilliseconds() });

    return (bool)allowed;
}
```

If `TryAcquireAsync` returns `false`, the submission task stays in `Pending` and is retried after the next backoff interval — it does not fail.

---

### 5. State Persistence and Status Tracking

Every state transition is written to the database and to the structured audit log. The submissions table is the authoritative view of what happened.

| Status      | Meaning                                           |
|-------------|---------------------------------------------------|
| `Pending`   | Enqueued, not yet attempted                       |
| `Submitted` | Accepted by KSeF; waiting for async result (UPO) |
| `Accepted`  | UPO received — invoice confirmed by the authority |
| `Rejected`  | KSeF returned a validation rejection              |
| `Failed`    | Terminal error after all retries exhausted        |

Only `Accepted` means the invoice is done. Everything else requires monitoring or human action.

```csharp
// After successful HTTP submission, update status and store the KSeF reference number.
submission.Status          = SubmissionStatus.Submitted;
submission.KSeFReferenceNo = response.ReferenceNumber;
submission.SubmittedAt     = DateTime.UtcNow;
await _db.SaveChangesAsync(ct);

_logger.LogInformation(
    "Invoice {InvoiceId} submitted to KSeF. Reference: {Ref}",
    submission.InvoiceId, submission.KSeFReferenceNo);
```

---

### 6. UPO Polling

Submission is step 1. KSeF processes asynchronously and the UPO (Urzędowe Poświadczenie Odbioru — the official receipt) arrives later. Without polling, invoices stay `Submitted` forever.

A separate polling job runs every N minutes and queries KSeF for the status of all `Submitted` records:

```csharp
public async Task PollPendingUpoAsync(CancellationToken ct)
{
    var submitted = await _db.KSeFSubmissions
        .Where(s => s.Status == SubmissionStatus.Submitted)
        .ToListAsync(ct);

    foreach (var submission in submitted)
    {
        var result = await _ksefClient.GetSubmissionStatusAsync(
            submission.KSeFReferenceNo, ct);

        if (result.IsAccepted)
        {
            submission.Status    = SubmissionStatus.Accepted;
            submission.UpoXml    = result.UpoXml;
            submission.AcceptedAt = DateTime.UtcNow;
        }
        else if (result.IsRejected)
        {
            submission.Status       = SubmissionStatus.Rejected;
            submission.RejectionCode = result.ErrorCode;
            submission.RejectionMsg  = result.ErrorMessage;

            _alerts.Raise(AlertLevel.Warning,
                $"KSeF rejected invoice {submission.InvoiceId}: {result.ErrorCode}");
        }
        // else still processing — leave as Submitted, poll again next cycle
    }

    await _db.SaveChangesAsync(ct);
}
```

UPO XML is stored in the database. An invoice without a stored UPO is not a completed invoice regardless of what the application UI shows.

---

### 7. Reconciliation Job

This is the piece most first-version integrations skip. Retry handles transient failures during submission. Reconciliation handles the failure of the entire system to maintain consistent state — stuck records, orphaned submissions, silently dropped transitions.

Run daily (or hourly for high-volume pipelines):

```csharp
// Flag anything stuck in Submitted for longer than a reasonable processing window.
var stuckThreshold = DateTime.UtcNow - TimeSpan.FromHours(2);

var stuckSubmissions = await _db.KSeFSubmissions
    .Where(s => s.Status == SubmissionStatus.Submitted
             && s.SubmittedAt < stuckThreshold)
    .ToListAsync(ct);

foreach (var s in stuckSubmissions)
{
    _alerts.Raise(AlertLevel.Error,
        $"Submission {s.KSeFReferenceNo} stuck in Submitted since {s.SubmittedAt:u}. " +
        $"Invoice {s.InvoiceId} — manual check required.");
}

// Also verify: every invoice in a terminal state (Accepted/Rejected)
// has a corresponding local record. Surface orphans if KSeF shows
// a reference number you have no record of.
```

The reconciliation job is the answer to "do you actually know what happened?" Without it, you are relying on nothing having gone wrong. In a distributed system that is not monitoring — it is hoping.

---

## Full Flow Summary

```
Web layer           DB                  Background worker          KSeF
    │                │                        │                      │
    │── EnqueueInvoice ──► Insert(Pending) ───►│                      │
    │                │                        │── AcquireRateLimit    │
    │                │                        │── HTTP POST ─────────►│
    │                │                        │◄── 202 + ReferenceNo ─│
    │                │◄── Update(Submitted) ───│                      │
    │                │                        │                       │
    │           [polling job]                 │                       │
    │                │                        │── GetStatus ─────────►│
    │                │                        │◄── UPO XML ───────────│
    │                │◄── Update(Accepted) ───│                       │
    │                │                        │                       │
    │           [reconciliation job]          │                       │
    │                │── Check stuck/orphans ──────────────────────►  │
```

---

## Production Evidence

This pattern has run **40,000+ invoices in production, 100% delivered**. During forensic recovery of a previous fire-and-forget pipeline, 15,141 invoices that had been silently lost were retrieved and reprocessed. Every failure mode described above — session expiry, 429 rate limiting, timeout during outage, missing UPO polling, absent reconciliation — corresponds to a real incident from that recovery.

---

## Related Reading

- [Robust KSeF Integration: Why Most Implementations Fail and How to Do It Right](https://www.solutionbox.cz/blog/robustni-ksef-integrace) — architecture overview and checklist
- [KSeF API in .NET: the most common errors and how to handle them](https://www.solutionbox.cz/blog/ksef-api-chyby-dotnet) — per-error-category patterns with code

---

Maintained by SolutionBox — https://www.solutionbox.cz
