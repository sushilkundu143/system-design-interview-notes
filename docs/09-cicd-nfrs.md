# CI/CD for Non-Functional Requirements: Interview Guide

## 1. What are NFRs?

Functional requirements describe what a system does. Non-functional requirements
describe the qualities and constraints under which it does it.

Examples:

- Accessibility and usability.
- Performance and scalability.
- Security and privacy.
- Reliability and availability.
- Maintainability, compatibility, and observability.

"The transfer form submits" is functional. "It is keyboard-operable, avoids
duplicate transfers, and meets an agreed latency target" describes quality
requirements around that function.

NFRs are not optional polish. They should become measurable acceptance criteria.

## 2. CI, delivery, and deployment

- **Continuous Integration:** Frequently integrate changes and run automated
  verification.
- **Continuous Delivery:** Keep verified changes ready for release; production
  approval can remain manual.
- **Continuous Deployment:** Automatically release changes that pass the pipeline.

CI/CD makes verification repeatable. It cannot prove every quality attribute or
replace production monitoring and manual evaluation.

## 3. Translate vague goals into evidence

| Goal | Concrete evidence | Important qualification |
| --- | --- | --- |
| Accessible | No newly detected serious violations; keyboard-flow checks | Automated scans do not prove WCAG conformance |
| Fast | Bundle budget and controlled lab checks; production Web Vitals | Lab results are not field percentiles |
| Secure | Reviewed high-risk findings and authorization tests | No findings is not proof of no vulnerabilities |
| Reliable | Error-rate/latency SLOs and recovery drills | Define traffic, time window, and excluded cases |
| Scalable | Load test at specified traffic mix and data size | Production capacity can differ from test |

Every gate needs an owner, measurement method, threshold, failure behavior, and
an exception policy.

## 4. Example pipeline

```text
Pull request
  -> Source/secret checks
  -> Dependency installation from lockfile
  -> Lint + types + unit tests
  -> Production build + bundle budget
  -> Integration + accessibility checks
  -> Preview/staging deployment
  -> Critical E2E + selected performance/security tests
  -> Reviewed release approval, if required
  -> Deploy immutable artifact to canary
  -> Observe health and quality
  -> Promote or roll back
```

Run independent checks in parallel where safe. Put fast, deterministic feedback
early; expensive load tests may run nightly or before high-risk releases.

Do not postpone all security or accessibility until a final test stage.

## 5. Accessibility gates

- Lint markup and component usage.
- Run automated checks on representative rendered states.
- Include open dialogs, validation errors, menus, and loading states.
- Test keyboard flows and focus restoration.
- Schedule manual screen-reader and zoom testing for significant UI changes.

A scan of the login page does not establish that the transfer workflow is
accessible. Acceptance criteria should name journeys and states.

Treat pre-existing violations transparently. A baseline can isolate regressions,
but needs a reduction plan and must not silently permit new failures.

## 6. Performance gates

### Build-time checks

- Track initial-route JavaScript and CSS.
- Define raw versus compressed sizes.
- Compare dependency changes and chunk growth.
- Fail significant regressions according to explicit policy.

### Lab checks

Use fixed browser versions, representative pages, controlled resources, and
production builds. Run multiple samples and use an agreed aggregate rather than
one unstable result.

Set both absolute limits and regression limits where useful:

```text
Illustrative policy:
  Initial route compressed JS <= agreed ceiling
  Lab LCP within agreed threshold under defined conditions
  No material regression relative to a comparable baseline
```

Do not present a Lighthouse score alone as a performance contract.

### Field checks

Track deployed LCP, INP, CLS, errors, and task latency by release/device/route.
Low-traffic canaries may not collect enough samples for reliable p75 conclusions.
Use sufficient observation windows and supplement with synthetic checks.

## 7. Security gates

Run source analysis, dependency scanning, secret scanning, and applicable
container/infrastructure checks.

Test actual authorization boundaries and security-sensitive business logic.
Use authorized staging targets for dynamic tests.

Secure the pipeline itself:

- Minimum job/token permissions.
- Reviewed immutable third-party action references.
- No secrets available to untrusted fork code.
- No privileged workflow that checks out and executes untrusted PR content.
- Short-lived deployment credentials through workload identity where supported.
- Protected environments and release branches.
- Artifact provenance and audit trails.

Masking logs is not enough if untrusted code can access credentials.

## 8. Reliability, compatibility, and load testing

Verify:

