# Implementation plan

Generated from TASK.md. Each increment is independently implementable.

Transition ids `T-1` to `T-9` come from the transition table in `TASK.md`. They are
referenced in scope and criteria alongside the `BR-*`, `A-*` and `AC-*` ids.

## Repository starting point

- `pom.xml`: Spring Boot 4.1.1, Java 21, starters for data-jpa, webmvc and validation,
  the PostgreSQL driver, `spring-boot-docker-compose` (runtime, optional), Lombok, H2
  (test), `spring-boot-starter-data-jpa-test` and `spring-boot-starter-webmvc-test`.
- `compose.yaml`: PostgreSQL 17, db/user/password `vetclinic`, port 5432.
- `src/main/java/com/example/demo/DemoApplication.java` and a `contextLoads` test only.
  `application.properties` contains only `spring.application.name=demo`.
- No entities, controllers, services, repositories, or test configuration exist yet.
- The Maven groupId is `com.siudek.vetclinic` but the Java package is
  `com.example.demo`. This plan keeps the existing package. Renaming it is not part of
  any increment.

## Coverage map

| Requirement | Increment |
|---|---|
| BR-1 | INC-11 |
| BR-2 | INC-12 |
| BR-3 | INC-11 |
| BR-4 | INC-12 |
| BR-5 | INC-12 |
| BR-6 | INC-4, INC-6, INC-8 |
| BR-7 | INC-8 |
| BR-8 | INC-13, INC-14 |
| BR-9 | INC-9, INC-15 |
| BR-10 | INC-3, INC-5, INC-15 |
| A-1 | INC-13, INC-14 |
| A-2 | INC-9, INC-13 |
| A-3 | INC-12 |
| A-4 | INC-11 |
| A-5 | INC-14 |
| A-6 | INC-8 |
| A-7 | INC-2 |
| AC-1 | INC-3 |
| AC-2 | INC-5 |
| AC-3 | INC-12 |
| AC-4 | INC-9 |
| AC-5 | INC-10 |
| AC-6 | INC-13, INC-14 |
| AC-7 | INC-14 |
| AC-8 | INC-11 |
| AC-9 | INC-11 |
| AC-10 | INC-11 |
| AC-11 | INC-12 |
| AC-12 | INC-12 |
| AC-13 | INC-14 |
| AC-14 | INC-14 |
| AC-15 | INC-14 |
| AC-16 | INC-4, INC-8 |
| AC-17 | INC-8 |

## Conventions for every increment

- Tests first: write the failing test for each criterion, then the implementation.
- Tests run with H2 (`./mvnw test`). Dev runs against PostgreSQL 17 from `compose.yaml`.
- Every status change goes through the single state-machine service introduced in
  INC-9. Controllers and repositories never inspect `status` (BR-9).
- Money is `BigDecimal` with scale 2.
- Do not add endpoints, fields or rules beyond what the increment lists. Anything
  undefined goes to `## Open questions`.
- "Writes no AuditEntry" can only be asserted once the `AuditEntry` entity exists
  (INC-13). Earlier increments therefore do not carry such criteria; INC-13 asserts it
  for every unaudited transition.

## Increments

### INC-1 — Test and dev configuration baseline

- **Status:** pending
- **Goal:** The application context starts under H2 in tests and is configured for PostgreSQL in development, so every later increment can add MockMvc and JPA tests.
- **Depends on:** none
- **Covers:** — (enabling work; no requirement id)
- **Scope:**
  - `src/test/resources/application.properties` with H2 in-memory datasource and `create-drop` schema generation.
  - `src/main/resources/application.properties` with the dev datasource and a schema strategy for PostgreSQL (`vetclinic` / `vetclinic`, port 5432, matching `compose.yaml`). Docker Compose support supplies the connection when running via `./mvnw spring-boot:run`.
  - Confirm the existing `DemoApplicationTests.contextLoads` passes without Docker running.
  - Confirm that a MockMvc-based test can be written with `spring-boot-starter-webmvc-test`.
- **Completion criteria:**
  - [ ] `./mvnw test` passes with Docker stopped and `contextLoads` green.
  - [ ] A trivial MockMvc test against a non-existent path returns 404 (proves the web test slice is wired).
  - [ ] `./mvnw spring-boot:run` starts against `compose.yaml` PostgreSQL (manual check, noted in the PR).

