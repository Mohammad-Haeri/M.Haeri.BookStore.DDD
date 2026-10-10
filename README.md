# 📚 M.Haeri.BookStore.DDD

A small bookstore Web API built with ASP.NET Core, developed step by step to explore Domain-Driven Design.

I started this project to work through DDD in a manageable application and share what I learn along the way. The aim is to make the code and the reasoning behind it easy to follow, including the decisions that need a bit of explanation.

## 🚧 Current status

The project is at the foundation stage.

The four projects, their references, and the shared build settings are in place. The API host includes OpenAPI support, but there are no bookstore endpoints yet. Domain logic, persistence, authentication, and tests will come as the project develops.

## Planned scope

The first version will focus on a small set of operations:

- Browse the book catalogue and view book details.
- Register books.
- Change a book's price.
- Replenish stock.
- Discontinue books.

Book details will include the title, author, publisher, price, and availability.

Browsing will be available without signing in. Management operations are planned to require administrator authorization.

Carts, checkout, payments, customer accounts, and reviews are outside the initial scope.

## Project structure

| Project | Responsibility |
| --- | --- |
| `M.Haeri.BookStore.Domain` | The domain model and its business rules. |
| `M.Haeri.BookStore.Application` | Application use cases and coordination of domain behavior. |
| `M.Haeri.BookStore.Infrastructure` | Persistence and implementations that depend on external technology. |
| `M.Haeri.BookStore.Api` | HTTP endpoints, application configuration, and dependency registration. |

The project references follow this direction:

- Application references Domain.
- Infrastructure references Application and Domain.
- API references Application and Infrastructure.
- Domain has no references to the other projects.

Shared build settings live in `Directory.Build.props`. Formatting and code conventions are defined in `.editorconfig`. C# files use CRLF line endings.

## How I'm approaching it

Business rules and terminology come first. Those decisions will guide the entities, value objects, and aggregate boundaries.

I'll introduce patterns as the use cases give them a purpose, and document the reasoning where a decision could reasonably go another way. Tests will accompany the features they cover.

The current structure gives us a place to start. The domain model is the next substantial piece of work.

## 🚀 Run locally

You'll need the .NET 10 SDK.

From a terminal:

```bash
git clone https://github.com/Mohammad-Haeri/M.Haeri.BookStore.DDD.git
cd M.Haeri.BookStore.DDD

dotnet restore M.Haeri.BookStore.DDD.slnx
dotnet build M.Haeri.BookStore.DDD.slnx --no-restore
```

For HTTPS, trust the local development certificate:

```bash
dotnet dev-certs https --trust
```

Then start the API:

```bash
dotnet run --project BookStore.Api/M.Haeri.BookStore.Api.csproj --launch-profile https
```

The HTTPS launch profile uses:

- HTTPS: `https://localhost:7251`
- HTTP: `http://localhost:5075`

In the Development environment, the OpenAPI JSON document is available at:

`https://localhost:7251/openapi/v1.json`

There are currently no bookstore endpoints to try.

## Feedback

If you spot a questionable design decision or an explanation that is hard to follow, please open an issue. A concrete example helps, especially while the model is still taking shape.

## License

This project is licensed under the [MIT License](LICENSE).