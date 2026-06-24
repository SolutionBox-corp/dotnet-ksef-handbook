# 02 — Authentication & Authorization

## Authentication Methods

KSeF supports several authentication mechanisms. For machine integration only two matter:

| Method | Identity | Carries permissions? | Valid until |
|---|---|---|---|
| **Token** | Declared at generation time | Yes — encoded into the token | 31 Dec 2026 |
| **KSeF certificate** | Issued inside KSeF | No — separate role assignment required | Max 2 years from issuance |
| Qualified electronic signature | Natural person | No | — |
| Qualified electronic seal | Organization (NIP) | No | — |
| Profil Zaufany | Natural person | No | — |

For a .NET application, the practical choice is token (2026) → KSeF certificate (2027 onwards).

---

## Token vs. KSeF Certificate

**Token** — a string generated inside the KSeF portal. Permissions (issue / view / manage) are
baked in at generation time. Present the token directly as a session credential. Simple to start
with; dead after 31 December 2026.

**KSeF certificate** — issued by KSeF, cryptographically bound to the taxpayer. Acts purely as
proof of identity; it carries no permissions on its own. Rights are assigned to the certificate
identity via roles inside KSeF (see [Roles and Permissions](#roles-and-permissions-uprawnienia)
below). Also required from 1 February 2026 to sign invoices issued in offline / emergency mode.
Valid for a maximum of 2 years from issuance (or from a taxpayer-chosen start date).

The difference is architectural: with a token the application is both authenticated and authorized
in one step; with a certificate authentication and authorization are decoupled. Plan your
authorization model around the certificate path from the start.

---

## The Hard Deadline: 1 January 2027

- Tokens and KSeF certificates coexist from **1 February 2026**.
- **Tokens stop working on 31 December 2026.** After midnight, any integration using only a token
  will fail to open a session and will not deliver a single invoice.
- **KSeF 1.0 tokens do not work in KSeF 2.0.** Regenerate them from 1 February 2026.

Use a token to get your integration running in 2026. Build the certificate path in parallel and
switch before the deadline — do not leave it until Q4 2026 when accounting-period pressure is high.

---

## Roles and Permissions (uprawnienia)

Authenticating is only half the problem. The authenticated identity also needs rights.

KSeF defines three permission classes:

- **Issue** (wystawianie) — send structured invoices (FA(3)) into KSeF
- **View** (przeglądanie) — read invoices belonging to the taxpayer
- **Manage** (zarządzanie) — administer KSeF settings and delegate permissions to others

### How permissions are assigned

**Company with a qualified electronic seal whose certificate includes the company NIP** — KSeF
grants access automatically. No ZAW-FA required.

**Company without a qualified seal** — must file a **ZAW-FA** notification with the tax office
(paper or electronic) designating a natural person (director, CFO, authorized employee) to act for
it. The updated ZAW-FA form applies from 1 February 2026. During the mandatory KSeF period the
right to issue is granted automatically based on the filed notification.

The designated person can then delegate issue / view / manage rights to further identities inside
KSeF — including the certificate identity used by your application. Your application does not need
a human to log in each session; it needs its own certificate identity with the correct rights
pre-assigned.

**Verify the application identity has the required rights before deploying to production.**
A session that opens successfully but lacks issue permission will still reject every invoice
submission.

---

## Session Flow Over the API

KSeF's API is session-based. The rough flow:

1. **Open session** — POST a signed XML challenge (or a token string) to the authentication
   endpoint. KSeF returns a **session token** (short-lived bearer).
2. **Use session token** — include it in every subsequent API call (invoice submission, status
   polling, UPO download).
3. **Session expires** — renew before expiry; expired sessions reject all operations silently or
   with a non-obvious error code.

```csharp
// Illustrative — adapt to the actual KSeF API client you use.
// Do not leak session tokens or certificate private keys outside this layer.

public sealed class KSeFSessionClient
{
    private readonly HttpClient _http;
    private readonly IKSeFCredential _credential; // token string or certificate

    public async Task<string> OpenSessionAsync(CancellationToken ct)
    {
        // Build the signed init request according to the current FA(3) schema.
        var initRequest = _credential.BuildInitRequest();

        var response = await _http.PostAsJsonAsync("/api/online/Session/InitSigned", initRequest, ct);
        response.EnsureSuccessStatusCode();

        var body = await response.Content.ReadFromJsonAsync<SessionResponse>(ct);
        return body!.SessionToken; // store and reuse until expiry
    }
}

// Wrap KSeF's SDK or raw HTTP behind your own interface.
// Error codes, session TTL quirks, and schema versions change between KSeF releases —
// isolate them here so a schema bump changes one file, not your entire codebase.
public interface IKSeFCredential
{
    object BuildInitRequest();
}
```

Keep the KSeF client behind an anti-corruption interface. KSeF's error strings, numeric codes,
and session TTL behaviour are unstable across releases; isolating them prevents leakage into
business logic.

---

## Certificate Lifecycle

The KSeF certificate is valid for at most 2 years. Treat renewal as an ongoing operational
process, not a one-time setup step:

- Track the expiry date in your deployment configuration or secrets manager.
- Alert well before expiry (30 days minimum). A lapsed certificate locks your integration
  completely until a new one is issued and its identity is re-granted the correct roles.
- After renewal, re-verify role assignments for the new certificate identity before cutover.

---

## Checklist for a .NET Integration

- [ ] **Application identity** — KSeF certificate in production; token only as a 2026 stopgap.
- [ ] **Roles assigned** — application certificate identity has issue + view rights in KSeF before
  first deploy.
- [ ] **ZAW-FA filed** (if the company has no qualified seal with NIP).
- [ ] **Session management** — open, reuse, renew; never fire-and-forget without a valid session.
- [ ] **Anti-corruption layer** — KSeF SDK wrapped behind your own interface.
- [ ] **Certificate expiry alert** — operational process, not a one-off.
- [ ] **Migration deadline** — certificate path ready before 1 January 2027.

---

Full write-up: https://www.solutionbox.cz/blog/ksef-token-certifikat

Maintained by SolutionBox — https://www.solutionbox.cz
