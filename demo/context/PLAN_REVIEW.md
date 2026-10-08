# Plan review

Reviewed `context/PLAN.md` against `TASK.md`.

## Verdict

READY WITH MINOR CHANGES

Every `BR-*`, `A-*` and `AC-*` id appears in the coverage map. The map agrees with the `Covers` lines, and the increments mostly test what they claim. The ordering is nearly sound, and the open questions are mostly real gaps in `TASK.md`. There are no blockers. There are five majors. Two increments (INC-9, INC-10) have criteria that cannot be asserted because the `AuditEntry` entity does not exist until INC-13. Two criteria (INC-10, INC-13) give no assertable status code. Several gating criteria quietly adopt answers to unresolved open questions. INC-14 has scope with no criteria. INC-15's matrix test has no defined expected result for part of its grid. These can be fixed by editing the plan, without re-slicing the increments.

## Findings

### F-1 — "Writes no AuditEntry" criteria precede the AuditEntry entity

- **Severity:** major
- **Category:** ordering
- **Where:** INC-9, INC-10
- **Problem:** `AuditEntry` and its repository are introduced in INC-13. INC-9 and INC-10 assert that certain transitions write no `AuditEntry`. Without the entity, nothing can be counted and the assertion cannot fail. INC-10 admits this ("nothing to count yet"). INC-9 does not. A criterion that cannot fail does not gate the increment.
- **Evidence:** INC-9: "`ON_HOLD` to `IN_PROGRESS` returns 200 ... and writes no AuditEntry (T-4, A-2)". INC-10: "Neither T-3 nor T-5 creates an AuditEntry (nothing to count yet; asserted again in INC-13)". INC-13 scope: "`AuditEntry` entity ... and repository."
- **Suggested fix:** Remove the no-audit assertions from INC-9 and INC-10. INC-13 already has "Transitions T-1 to T-7 create no AuditEntry". Extend it to name T-3, T-4 and T-5 explicitly. INC-14 already asserts T-4 is unaudited. Alternatively, move the `AuditEntry` entity earlier, but only if there is a reason to.

### F-2 — Criteria with no assertable status code

- **Severity:** major
- **Category:** testability
- **Where:** INC-10 (4th criterion), INC-13 (4th criterion)
- **Problem:** The criteria say the request is "rejected ... (status code per the open question)". A test cannot assert a status code that is not stated. Tests come first under the plan's own convention. These T-3 and T-8 missing-reason paths therefore have no gating test. The only status `TASK.md` gives for a missing reason is 400 for rollbacks (AC-14), so the question is genuine. The plan still needs to either resolve it or assert something checkable.
- **Evidence:** INC-10: "`IN_PROGRESS` to `ON_HOLD` without a reason is rejected and the status is unchanged (status code per the open question)." INC-13: "Cancellation without a reason is rejected, the status is unchanged and no AuditEntry is written (status code per the open question)."
- **Suggested fix:** Pick a provisional code (400, by analogy with AC-14) and mark it as an assumption in the criterion. Or weaken the criterion to "returns a 4xx and the status is unchanged". Resolve open question 4 before INC-10 starts.

### F-3 — Gating criteria adopt answers to open questions, and open questions are not tied to increments

- **Severity:** major
- **Category:** assumptions
- **Where:** INC-9, INC-10, INC-12, INC-13, INC-4, INC-6, open questions 1, 3, 15, 16, 19
- **Problem:** The open questions section says no rule has been given beyond "neutral defaults". Several completion criteria nonetheless assert a specific answer:
  - Wrong role gives 403 on non-rollback transitions (INC-9, INC-10, INC-12, INC-13). This is open question 3. `TASK.md` states 403 only for rollbacks.
  - The PATCH body is `{status, reason}` (open question 1).
  - A `GET` on a soft-deleted pet or owner returns 200 with `deletedAt` (INC-4 scope, INC-6 criterion). This is open question 15. BR-6 requires only that historical appointments stay readable, not the pet or owner themselves.
  - The INC-4 and INC-6 search criteria require a path and an exact-versus-partial match. Both are open question 16.
  - The clinical-record method and path (INC-10) are open question 19.
  - A-2 says the resume is by "the assigned doctor", yet T-4 is tested for any DOCTOR (open question 9).
  
  The questions are not mapped to the increment that needs them answered. An implementer writing tests first cannot proceed on these without making the decision themselves. That is the silent decision the plan says it avoids.
- **Evidence:** INC-6: "`GET /pets/{id}` of a soft-deleted pet still returns the pet with `deletedAt` populated." INC-9: "A registered pair called by a role not permitted for it returns 403." Open questions preamble: "None has been given a rule in this plan beyond the neutral defaults stated in the increments."
- **Suggested fix:** For each open question, name the first increment that is blocked by it ("must be answered before INC-N"). Mark criteria that rest on an assumed answer, for example "(assumes OQ-3: 403)". Or ask for the decisions up front.

### F-4 — INC-14 scope lists statuses and transitions that have no criteria