### INC-2 — Error model and role resolution

- **Status:** pending
- **Goal:** A uniform mapping from domain failures to HTTP status codes, plus a resolved caller role available to every request.
- **Depends on:** INC-1
- **Covers:** A-7
- **Scope:**
  - A `Role` enum (`ADMIN`, `DOCTOR`) and a resolver that reads `X-Role` and trusts it, with no authentication (A-7).
  - Exceptions and a global `@RestControllerAdvice` mapping: bean-validation and malformed-body failures to 400, not-found to 404, state/business conflict to 409, role not permitted to 403.
  - A missing or unparseable `X-Role` on endpoints that need a role is handled as per the open question below. Until it is answered, only the endpoints that need a role consult it.
  - Test-only controller or `@WebMvcTest` fixtures are acceptable to prove the mapping. They must not ship in `src/main`.
- **Completion criteria:**
  - [ ] A request with `X-Role: ADMIN` and one with `X-Role: DOCTOR` each resolve to the matching `Role`.
  - [ ] Each of the four exception types maps to 400, 403, 404 and 409 respectively, asserted in tests.
  - [ ] Error bodies are JSON and contain a status and a message.
  - [ ] No authentication filter, token handling or user store exists (A-7).

### INC-3 — Owner registration

- **Status:** pending
- **Goal:** Reception can register and fetch an owner, with validation and an index on phone.
- **Depends on:** INC-2
- **Covers:** AC-1, BR-10
- **Scope:**
  - `Owner` entity (fullName, phone, email, deletedAt) and repository.
  - `POST /owners` and `GET /owners/{id}`.
  - Bean Validation on request DTOs: `fullName` not blank. Format constraints for phone and email are an open question.
  - Index on `Owner.phone` declared on the entity.
- **Completion criteria:**
  - [ ] `POST /owners` with a blank `fullName` returns 400 (AC-1).
  - [ ] `POST /owners` with a missing `fullName` returns 400.
  - [ ] `POST /owners` with a valid body returns 201 and the body contains an id; `GET /owners/{id}` returns the same data.
  - [ ] `GET /owners/{unknownId}` returns 404.
  - [ ] A schema-metadata test confirms an index exists on the `phone` column of the owner table (BR-10).

### INC-4 — Owner search and soft delete

- **Status:** pending
- **Goal:** Owners can be found by phone or name and soft-deleted without removing the row.
- **Depends on:** INC-3
- **Covers:** BR-6, AC-16
- **Scope:**
  - Search endpoint for owners by phone or by name (path and match semantics under open questions).
  - `DELETE /owners/{id}` sets `deletedAt` and does not remove the row.
  - Behaviour of deleted owners in search and `GET` follows the open-questions answer. Until then, `GET /owners/{id}` of a deleted owner still returns the owner with `deletedAt` populated, so that history stays readable (BR-6).
  - The "past appointments remain retrievable" half of AC-16 is verified in INC-8.
- **Completion criteria:**
  - [ ] `DELETE /owners/{id}` returns 204 and the persisted row has a non-null `deletedAt` (AC-16).
  - [ ] After deletion the row still exists in the database (count unchanged).
  - [ ] `DELETE /owners/{unknownId}` returns 404.
  - [ ] Searching by an existing phone returns that owner.
  - [ ] Searching by a name returns the matching owner(s) and no non-matching ones.
  - [ ] Searching with no match returns an empty list with status 200.

### INC-5 — Pet registration

- **Status:** pending
- **Goal:** Reception can register a pet for an existing owner and fetch it, with indexes on chip number and name.
- **Depends on:** INC-3
- **Covers:** AC-2, BR-10
- **Scope:**
  - `Pet` entity (name, species, breed, sex, birthDate, weight, chipNumber, deletedAt), N to 1 Owner.
  - `POST /pets` and `GET /pets/{id}`.
  - Validation: `name` not blank; `owner` reference required. Other field constraints are open questions.
  - Indexes on `Pet.chipNumber` and `Pet.name`.
- **Completion criteria:**
  - [ ] `POST /pets` referencing an unknown owner id returns 404 (AC-2).
  - [ ] `POST /pets` with a blank `name` returns 400.
  - [ ] `POST /pets` for an existing owner returns 201; `GET /pets/{id}` returns the pet with its owner id.
  - [ ] A schema-metadata test confirms indexes on the `chip_number` and `name` columns of the pet table (BR-10).

