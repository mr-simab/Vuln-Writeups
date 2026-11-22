# Email Case Variation and Login Rate Limiting

> **Question:** Can different capitalization of the same email address evade the login endpoint's failed-attempt counter?
>
> **Result:** The original assessment reported a rate-limit bypass. Its captured responses show an initial `401`, a rate-limited `429`, a subsequent `401` for a case-variant email, and then a `200` with authentication cookies. However, the `429` response indicated **13 seconds until reset**, while the next captured request occurred **18 seconds later**.
> 
> **Scope:** Authorized testing conducted on **3 September 2026**. The vulnerability was subsequently remediated by the organization following this disclosure.

## 1. At a glance

| Field | Finding |
| --- | --- |
| Research topic | Case-sensitive email identifiers and login rate-limit counters |
| Application | UpToSkills |
| Observed endpoint | `POST https://gateway.uptoskills.com/api/auth/login` |
| Base URL in written report | `https://gateway.uptoskills.in` |
| Class | Rate-limit logic / identifier canonicalization |
| Severity | **Medium** |
| Methodology | Authorized, manual web/API testing using Burp Suite Repeater |
| Observed server policy header | `RateLimit-Policy: 5;w=120` |
| Evidence status | Rate limiting confirmed, capitalization-based bypass **inconclusive** |

## 2. Background: why email canonicalization matters

A login rate limiter should protect the *account*, not merely a particular spelling of its identifier. If the login system recognizes `researcher@example.invalid` and `Researcher@example.invalid` as the same account, then per-account failed-login counters generally need to treat those inputs as the same identity as well.

A possible implementation mistake is to derive the counter key directly from the raw, unnormalized email string:

```text
Hypothetical weak design                 Intended account-level behavior

"researcher@example.invalid" ──► key A   "researcher@example.invalid" ─┐
"Researcher@example.invalid" ──► key B   "Researcher@example.invalid" ─┴─► one account counter
```

These example addresses are illustrative and are **not** the researcher's original credentials. This is a description of a possible failure mode, **not** proof of the server's internal implementation.

## 3. Test setup and evidence handling

The assessment was performed through the login endpoint with Burp Suite Repeater. The test account's email capitalization was changed between requests. The original screenshots show the same API path, rate-limit headers, and HTTP outcomes.

All four figures below were extracted from the source Word document and **sanitized with opaque masks** covering account identifiers, passwords, cookies, CAPTCHA/challenge content, and unrelated identifying details. They have not been recreated from scratch, and no server response values were changed. Screenshots are included to document the historical observation, not to establish current exploitability.

The written report lists `gateway.uptoskills.in`, whereas the screenshots' host/target is `gateway.uptoskills.com`. **This inconsistency is retained rather than silently reconciled**. The observed endpoint above follows the HTTP screenshots.

## 4. Reproduction sequence and recorded results

### Step 1 - Capture an invalid login attempt

A login request was captured for an test account. The server denied the attempt with a generic authentication failure.

**Recorded response excerpt:**

```http
HTTP/2 401 Unauthorized
Date: Thu, 03 Sep 2026 17:22:21 GMT
RateLimit-Policy: 5;w=120
RateLimit-Limit: 5
RateLimit-Remaining: 3
RateLimit-Reset: 119
Content-Type: application/json; charset=utf-8

{"error":"Invalid email or password"}
```

![Sanitized Burp Suite evidence showing an initial HTTP 401 and rate-limit headers](assets/01-initial-401.png)

*Figure 1 - Initial `401 Unauthorized`. The server reports an attempt budget and a 120-second window.*

### Step 2 - Observe the rate limiter activate

After additional failed attempts, the same endpoint returned `429 Too Many Requests`. The evidence confirms that **a rate limiter exists**.

**Recorded response excerpt:**

```http
HTTP/2 429 Too Many Requests
Date: Thu, 03 Sep 2026 17:24:07 GMT
RateLimit-Policy: 5;w=120
RateLimit-Remaining: 0
RateLimit-Reset: 13
Retry-After: 13
Content-Type: application/json; charset=utf-8

{
  "error": "Too many login attempts. Please try again after 2 minutes.",
  "retryAfter": 120
}
```

![Sanitized Burp evidence showing HTTP 429 with Retry-After 13 seconds](assets/02-throttle-429.png)

*Figure 2 - Protection activated. Crucially, `RateLimit-Reset` and `Retry-After` report only **13 seconds** remaining.*

**Observation:** The human-readable error and JSON `retryAfter` mention 120 seconds, while the HTTP `Retry-After` and `RateLimit-Reset` headers show 13 seconds. This distinction affects interpretation of the later response.

### Step 3 - Change the email's letter case and retry

The report states that the researcher changed the capitalization of alphabetic characters in the email address, without treating it as a different logical account. The next captured request returned `401 Unauthorized` instead of the previous `429`.

**Recorded response excerpt:**

```http
HTTP/2 401 Unauthorized
Date: Thu, 03 Sep 2026 17:24:25 GMT
RateLimit-Policy: 5;w=120
RateLimit-Remaining: 4
RateLimit-Reset: 120

{"error":"Invalid email or password"}
```

![Sanitized Burp screenshot showing a case-variant request receiving 401 and a renewed counter](assets/03-case-variant-401.png)

*Figure 3 - Case-variant request returned `401`, with `RateLimit-Remaining: 4` and `RateLimit-Reset: 120`.*