- **Severity:** major
- **Category:** sizing
- **Where:** INC-14
- **Problem:** The scope makes BOOKED, CHECKED_IN, IN_PROGRESS, ON_HOLD, READY and PAID rollback sources. The criteria cover only PAID to READY, READY to IN_PROGRESS and CHECKED_IN to BOOKED. BOOKED has no previous state. IN_PROGRESS has two candidates (CHECKED_IN or ON_HOLD). ON_HOLD to IN_PROGRESS is T-4 by A-2, so a separate rollback from ON_HOLD cannot exist. The increment therefore cannot be called finished and verified against its own scope.
- **Evidence:** INC-14 scope: "Non-terminal statuses for T-9: BOOKED, CHECKED_IN, IN_PROGRESS, ON_HOLD, READY, PAID." Also: "How 'previous state' is determined ... is an open question. The criteria below use only the cases that are unambiguous".
- **Suggested fix:** Reduce the INC-14 scope to the three unambiguous rollbacks, and state that rollback from BOOKED, IN_PROGRESS and ON_HOLD is deferred pending open question 2. Or require open question 2 to be answered before INC-14 and add criteria for the remaining sources.

### F-5 — INC-15 matrix test depends on unresolved questions

- **Severity:** major
- **Category:** testability
- **Where:** INC-15 (matrix criterion)
- **Problem:** The criterion requires all 8 x 8 pairs for both roles to match the `TASK.md` transition table. `TASK.md` does not define the expected result for several cells. Rollback pairs such as IN_PROGRESS to CHECKED_IN or ON_HOLD to anything beyond T-4 depend on open question 2. Unlisted pairs attempted by the wrong role depend on open question 5 (403 or 409). The expected-value table for the test cannot be written from `TASK.md` alone, so the test would encode the implementer's choice. The diagonal (self-transition) cells are also unspecified, though 409 follows from "any transition not listed".
- **Evidence:** "The matrix test covers all 8 x 8 status pairs for both roles and passes." Open question 5: "the precedence of 403, 400, 409 is not defined."
- **Suggested fix:** Limit the matrix to cells determined by the table: listed pairs for each allowed and disallowed role, and unlisted pairs. State the expected code for unlisted pairs per role. Exclude the T-9 cells until open question 2 is resolved.

### F-6 — The BR-9 structural test is vague and self-contradictory

- **Severity:** minor
- **Category:** testability
- **Where:** INC-9, INC-15
- **Problem:** "No controller or repository class references `AppointmentStatus` for decision-making" cannot be asserted by a source or reflection test. Such a test can detect only a reference, not "decision-making". The same increment requires the controller to turn an unknown status string into a 400, which needs the enum or a DTO field of that type. Response DTOs and repository derived queries (for example a status filter) also reference it. As written, the test will either fail on legitimate code or be too loose to mean anything.
- **Evidence:** INC-9: "(verified by a simple source or reflection test)". INC-15: "A structural test that no class outside the state-machine service decides on status transitions".
- **Suggested fix:** Define the check in terms a test can assert. For example, the status field is changed only through a single method, and the setter is not public or is called only from the state-machine package. Or use an ArchUnit-style rule on calls to that method.

### F-7 — Manual, vague or conditional completion criteria

- **Severity:** minor
- **Category:** testability
- **Where:** INC-1, INC-2, INC-8, INC-11, INC-15
- **Problem:** Several criteria cannot gate an increment:
  - INC-1: "`./mvnw spring-boot:run` starts against `compose.yaml` PostgreSQL (manual check, noted in the PR)" is explicitly manual.
  - INC-2: "No authentication filter, token handling or user store exists" is a negative claim with no stated check. One option is to assert that no Spring Security dependency is on the classpath.
  - INC-8: "An appointment's request payload has no field to attach more than one pet" is a design statement. It can only be tested through a reflection check or a rejected-payload test.
  - INC-11: "Any update or removal endpoint added for line items also returns 409 ... subject to the open question about whether such endpoints exist" is conditional and may apply to nothing.
  - INC-15: "A traceability check lists a test for each of AC-1 to AC-17" names no mechanism, such as a naming convention checked by a script or test.
- **Evidence:** The quoted lines above.
- **Suggested fix:** Make INC-1 a Testcontainers or compose-backed integration test, or move the PostgreSQL check out of the gating criteria. State a concrete mechanism for the INC-2 and INC-15 checks. Drop or concretise the INC-8 and INC-11 criteria.

### F-8 — Criteria for AC-3 and AC-5 omit the caller role

- **Severity:** minor
- **Category:** testability
- **Where:** INC-10, INC-12
- **Problem:** AC-5 (READY with empty diagnosis) is a DOCTOR-only transition. AC-3 (PAID with balance > 0) is ADMIN-only. The criteria do not say which role the test uses. If a test uses the wrong role, the result depends on the unresolved 403-versus-409 precedence (open question 5), and the AC could pass or fail for the wrong reason.
- **Evidence:** INC-10: "`IN_PROGRESS` to `READY` with an empty diagnosis returns 409 and the status stays `IN_PROGRESS` (AC-5)." INC-12: "`READY` to `PAID` while `balance > 0` returns 409 ... (AC-3, BR-4)."
- **Suggested fix:** State the permitted role in both criteria (`X-Role: DOCTOR` and `X-Role: ADMIN` respectively).