### INC-6 — Pet search and soft delete

- **Status:** pending
- **Goal:** Pets can be found by name or chip number and soft-deleted without removing the row.
- **Depends on:** INC-5
- **Covers:** BR-6
- **Scope:**
  - Search endpoint for pets by name or by chip number.
  - `DELETE /pets/{id}` sets `deletedAt`.
  - Whether deleting an owner also soft-deletes their pets is an open question. Do not cascade until it is answered.
- **Completion criteria:**
  - [ ] `DELETE /pets/{id}` returns 204 and the persisted row has a non-null `deletedAt`; the row still exists.
  - [ ] `DELETE /pets/{unknownId}` returns 404.
  - [ ] Searching by an existing chip number returns that pet.
  - [ ] Searching by name returns matching pets and no non-matching ones.
  - [ ] `GET /pets/{id}` of a soft-deleted pet still returns the pet with `deletedAt` populated.

### INC-7 — Service price list

- **Status:** pending
- **Goal:** Services (code, name, duration, price) can be created, read and have their price changed, so line items have something to copy from.
- **Depends on:** INC-2
- **Covers:** — (prerequisite for BR-3 and AC-10)
- **Scope:**
  - `Service` entity (code, name, durationMinutes, price) and repository.
  - Minimal endpoints: create, get by id or code, update. `TASK.md` defines no Service endpoints; these are the smallest set needed for AC-10 (see open questions).
  - Validation: code and name not blank; price and duration not negative (the exact bounds are an open question).
- **Completion criteria:**
  - [ ] Creating a Service with valid data returns 201 and it can be fetched.
  - [ ] Creating a Service with a blank `code` returns 400.
  - [ ] Updating the price of a Service persists the new price on the next fetch.
  - [ ] Fetching an unknown Service returns 404.

### INC-8 — Appointment booking and retrieval

- **Status:** pending
- **Goal:** Reception can book an appointment for a live pet, and appointments (including those of deleted owners and pets) can be read and listed.
- **Depends on:** INC-4, INC-6
- **Covers:** BR-6, BR-7, A-6, AC-16, AC-17
- **Scope:**
  - `Appointment` entity with the fields in the domain model: scheduledAt, status, complaint, anamnesis, diagnosis, notes, assignedDoctorId, total. Exactly one Pet per appointment (A-6). `AppointmentStatus` enum with all lifecycle states.
  - `POST /appointments` creates an appointment in `BOOKED` with `total` 0.00. Creation is the only place status is set outside the state-machine service, and it only sets the initial value.
  - `GET /appointments/{id}`.
  - List appointments assigned to a doctor (`assignedDoctorId` filter; path under open questions).
  - BR-7 guard: a soft-deleted pet, or a pet whose owner is soft-deleted, cannot be booked.
  - Validation: pet reference, `scheduledAt` and `assignedDoctorId` required.
- **Completion criteria:**
  - [ ] `POST /appointments` for a live pet returns 201 with status `BOOKED` and `total` 0.00.
  - [ ] `POST /appointments` referencing a soft-deleted pet returns 409 (AC-17).
  - [ ] `POST /appointments` referencing a pet whose owner is soft-deleted returns 409 (BR-7).
  - [ ] `POST /appointments` referencing an unknown pet returns 404.
  - [ ] `POST /appointments` without `scheduledAt` returns 400.
  - [ ] After `DELETE /owners/{id}` and `DELETE /pets/{id}`, `GET /appointments/{id}` of a previously booked appointment still returns 200 with its data (AC-16, BR-6).
  - [ ] The doctor listing returns only appointments with the requested `assignedDoctorId`.
  - [ ] An appointment's request payload has no field to attach more than one pet (A-6).

### INC-9 — State machine core and the first transitions

