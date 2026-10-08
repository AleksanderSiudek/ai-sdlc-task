# TASK.md — Veterinary Clinic Backend

## Overview

A REST backend for managing the day-to-day operations of a veterinary clinic:
pet owners, their pets, appointments and payments. It replaces paper cards and
spreadsheets with one system that keeps the full visit history of every pet and
enforces a controlled appointment lifecycle.

The core of the system is the appointment state machine: every status change is
guarded by explicit, documented rules collected in one place, never scattered
across the codebase as ad-hoc conditionals.

## User flows

**Reception**
1. Registers an owner (name, phone, email) and their pets.
2. Books an appointment for a pet: date, time, assigned doctor, owner complaint.
3. Checks the pet in on arrival.
4. Adds services from the price list to the appointment.
5. Registers payments and closes the appointment once it is fully paid.

**Veterinarian**
1. Lists the appointments assigned to them.
2. Starts the examination of a checked-in pet.
3. Records anamnesis, diagnosis and examination notes.
4. Puts an appointment on hold while waiting for test results, and resumes it.
5. Marks the appointment as ready for discharge.

**Search**
- Owners by phone or name; pets by name or chip number.

## Domain model

| Entity | Key fields | Relations |
|---|---|---|
| Owner | fullName, phone, email, deletedAt | 1 → N Pet |
| Pet | name, species, breed, sex, birthDate, weight, chipNumber, deletedAt | N → 1 Owner |
| Service | code, name, durationMinutes, price | referenced by LineItem |
| Appointment | scheduledAt, status, complaint, anamnesis, diagnosis, notes, assignedDoctorId, total | N → 1 Pet, 1 → N LineItem, 1 → N Payment |
| LineItem | serviceCode, serviceName, unitPrice, quantity, lineTotal | N → 1 Appointment |
| Payment | amount, paidAt | N → 1 Appointment |
| AuditEntry | appointmentId, fromStatus, toStatus, actorRole, reason, occurredAt | N → 1 Appointment |

`assignedDoctorId` is a plain identifier; there is no Employee entity (see Out of scope).

## Appointment lifecycle

```
BOOKED → CHECKED_IN → IN_PROGRESS → READY → PAID → CLOSED
                           ↕
                        ON_HOLD

BOOKED, CHECKED_IN → CANCELLED
```

### Transition table

| # | From | To | Allowed role | Guard | Audited |
|---|---|---|---|---|---|
| T-1 | BOOKED | CHECKED_IN | admin | — | no |
| T-2 | CHECKED_IN | IN_PROGRESS | admin, doctor | — | no |
| T-3 | IN_PROGRESS | ON_HOLD | doctor | reason required | no |
| T-4 | ON_HOLD | IN_PROGRESS | admin, doctor | — | no |
| T-5 | IN_PROGRESS | READY | doctor | diagnosis not empty | no |
| T-6 | READY | PAID | admin | balance == 0 | no |
| T-7 | PAID | CLOSED | admin | — | no |
| T-8 | BOOKED, CHECKED_IN | CANCELLED | admin | reason required | yes |
| T-9 | any non-terminal → previous state | admin | reason required | yes |

Any transition not listed above is rejected with HTTP 409.

## Business rules

- **BR-1** `lineTotal = unitPrice × quantity`; `Appointment.total = Σ lineTotal`.
- **BR-2** `balance = total − Σ Payment.amount`.
- **BR-3** When a line item is added, `unitPrice` and `serviceName` are copied
  from the Service at that moment and never re-read afterwards. Later price-list
  changes must not alter past appointments.
- **BR-4** An appointment cannot enter PAID while `balance > 0`.
- **BR-5** Partial payments are allowed; several Payments may belong to one
  appointment.
- **BR-6** Owners and pets are soft-deleted (`deletedAt`). Historical
  appointments must remain readable after deletion.
- **BR-7** A soft-deleted owner or pet cannot be used in a new appointment.
- **BR-8** Every rollback (T-9) and every cancellation (T-8) writes an
  AuditEntry with actor role, reason and timestamp.
- **BR-9** All transition logic lives in a single state-machine service; no
  status checks in controllers or repositories.
- **BR-10** Indexes on `Owner.phone`, `Pet.chipNumber`, `Pet.name`.

## Technology constraints

- Java 21, Spring Boot 4.1, Maven
- Spring Web MVC, Spring Data JPA, Bean Validation
- PostgreSQL 17 in development (via Docker Compose); H2 in tests
- REST over JSON; errors as HTTP status codes
- JUnit 5
- Caller's role supplied in the `X-Role` header (`ADMIN` or `DOCTOR`)

## Assumptions

- **A-1** CANCELLED is terminal. No transition out of it is allowed; a new
  appointment must be created instead.
- **A-2** ON_HOLD may be resumed to IN_PROGRESS by the assigned doctor or an
  administrator. Resuming is not a rollback and is not audited.
- **A-3** Overpayment is rejected: a payment that would make the sum of
  payments exceed `total` is refused.
- **A-4** Line items cannot be added, changed or removed once the status is
  READY or later. The bill is frozen when the owner sees the amount.
- **A-5** CLOSED is terminal. The administrator rollback privilege of T-9 does
  not apply to it. The source concept contradicts itself here — it calls CLOSED
  terminal while also allowing rollbacks from any stage — and this resolves the
  contradiction in favour of terminality.
- **A-6** One appointment concerns exactly one pet.
- **A-7** Authentication is out of scope; the `X-Role` header is trusted.

## Acceptance criteria

- **AC-1** `POST /owners` with a blank `fullName` returns 400.
- **AC-2** `POST /pets` referencing an unknown owner returns 404.
- **AC-3** `PATCH /appointments/{id}/status` to `PAID` while `balance > 0`
  returns 409 and the status is unchanged.
- **AC-4** `PATCH /appointments/{id}/status` to `IN_PROGRESS` from `BOOKED`
  returns 409 (CHECKED_IN is required first).
- **AC-5** `PATCH /appointments/{id}/status` to `READY` with an empty diagnosis
  returns 409.
- **AC-6** Any transition out of `CANCELLED` returns 409.
- **AC-7** Any transition out of `CLOSED` returns 409.
- **AC-8** `POST /appointments/{id}/line-items` on an appointment in `READY`
  returns 409.
- **AC-9** Adding two line items of 100.00 ×1 and 50.00 ×2 sets `total` to
  200.00.
- **AC-10** After a Service price changes, the `unitPrice` of an existing line
  item is unchanged and `total` is unchanged.
- **AC-11** A payment exceeding the outstanding balance returns 409.
- **AC-12** Two payments of 50.00 against a total of 100.00 bring `balance` to
  0.00 and allow the transition to `PAID`.
- **AC-13** A rollback performed with `X-Role: DOCTOR` returns 403.
- **AC-14** A rollback performed with `X-Role: ADMIN` and no reason returns 400.
- **AC-15** A successful rollback creates exactly one AuditEntry recording the
  from-status, to-status, role and reason.
- **AC-16** `DELETE /owners/{id}` sets `deletedAt`; the owner's past
  appointments remain retrievable.
- **AC-17** `POST /appointments` referencing a soft-deleted pet returns 409.

## Out of scope

- Web or mobile frontend
- Authentication and token issuance; the `X-Role` header is trusted
- Medications, prescriptions and stock management
- Employee records, rates and specializations
- File attachments (photos, scans, lab results)
- Vaccination schedules and reminders
- Reporting, analytics and invoicing documents
- Multi-clinic or multi-tenant support