# Project: ShelfLife — a Library Lending API

You'll build the backend for a small library: members borrow books, return them,
get fined when they're late, and staff manage the catalogue. It's a REST API
(no frontend needed) built with Spring Boot.

Why this project: it's small enough to finish, but it has real business rules
(borrow limits, due dates, fines, roles), so you'll touch almost every part of
Spring that jobs actually use. No tutorials-copying: each milestone tells you
**what** to build and **what to research**, never the code.

---

## Tech stack (pin these)

| Thing | Choice |
|---|---|
| Language | Java 21 (records, `var`, switch expressions, text blocks) |
| Framework | Spring Boot 3.x (latest stable) |
| Build | Maven (simpler to start than Gradle) |
| DB | H2 in-memory first, PostgreSQL later (M7) |
| Testing | JUnit 5, Mockito, Spring Boot Test, Testcontainers |
| Tools | IntelliJ IDEA Community, Postman or `curl`, Docker |

Generate the starting project at **start.spring.io**. Dependencies to pick at the
start: *Spring Web, Spring Data JPA, H2, Validation, Lombok (optional), Spring Boot DevTools*.
Add the rest when a milestone needs them — knowing *why* you add a dependency is part of the lesson.

---

## Domain model

```
Author 1───* Book *───1 Category
              │
              1
              │
              * Copy            (a physical copy of a book; a book can have 3 copies)
              │
              1
              │
              * Loan *───1 Member
```

- **Author**: id, name, bio
- **Category**: id, name (unique)
- **Book**: id, isbn (unique), title, publishedYear, author, category
- **Copy**: id, book, barcode (unique), status (`AVAILABLE`, `ON_LOAN`, `LOST`)
- **Member**: id, name, email (unique), password, role (`MEMBER`, `LIBRARIAN`), joinedAt
- **Loan**: id, copy, member, borrowedAt, dueAt, returnedAt (nullable), fine

### Business rules
1. A member can have at most **3 active loans**.
2. A loan lasts **14 days**.
3. Late returns cost **$0.50 per day**, capped at **$20**.
4. A member with **unpaid fines over $10** cannot borrow.
5. Only an `AVAILABLE` copy can be borrowed.
6. A book can't be deleted while any of its copies is on loan.

---

## Milestones

Work through them in order. Don't move on until the "Done when" list passes.
Commit after each milestone (`git commit -m "M2: catalogue CRUD"`) so you can see your progress.

### M0 — Java warm-up (½ day, skip if you're comfortable)
Before Spring, write a plain Java console program (no frameworks) that holds a
`List<Book>` in memory and lets you add, list, search by title, and sort by year.

**Research:** classes vs records, interfaces, `List`/`Map`/`Optional`, Streams
(`filter`, `map`, `sorted`, `collect`), exceptions, `equals`/`hashCode`.

**Done when:** you can explain what a Stream pipeline does line by line.

---

### M1 — Hello Spring (½ day)
Create the project, run it, and add `GET /api/health` returning `{"status":"UP"}`.

**Research:** what `@SpringBootApplication` does, what an embedded Tomcat is,
`@RestController`, `@GetMapping`, `application.properties`/`application.yml`,
how Jackson turns an object into JSON.

**Done when:**
- `mvn spring-boot:run` starts the app on port 8080
- `curl localhost:8080/api/health` returns the JSON
- You can explain in one sentence what "auto-configuration" is

---

### M2 — Catalogue CRUD, in memory (1 day)
Build full CRUD for **Book** with no database: store books in a `Map` inside a
service class.

Endpoints:
- `GET /api/books` — list
- `GET /api/books/{id}` — one book (404 if missing)
- `POST /api/books` — create (return **201** + `Location` header)
- `PUT /api/books/{id}` — update
- `DELETE /api/books/{id}` — delete (**204**)

Structure it in layers: `controller` → `service` → (later) `repository`.

**Research:** dependency injection and **constructor injection**, `@Service`,
beans and the ApplicationContext, `@PathVariable`, `@RequestBody`,
`ResponseEntity`, HTTP status codes, why controllers shouldn't hold business logic.

**Done when:** every endpoint works from Postman/curl with correct status codes.