- **Status:** pending
- **Goal:** One state-machine service owns all status changes; `PATCH /appointments/{id}/status` supports T-1, T-2 and T-4 and rejects everything not listed.
- **Depends on:** INC-8, INC-2
- **Covers:** BR-9, A-2, AC-4
- **Scope:**
  - A single `AppointmentStateMachine` service (name is free) holding a declarative transition table: from, to, allowed roles, guard hook. Only rows implemented so far are registered.
  - `PATCH /appointments/{id}/status` with a JSON body containing the target status and an optional reason. The controller only parses the request, resolves the role and delegates.
  - T-1 BOOKED to CHECKED_IN (ADMIN). T-2 CHECKED_IN to IN_PROGRESS (ADMIN, DOCTOR). T-4 ON_HOLD to IN_PROGRESS (ADMIN, DOCTOR; resume, not audited, A-2).
  - Any `(from, to)` pair that is not registered returns 409. A registered pair called by a role not permitted for it returns 403.
  - Unknown appointment returns 404. Unknown status value returns 400.
  - Failed transitions leave the status unchanged.
- **Completion criteria:**
  - [ ] `BOOKED` to `CHECKED_IN` with `X-Role: ADMIN` returns 200 and the status is `CHECKED_IN` (T-1).
  - [ ] `BOOKED` to `CHECKED_IN` with `X-Role: DOCTOR` returns 403 and the status is unchanged.
  - [ ] `CHECKED_IN` to `IN_PROGRESS` returns 200 for ADMIN and for DOCTOR (T-2).
  - [ ] `BOOKED` to `IN_PROGRESS` returns 409 and the status stays `BOOKED` (AC-4).
  - [ ] `ON_HOLD` to `IN_PROGRESS` returns 200 for ADMIN and for DOCTOR (T-4, A-2). Setting up an `ON_HOLD` appointment may use direct persistence in the test until INC-10. That T-4 writes no AuditEntry is asserted in INC-13, once the entity exists.
  - [ ] A parameterised unit test of the state machine iterates every `(from, to)` pair of the statuses and asserts that only the registered pairs succeed.
  - [ ] No controller or repository class references `AppointmentStatus` for decision-making (verified by a simple source or reflection test).
  - [ ] `PATCH` on an unknown id returns 404; an unknown status string returns 400.

### INC-10 — Clinical record and the doctor-side transitions T-3, T-5

- **Status:** pending
- **Goal:** Doctors can record anamnesis, diagnosis and notes, put an appointment on hold with a reason, and mark it READY only with a diagnosis.
- **Depends on:** INC-9
- **Covers:** AC-5
- **Scope:**
  - An endpoint to update `anamnesis`, `diagnosis` and `notes` on an appointment (method and path under open questions). It does not touch status.
  - T-3 IN_PROGRESS to ON_HOLD: DOCTOR only; `reason` required. Not audited.
  - T-5 IN_PROGRESS to READY: DOCTOR only; guard `diagnosis` not empty. Not audited.
  - Both rows are registered in the state machine from INC-9. No status logic goes into the clinical-record endpoint.
- **Completion criteria:**
  - [ ] Updating anamnesis, diagnosis and notes persists them and `GET /appointments/{id}` returns them.
  - [ ] `IN_PROGRESS` to `ON_HOLD` with `X-Role: DOCTOR` and a reason returns 200 (T-3).
  - [ ] `IN_PROGRESS` to `ON_HOLD` with `X-Role: ADMIN` returns 403.
  - [ ] `IN_PROGRESS` to `ON_HOLD` without a reason is rejected and the status is unchanged (status code per the open question).
  - [ ] `IN_PROGRESS` to `READY` with `X-Role: DOCTOR` and an empty diagnosis returns 409 and the status stays `IN_PROGRESS` (AC-5).
  - [ ] `IN_PROGRESS` to `READY` with a non-empty diagnosis and `X-Role: DOCTOR` returns 200 (T-5).
  - [ ] `IN_PROGRESS` to `READY` with `X-Role: ADMIN` returns 403.
  - [ ] That T-3 and T-5 write no AuditEntry is asserted in INC-13, once the entity exists.

### INC-11 — Line items and totals

- **Status:** pending
- **Goal:** Services from the price list can be added to an appointment with price snapshotting and a correct total, and the bill is frozen from READY onwards.
- **Depends on:** INC-10, INC-7
- **Covers:** BR-1, BR-3, A-4, AC-8, AC-9, AC-10
- **Scope:**
  - `LineItem` entity (serviceCode, serviceName, unitPrice, quantity, lineTotal), N to 1 Appointment.
  - `POST /appointments/{id}/line-items` with a service code (or id) and quantity. `unitPrice` and `serviceName` are copied from the Service at that moment and never re-read (BR-3).
  - `lineTotal = unitPrice x quantity`; `Appointment.total = sum of lineTotal`, recalculated on every change (BR-1).
  - Freeze guard (A-4): any add, change or remove is refused with 409 when status is READY or later. The guard lives in a collaborator of the state machine, not in the controller. The status-to-editable decision for the remaining statuses is an open question.
  - The appointment response includes its line items and `total`.
