---
name: spring-boot-best-practices
description: Use when writing or reviewing Spring Boot / Java backend code (controllers, services, repositories, configuration, messaging). Covers layered architecture, dependency injection, exception handling, validation, transaction boundaries, persistence performance (N+1 queries), configuration/secrets, and testing conventions.
---

# Spring Boot best practices

## Layered architecture & separation of concerns

- **Controllers are thin.** A controller method validates/binds the request, delegates to a service, and maps the result to a response — it contains no business logic.
- **Service layer holds business logic.** Business rules, orchestration across multiple repositories, and transaction boundaries live here, not in the controller or repository.
- **Repositories only do data access.** No business logic in a custom repository method beyond query construction.
- **Never expose JPA entities directly in API request/response bodies.** Use DTOs (request/response records or classes) at the API boundary, and map between entity and DTO explicitly (manually, or via MapStruct/a mapper if the project already uses one). Exposing entities directly leaks persistence structure into the API contract, risks serializing lazy-loaded associations unintentionally (causing `LazyInitializationException` or accidental N+1s), and makes the API contract and the DB schema impossible to evolve independently.

## Dependency injection

- **Use constructor injection, always** — not field injection (`@Autowired` directly on a field). Constructor injection makes dependencies explicit, enables `final` fields (immutability), makes the class trivially testable without a Spring context (just `new` it with mocks), and fails fast at construction if a dependency is missing.
- With Lombok available, `@RequiredArgsConstructor` on `final` fields is the common shorthand for this — don't hand-write boilerplate constructors if the project already uses Lombok elsewhere.
- Avoid injecting `ApplicationContext` to manually look up beans as a workaround for a DI design issue — restructure the dependency graph instead.

## Exception handling

- Centralize exception-to-HTTP-response mapping in a `@ControllerAdvice`/`@RestControllerAdvice` with `@ExceptionHandler` methods, rather than try/catch blocks scattered across controllers returning ad hoc error responses.
- Define a small hierarchy of meaningful custom exceptions (e.g. `ResourceNotFoundException`, `ValidationException`) rather than throwing generic `RuntimeException` everywhere — this is what makes centralized handling actually work well.
- **Never leak internal details in an error response to the client**: no raw stack traces, no internal exception messages that reveal implementation/schema details, no SQL fragments. Log the full detail server-side; return a sanitized, consistent error shape to the client.
- Don't catch an exception just to log and rethrow it unchanged — that adds noise without value. Catch where you can either add context, recover, or translate it into a different exception type the layer above expects.

## Validation

- Use Bean Validation annotations (`@NotNull`, `@Size`, `@Email`, `@Valid`, custom constraint annotations, etc.) on request DTOs, and `@Valid`/`@Validated` at the controller method parameter, so invalid requests are rejected before reaching business logic — don't hand-roll validation checks in the service layer for things Bean Validation already covers declaratively.
- Validate at the boundary (the DTO entering the controller), not deep inside the service — fail fast with a clear 400-level response rather than discovering invalid data mid-transaction.

## Transactions

