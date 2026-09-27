# EF Core (Entity Framework Core)

*A deep-dive walkthrough of EF Core — covering `DbContext`/`DbSet<T>` and the change tracker as the actual mechanism underneath "just use C# objects," why `DbContext` is the textbook example of a scoped service, LINQ-to-Entities translation revisited with EF Core's specific behavior, navigation properties and loading strategies (including the N+1 problem in depth), tracking vs. no-tracking queries, `SaveChanges` and optimistic concurrency, migrations, and the disciplined cases where dropping to raw SQL is the right call rather than a failure of the ORM.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [DbContext and DbSet: What They Actually Are](#1-dbcontext-and-dbset-what-they-actually-are)
3. [DbContext Lifetime: The Textbook Scoped Service](#2-dbcontext-lifetime-the-textbook-scoped-service)
4. [The Change Tracker: How EF Core Knows What Changed](#3-the-change-tracker-how-ef-core-knows-what-changed)
5. [LINQ to Entities, Revisited for EF Core Specifically](#4-linq-to-entities-revisited-for-ef-core-specifically)
6. [Navigation Properties and Relationships](#5-navigation-properties-and-relationships)
7. [Loading Strategies: Eager, Lazy, and Explicit](#6-loading-strategies-eager-lazy-and-explicit)
8. [The N+1 Problem, In Depth](#7-the-n1-problem-in-depth)
9. [Tracking vs. No-Tracking Queries](#8-tracking-vs-no-tracking-queries)
10. [SaveChanges: How Persistence Actually Happens](#9-savechanges-how-persistence-actually-happens)
11. [Optimistic Concurrency](#10-optimistic-concurrency)
12. [Migrations](#11-migrations)
13. [Transactions Beyond a Single SaveChanges](#12-transactions-beyond-a-single-savechanges)
14. [Raw SQL: When and How to Drop Down](#13-raw-sql-when-and-how-to-drop-down)
15. [Common Pitfalls](#14-common-pitfalls)
16. [Quick Reference Table](#quick-reference-table)
17. [Conclusion](#conclusion)

---

## Introduction

EF Core lets you work with a database through ordinary C# objects and LINQ queries rather than hand-written SQL — but "ORM" undersells what's actually happening underneath: a `DbContext` is a genuine unit-of-work implementation with its own change-tracking system, its query methods return `IQueryable<T>` that gets translated into SQL via the exact expression-tree mechanism this series' IEnumerable/IQueryable guide covers in full depth, and its `SaveChanges` call is where an entire batch of in-memory object mutations gets diffed against tracked state and turned into the minimal set of `INSERT`/`UPDATE`/`DELETE` statements actually needed. This guide goes deep on all of that machinery — the change tracker specifically, since it's the single most important mental model for understanding EF Core correctly, `DbContext`'s lifetime (a direct, canonical application of this series' ASP.NET Core Dependency Injection guide's Scoped lifetime), and the N+1 problem, which is the most common, most costly real-world EF Core mistake.

```plaintext
DbContext (Scoped, per this series' ASP.NET Core Dependency Injection guide)
   ↓
DbSet<Product> Products → IQueryable<Product> (per this series' IEnumerable/IQueryable guide)
   ↓
.Where(p => p.Price > 100)  →  EXPRESSION TREE  →  translated to SQL  →  executed on the DATABASE
   ↓
Tracked entities, mutated in memory  →  SaveChanges()  →  the CHANGE TRACKER diffs them  →
   →  the MINIMAL set of INSERT/UPDATE/DELETE statements, in ONE transaction
```

---

## 1. DbContext and DbSet: What They Actually Are

### `DbContext`: a unit of work, not just a "database connection wrapper"

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; } = null!;
    public DbSet<Order> Orders { get; set; } = null!;

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Product>().HasKey(p => p.Id); // and further mapping configuration
    }
}
```

A `DbContext` represents one coherent, bounded session of work against the database — it holds a connection, tracks every entity you've loaded or added through it (Section 3), and is responsible for translating a single call to `SaveChanges()` into a consistent, atomic set of database writes. This is precisely why "unit of work" is the accurate description, not "connection wrapper" — the connection itself is almost incidental to what a `DbContext` actually manages.

### `DbSet<T>`: the entry point into a specific entity type, and an `IQueryable<T>` itself

```csharp
public DbSet<Product> Products { get; set; } = null!;

// DbSet<T> IS an IQueryable<T> — everything this series' IEnumerable/IQueryable guide covers applies directly:
var expensiveProducts = context.Products.Where(p => p.Price > 100); // still just building an EXPRESSION TREE
```

Each `DbSet<T>` property corresponds to a table (or, per Section 5, part of a mapped relationship) and implements `IQueryable<T>` directly — this is worth stating explicitly because it means every mechanic this series' IEnumerable/IQueryable guide covers (deferred execution, expression trees, the provider translating your LINQ into SQL) applies to EF Core queries exactly as described there, with EF Core's own `IQueryProvider` implementation being the specific translator doing the work.

---

## 2. DbContext Lifetime: The Textbook Scoped Service

### Why `DbContext` is registered `Scoped` by convention, and why that's not arbitrary

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)); // registers AppDbContext as SCOPED, by default
```

This series' ASP.NET Core Dependency Injection guide's Section 4 explicitly names `DbContext` as the textbook example of why the `Scoped` lifetime exists — worth restating precisely why here: a `DbContext`'s change tracker (Section 3) accumulates state over its lifetime, and that accumulated state needs to correspond to *one coherent unit of work*. One HTTP request represents exactly that — a single, bounded operation — which is precisely why "one `DbContext` instance per request" (what `Scoped` mechanically provides, per that guide's Section 5) is the natural, correct fit.

### Why `DbContext` as a Singleton is a direct, serious instance of the captive dependency problem

```csharp
// ❌ Registering AppDbContext as a SINGLETON is a severe, common mistake
builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString),
    ServiceLifetime.Singleton); // DON'T do this
```

This series' ASP.NET Core Dependency Injection guide's Section 7 covers the captive dependency problem in the abstract; a Singleton `DbContext` is one of the most damaging concrete instances of it — a single `DbContext` instance shared across every concurrent request would accumulate change-tracked entities from every request that ever touched it, entangling entirely unrelated units of work together, and (per this series' Threading guide's Section 3) `DbContext` is explicitly **not thread-safe**, meaning concurrent requests sharing one instance risk genuine race conditions and exceptions, not just logically confused data.

### Why `DbContext` as Transient is *safer* than Singleton but still usually wrong

```plaintext
A Transient DbContext avoids the SHARING danger (each resolution gets
  its OWN instance) — but it discards the entire benefit of Scoped's
  "one consistent unit of work per request" model, since MULTIPLE
  DbContext instances resolved within the SAME request would each have
  their OWN, independent change tracker, unaware of entities another
  instance already loaded or is tracking — undermining change tracking's
  whole point (Section 3) for anything spanning more than one resolution.
```

This is worth knowing as the genuine, precise reason Scoped — not just "anything other than Singleton" — is the specifically correct lifetime: Transient avoids the thread-safety and cross-request-entanglement dangers of Singleton, but still fails to provide the single, coherent unit-of-work guarantee a request genuinely benefits from.

---

## 3. The Change Tracker: How EF Core Knows What Changed

### Every entity loaded through a tracking query is registered with the context's change tracker

```csharp
var product = await context.Products.FirstAsync(p => p.Id == 42); // TRACKED, automatically, by default
product.Price = 29.99m; // just an ORDINARY C# property assignment — no explicit "mark as changed" call needed

await context.SaveChangesAsync(); // the CHANGE TRACKER detects Price differs from its ORIGINAL, loaded value
```

This is the actual mechanism underneath "just use C# objects" — when a tracking query (Section 8 distinguishes this from no-tracking queries) loads an entity, EF Core keeps a snapshot of its original values *and* a reference to the live object; mutating the object's properties directly, with ordinary C# assignment, is all that's needed — the change tracker compares the live object's current values against its own snapshot when `SaveChanges` runs, and generates an `UPDATE` only for the properties that genuinely differ.

### Entity states: what the change tracker actually records per entity

```plaintext
Added:     a NEW entity, not yet in the database — SaveChanges produces an INSERT
Unchanged: loaded, tracked, and its properties still match the ORIGINAL snapshot
Modified:  loaded, tracked, and AT LEAST ONE property now differs from the snapshot — produces an UPDATE
Deleted:   marked for removal via context.Remove(entity) — produces a DELETE
Detached:  NOT tracked at all (Section 8's no-tracking queries produce entities in this state)
```

Every tracked entity is in exactly one of these states at any given moment — this is the literal, concrete data structure the change tracker maintains, and `SaveChanges` (Section 9) is fundamentally just "for every entity currently in `Added`, `Modified`, or `Deleted` state, generate and execute the corresponding SQL statement."

### `context.Entry(entity)`: inspecting or manually controlling an entity's tracked state directly

```csharp
var entry = context.Entry(product);
Console.WriteLine(entry.State); // e.g., EntityState.Modified
Console.WriteLine(entry.Property(p => p.Price).OriginalValue); // the value BEFORE this session's mutation
Console.WriteLine(entry.Property(p => p.Price).CurrentValue);   // the value AFTER
```

Worth knowing this exists for the genuinely useful cases where you need to inspect or override the change tracker's own determination directly — comparing an original and current value explicitly, or forcing a specific entity's state manually (useful, for instance, when re-attaching a previously-detached entity that came from outside the current `DbContext`'s own query, per Section 8).

---

## 4. LINQ to Entities, Revisited for EF Core Specifically

### Everything this series' IEnumerable/IQueryable guide covers applies directly — worth restating the core mechanic briefly

```csharp
var query = context.Products.Where(p => p.Price > 100); // an EXPRESSION TREE, not yet executed
var results = await query.ToListAsync(); // NOW translated to SQL and executed — per that guide's Section 3
```

This series' IEnumerable/IQueryable guide's Sections 3-4 cover, in full depth, exactly why `Where(p => p.Price > 100)` against `IQueryable<T>` compiles the lambda to an expression tree rather than a delegate, and exactly how that lets EF Core translate it into a SQL `WHERE` clause rather than filtering in memory — that entire discussion applies to EF Core specifically and directly; this section covers only what's genuinely EF Core-specific on top of that shared foundation.

### `Include`: EF Core's own operator, extending LINQ specifically for loading related data

```csharp
var order = await context.Orders
    .Include(o => o.Customer)        // EAGER-loads the related Customer, per Section 6
    .Include(o => o.Items)             // and the related Items collection
    .FirstAsync(o => o.Id == 42);
```

`Include` (and `ThenInclude`, for going one level deeper into a nested relationship) is EF Core's own addition to the LINQ vocabulary, specifically for controlling Section 6's loading strategy — it has no equivalent in this series' LINQ guide's standard operator set because it's not really about filtering or projecting data; it's an instruction to EF Core's SQL-generation about which related tables to `JOIN` into the same query.

### Which LINQ expressions genuinely translate, and this series' IEnumerable/IQueryable guide's Section 8 warning applies with real, specific force here

```csharp
// ❌ EF Core cannot translate an arbitrary C# method call into SQL
var results = await context.Products.Where(p => SomeCustomBusinessLogic(p)).ToListAsync(); // throws at runtime

// ✅ EF Core DOES translate a meaningful, growing subset of built-in .NET methods
var results2 = await context.Products.Where(p => p.Name.Contains("Pro") && p.Name.ToUpper() != "TEST").ToListAsync();
```

This series' IEnumerable/IQueryable guide's Section 8 already covers the general problem; worth being concrete here about EF Core's own, specific, and genuinely evolving support — each EF Core version expands the set of translatable `string`/`DateTime`/math methods, but the underlying limitation (arbitrary custom methods can never translate) is permanent and structural, not something a future EF Core version will ever fully close.

---

## 5. Navigation Properties and Relationships

### A navigation property represents a relationship as an ordinary C# reference or collection

```csharp
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; } // the FOREIGN KEY itself
    public Customer Customer { get; set; } = null!; // the NAVIGATION PROPERTY — a reference to the related entity
    public List<OrderItem> Items { get; set; } = new(); // a COLLECTION navigation property (one-to-many)
}
```

This is precisely the "interact with the database using C# objects" promise this topic's own framing describes — rather than manually writing a `JOIN` and mapping the result yourself, `order.Customer` is just an ordinary object reference, and `order.Items` is just an ordinary `List<T>`, both of which EF Core populates according to whatever loading strategy (Section 6) is actually in effect.

### Configuring relationships: convention, data annotations, or the Fluent API

```csharp
// Fluent API, in OnModelCreating — the most explicit, most commonly recommended approach for real applications
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId);
```

EF Core can often infer relationships purely from naming convention (a `CustomerId` property on `Order` alongside a `Customer` navigation property is enough for EF Core to infer the relationship automatically) — but explicit Fluent API configuration in `OnModelCreating` is generally the more maintainable, more explicit choice for any relationship whose cardinality or behavior (cascade delete, required vs. optional) genuinely matters to get exactly right, rather than relying on convention-based inference remaining correct as the model evolves.

---

## 6. Loading Strategies: Eager, Lazy, and Explicit

### Eager loading: `Include`, fetching related data as part of the original query

```csharp
var order = await context.Orders.Include(o => o.Customer).FirstAsync(o => o.Id == 42);
// order.Customer is ALREADY populated — a SINGLE query (with a JOIN) fetched BOTH
```

This is generally the recommended default strategy — you know upfront exactly what related data you need, and EF Core fetches it in the same round trip via a SQL `JOIN`, avoiding Section 7's N+1 problem entirely for the data you explicitly requested.

### Lazy loading: related data fetched automatically, on first access, via a SEPARATE query

```csharp
// requires an explicit opt-in (a NuGet package + virtual navigation properties):
public virtual Customer Customer { get; set; } = null!; // "virtual" enables EF Core's lazy-loading PROXY

var order = await context.Orders.FirstAsync(o => o.Id == 42); // Customer NOT loaded yet
var customerName = order.Customer.Name; // ⚠️ triggers a SEPARATE, ADDITIONAL query RIGHT HERE, on first access
```

Lazy loading is convenient — you never need to remember `Include` for data you might need — but this convenience is precisely the mechanism underneath Section 7's N+1 problem: each access to an unloaded navigation property triggers its own additional round trip, invisibly, at the exact point of access, which is genuinely easy to miss during development (a single test record) and genuinely costly once real, larger datasets are involved.

### Explicit loading: manually, deliberately triggering a load for an already-fetched entity

```csharp
var order = await context.Orders.FirstAsync(o => o.Id == 42);
await context.Entry(order).Reference(o => o.Customer).LoadAsync(); // explicitly, deliberately load it NOW
await context.Entry(order).Collection(o => o.Items).LoadAsync();     // same, for a collection navigation
```

A middle ground — the initial query doesn't eagerly fetch everything, but a specific, later point in the code explicitly and visibly triggers a load for exactly the navigation property needed, at exactly the moment it's needed — worth knowing as an option for cases where eager-loading everything upfront would be wasteful, but lazy loading's invisible, automatic triggering is too risky to rely on.

---

## 7. The N+1 Problem, In Depth

### The concrete anti-pattern: one query to get a list, then one MORE query per item to get its related data

```csharp
var orders = await context.Orders.ToListAsync(); // query #1 — fetches N orders

foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // ⚠️ with LAZY LOADING enabled, this is ONE ADDITIONAL
                                               //  query PER ITERATION — N additional queries total
}
// TOTAL: 1 + N queries, where a SINGLE, JOINED query could have fetched everything in ONE round trip
```

This is the single most common, most costly real-world EF Core performance mistake, and it's worth naming precisely why it's so easy to fall into: lazy loading (Section 6) makes each individual access *look* like an ordinary, cheap property read in the code — `order.Customer.Name` reads exactly like accessing any other object property — while actually triggering a full, separate database round trip, invisibly, every single time.

### Why this is disproportionately damaging at real scale, not just "a bit slower"

```plaintext
Per this series' High-Volume Transaction Processing guide's own framing
  of network round trips as a genuine, first-class cost: N+1 doesn't
  scale LINEARLY in a forgiving way — it turns a query that SHOULD take
  one network round trip into N+1 separate ones, each carrying its OWN
  latency overhead, which at real production data volumes (hundreds or
  thousands of orders, not the handful used in local testing) can turn a
  sub-second operation into one taking many seconds or worse.
```

This is precisely why N+1 is so often the specific, identifiable cause behind "the app works fine in development but is unbearably slow in production" — the bug's severity scales directly with data volume, and development/testing environments routinely have far too little data for the problem to be visible at all.

### The fix: `Include` (eager loading), applied deliberately, based on what the calling code actually needs

```csharp
var orders = await context.Orders.Include(o => o.Customer).ToListAsync(); // ONE query, with a JOIN — no N+1

foreach (var order in orders)
{
    Console.WriteLine(order.Customer.Name); // order.Customer is ALREADY populated — no additional query
}
```

The general remediation is straightforward once identified: eagerly include exactly the related data the subsequent code will actually touch — worth pairing this with genuinely disabling lazy loading entirely in many real applications (not installing the lazy-loading proxy package, and not marking navigation properties `virtual`), which converts N+1 from "a mistake that's easy to make invisibly" into "a mistake that throws an immediate, loud `NullReferenceException` the moment an un-included navigation property is accessed" — a considerably safer failure mode.

---

## 8. Tracking vs. No-Tracking Queries

### Tracking queries (the default): every loaded entity is registered with the change tracker (Section 3)

```csharp
var product = await context.Products.FirstAsync(p => p.Id == 42); // TRACKED, by default
```

This is what makes Section 3's "just mutate the property, `SaveChanges` figures out what changed" workflow possible — but it comes with real, non-trivial overhead: the change tracker maintains an original-value snapshot for every tracked entity, and comparing against that snapshot on every `SaveChanges` call has a genuine cost proportional to how many entities are currently being tracked.

### No-tracking queries: explicitly opting out, for read-only scenarios

```csharp
var products = await context.Products.AsNoTracking().Where(p => p.Price > 100).ToListAsync();
// EF Core does NOT register these entities with the change tracker at all — genuinely cheaper,
// but MUTATING one of these objects and calling SaveChanges() will NOT persist that change
```

For a genuinely read-only query — displaying a list, returning data via an API that will never write it back through this same context — `AsNoTracking()` skips the change-tracking overhead entirely, which is a real, measurable performance improvement for read-heavy workloads; the trade-off is explicit and precise: a no-tracking entity's mutations are simply invisible to `SaveChanges`, since it was never registered to be compared against in the first place.

### `AsNoTrackingWithIdentityResolution`: a middle ground worth knowing about

```csharp
var orders = await context.Orders.Include(o => o.Customer).AsNoTrackingWithIdentityResolution().ToListAsync();
// avoids tracking overhead, but STILL ensures the SAME logical entity (e.g., the same Customer,
// referenced by multiple Orders) is represented by the SAME object instance across the result set
```

Worth knowing this exists for a genuinely specific edge case plain `AsNoTracking()` can otherwise produce: without identity resolution, two different `Order`s referencing the *same* `Customer` in the database would, under plain no-tracking, each get their *own*, separate `Customer` object instance in memory — this variant avoids that duplication while still skipping the full change-tracking overhead.

### Making no-tracking the default for read-heavy contexts

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString);
    options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking); // flips the DEFAULT for this context
});
```

For a `DbContext` (or a whole application) that's predominantly read-only — most reporting or query-heavy services — flipping the tracking default and opting *into* tracking explicitly (via `.AsTracking()`) for the specific queries that genuinely need it is often the more deliberate, more performant overall design than tracking by default and remembering `AsNoTracking()` on every read-only query individually.

---

## 9. SaveChanges: How Persistence Actually Happens

### One call, potentially many statements, all in a single, implicit transaction

```csharp
context.Products.Add(newProduct);      // state: Added
existingProduct.Price = 19.99m;          // state: Modified (detected automatically)
context.Orders.Remove(oldOrder);          // state: Deleted

await context.SaveChangesAsync(); // ONE call — generates INSERT, UPDATE, and DELETE statements,
                                     //  all executed within a SINGLE, IMPLICIT database transaction
```

This is worth stating explicitly: every change accumulated across a `DbContext`'s change tracker since the last `SaveChanges` call is committed together, atomically, in one implicit transaction — if any single statement fails, the entire batch rolls back, leaving the database exactly as it was before the call, which is precisely the all-or-nothing guarantee `SaveChanges`'s "unit of work" framing (Section 1) promises.

### Why `SaveChangesAsync` (not the synchronous version) is the standard, idiomatic choice

```plaintext
Per this series' async/await guide's Section 1: SaveChangesAsync performs
  genuine, I/O-bound database round trips — using the ASYNC version lets
  the calling thread be freed while waiting on that I/O, exactly the
  efficiency benefit that whole guide is built around, rather than
  blocking a thread-pool thread (per this series' Task guide's Section 6)
  for the duration of the database round trip.
```

### Interceptors: hooking into the SaveChanges pipeline itself

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(DbContextEventData eventData, InterceptionResult<int> result)
    {
        // inspect eventData.Context.ChangeTracker.Entries() and add audit fields, log, etc.
        return result;
    }
}
```

Worth knowing this extension point exists for genuinely cross-cutting concerns around persistence itself (automatically stamping `ModifiedAt`/`ModifiedBy` fields on every changed entity, comprehensive audit logging) — a direct architectural parallel to this series' Middleware guide's own cross-cutting concerns, just applied at the data-access layer's single choke point (`SaveChanges`) rather than the HTTP pipeline's.

---

## 10. Optimistic Concurrency

### The problem: two requests loading and modifying the SAME row, concurrently, without either knowing about the other

```plaintext
Per this series' Threading guide's Section 3 and High-Volume Transaction
  Processing guide's Section 7: this is the classic lost-update race
  condition, just occurring across TWO SEPARATE database round trips
  (and often two separate HTTP requests) rather than within a single
  process's memory.
```

### A concurrency token: a column EF Core checks hasn't changed since it was originally read

```csharp
public class Product
{
    public int Id { get; set; }
    public decimal Price { get; set; }
    [Timestamp]
    public byte[] RowVersion { get; set; } = null!; // a CONCURRENCY TOKEN — SQL Server's `rowversion` type
}
```

```csharp
var product = await context.Products.FirstAsync(p => p.Id == 42); // RowVersion captured AS OF this read
product.Price = 29.99m;
await context.SaveChangesAsync(); // the generated UPDATE includes: "WHERE Id = 42 AND RowVersion = <the ORIGINAL value>"
                                     // if ANOTHER process already changed this row, RowVersion no longer matches,
                                     // ZERO rows are affected, and EF Core throws DbUpdateConcurrencyException
```

This is the concrete mechanism underneath optimistic concurrency: rather than locking the row for the duration of the "read, modify, write" sequence (pessimistic locking, which this series' Threading guide's Section 4/6 covers in the general case), EF Core includes the originally-read concurrency token's value in the `UPDATE`'s `WHERE` clause — if the row has genuinely changed since it was read (by anyone), the `WHERE` clause matches zero rows, and EF Core detects this and throws, letting your application code decide how to handle the conflict (retry, merge, or surface it to the user), rather than silently overwriting someone else's concurrent change.

### Handling the conflict: what actually happens when `DbUpdateConcurrencyException` is thrown

```csharp
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var databaseValues = await entry.GetDatabaseValuesAsync(); // the CURRENT, actual database state
    // decide: reload and retry, merge the two sets of changes, or surface a "someone else already
    // changed this" message to the user — per this series' Exception Handling guide's Section 2
    // custom-exception-hierarchy discipline, this is a GENUINE, meaningful domain condition worth
    // its own specific handling, not a generic 500
}
```

---

## 11. Migrations

### Code-first schema evolution: the C# model is the source of truth, migrations bring the database in line with it

```bash
dotnet ef migrations add AddProductDescription  # generates a migration FILE, comparing the current
                                                   #  model against the LAST migration's snapshot
dotnet ef database update                          # APPLIES pending migrations to the actual database
```

This is the standard "code-first" EF Core workflow — you change the C# entity classes (add a property, change a relationship), and `dotnet ef migrations add` generates a migration file containing the specific `Up()`/`Down()` schema-change operations (in C#, itself, generating the corresponding SQL) needed to bring the database schema in line with the model's current shape.

### Migrations in a real deployment pipeline: applying them deliberately, not automatically on every app startup

```plaintext
Auto-applying migrations on EVERY application startup (a common,
  convenient pattern for LOCAL development) is generally considered
  risky for PRODUCTION deployments — multiple application instances
  starting up simultaneously (a typical scaled-out deployment) could
  attempt to apply the SAME migration concurrently, and a schema change
  genuinely warrants its own deliberate, reviewed, and TIMED rollout
  step, separate from application code deployment.
```

Worth knowing this as a genuine, real production concern rather than paranoia — most mature deployment pipelines apply migrations as an explicit, separate, controlled step (often before the new application version is even deployed), rather than letting every scaled-out instance of the application attempt schema changes on its own, independent startup.

---

## 12. Transactions Beyond a Single SaveChanges

### `BeginTransaction`: explicitly grouping MULTIPLE SaveChanges calls into one atomic unit

```csharp
using var transaction = await context.Database.BeginTransactionAsync();
try
{
    context.Orders.Add(newOrder);
    await context.SaveChangesAsync(); // SaveChanges #1

    context.InventoryLog.Add(new InventoryLogEntry(/* ... */));
    await context.SaveChangesAsync(); // SaveChanges #2 — needs to be ATOMIC together with #1

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Section 9 already establishes that a single `SaveChanges` call is inherently atomic — this is for the genuinely different situation where *multiple, separate* `SaveChanges` calls (perhaps because intermediate logic needs the database-generated ID from the first save before proceeding to the second) need to be treated as one all-or-nothing unit together — explicit transaction control is the correct tool for exactly this case.

---

## 13. Raw SQL: When and How to Drop Down

### `FromSqlRaw`/`FromSqlInterpolated`: raw SQL that still returns tracked, mapped entities

```csharp
var products = await context.Products
    .FromSqlInterpolated($"SELECT * FROM Products WHERE Price > {minPrice}") // SAFELY parameterized —
                                                                                 //  NOT string concatenation
    .ToListAsync();
```

`FromSqlInterpolated` (preferred over `FromSqlRaw` with manual parameter objects, specifically because interpolated values are automatically, safely parameterized rather than risking SQL injection) is the right tool when a query's logic is genuinely too complex or performance-sensitive to express cleanly through LINQ, while still wanting the result mapped back into tracked, ordinary entity objects.

### `ExecuteSqlInterpolated`/`ExecuteUpdate`/`ExecuteDelete`: bulk operations bypassing the change tracker entirely

```csharp
// A BULK update, executed DIRECTLY on the database, with NO entities loaded into memory or tracked at all
await context.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(setters => setters.SetProperty(p => p.CategoryId, newCategoryId));
```

For a genuinely bulk operation — updating or deleting potentially many thousands of rows based on a condition — loading every matching entity into memory, mutating each one, and letting the change tracker diff them individually (Section 3) is real, unnecessary overhead; `ExecuteUpdate`/`ExecuteDelete` (added in EF Core 7) translate directly into a single bulk `UPDATE`/`DELETE` SQL statement, with no entities ever loaded, tracked, or compared at all — worth reaching for specifically once a bulk operation's scale makes the ordinary load-mutate-`SaveChanges` pattern genuinely wasteful.

### When reaching for raw SQL is the right call, not a failure of the ORM

```plaintext
A genuinely complex reporting query, a database-specific feature LINQ
  has no vocabulary for, or a query where measured, PROVEN performance
  matters enough to hand-tune the exact SQL — these are all legitimate,
  deliberate reasons to drop to raw SQL for a SPECIFIC query, while the
  rest of the application continues using ordinary LINQ-to-Entities.
  EF Core is a tool for the COMMON case, not a mandate to avoid SQL entirely.
```

---

## 14. Common Pitfalls

| Pitfall | Why it hurts | Better approach |
|---|---|---|
| Registering `DbContext` as Singleton | Not thread-safe; entangles unrelated requests' change-tracked state together, risking genuine race conditions | Register `DbContext` as Scoped (the default with `AddDbContext`), matching one instance to one request (Section 2) |
| Enabling lazy loading without a deliberate reason | Makes an invisible, per-access database round trip look like an ordinary, cheap property read — the direct cause of the N+1 problem | Prefer eager loading (`Include`) deliberately; consider disabling lazy loading entirely so missing includes fail loudly instead of silently (Section 6-7) |
| Iterating a collection and accessing an un-included navigation property per item | Turns one query into N+1 separate round trips, scaling disastrously with real data volume | Eagerly include exactly the related data the subsequent code will touch, in the original query (Section 7) |
| Using tracking queries for genuinely read-only data | Pays real, avoidable change-tracking overhead for entities that will never be mutated or saved | Use `AsNoTracking()` (or a no-tracking default) for read-only query paths (Section 8) |
| Assuming `SaveChanges` alone guarantees atomicity across MULTIPLE separate save calls | Each `SaveChanges` call is its own implicit transaction; several calls in sequence are NOT automatically atomic together | Use explicit `BeginTransaction`/`CommitAsync` when multiple `SaveChanges` calls need to succeed or fail as one unit (Section 12) |
| Ignoring `DbUpdateConcurrencyException` or treating it as a generic error | Silently overwriting another process's concurrent change, or surfacing an unhelpful generic 500 instead of a meaningful conflict response | Handle the exception explicitly, inspecting the actual current database values and deciding how to proceed (Section 10) |
| Auto-applying migrations on every application startup in production | Multiple scaled-out instances starting simultaneously can race to apply the same migration; schema changes deserve their own deliberate rollout step | Apply migrations as an explicit, controlled deployment step, separate from application startup (Section 11) |
| Loading entities into memory just to bulk-update or bulk-delete them based on a condition | Real, unnecessary overhead — loading, tracking, and diffing potentially thousands of entities individually for what's really one bulk operation | Use `ExecuteUpdate`/`ExecuteDelete` for genuinely bulk operations, bypassing the change tracker entirely (Section 13) |

---

## Quick Reference Table

| Concept | Syntax | Purpose |
|---|---|---|
| Unit of work | `DbContext` (registered Scoped) | One coherent session of tracked changes, matched to one request |
| Query entry point | `DbSet<T>` (an `IQueryable<T>`) | LINQ-to-Entities queries, translated to SQL via expression trees |
| Change tracking | Automatic, on tracking queries | Detects mutated properties, generates minimal `UPDATE` statements |
| Eager loading | `.Include(x => x.Related)` | Fetches related data in the same query, avoiding N+1 |
| No-tracking query | `.AsNoTracking()` | Skips change-tracking overhead for read-only data |
| Persistence | `await context.SaveChangesAsync();` | Commits all accumulated changes atomically, in one transaction |
| Optimistic concurrency | `[Timestamp]` / `IsRowVersion()` | Detects and rejects saves based on stale, already-changed data |
| Schema evolution | `dotnet ef migrations add ...` | Generates code-first schema changes from the current model |
| Bulk operations | `.ExecuteUpdateAsync(...)` / `.ExecuteDeleteAsync(...)` | Direct, single-statement bulk changes, bypassing the change tracker |

---

## Conclusion

EF Core's real depth lives in the change tracker — the mechanism that turns ordinary C# property mutation into the correct, minimal set of SQL statements, and turns `DbContext` into a genuine unit-of-work implementation rather than just a connection wrapper — and understanding it precisely is what makes `Scoped` lifetime, tracking vs. no-tracking queries, and optimistic concurrency all make sense as expressions of the same underlying model, rather than as separate, unrelated features to memorize individually. Everything this guide covers about LINQ-to-Entities translation builds directly on this series' IEnumerable/IQueryable guide's expression-tree mechanics, applied specifically to EF Core's own provider and its particular, evolving set of translatable operations.

The N+1 problem deserves the attention this guide gives it because it's genuinely the most common, most consequential real-world EF Core mistake — not because it's exotic, but precisely because it's invisible in the code (an ordinary-looking property access) and invisible in typical development testing (too little data for the cost to be noticeable), surfacing only once real production data volume makes the gap between "one query" and "one query per item" impossible to ignore. Knowing when to reach for eager loading, when no-tracking queries are the right default, and when a genuinely bulk operation warrants bypassing the change tracker entirely via `ExecuteUpdate`/`ExecuteDelete` is what separates EF Core used well from EF Core used as a black box that happens to work until it's tested against real scale.

---

*Found this useful? Feel free to star the repo, open an issue with corrections, or share the innocent-looking-foreach-that-turned-into-thousands-of-queries incident that made the N+1 problem click far better than any performance profiler screenshot ever could.*