- **Completion criteria:**
  - [ ] Adding line items of 100.00 x 1 and 50.00 x 2 results in `total` 200.00 (AC-9).
  - [ ] Each line item's `lineTotal` equals `unitPrice x quantity` (BR-1).
  - [ ] After a Service price is updated, the existing line item's `unitPrice` and the appointment `total` are unchanged (AC-10, BR-3).
  - [ ] After a Service name is updated, the existing line item's `serviceName` is unchanged (BR-3).
  - [ ] A line item added after a price change uses the new price.
  - [ ] `POST /appointments/{id}/line-items` on an appointment in `READY` returns 409 and `total` is unchanged (AC-8).
  - [ ] The same call returns 409 for `PAID` and `CLOSED` (A-4).
  - [ ] An unknown service code returns 404; `quantity` of 0 or negative returns 400.
  - [ ] Any update or removal endpoint added for line items also returns 409 in READY or later (A-4), subject to the open question about whether such endpoints exist.

### INC-12 — Payments, balance, PAID and CLOSED

- **Status:** pending
- **Goal:** Payments can be registered against an appointment, the balance is computed, and the appointment moves through PAID to CLOSED under the guard.
- **Depends on:** INC-11
- **Covers:** BR-2, BR-4, BR-5, A-3, AC-3, AC-11, AC-12
- **Scope:**
  - `Payment` entity (amount, paidAt), N to 1 Appointment.
  - `POST /appointments/{id}/payments`. Several payments per appointment are allowed (BR-5).
  - `balance = total - sum of payments` (BR-2), exposed on the appointment response.
  - Overpayment guard (A-3): a payment that would make the sum of payments exceed `total` returns 409.
  - T-6 READY to PAID (ADMIN), guard `balance == 0` (BR-4). T-7 PAID to CLOSED (ADMIN). Both registered in the state machine.
  - Payment amount must be positive (400 otherwise). Whether `paidAt` is client-supplied or server-set, and which statuses accept payments, are open questions.
- **Completion criteria:**
  - [ ] An appointment with total 100.00 and one payment of 40.00 shows `balance` 60.00 (BR-2).
  - [ ] `READY` to `PAID` with `X-Role: ADMIN` while `balance > 0` returns 409 and the status stays `READY` (AC-3, BR-4).
  - [ ] A payment of 100.01 against a total of 100.00 returns 409 and no Payment row is created (AC-11, A-3).
  - [ ] A second payment that would take the sum above the total returns 409 even though each payment alone is below the total.
  - [ ] Two payments of 50.00 against a total of 100.00 bring `balance` to 0.00, and `READY` to `PAID` with `X-Role: ADMIN` then returns 200 (AC-12, BR-5).
  - [ ] `READY` to `PAID` with `X-Role: DOCTOR` returns 403.
  - [ ] `PAID` to `CLOSED` with `X-Role: ADMIN` returns 200 (T-7); with `X-Role: DOCTOR` returns 403.
  - [ ] A payment of 0 or a negative amount returns 400.
  - [ ] `balance` is computed from persisted payments, not stored independently of them.

### INC-13 — Audit trail and cancellation

- **Status:** pending
- **Goal:** BOOKED or CHECKED_IN appointments can be cancelled with a reason, the cancellation is audited, CANCELLED is terminal, and the unaudited transitions are proved to write no AuditEntry.
- **Depends on:** INC-12
- **Covers:** BR-8, A-1, A-2, AC-6
- **Scope:**
  - `AuditEntry` entity (appointmentId, fromStatus, toStatus, actorRole, reason, occurredAt) and repository.
  - T-8 BOOKED or CHECKED_IN to CANCELLED: ADMIN only, `reason` required, writes exactly one AuditEntry with the actor role, reason and timestamp (BR-8).
  - The AuditEntry is written in the same transaction as the status change. A failed transition writes nothing.
  - CANCELLED has no outgoing transition (A-1); the default 409 from INC-9 covers it.
  - No audit-read endpoint is added, because `TASK.md` does not define one. Tests read the repository.
  - This is the first increment where the AuditEntry count can be asserted, so it owns the "unaudited" assertions for T-1 to T-7, including T-3, T-4 and T-5 whose own increments (INC-9, INC-10) cannot check it.
