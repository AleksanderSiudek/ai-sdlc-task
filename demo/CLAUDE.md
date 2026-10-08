# Vet Clinic — project rules

## What this repository is

A Spring Boot backend for a veterinary clinic. The requirements live in
`TASK.md`; the implementation plan lives in `context/PLAN.md`.

**Application implementation is out of scope for the current phase.** Do not
write entities, controllers, services or repositories unless explicitly asked.
The deliverables right now are planning artifacts.

## Stack

- Java 21, Spring Boot 4.1, Maven (`./mvnw`)
- Spring Web MVC, Spring Data JPA, Bean Validation
- PostgreSQL 17 via `compose.yaml` for development; H2 for tests
- JUnit 5

## Conventions

- Tests first, implementation second.
- `TASK.md` is the single source of truth for requirements. If something is
  unclear, it is a gap in `TASK.md` — report it, do not invent a rule.
- Business rules are referenced by their ids (`BR-1`, `A-3`, `AC-7`), never
  restated in prose.
- All appointment state transitions belong in one state-machine service.