### F-9 — INC-6 declares an unneeded dependency on INC-4

- **Severity:** minor
- **Category:** ordering
- **Where:** INC-6, INC-8
- **Problem:** No INC-6 criterion uses owner search or owner deletion. The scope explicitly forbids cascading. The real dependency is INC-5. The INC-4 link hides that INC-8's AC-16 criterion needs `DELETE /owners/{id}`, which comes from INC-4.
- **Evidence:** INC-6: "Depends on: INC-5, INC-4". INC-8: "After `DELETE /owners/{id}` and `DELETE /pets/{id}`, `GET /appointments/{id}` ... still returns 200".
- **Suggested fix:** Change INC-6 to depend on INC-5 only. Change INC-8 to depend on INC-4 and INC-6.

### F-10 — Rules added or inferred that are not in TASK.md and not in the open questions

- **Severity:** minor
- **Category:** assumptions
- **Where:** INC-1, INC-2, INC-8, INC-11, INC-12, open questions 21, 22
- **Problem:**
  - Several validation rules appear only in criteria: line-item `quantity` must be positive (INC-11); payment `amount` must be positive, with 400 (INC-12); `scheduledAt` is required, with 400 (INC-8).
  - The error body must "contain a status and a message" (INC-2), though `TASK.md` says errors are HTTP status codes.
  - INC-1 asks for "a schema strategy for PostgreSQL" without naming one, while open question 22 says Hibernate generation is assumed. BR-10 index creation in dev depends on this.
  - Open question 21 (package naming) is a repository matter, not a gap in `TASK.md`.
  
  The positive-amount and positive-quantity rules are reasonable, but they are decisions where `TASK.md` is silent.
- **Evidence:** INC-12: "Payment amount must be positive (400 otherwise)." INC-1: "a schema strategy for PostgreSQL".
- **Suggested fix:** List the invented validations as open questions or mark them as assumptions. Name the INC-1 schema strategy (`ddl-auto` value). Move open question 21 out of the requirements gaps.

### F-11 — Business rules are restated in prose

- **Severity:** minor
- **Category:** scope
- **Where:** INC-11, INC-12, INC-13
- **Problem:** The project rule is "Business rules are referenced by their ids ... never restated in prose". The plan restates the rule bodies as well as citing the ids: the `lineTotal` and `total` formulas, the `balance` formula, and the overpayment condition. This risks drift from `TASK.md`, the single source of truth.
- **Evidence:** INC-11: "`lineTotal = unitPrice x quantity`; `Appointment.total = sum of lineTotal`". INC-12: "`balance = total - sum of payments` (BR-2)".
- **Suggested fix:** Replace the restatements with the id references (BR-1, BR-2, A-3). Keep only the concrete numeric examples used in tests.

## Checked and sound

- **Coverage:** All 10 BR, 7 A and 17 AC ids appear in the coverage map. Every map entry matches the `Covers` line of the increment it points to, and that increment has a criterion that exercises the requirement. The weakest link is AC-16, split between INC-4 (deletion) and INC-8 (history readable). The split is stated honestly in INC-4.
- **Scope:** INC-7 (Service endpoints) is the only addition that `TASK.md` does not define. It is needed for AC-10 and for the "adds services from the price list" flow. It is flagged as open question 10. Nothing from the Out of scope list is planned (no auth, employees, attachments, reports or multi-tenancy). No requirement was silently dropped. The search, doctor listing, clinical record and payment flows are all present.
- **Ordering:** The rest of the dependency chain is real: the state machine (INC-9) precedes T-3 and T-5, line items need a READY state, payments need totals, and rollback needs the audit entity. INC-9's T-4 test sets up its ON_HOLD state by direct persistence. This is acknowledged and acceptable.
- **Open questions:** Questions 2, 4, 5, 6, 7, 8, 11, 13, 14, 18 and 19 are genuine gaps. None has its answer written in `TASK.md`. The ambiguity around T-9 and "previous state" is accurately described, including the T-4 and A-2 overlap and A-5 on CLOSED.
- **Increment size:** INC-3 to INC-8, INC-11, INC-12 and INC-13 are each standalone and verifiable. INC-9 is the largest but is coherent. INC-1 is small, but it is the baseline that later increments need.
- **Criteria:** Most criteria are executable assertions with concrete values (AC-9's 200.00, AC-11's 100.01 versus 100.00, AC-12's two payments of 50.00, the snapshotting checks for price and name). The overpayment case also covers cumulative payments.

## Human review

Reviewed by Aleksander Siudek, 2026-10-08.

- **F-1 — agree.** INC-9 has a criterion saying no AuditEntry is created, but the
  entity does not exist yet, so the test has nothing to check.
- **F-8 — agree.** The criteria for AC-3 and AC-5 do not say which role to send in
  `X-Role`. Sitting down to write the test, I would not know what to put in the
  request.
- **F-9 — agree.** INC-6 declares a dependency on INC-4, but none of its criteria
  use anything from INC-4.

**Decision:** run the planner in revision mode on F-1, F-8 and F-9. The remaining
findings are noted but not acted on in this phase.