- **Completion criteria:**
  - [ ] `BOOKED` to `CANCELLED` with `X-Role: ADMIN` and a reason returns 200 and exactly one AuditEntry exists with from `BOOKED`, to `CANCELLED`, role `ADMIN`, the reason and a non-null `occurredAt` (T-8, BR-8).
  - [ ] The same from `CHECKED_IN` also works.
  - [ ] Cancellation with `X-Role: DOCTOR` returns 403 and no AuditEntry is written.
  - [ ] Cancellation without a reason is rejected, the status is unchanged and no AuditEntry is written (status code per the open question).
  - [ ] Cancellation from `IN_PROGRESS`, `READY`, `PAID` or `CLOSED` returns 409.
  - [ ] Every transition from `CANCELLED` to each of the other statuses returns 409 (AC-6, A-1).
  - [ ] Each of T-1 to T-7 (the "Audited: no" column), including T-3, T-4 and T-5 introduced in INC-9 and INC-10, is performed once successfully and afterwards the AuditEntry count is 0. T-4 is the resume of A-2.

### INC-14 — Rollback (T-9)

- **Status:** pending
- **Goal:** An administrator can roll a non-terminal appointment back to its previous state with a reason, audited, while CLOSED and CANCELLED stay terminal.
- **Depends on:** INC-13
- **Covers:** BR-8, A-1, A-5, AC-6, AC-7, AC-13, AC-14, AC-15
- **Scope:**
  - T-9 any non-terminal status to its previous state: ADMIN only, `reason` required, writes one AuditEntry.
  - Non-terminal statuses for T-9: BOOKED, CHECKED_IN, IN_PROGRESS, ON_HOLD, READY, PAID. CANCELLED (A-1) and CLOSED (A-5) are excluded.
  - How "previous state" is determined, and how a rollback request is told apart from a forward request, is an open question. The criteria below use only the cases that are unambiguous in `TASK.md`: READY to IN_PROGRESS, PAID to READY, CHECKED_IN to BOOKED.
  - Evaluation order for 403, 400 and 409 on a rollback is an open question. The criteria below avoid cases that depend on it.
- **Completion criteria:**
  - [ ] `PAID` to `READY` with `X-Role: DOCTOR` returns 403 and the status is unchanged (AC-13).
  - [ ] `PAID` to `READY` with `X-Role: ADMIN` and no reason returns 400 and the status is unchanged (AC-14).
  - [ ] `PAID` to `READY` with `X-Role: ADMIN` and a reason returns 200 and exactly one AuditEntry exists with from `PAID`, to `READY`, role `ADMIN` and that reason (AC-15).
  - [ ] The same holds for `READY` to `IN_PROGRESS` and `CHECKED_IN` to `BOOKED`.
  - [ ] A failed rollback (403, 400 or 409) creates no AuditEntry.
  - [ ] Any transition out of `CLOSED`, including a rollback to `PAID` by an administrator with a reason, returns 409 and creates no AuditEntry (AC-7, A-5).
  - [ ] Any transition out of `CANCELLED`, including a rollback with a reason, returns 409 (AC-6, A-1).
  - [ ] After a rollback from `PAID` to `READY`, line items remain frozen (A-4), and after `READY` to `IN_PROGRESS` they can be added again.
  - [ ] `ON_HOLD` to `IN_PROGRESS` remains the unaudited T-4 and writes no AuditEntry (A-2).

### INC-15 — Hardening and end-to-end verification

- **Status:** pending
- **Goal:** The full lifecycle is proved end to end, and the structural requirements (single state machine, indexes) are guarded by tests.
- **Depends on:** INC-14
- **Covers:** BR-9, BR-10
- **Scope:**
  - One end-to-end MockMvc scenario for the reception and veterinarian flows: register owner and pet, book, check in, start, add services, record diagnosis, mark READY, pay, PAID, CLOSED.
  - An end-to-end scenario with ON_HOLD and resume, and one with cancellation.
  - A structural test that no class outside the state-machine service decides on status transitions (BR-9). This extends the INC-9 check to the final code.
  - A schema test asserting all three BR-10 indexes together.
  - A full unit-level transition matrix for every `(from, to)` pair and each role, compared against the transition table of `TASK.md`.
  - Run `./mvnw verify` and confirm every AC-1 to AC-17 has at least one named test.