- Timeouts, retries, and recovery paths.
- Dependency failure behavior.
- Supported browser/device combinations.
- API/schema compatibility.
- Idempotency and duplicate-event handling.
- Bounded concurrency and overload behavior.
- Backup restore and rollback procedures where relevant.

Load tests need defined arrival rates, traffic mix, data volume, warm/cold cache
states, and duration. A tiny test database with a warm cache can hide bottlenecks.

Run fault-injection tests only within an approved scope. Reliability testing
must not put shared environments or real customer operations at risk.

## 9. Artifacts, environments, and secrets

Build once and promote the same immutable artifact where practical.
Record source revision, dependency versions, configuration, and test evidence.

Separate public runtime configuration from secrets. Avoid rebuilding an
unverified artifact differently for production.

Use synthetic test accounts and data. Never copy confidential production
banking data into a broadly accessible preview environment.

A feature flag can reduce rollout exposure, but disabled code still needs
security review and server-side authorization.

## 10. Safe deployment strategies

| Strategy | Mechanism | Trade-off |
| --- | --- | --- |
| Rolling | Gradually replace instances | Old/new versions coexist |
| Blue-green | Switch traffic between two environments | Extra capacity and careful state handling |
| Canary | Release to a small cohort before widening | Requires meaningful monitoring and routing |
| Feature-flag rollout | Enable functionality by cohort | Adds flag lifecycle and compatibility concerns |

For every strategy, verify old/new API compatibility.

Database migrations often need an expand-and-contract sequence:

1. Add compatible schema.
2. Deploy code that tolerates old/new representations.
3. Migrate data.
4. Remove obsolete structures after older code is gone.

Rolling back application code does not undo a destructive database migration.

## 11. SLI, SLO, and error budgets

- **SLI:** The measured indicator, such as successful eligible requests.
- **SLO:** The target over a defined window, such as 99.9% success over 30 days.
- **Error budget:** The amount of failure allowed by that SLO.

An illustrative 99.9% request-success SLO permits 0.1% unsuccessful eligible
requests in the window. It is not automatically equivalent to a fixed number of
outage minutes unless the SLI is time-based.

Define a "good" request precisely: status, latency, and business outcome may all
matter. HTTP 200 alone can hide a failed operation.

Use budget burn and release-specific signals to guide promotion, rollback, or a
pause in risky releases. Alerts should be actionable rather than noisy.

## 12. Flakiness and exceptions

If a test fails nondeterministically:

1. Preserve logs, screenshots/traces, and environment details.
2. Investigate race conditions or unreliable dependencies.
3. Assign ownership.
4. Quarantine only with explicit justification and expiry.
5. Restore the meaningful gate after fixing the cause.

Repeatedly retrying until green can conceal real defects.

Exceptions for security/performance/accessibility gates should include impact,
mitigation, approver, and expiration. Do not hide them in a pipeline setting.

## 13. Interview scenario

**Release:** A new React transaction-history page.

PR checks:

- Unit and integration behavior tests.
- Accessible filtering and error states.
- Bundle-growth budget.
- Security tests for account ownership.

Staging:

- Keyboard and E2E flow verification.
- Realistic pagination/load conditions.
- Browser and API compatibility.

Canary:

- Compare errors and task latency.
- Monitor real-user responsiveness.
- Roll back or disable the feature if defined guards fail.

Document what can roll back safely and how saved data remains compatible.

## 14. Interview questions

**Can every NFR be checked on each PR?**

No. Put fast deterministic checks on PRs, expensive checks on suitable schedules,
manual checks at appropriate milestones, and field validation after deployment.

**Why build once?**

It reduces differences between tested and deployed artifacts and improves
traceability.

**What if a gate blocks an urgent fix?**

Use a documented exception with accountable approval and bounded risk, not an
untracked bypass. Keep the remediation deadline explicit.

**How do you know a deployment is healthy?**

Use business-relevant SLIs, errors, latency, synthetic journeys, and sufficient
field samples, not just a process health check.

## 15. Interview summary

> I turn NFRs into measurable acceptance criteria and layer fast PR checks,
> deeper staging tests, manual evaluation, and production monitoring. I protect
> pipeline credentials, promote traceable artifacts, and use canaries,
> compatibility planning, and defined rollback criteria for safe releases.

## References

- [GitHub Actions security](https://docs.github.com/en/actions/security-for-github-actions)
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [Google SRE: service-level objectives](https://sre.google/sre-book/service-level-objectives/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