---

### M3 — Real persistence with JPA (1–2 days)
Replace the `Map` with **Spring Data JPA + H2**. Add `Author`, `Category`, and
`Copy` entities with the relationships from the diagram.

Extra endpoints:
- `GET /api/books?title=spring&category=Programming` — filter
- `GET /api/books?page=0&size=10&sort=title` — pagination & sorting
- `POST /api/books/{id}/copies` — add a physical copy
- Seed some data on startup (`data.sql` or a `CommandLineRunner`)

**Research:** `@Entity`, `@Id`, `@GeneratedValue`, `@ManyToOne`, `@OneToMany`,
`mappedBy`, `FetchType.LAZY` vs `EAGER`, `JpaRepository`, **derived query methods**
(`findByTitleContainingIgnoreCase`), `@Query`, `Pageable`/`Page`, the H2 console,
`spring.jpa.show-sql`.

**Done when:**
- Data survives between requests (but resets on restart — that's fine for H2)
- You've looked at the SQL Hibernate generates and seen an **N+1 query problem**
  at least once, then fixed it (research: `JOIN FETCH` / `@EntityGraph`)

---

### M4 — DTOs, validation & error handling (1 day)
Stop returning entities directly from controllers.

- Create request/response **DTOs as Java records** (`CreateBookRequest`, `BookResponse`)
- Write a mapper (by hand first; optionally try MapStruct after)
- Validate input: ISBN not blank and correct format, title ≤ 200 chars, year not in the future
- One **global exception handler** that returns errors in a consistent shape:

```json
{
  "status": 400,
  "error": "Validation failed",
  "details": { "title": "must not be blank" },
  "timestamp": "2026-11-25T10:00:00Z"
}
```

- Custom exceptions: `ResourceNotFoundException` (404), `BusinessRuleException` (409/422)

**Research:** why entities shouldn't leak out of the API, Bean Validation
(`@NotBlank`, `@Size`, `@Pattern`, `@Valid`), `@RestControllerAdvice`,
`@ExceptionHandler`, `ProblemDetail` (Spring 6's built-in RFC 7807 support).

**Done when:** sending garbage JSON never produces a stack trace or a 500.

---

### M5 — Loans: the business logic (1–2 days)
This is the heart of the app. Add `Member` and `Loan`.

Endpoints:
- `POST /api/members` — register
- `POST /api/loans` `{ "memberId": 1, "copyId": 5 }` — borrow
- `POST /api/loans/{id}/return` — return (calculate fine)
- `GET /api/members/{id}/loans?active=true` — a member's loans
- `GET /api/loans/overdue` — all overdue loans

Enforce **all six business rules** above in the service layer.

**Research:** `@Transactional` (what happens if an exception is thrown halfway?),
`java.time` (`LocalDate`, `Instant`, `Duration`, `ChronoUnit.DAYS.between`),
`BigDecimal` for money (and **why not `double`**), injecting a `Clock` so time is testable,
optimistic locking with `@Version` (two people borrowing the last copy at the same time).

**Done when:** you can't break any rule via the API, no matter what you send.

---

### M6 — Testing (1–2 days, don't skip this)
Employers care about this more than almost anything else on the list.

- **Unit tests** for `LoanService` with Mockito — one test per business rule,
  plus fine calculation edge cases (on time, 1 day late, 100 days late → capped)
- **Slice test** for a controller with `@WebMvcTest` + `MockMvc`
- **Repository test** with `@DataJpaTest`
- **One full integration test** with `@SpringBootTest` that borrows and returns a book

**Research:** the test pyramid, Arrange/Act/Assert, `@Mock` / `@InjectMocks`,
`when(...).thenReturn(...)`, `verify(...)`, AssertJ (`assertThat`), `@MockitoBean`,
`jsonPath` assertions, a fixed `Clock` in tests.

**Done when:** `mvn test` is green and `LoanService` has tests for every rule.

---

### M7 — PostgreSQL, Docker, profiles & migrations (1 day)
- Run PostgreSQL in **Docker** with a `docker-compose.yml`
- Use **Spring profiles**: `dev` (H2) and `prod` (Postgres)
- Manage the schema with **Flyway** migrations instead of `ddl-auto=update`
- Switch your integration test to **Testcontainers** (real Postgres in tests)

**Research:** `application-{profile}.yml`, `@Profile`, `spring.profiles.active`,
environment variables for secrets, Flyway naming (`V1__init.sql`), why
`ddl-auto=update` is dangerous in production.

**Done when:** `docker compose up` + running with the `prod` profile works, and
restarting the app keeps your data.

---

### M8 — Security with JWT (2 days)
Add **Spring Security**.

- `POST /api/auth/register` and `POST /api/auth/login` (login returns a JWT)
- Passwords stored with **BCrypt**, never plain text
- Rules:
  - Anyone: browse books
  - `MEMBER`: borrow/return *their own* loans, see *their own* history
  - `LIBRARIAN`: manage catalogue, see all loans, register copies
- `POST /api/loans` should now take the member from the token, not the request body

**Research:** the `SecurityFilterChain` bean, `UserDetailsService`,
`PasswordEncoder`, stateless sessions, JWT structure (header.payload.signature),
OAuth2 Resource Server (`spring-boot-starter-oauth2-resource-server`) **or** a
hand-written JWT filter (try the filter first to understand it), `@PreAuthorize`,
401 vs 403.

**Done when:** a member gets **403** trying to delete a book, and **401** with no token.

---

### M9 — Production polish (1 day, pick at least 3)
- [ ] **OpenAPI/Swagger UI** docs (`springdoc-openapi`)
- [ ] **Actuator** health/metrics endpoints
- [ ] A **`@Scheduled`** job that runs nightly and logs/flags overdue loans
- [ ] **Caching** on `GET /api/books/{id}` with `@Cacheable`
- [ ] **Structured logging** with SLF4J; log every borrow/return
- [ ] An **event**: when a book is returned, publish `BookReturnedEvent` and handle it
      in a separate listener (`ApplicationEventPublisher`, `@EventListener`)
- [ ] A **Dockerfile** for the app itself, so `docker compose up` runs everything
- [ ] A **GitHub Actions** workflow that runs `mvn verify` on every push

---

## Stretch goals (after you're done)
- **Reservations**: if no copy is available, join a waitlist; first in line gets
  notified when a copy comes back
- Send a "your book is due tomorrow" **email** (use MailHog/Mailpit in Docker)
- Rewrite one module with **Spring WebFlux** to see reactive programming
- Split notifications into a second service talking over **Kafka or RabbitMQ**

---

## Suggested package layout

```
com.yourname.shelflife
├── ShelfLifeApplication.java
├── config/          (security, clock, openapi beans)
├── catalogue/       (Book, Author, Category, Copy: entity, repo, service, controller, dto)
├── member/
├── loan/
├── auth/
└── common/          (exceptions, error handler)
```

(This is "package by feature" — research how it differs from "package by layer".)

---

## Rough timeline

| Days | Milestones |
|---|---|
| 1 | M0, M1, M2 |
| 2–3 | M3, M4 |
| 4–5 | M5, M6 |
| 6 | M7 |
| 7–8 | M8 |
| 9 | M9 |

About **2 weeks part-time** or **8–9 full days**.

---

## Learning resources
- **Spring guides**: spring.io/guides ("Building a RESTful Web Service", "Accessing Data with JPA")
- **Reference docs**: docs.spring.io/spring-boot/reference
- **Baeldung** (baeldung.com): best place to look up any specific Spring topic
- **Amigoscode / Dan Vega** on YouTube: modern Spring Boot 3 content
- **Book (optional)**: *Spring in Action, 6th ed.* (Craig Walls)

## How to learn fast
1. **Read the error message first.** Spring stack traces are long, but the root cause is usually the last `Caused by:`.
2. **Type everything yourself.** No copy-paste, not even from docs.
3. **Before each milestone, write down 3 questions** you expect to answer. Check them off at the end.
4. **Break things on purpose.** Remove `@Transactional`, drop `@Service`, make a field private with no getter. Watch what happens.
5. **Ask for a review.** When you finish a milestone, push it and ask me to review it.
   I'll point out what a senior dev would flag without rewriting it for you.