- **Completion criteria:**
  - [ ] The end-to-end happy-path scenario finishes with status `CLOSED`, `balance` 0.00 and no AuditEntry.
  - [ ] The hold and resume scenario ends in `IN_PROGRESS` with no AuditEntry.
  - [ ] The cancellation scenario ends in `CANCELLED` with exactly one AuditEntry.
  - [ ] The matrix test covers all 8 x 8 status pairs for both roles and passes.
  - [ ] The index test finds indexes on `Owner.phone`, `Pet.chipNumber` and `Pet.name` (BR-10).
  - [ ] `./mvnw verify` is green.
  - [ ] A traceability check lists a test for each of AC-1 to AC-17.

## Open questions

Each item is undefined in `TASK.md`. None has been given a rule in this plan beyond the
neutral defaults stated in the increments.

1. **Status request shape.** The body of `PATCH /appointments/{id}/status` is not
   defined (field names for target status and reason). Increment INC-9 assumes
   `{ "status": "...", "reason": "..." }`.
2. **Rollback identification and "previous state".** T-9 says "previous state". It is
   not defined whether that is the preceding step in the lifecycle chain, or the
   actual last status from history. The domain model has no status-history entity, and
   only T-8 and T-9 are audited. IN_PROGRESS is ambiguous (CHECKED_IN or ON_HOLD).
   ON_HOLD to IN_PROGRESS is both T-4 and a rollback. A-2 says it is a resume, not a
   rollback, which this plan follows. It is also not stated whether a request to a
   previous state is always a rollback, or whether an explicit flag is needed.
3. **Role violations on non-rollback transitions.** `TASK.md` states 403 only for
   rollbacks (AC-13). Using 403 for any role not in the "Allowed role" column is
   assumed in INC-9, but "Any transition not listed above is rejected with HTTP 409"
   could be read to include a listed transition with a wrong role.
4. **Missing reason on T-3 and T-8.** AC-14 gives 400 for a rollback without a reason.
   The status code for a missing reason on T-3 (hold) and T-8 (cancel) is not stated.
   The plan leaves it open (400 by analogy or 409 as a guard failure).
5. **Check ordering.** When a request has several faults (wrong role, missing reason,
   unlisted transition, terminal state), the precedence of 403, 400, 409 is not
   defined.
6. **Missing or invalid `X-Role`.** Behaviour (400, 403, or default) is not defined.
7. **Reason storage for non-audited transitions.** T-3 requires a reason but is not
   audited, and no reason field exists on Appointment. It is unclear whether the hold
   reason is stored anywhere.
8. **"Diagnosis not empty" (T-5).** Whether whitespace-only counts as empty.
9. **Assigned doctor identity.** A-2 and the veterinarian flow refer to the "assigned
   doctor", but the only caller identity is the role in `X-Role`; there is no doctor id
   header. It is unclear whether DOCTOR transitions must be restricted to the assigned
   doctor, and how "lists appointments assigned to them" identifies the caller
   (assumed a query parameter `assignedDoctorId`).
10. **Service endpoints.** No endpoints for the price list are defined, yet AC-10
    needs a price change. INC-7 adds the smallest set. Whether a Service can be
    deleted, whether `code` is unique, and whether Service is soft-deleted are not
    specified.
11. **Payment details.** Endpoint path, whether `paidAt` is supplied by the client or
    set by the server, and which statuses accept payments (for example BOOKED,
    CANCELLED, CLOSED) are not defined.
12. **Line-item editing.** A-4 mentions "changed or removed", but no update or delete
    endpoints are specified. Which statuses before READY accept line items (for
    example BOOKED, ON_HOLD) is also not stated; CANCELLED is "later" than READY only
    by a loose reading.
13. **Zero-total appointments.** An appointment with no line items has balance 0 and
    would satisfy BR-4/T-6. Whether that is intended is not stated.
14. **Cancelling after payments.** Behaviour of T-8 when payments already exist
    (refunds, balance) is not defined.
