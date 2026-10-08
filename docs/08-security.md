# Frontend and Web Application Security: Interview Guide

## 1. Security starts with trust boundaries

The browser is controlled by the user. Requests, local storage, hidden controls,
and frontend code can be inspected or modified.

```text
Untrusted browser -> Authenticated API -> Authorized business logic
                  -> Data services and database
```

Frontend validation improves UX but cannot enforce authorization or financial
correctness. Protect sensitive operations on the server.

Begin with a threat model: assets, actors, entry points, trust boundaries,
abuse scenarios, and controls. Consider account data, authentication sessions,
payment integrity, and dependencies.

This is defensive interview guidance, not a vulnerability review of a project.

## 2. Authentication versus authorization

- **Authentication:** Who is making the request?
- **Authorization:** Is that identity allowed to perform this operation on this
  specific resource?

A logged-in user must not access another user's statement by changing an ID.
Every relevant backend operation needs object-level and tenant-level checks.

Role-based access is useful, but roles alone may not capture account ownership,
transaction limits, or business conditions.

Hiding a menu item or using a route guard is a UX measure, not an access-control
boundary.

## 3. XSS: Cross-Site Scripting

XSS occurs when untrusted content executes as script in the application's origin.
It can compromise visible data and authenticated actions.

Common sources include rendered HTML, unsafe DOM APIs, third-party scripts, and
incorrectly handled URLs.

### React protections and limits

React escapes ordinary text interpolation:

```jsx
<p>{userProvidedComment}</p>
```

That does not make every React application immune to XSS. Raw HTML insertion,
unsafe URL handling, direct DOM manipulation, and compromised dependencies
remain risks.

If rich HTML is required, sanitize with a maintained, appropriate sanitizer
under a narrowly defined policy. Output encoding must match the context; HTML
encoding is not a universal solution for JavaScript or URLs.

### Content Security Policy

CSP can restrict script execution and other resource sources. A nonce- or
hash-based policy can reduce exposure, but requires integration with actual
rendering and asset delivery.

Roll out with reporting and test functionality. Avoid broad allowances that
defeat the intended protection. CSP is defense in depth, not a replacement for
safe rendering.

## 4. CSRF: Cross-Site Request Forgery

CSRF can cause a browser to send an unwanted authenticated request when
credentials such as cookies are attached automatically.

Controls may include:

- Framework-supported CSRF tokens and validation.
- Appropriate SameSite cookie policy.
- Origin checks as part of a suitable server policy.
- No state-changing operations through GET.

SameSite reduces risk but is not a universal substitute for CSRF protections,
especially when deployments and cross-site workflows require weaker settings.

An explicitly attached authorization token changes the CSRF threat model,
but does not eliminate XSS or token-theft concerns.

## 5. Sessions, tokens, and browser storage

Where the architecture permits, secure cookies can reduce direct token exposure:

```http
Set-Cookie: __Host-session=<opaque-value>; Path=/; Secure; HttpOnly; SameSite=Lax
```

This is an illustrative host-only cookie. Choose SameSite and session policies
according to the actual authentication flow.

- **Secure:** Send over HTTPS.
- **HttpOnly:** Prevent JavaScript from reading the cookie.
- **SameSite:** Restrict certain cross-site cookie sending.
- **__Host- prefix:** Adds browser-enforced constraints in supporting browsers,
  including no Domain attribute and Path=/ with Secure.

HttpOnly does not stop injected JavaScript from making authenticated requests.

Avoid treating localStorage as a safe vault for credentials. It is readable by
JavaScript in the origin and persists beyond a tab.

Plan session expiry, logout/revocation, rotation, refresh behavior, and sensitive
operation reauthentication.

For OAuth/OIDC browser flows, use current provider/library guidance, commonly
Authorization Code with PKCE. Validate redirect destinations and protocol
parameters appropriately. PKCE does not solve XSS, and browser apps cannot keep
a client secret private.

## 6. CORS is not authentication

CORS controls whether browser JavaScript can read certain cross-origin
responses. It does not prevent every request from being sent or stop non-browser
clients from calling an endpoint.

Use an explicit origin policy where cross-origin access is required.
Credentialed access cannot use a wildcard allowed origin.

Server authorization is required regardless of CORS.

## 7. API and transaction safety

At the server boundary:

