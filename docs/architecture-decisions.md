# 🧭 Architecture decisions

This document records the main technical choices behind the project and the trade-offs that come with them.

The [domain discovery notes](domain-discovery.md) describe the bookstore's scope and business rules. These notes explain how the application will support those rules.

The decisions below are the agreed direction for the first version. They do not imply that every part has already been implemented.

## 1. Keep the application in one deployment

The bookstore will run as one ASP.NET Core API backed by one PostgreSQL database.

The solution has four technical layers:

| Layer | Responsibility | Explicit project references |
|---|---|---|
| Domain | Business behaviour, value objects, and aggregate rules | None |
| Application | Use cases and the contracts needed to carry them out | Domain |
| Infrastructure | Persistence and implementations of application and domain contracts | Application, Domain |
| API | HTTP endpoints, request handling, and dependency registration | Application, Infrastructure |

The Domain layer stays independent of ASP.NET Core, EF Core, and database providers. Application use cases coordinate work without depending on Infrastructure implementations.

The API wires those implementations together when the application starts.

This structure gives the project clear dependency boundaries while keeping deployment and local development manageable. The layers still share one process and one release cycle, so separating them into projects does not make them independently deployable services.

## 2. Use PostgreSQL with EF Core

PostgreSQL is the database, and EF Core is the persistence tool for the first version.

EF Core fits the work this project needs: loading aggregates, tracking changes, mapping value objects, applying migrations, and committing related changes together. It also leaves enough room to concentrate on domain modelling and application behaviour.

Database configuration and EF Core mappings belong in Infrastructure.

Choosing an ORM still leaves us responsible for the SQL it produces. Query shape, indexes, tracking, and the amount of data loaded will need attention as endpoints are implemented.

There is no measured requirement for a second data-access tool at this stage.

## 3. Treat Book as the aggregate root

A Book represents one sellable edition in the catalogue. Its metadata, price, stock quantity, and lifecycle status belong within the same aggregate.

Changes go through Book's behaviour so that it can enforce the rules governing its state. For example, replenishment must receive a positive quantity and must reject a discontinued book.

A failed operation must leave the Book unchanged. Methods that modify several values need to validate the proposed change before assigning any of them.

The model will distinguish identity from descriptive values:

- BookId is the stable internal identity.
- ISBN is a required, unique business identifier that can be corrected.
- ISBN and price will be represented as immutable value objects.
- Author information belongs within the Book aggregate for this milestone.

Correcting an ISBN does not create a different Book or change its BookId.

Keeping these values together makes their consistency rules straightforward to enforce. It also means concurrent changes to different parts of a Book may affect the same aggregate version. That is an acceptable starting point for this scope.

The detailed business rules remain in the domain discovery notes.

## 4. Put command persistence behind a focused Book repository

The Book repository contract belongs in Domain. Its EF Core implementation belongs in Infrastructure.

Application command handlers use it to load and persist Book aggregates. The contract should express the operations the application needs without exposing DbContext, EF tracking, or database query types.

There is one repository for the Book aggregate type. Value objects do not need repositories of their own.

The repository is also not the entry point for every catalogue read. Queries can use separate read contracts that return the data an endpoint needs.

This abstraction has a cost: there is another contract and implementation to maintain, while EF Core already provides repository-like capabilities. Here, it gives aggregate persistence a clear boundary and makes the dependency direction explicit.

We will keep it specific to Book rather than building a generic repository framework.

## 5. Let the application choose when to commit

Application use cases decide when their work is ready to be saved.

A small Unit of Work contract belongs in Application. Infrastructure implements that contract through the same DbContext used by the repositories.

Repositories do not save automatically after each method call. A handler can complete the required domain operations and then commit once.

For each attempt, the repository and Unit of Work must share the same DbContext. Otherwise, the apparent commit boundary would not cover the same tracked changes.

EF Core supplies the actual Unit of Work behaviour. The application contract exposes the commit boundary without requiring handlers to depend on EF Core.

A transactional save prevents a partially persisted operation. Concurrent updates still require their own protection.

## 6. Separate commands and queries while sharing one database

Commands and queries will have separate handlers and contracts.

Commands express an intention to change state. They load the aggregate, invoke its behaviour, and commit the result.

Queries return read models shaped for their callers. Infrastructure can project directly from the database without constructing a full Book aggregate for every catalogue request.

Both paths use EF Core and the same PostgreSQL database. The API calls ordinary application handlers directly.

This gives reads and writes room to have different needs without introducing synchronisation between separate data stores.