**Important timing check:** This response was captured at **17:24:25 GMT**, **18 seconds** after the `429` response at **17:24:07 GMT**. Because the earlier headers stated that reset was **13 seconds** away, the original limit could have expired naturally before the case-variant request. A new full window is therefore *not conclusive evidence* of a bypass.

### Step 4 - Review the later successful login response

A further request returned `HTTP 200 OK`, with `Set-Cookie` headers for authentication tokens shown in the original screenshot.

**Recorded response excerpt:**

```http
HTTP/2 200 OK
Date: Thu, 03 Sep 2026 17:24:50 GMT
RateLimit-Policy: 5;w=120
RateLimit-Remaining: 3
RateLimit-Reset: 95
Set-Cookie: token=<REDACTED>
Set-Cookie: refreshToken=<REDACTED>
```

![Sanitized Burp Suite evidence showing a subsequent 200 with rate-limit headers](assets/04-later-200.png)

*Figure 4 - Later `200 OK`; credential-bearing cookies and response data are completely covered in the public copy.*

A `200 OK` accompanied by authentication cookies indicates a successful authentication response in this screenshot. It should **not** be described as an authentication bypass or proof that invalid passwords were accepted. This response was also recorded **after** the original rate-limit window could already have reset.

## 5. Evidence timeline

| Response | Screenshot timestamp (UTC) | Time since `429` | Remaining/reset reported | Interpretation |
| --- | --- | --- | --- | --- |
| Initial `401` | 17:22:21 | - | 3 remaining; 119 s reset | Authentication failure with rate tracking |
| `429` | 17:24:07 | 0 s | 0 remaining; **13 s reset** | Rate limiting confirmed |
| Case-variant `401` | 17:24:25 | **+18 s** | 4 remaining; 120 s reset | Consistent with a new window **or** an identifier-based reset; not distinguishable |
| Later `200` | 17:24:50 | **+43 s** | 3 remaining; 95 s reset | Successful login in the new window; not an auth bypass |

The key analytical constraint is simple: **18 seconds elapsed after a response advertising a 13-second remaining cooldown**. That prevents the screenshots alone from proving that capitalization was the cause of renewed attempts.

## 6. Expected behavior vs. what the test establishes

| Question | Expected behavior | What the evidence establishes |
| --- | --- | --- |
| Are repeated login attempts limited? | Eventually reject excessive failures | **Yes** - the endpoint returned `429` |
| Do case variants share one account-level limiter? | Yes, if they refer to the same account | **Not conclusively tested within a still-active cooldown** |
| Did the case variant obtain additional attempts before reset? | Should not, if account-wide throttling is active | **Unresolved** - captured attempt followed the reported reset time |
| Did any test bypass password authentication? | No | **No evidence of authentication bypass** |
| Is the limiter keyed by raw email strings? | Ideally use a stable account identifier | **Hypothesis; backend implementation not inspected** |

## 7. Security impact and limits

**Confirmed:** A login rate limiter was present, visibly blocked attempts with HTTP 429, and later accepted new requests after its advertised short remaining window elapsed.

**Potential impact if subsequently confirmed:** A limiter keyed to raw email input could grant additional password-guessing attempts through case variations for the same underlying account, reducing the effectiveness of per-account throttling and credential-stuffing defenses.

## 8. What a conclusive retest would require

The original hypothesis is straightforward to validate in an **explicitly authorized, controlled environment**:

1. Establish an active rate-limit window using one authorized test account, and record the precise server-side reset deadline.
2. Verify that the original identifier is still blocked **before the reset deadline**.
3. Compare an email case-variant request **within that same active window**, while keeping all other meaningful conditions consistent (account, source, session, CAPTCHA state, and timing).
4. Compare status, `Retry-After`, `RateLimit-Remaining`, `RateLimit-Reset`, and authentication behavior; repeat with a control request using the original casing.
5. Confirm whether both spellings resolve to the same application account; otherwise, they are not a valid per-account comparison.

A finding would be strengthened by server-side logs or implementation evidence showing that the limit is keyed on unnormalized email strings. No such retest or internal evidence is supplied with the report, and **none was performed for this write-up**.

## 9. Recommended hardening

- **Key throttling to a stable account identity:** For known accounts, derive the failed-login counter from an internal account ID rather than the raw request email.
- **Normalize consistently:** Apply the application's documented email-identity rules before account lookup and rate-limit evaluation. Handle provider-specific email semantics deliberately; do not assume every provider treats all local-part variations identically.
- **Combine signals:** Maintain suitable per-account and per-source protections to deter distributed guessing and identifier variation.
- **Make cooldown information consistent:** Align `Retry-After`, body `retryAfter`, and UI messaging so analysts and users see one coherent duration.
- **Add regression tests:** Verify different capitalization, whitespace handling, and permitted equivalent identifiers share the intended counters and cannot silently reset protection.
- **Instrument safely:** Log rate-limit decisions and canonical account identifiers without exposing passwords, CAPTCHA tokens, or authentication cookies.

## 10. Conclusion

This assessment highlights a subtle authentication-security question: **whether failed-login protection follows a normalized account identity or a raw input string**. The test positively establishes that throttling exists. The original report attributes later accepted requests to changing the email's case, but **the screenshot timeline also supports ordinary cooldown expiration**.

Instead of claiming an unproven exploit, the strongest conclusion is that the potential canonicalization weakness **requires a controlled within-window comparison**. This distinction matters when writing reproducible, trustworthy security research.

---
