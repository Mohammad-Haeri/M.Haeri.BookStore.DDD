# 📚 Domain Discovery

This document records the decisions behind the first version of the bookstore. It defines the scope, the language we use, and the rules the implementation must preserve.

The application is intentionally small. The aim is to demonstrate DDD through clear business behavior, controlled changes, and meaningful boundaries.

These decisions describe the intended behavior. Implementation and tests will follow.

## 🎯 Scope

The first version covers a public book catalogue and a small set of administrative operations.

Visitors can browse books, find a book by ISBN, and view its details. Administrators manage book information, prices, stock, and whether a book is offered by the store.

Purchasing is outside this version. Stock management initially covers registering the starting quantity and receiving more copies.

## Shared language

| Term | Meaning in this project |
|---|---|
| Book | A catalogue entry representing a sellable book edition. Identical physical copies share this entry. |
| BookId | The permanent identifier assigned to a book by our application. |
| ISBN | The book's unique publishing identifier. |
| Stock quantity | The number of copies currently held for that book. |
| Replenishment | Receiving additional copies and increasing the stock quantity. |
| Active | The store currently offers the book in its public catalogue. |
| Out of stock | The book has zero copies available. This does not change its Active or Discontinued status. |
| Discontinued | The store has stopped offering the book. Its record and remaining stock are retained. |
| Reactivation | Returning a discontinued book to Active status. |

Cover types and format variants are outside the initial scope.

## People and use cases

| Actor | Use cases |
|---|---|
| Visitor | Browse the public catalogue, find a book by ISBN, and view book details. |
| Administrator | Register a book, correct its information, change its price, replenish stock, discontinue it, reactivate it, and view books in either status. |

Public browsing does not require an account.

Administrative operations require authentication and administrator authorization. The authentication mechanism and provider will be finalized during the authentication phase.

Authentication remains outside the bookstore domain. A book enforces valid changes to its state; access control determines who may request those changes.

## 🔎 Identity and ISBN

Each book has a permanent `BookId` and a required, unique ISBN.

`BookId` identifies the entity inside our application. ISBN is its business identifier and supports catalogue lookup.

The rules are:

- Two book entries cannot share the same normalized ISBN.
- Accepted spaces and hyphens must not allow equivalent ISBNs to be treated as different values.
- Registering an existing ISBN is rejected.
- Receiving more copies of an existing book is a replenishment operation.
- Correcting an ISBN leaves the book's `BookId` unchanged.

The domain validates the ISBN value. The application and persistence layer coordinate uniqueness across the catalogue, with a database constraint providing the final guarantee when requests arrive concurrently.

The unique index also supports efficient ISBN lookups.

## Book information

A book requires:

- A non-empty title.
- At least one author.
- A non-empty name for every author.
- A non-empty publisher.
- A valid ISBN.
- A valid price.
- A non-negative stock quantity.

A book may have several authors.

Author profiles and separate author-management operations are outside this version. Authors are recorded as part of the book's information.

## 💵 Price

The first version uses USD for all book prices.

- A price must be greater than zero.
- A price may have up to two decimal places.
- Values requiring more precision are rejected rather than silently rounded.
- Changing a price must preserve these rules.

For example, USD 19.95 is valid. Zero, negative prices, and USD 19.955 are rejected.

Currency conversion, discounts, and tax calculations are outside the initial scope.

## Stock

A book can be registered with zero stock. This allows the catalogue entry to exist before any copies arrive.

Stock follows these rules:

- The quantity is a whole number.
- It cannot be negative.
- Replenishment requires a positive quantity.
- Replenishment adds to the existing quantity.
- A discontinued book cannot be replenished until it is reactivated.

For example, receiving five copies when the current quantity is ten produces a quantity of fifteen.

Out-of-stock status is derived from the quantity. It is not a separate flag that administrators edit.

## Book lifecycle

A book is either Active or Discontinued.

Discontinuing a book:

- Preserves its identity, information, and remaining stock.
- Removes it from the public catalogue.
- Prevents replenishment.

An administrator can reactivate it later.

Stock and lifecycle status describe different things. An active book can have zero stock, and a discontinued book can still have copies remaining.

## Public visibility

| Status | Stock | Public behavior |
|---|---:|---|
| Active | Greater than zero | Listed and shown as in stock. |
| Active | Zero | Listed and shown as out of stock. |
| Discontinued | Any quantity | Excluded from public listings, ISBN lookup, and detail requests. |

Administrators can view both active and discontinued books.

Keeping active books visible when stock reaches zero lets visitors still find their information.

## Correcting information

Administrators can correct a book's title, authors, publisher, and ISBN.

Corrections are allowed for both active and discontinued books. Required information must remain valid, at least one author must remain, and the ISBN must remain unique.

The `BookId` stays unchanged.

A correction must still describe the same book. A different book requires a new catalogue entry.

Price changes, replenishment, discontinuation, and reactivation remain separate operations with their own rules.

## 🧪 Examples to carry into tests

| Situation | Expected result |
|---|---|
| Register a valid book with zero stock | Registration succeeds. The active book appears as out of stock. |
| Register a book using an existing ISBN | Registration is rejected. |
| Register a book without any authors | Registration is rejected. |
| Change a price to zero | The change is rejected and the existing price remains. |
| Replenish ten copies with another five | Stock becomes fifteen. |
| Replenish with zero or a negative quantity | The operation is rejected. |
| Discontinue a book with ten copies | Stock remains ten, and the book becomes hidden publicly. |
| Replenish a discontinued book | The operation is rejected. |
| Reactivate that book and replenish it | Replenishment is allowed. |
| Correct an ISBN to another book's ISBN | The correction is rejected. |

These examples will guide domain and integration tests as the features are implemented.

## Deliberate limits

The first version does not include:

- Carts, orders, checkout, payments, or returns.
- Customer accounts, reviews, or wish lists.
- Individual-copy tracking, stock reservations, or multiple warehouses.
- Cover and format variants.
- Multiple currencies, discounts, or tax calculations.

Those features introduce additional business rules. They can be considered later if they help the educational purpose of the project.

## Technical direction

PostgreSQL and EF Core, through the Npgsql provider, are the chosen persistence tools.

Domain rules will live in the domain model. EF Core configuration and database constraints will support those rules from Infrastructure.

Aggregate boundaries, value objects, repository contracts, detailed ISBN validation, and field limits will be worked through in the next design steps.