- Validate schemas, sizes, types, and business rules.
- Authorize each requested object and action.
- Use parameterized database operations.
- Enforce transaction limits and authoritative amounts.
- Use idempotency for retryable operations such as payment submission.
- Apply rate limits and bounded concurrency.
- Return safe error responses while retaining diagnostic server logs.

Network timeouts can leave an operation's outcome uncertain. Do not retry a
non-idempotent transfer as though it definitely failed.

Client-generated IDs and prices are inputs, not authoritative values.

## 8. HTTPS, headers, and content boundaries

Use HTTPS throughout appropriate transport paths. HSTS can encourage browsers
to use HTTPS, but preload and subdomain policies need deliberate planning.

Other protections include:

- CSP `frame-ancestors` to restrict embedding and reduce clickjacking risk.
- Suitable `X-Content-Type-Options: nosniff`.
- An intentional Referrer-Policy.
- Appropriate cache controls for private responses.
- Safe redirect allowlists.

Security headers are configuration-specific; copying a generic header set does
not establish that the deployment is safe.

Treat file uploads as untrusted: validate type and size, scan where necessary,
store safely, and avoid serving active content in a privileged application
origin.

Server-side fetching also needs protections against unintended internal
destinations. A frontend URL check cannot prevent server-side request forgery.

## 9. Dependencies and supply chain

Third-party JavaScript can run with the application's privileges.

Practices:

- Minimize unnecessary dependencies and third-party scripts.
- Use lockfiles and reproducible installation.
- Review updates and automated vulnerability findings.
- Verify provenance/signatures where supported.
- Pin CI actions to reviewed immutable revisions.
- Protect publishing and deployment credentials.
- Use integrity checks for suitable external static resources.

A vulnerability scanner's severity is one input. Evaluate reachability,
exposure, available fixes, and compensating controls. Exceptions need ownership
and expiry, not silent suppression.

## 10. Secrets and privacy

Never put secrets in frontend bundles. Build-time environment variables exposed
to client code are still public even if their names look private.

Keep credentials server-side and use a managed secret store for deployment.

Avoid collecting:

- Authentication tokens.
- Full account identifiers.
- Sensitive request bodies.
- Personal data unnecessary for debugging.

Apply telemetry redaction, retention limits, access controls, and logout cleanup
of user-specific caches. Do not expose stack traces or confidential integration
details in browser errors.

## 11. Security testing and operational response

Use complementary techniques:

| Technique | Purpose |
| --- | --- |
| Threat modeling | Identify design-level risks |
| SAST | Inspect code for risky patterns |
| SCA | Inspect dependency vulnerabilities/licenses |
| Secret scanning | Detect exposed credentials |
| DAST | Test deployed behavior in an authorized environment |
| Authorization tests | Verify role, ownership, and tenant boundaries |
| Expert review | Evaluate business logic and contextual risks |

Test that an authenticated user cannot access another user's resources, not
just that anonymous requests fail.

Restrict dynamic tests to approved targets and avoid destructive production
operations.

Prepare incident response: identify scope, revoke exposed credentials, contain
the issue, communicate through the required process, and verify remediation.

## 12. Interview scenario

**Problem:** A statement download button is hidden for unauthorized users.

**Why that is insufficient:** The user can call the download API directly.

**Correct design:** The server checks session, tenant, and statement ownership
before serving it, and uses a safe cache/storage policy. The UI hides unavailable
actions only to improve usability.

## 13. Interview questions

**Does React prevent all XSS?**

No. It escapes ordinary text, but unsafe HTML, DOM operations, URLs, and external
code can still introduce vulnerabilities.

**Does HttpOnly solve XSS?**

No. It limits cookie reading, but injected scripts can still act in the user's
session.

**Is CORS an API security boundary?**

It is a browser cross-origin response policy, not a replacement for identity
and authorization checks.

**How do you protect banking operations?**

Enforce server authorization and business validation, use secure sessions and
transport, prevent injection/CSRF, and design idempotency and auditability.

## 14. Interview summary

> I treat the browser as untrusted, enforce authorization and business rules on
> the server, and use safe rendering, session controls, CSRF protection, secure
> transport, and dependency hygiene. I combine threat modeling, targeted tests,
> scanning, and operational controls rather than relying on one framework.

## References

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP XSS prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP CSRF prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [MDN Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP)