15. **Soft delete cascade and visibility.** Whether deleting an owner also soft-deletes
    their pets, whether deleted owners and pets appear in search results, whether
    `GET` on them returns 200 or 404, and whether a pet can be created for a
    soft-deleted owner (404 or 409) are not defined. INC-8 treats booking against a
    pet of a deleted owner as 409 by BR-7.
16. **Search semantics.** Endpoint paths, exact versus partial match, case sensitivity,
    and pagination for owner and pet search are not defined.
17. **Field constraints.** Format rules for phone, email, chip number, species, sex,
    weight and birthDate, and uniqueness of phone, email and chipNumber, are not
    defined. Only `fullName` not blank is stated (AC-1).
18. **Scheduling rules.** Whether `scheduledAt` may be in the past, and whether a
    doctor can be double-booked (using `Service.durationMinutes`), are not defined.
19. **Clinical record endpoint.** Method and path for recording anamnesis, diagnosis
    and notes are not defined, nor which statuses allow editing them.
20. **Audit read access.** No endpoint to read AuditEntry rows is defined. Tests read
    the repository directly.
21. **Package naming.** The Maven groupId is `com.siudek.vetclinic`, the Java package
    is `com.example.demo` and `spring.application.name=demo`. Whether to rename is not
    in `TASK.md`.
22. **Schema management.** No migration tool (Flyway or Liquibase) is in `pom.xml`.
    The plan assumes Hibernate schema generation. Confirm whether migrations are
    wanted.

## Revision log

Revision 1, driven by `context/PLAN_REVIEW.md` and the human decision recorded in its
"Human review" section: act on F-1, F-8 and F-9 only.

### Applied

- **F-1 — "Writes no AuditEntry" criteria precede the AuditEntry entity.**
  - INC-9: removed "and writes no AuditEntry" from the T-4 criterion and replaced it with
    a pointer to INC-13.
  - INC-10: replaced the "Neither T-3 nor T-5 creates an AuditEntry (nothing to count
    yet)" criterion with a pointer to INC-13, so no criterion remains that cannot fail.
  - INC-13: rewrote the "T-1 to T-7 create no AuditEntry" criterion so it explicitly names
    T-3, T-4 and T-5 and states a checkable procedure (perform each transition once,
    then assert the AuditEntry count is 0). Added a matching scope line and extended the
    goal. Added A-2 to INC-13's `Covers` line and to the coverage map, because the
    unaudited-resume half of A-2 is now asserted there.
  - Added a convention in "Conventions for every increment" so later edits do not
    reintroduce audit assertions before INC-13.
  - Why: the entity and repository are introduced in INC-13, so earlier tests had
    nothing to count. The reviewer's first suggested fix (move the assertion to INC-13)
    was chosen over moving the entity earlier, since there was no other reason to move
    it. INC-14's existing T-4 no-audit criterion is unchanged.
- **F-8 — Criteria for AC-3 and AC-5 omit the caller role.**
  - INC-10: the AC-5 criterion now specifies `X-Role: DOCTOR` with the empty diagnosis.
  - INC-12: the AC-3 criterion now specifies `X-Role: ADMIN`.
  - Why: each uses the one role permitted for the transition, so the 409 comes from the
    guard and not from an unresolved 403-versus-409 precedence (open question 5).
- **F-9 — INC-6 declares an unneeded dependency on INC-4.**
  - INC-6 now depends on INC-5 only.
  - INC-8 now depends on INC-4 and INC-6, since its AC-16 criterion uses
    `DELETE /owners/{id}` from INC-4.
  - Why: matches which increment actually supplies what each criterion uses. The order
    stays valid, INC-4 still precedes INC-8.

### Not acted on in this phase

Per the human decision, the following findings are noted and deliberately left
unchanged. They remain open for a later revision. None was rejected as wrong.

- **F-2** (criteria with no assertable status code, INC-10 and INC-13): not changed.
  The affected criteria still read "status code per the open question".
- **F-3** (gating criteria adopt answers to open questions; questions not tied to
  increments): not changed.
- **F-4** (INC-14 scope broader than its criteria): not changed.
- **F-5** (INC-15 matrix depends on unresolved questions): not changed.
- **F-6** (BR-9 structural test is vague): not changed.
- **F-7** (manual, vague or conditional criteria): not changed.
- **F-10** (rules inferred that are not in `TASK.md`): not changed.
- **F-11** (business rules restated in prose): not changed.