- Put `@Transactional` at the **service layer**, on the method that represents one logical unit of work — not on the repository (that's usually redundant with Spring Data's own transaction handling) and not on the controller (too broad a scope, and HTTP-layer concerns shouldn't dictate transaction boundaries).
- Be aware of the **self-invocation problem**: calling a `@Transactional` method from another method in the *same* class (via `this.`) bypasses the Spring proxy and the transaction annotation has no effect. If this pattern is needed, either restructure so the call goes through a separate bean, or be explicit that the lack of transactionality here is intentional.
- Mark read-only query methods `@Transactional(readOnly = true)` — this is a real optimization hint to the persistence provider (and some drivers), not just documentation.
- Keep transactions as short as possible — don't do slow I/O (external HTTP calls, file access) inside a transactional method unless genuinely necessary, since that holds a DB connection/lock for the duration.

## Persistence & query performance

- **Watch for N+1 query problems**: iterating a collection of entities and accessing a lazy-loaded association inside the loop triggers one query per entity. Fix with a fetch join (`JOIN FETCH` in JPQL), an `@EntityGraph`, or a dedicated projection query that gets everything needed in one query — don't just "see if it's slow in prod" before checking for this pattern.
- Use **projections or DTO-returning queries** for read-heavy, display-only endpoints instead of loading full entity graphs and mapping afterward — this avoids pulling columns/associations the endpoint doesn't need.
- **Paginate** any endpoint/query that can return an unbounded number of rows (`Pageable`/`Page<T>` in Spring Data) — don't load an entire table into memory.
- Be deliberate about `FetchType.EAGER` vs `LAZY` on entity associations — default to `LAZY` for collections and most `@ManyToOne`/`@OneToOne` associations unless there's a specific, justified reason for eager loading; eager-by-default is a common source of accidental over-fetching.

## Configuration & secrets

- Externalize configuration via `application.yml`/`application.properties` with Spring profiles (`dev`, `test`, `prod`) rather than hardcoding environment-specific values.
- **Never commit a real secret (DB password, API key, signing key) into `application.yml`** — source secrets from environment variables, a secrets manager, or the project's established pattern (e.g. Kubernetes `Secret`s injected as env vars — see the project's infra conventions if any). This overlaps with the `app-security-review` skill; apply both when a change touches configuration.

## Logging

- Use the SLF4J API (`LoggerFactory.getLogger(...)`) rather than `System.out.println` — gives you levels, structured output, and configurable sinks for free.
- Never log sensitive data (passwords, tokens, full card numbers, decrypted content) even at DEBUG level — a DEBUG log level getting accidentally enabled in production is a realistic failure mode, not a hypothetical.
- Log at the right level: ERROR for things requiring attention, WARN for recoverable/unexpected-but-handled conditions, INFO for significant lifecycle events, DEBUG for diagnostic detail — avoid defaulting everything to INFO or ERROR regardless of severity.

## Messaging (if Kafka or similar is in use)

- Design consumers to be **idempotent** — a message can be redelivered (at-least-once delivery is the common guarantee), so processing the same message twice should not corrupt state (e.g. use an idempotency key, or make the operation naturally idempotent like an upsert).
- Be deliberate about offset-commit strategy (auto-commit vs manual commit after successful processing) — auto-commit before processing completes can silently lose messages on a crash; committing after processing (manually) is usually safer for anything where message loss matters.
- Keep consumer processing fast or move heavy work off the consumer thread — a slow consumer can trigger rebalancing/lag issues in the broader pipeline.

## Testing

- Use **slice tests** (`@WebMvcTest` for controller-layer tests with mocked service dependencies, `@DataJpaTest` for repository-layer tests against an in-memory or test DB) instead of a full `@SpringBootTest` for most unit-level testing — full context loads are slow and test more than the unit under test actually needs.
- Reserve `@SpringBootTest` for genuine integration tests that need the full wired application context.
- Prefer **Testcontainers** (a real containerized Postgres/Kafka/etc.) over H2/in-memory substitutes for integration tests where the real engine's behavior (SQL dialect quirks, constraints) matters — if the project doesn't have Testcontainers set up yet, this is worth flagging as a suggestion rather than introducing unprompted mid-task.
- Test services with mocked repositories/dependencies (constructor injection makes this trivial) rather than always spinning up a full context — fast unit tests should be the majority of the suite.

## How to run this review

1. Walk the sections above against the actual files touched (controller, service, repository, config, entity/DTO).
2. Prioritize: entity-leaking-through-API, N+1 queries, field injection, and hardcoded secrets are the highest-impact, most common findings — check those first.
3. Fix inline where straightforward; flag anything requiring a broader architectural change (e.g. introducing Testcontainers, restructuring a transaction boundary that affects callers) rather than doing it unprompted.