The additional handlers and read contracts create some structure to maintain. They should follow actual use cases, with shared code introduced where there is a clear benefit.

Public queries must consistently apply the visibility rules. A discontinued book must stay hidden whether the caller browses the catalogue, searches by ISBN, or requests its details directly.

## 7. Handle concurrent changes according to the operation

Book updates will use optimistic concurrency.

When a handler saves a Book, persistence checks that the aggregate version still matches the version it loaded. If another operation changed it first, the stale save is rejected.

That rejects an unsafe attempt. The application then decides whether the original operation can be tried again.

| Concurrent situation | Intended handling |
|---|---|
| Two valid stock increases | Reload and retry the original quantity increase within a bounded retry policy |
| A book is discontinued during replenishment | Reload, recheck the rule, and reject replenishment |
| ISBN or descriptive information changes during replenishment | Preserve the latest information and retry if replenishment remains valid |
| Competing price or ISBN edits | Report a conflict so the change can be reviewed |
| A request explicitly depends on information that has changed | Report the failed precondition rather than silently changing its meaning |

For example, two requests may load a stock quantity of 10. One adds 5 and saves 15. The other originally intended to add 3.

After a concurrency conflict, the second request must reload 15 and apply the original increase of 3, producing 18. It must not save its earlier calculated total of 13.

Retries must:

- Reload using the original BookId.
- Work from the latest state.
- Reapply the original business intention.
- Run the relevant domain checks again.
- Stop after a defined limit.

Reloading by BookId matters because ISBN can be corrected while an operation is running.

Updates should change only the intended properties. That alone does not protect an operation whose validity depends on another property, such as stock replenishment depending on lifecycle status.

The persistence mapping must make every aggregate change participate in concurrency protection. The exact token mapping and retry limits will be settled during implementation.

ISBN uniqueness needs separate protection. Application checks can provide an early, friendly rejection, but PostgreSQL must enforce uniqueness on the normalised ISBN to cover competing requests.

A queue was considered as a way to serialise operations. The current API will use database constraints and optimistic concurrency. Durable messaging can be reconsidered when there is a concrete processing or integration requirement.

## 8. Give validation a clear responsibility at each boundary

Validation has several responsibilities:

| Boundary | Responsibility |
|---|---|
| API | Request shape, binding, and transport-level checks |
| Application | Use-case prerequisites, record lookup, and checks requiring stored data |
| Domain | Rules that must hold whenever a Book is created or changed |
| Database | Persistent constraints, including ISBN uniqueness |

A valid HTTP request can still describe an invalid business operation. Domain behaviour must therefore enforce its rules regardless of where the request originated.

Application checks can improve feedback, particularly for missing books or an ISBN already in use. Checks against stored data must still account for concurrent changes.

Database constraints provide the final protection for rules that persistence must guarantee.

Some checks may appear at more than one boundary because they serve different purposes. Their meaning should remain consistent.

## 9. Represent expected failures explicitly

Expected validation and business failures will use explicit results with stable error codes and clear messages.

Examples include:

- Invalid ISBN or price.
- Book not found.
- ISBN already assigned to another Book.
- Replenishment rejected because a Book is discontinued.
- A concurrent change that could not be resolved safely.

Domain and Application results will not contain HTTP status codes or ASP.NET Core response types. The API translates those results into appropriate responses, using Problem Details for errors.

Unexpected failures continue through exception handling. Infrastructure should recognise specific persistence failures, such as a unique-constraint violation, and translate them where their meaning is understood.

It must not turn every database exception into a business rejection.

The exact Result types and their placement will be decided when the first use cases are implemented.

## 10. Keep access control outside the Book model

Public catalogue browsing is anonymous. Management use cases require administrator access.

Authentication and authorisation belong at the API and application boundary. Book enforces its business rules independently of the caller's authentication mechanism.

An authorised administrator can still request an invalid operation. Authorisation allows the caller to attempt it; the domain decides whether the requested state change is valid.

The identity provider and authentication setup remain open until the authentication phase.

## 11. Leave detailed implementation choices to the relevant work

The architecture gives us enough direction to start the domain model. Several details still need to be designed as their implementation approaches:

- BookId representation and construction.
- Exact ISBN validation and normalisation rules.
- Field limits and value-object APIs.
- EF Core mappings and concurrency-token configuration.
- Retry limits and conflict responses.
- Authentication provider and configuration.

Domain events can be introduced when an agreed use case needs a separate business reaction to a domain change. There is no such reaction in the current scope.

These notes should evolve with the project. If a significant choice changes, record what replaced it and why, keeping enough of the earlier reasoning for a reader to understand the change.