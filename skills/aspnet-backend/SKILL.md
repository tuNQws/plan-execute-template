# Skill: aspnet-backend

> Backend conventions for ASP.NET Core (C#) — layering, transactions, and
> function decomposition.
> Last updated: 2026-08-09 | Confidence: high | Status: stable

---

## Purpose

Layering and code-structure rules for the C# backend: DB access lives in
repositories, shared logic lives in helpers, every write is wrapped in a
transaction, every write and exception is logged structurally for traceability,
long methods are split along semantic seams, and every commit describes the
tasks it completed.

---

## When to use

Load this skill before planning or implementing anything that touches the
ASP.NET Core backend:

- "add an endpoint / API / controller for …"
- "save / update / insert / delete … in the database"
- "add a service / repository / business logic for …"
- Any request touching `Controllers/`, `Services/`, `Repositories/`, `Helpers/`,
  or `Data/`
- Any commit made after finishing a plan

---

## Layer boundaries

```
Controller  ← HTTP only: bind, validate model, call service, map to response
    │         no business logic, no DbContext, no LINQ over entities
    ▼
Service     ← business logic + TRANSACTION BOUNDARY
    │         orchestrates repositories, no raw DbContext queries
    ▼
Repository  ← all DB access: every query and every write
    │         one repository per aggregate/entity, no business rules
    ▼
DbContext / EF Core
```

`Helpers/` sits outside this stack — pure, stateless functions callable from any
layer.

---

## Rule 1 — All DB logic goes in a repository method

**Never** query or write through `DbContext` from a controller or service. If
the operation you need does not exist yet, add a method to the repository
interface and implement it.

### Do

```csharp
// Repositories/Interfaces/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<Order>> GetPendingByCustomerAsync(int customerId, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task UpdateStatusAsync(int orderId, OrderStatus status, CancellationToken ct = default);
}

// Repositories/OrderRepository.cs
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context) => _context = context;

    public async Task<List<Order>> GetPendingByCustomerAsync(
        int customerId, CancellationToken ct = default)
    {
        return await _context.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Pending)
            .ToListAsync(ct);
    }
}
```

### Don't

```csharp
// ✗ Service reaching into DbContext directly
public class OrderService
{
    private readonly AppDbContext _context;   // ✗ wrong dependency

    public async Task<List<Order>> GetPendingAsync(int customerId)
        => await _context.Orders.Where(o => o.CustomerId == customerId).ToListAsync();
}
```

### Rules

- One repository per aggregate root (`OrderRepository`, not `OrderItemRepository`
  when `OrderItem` is only ever loaded through `Order`)
- Repository methods return entities or DTOs — **never** `IQueryable`. Returning
  `IQueryable` leaks the query outside the layer and defeats the boundary.
- Read methods use `.AsNoTracking()` unless the caller will mutate the result
- No business rules in a repository. `UpdateStatusAsync` sets a status; deciding
  *whether* the status may change is the service's job.
- Pass `CancellationToken` through every async method

### Before adding a method

Search first — the method may already exist under a different name:

```bash
grep -rn "interface I.*Repository" --include=*.cs .
grep -rn "Task<.*> Get.*Async" --include=*.cs Repositories/
```

---

## Rule 2 — Shared logic goes in a helper

If a piece of logic is used in more than one place and it does not touch the
database or HTTP, it belongs in `Helpers/`.

```csharp
// Helpers/DateTimeHelper.cs
public static class DateTimeHelper
{
    public static DateTime ToVietnamTime(this DateTime utc) =>
        TimeZoneInfo.ConvertTimeFromUtc(
            utc, TimeZoneInfo.FindSystemTimeZoneById("SE Asia Standard Time"));

    public static bool IsBusinessDay(DateTime date) =>
        date.DayOfWeek is not (DayOfWeek.Saturday or DayOfWeek.Sunday);
}
```

### Rules

- Helpers are `static` classes with **pure** functions: same input → same output,
  no side effects
- No `DbContext`, no `HttpContext`, no `ILogger`, no file or network I/O in a
  helper. If it needs any of those, it is a service, not a helper.
- One helper class per concern (`StringHelper`, `DateTimeHelper`,
  `ValidationHelper`) — never a single `Utils` dumping ground
- Every helper method is directly unit-testable with no mocks

### When *not* to extract a helper

**Wait for the third caller.** A "helper" with one caller is a premature
abstraction — inline it. With two callers, judge by whether the logic is a named
concept in the domain. With three, extract it.

This is deliberately stricter than "don't repeat yourself": a wrong abstraction
is more expensive to unwind than duplicated code.

---

## Rule 3 — Every insert / update / delete is wrapped in a transaction

### The transaction boundary is the SERVICE, not the repository

One business operation = one transaction, even when it spans several repository
calls. Putting `BeginTransactionAsync` inside repository methods creates nested
transactions and makes multi-step operations non-atomic.

```csharp
// Services/OrderService.cs
public async Task<Result> CancelOrderAsync(int orderId, CancellationToken ct = default)
{
    await using var transaction = await _context.Database.BeginTransactionAsync(ct);
    try
    {
        await _orderRepository.UpdateStatusAsync(orderId, OrderStatus.Cancelled, ct);
        await _inventoryRepository.RestoreStockAsync(orderId, ct);
        await _auditRepository.AddAsync(AuditEntry.ForCancel(orderId), ct);

        await _unitOfWork.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);

        return Result.Ok();
    }
    catch
    {
        await transaction.RollbackAsync(ct);
        throw;
    }
}
```

`await using` rolls back automatically if `CommitAsync` is never reached, so the
explicit `RollbackAsync` is belt-and-braces — keep it, it documents intent.

### Rules

- `await using`, never plain `using`, for `IDbContextTransaction`
- Commit **once**, at the end, after all writes succeed
- Never `return` from inside the `try` before `CommitAsync`
- Never swallow the exception — rethrow so the caller sees the failure. Log at
  the boundary, not here.
- Call `SaveChangesAsync` once per transaction where possible, not per repository
  method

### The single exception

A single-statement write with no other work in the same operation does not need
an explicit transaction — EF Core's `SaveChangesAsync` is already atomic for one
call. Do not add ceremony where nothing can interleave:

```csharp
// Fine — one SaveChangesAsync, nothing else in the operation
public async Task MarkAsReadAsync(int notificationId, CancellationToken ct = default)
{
    await _notificationRepository.MarkReadAsync(notificationId, ct);
    await _unitOfWork.SaveChangesAsync(ct);
}
```

The moment a second write joins it, wrap both.

### Concurrency

For read-then-write operations where two requests could race, take the row lock
inside the transaction rather than relying on it implicitly:

```csharp
var order = await _orderRepository.GetByIdForUpdateAsync(orderId, ct); // SELECT ... FOR UPDATE
```

---

## Rule 4 — Log every write and every exception, structurally

Logs exist to answer one question after the fact: *what happened to this record,
when, at whose request?* That only works if logs are **structured** — named
fields a log query can filter on, not sentences a human has to read.

### Structured, not interpolated

```csharp
// ✓ Message template with named placeholders — each becomes a queryable field
_logger.LogInformation(
    "Order {OrderId} status changed to {NewStatus} by user {UserId}",
    orderId, newStatus, currentUserId);

// ✗ String interpolation — collapses to one opaque string, nothing is queryable
_logger.LogInformation($"Order {orderId} status changed to {newStatus}");
```

Both print the same text. Only the first lets you run
`OrderId = 4711` in Seq / Application Insights / CloudWatch. **Never use `$""`
inside a logging call.** This is the single most common mistake here and it is
invisible until the day you need to investigate something.

Placeholder names are `PascalCase` and stable across the codebase — always
`{OrderId}`, never `{orderId}` in one place and `{id}` in another. Inconsistent
names mean no single query can follow one entity across layers.

### Correlation — tie every log line of one request together

Open a log scope at the service entry point. Every line logged inside the scope
inherits its fields, including lines from code you did not write.

```csharp
// Services/OrderService.cs
public async Task<Result> CancelOrderAsync(int orderId, CancellationToken ct = default)
{
    using var scope = _logger.BeginScope(new Dictionary<string, object>
    {
        ["Operation"]     = nameof(CancelOrderAsync),
        ["EntityType"]    = nameof(Order),
        ["EntityId"]      = orderId,
        ["UserId"]        = _currentUser.Id,
        ["CorrelationId"] = _correlationId,
    });

    _logger.LogInformation("Cancelling order {OrderId}", orderId);

    await using var transaction = await _context.Database.BeginTransactionAsync(ct);
    try
    {
        await _orderRepository.UpdateStatusAsync(orderId, OrderStatus.Cancelled, ct);
        await _inventoryRepository.RestoreStockAsync(orderId, ct);

        await _unitOfWork.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);

        _logger.LogInformation(
            "Order {OrderId} cancelled, {ItemCount} items restocked",
            orderId, order.Items.Count);

        return Result.Ok();
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync(ct);

        _logger.LogError(ex,
            "Failed to cancel order {OrderId} — transaction rolled back", orderId);

        throw;
    }
}
```

`CorrelationId` comes from the incoming request (`X-Correlation-Id` header) or is
generated per request in middleware and stored in a scoped `ICorrelationContext`.
Return it in the error response so a user's screenshot is enough to find the
exact log lines.

### What to log on insert / update

| When | Level | Content |
|---|---|---|
| Before the write | `Information` | Operation + entity type + id (or "new") + actor |
| After commit succeeds | `Information` | Entity id (the **generated** id, for inserts) + what changed |
| Validation rejected the write | `Warning` | Which rule failed and the offending value |
| Write threw | `Error` | The exception object + entity id + "rolled back" |

Log **after** commit, not before, for the success line — a line saying "order
created" for a transaction that later rolled back is worse than no line at all.

For updates, log which fields changed, not the whole entity:

```csharp
_logger.LogInformation(
    "Order {OrderId} updated: {ChangedFields}",
    orderId, string.Join(",", changedFields));
```

### Exceptions — pass the exception object, log once

```csharp
// ✓ Exception as the first argument — stack trace, inner exceptions, type all preserved
_logger.LogError(ex, "Failed to cancel order {OrderId}", orderId);

// ✗ Stack trace destroyed — you get one line of text and no idea where it came from
_logger.LogError("Failed to cancel order {OrderId}: {Error}", orderId, ex.Message);
```

**Log an exception once.** The rule: log where you *handle* it, rethrow where you
don't. Logging at every layer on the way up produces three entries for one fault
and makes the error count meaningless.

- **Service** — logs the `Error` with entity context, then rethrows. This is the
  layer that knows *which record* failed, so this is where the context lives.
- **Global exception middleware** — catches what nobody handled, logs anything
  not yet logged, converts it to a response. Never let a raw exception reach the
  client.
- **Controller** — no `try/catch` at all. If you find one, delete it and let the
  middleware do its job.

```csharp
// Middleware/ExceptionHandlingMiddleware.cs
public async Task InvokeAsync(HttpContext context, RequestDelegate next)
{
    try
    {
        await next(context);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex,
            "Unhandled exception on {Method} {Path} — correlation {CorrelationId}",
            context.Request.Method, context.Request.Path, _correlationId);

        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Title  = "An unexpected error occurred",
            Status = StatusCodes.Status500InternalServerError,
            // The client gets the id, never the message or stack trace
            Extensions = { ["correlationId"] = _correlationId },
        });
    }
}
```

### Log levels

| Level | Use for | Example |
|---|---|---|
| `Trace` / `Debug` | Local diagnosis only — never enabled in production | Query parameters |
| `Information` | Business events worth auditing | Order created, status changed |
| `Warning` | Expected failures — the caller did something invalid | Validation failed, not found |
| `Error` | Unexpected failure of one operation | DB write threw, external API 500 |
| `Critical` | The process cannot continue | Cannot reach the database at all |

A failed *request* is `Warning`, not `Error`. Reserve `Error` for faults that
need a human — otherwise alerts get ignored, which is the same as having none.

### Never log

- Passwords, tokens, API keys, connection strings, `Authorization` headers
- Full card numbers, national IDs, or any regulated PII
- Whole request/response bodies on write endpoints — log the ids, not the payload
- Anything inside a loop over rows: log the aggregate (`{RowCount}`), not each row

Log a stable reference (`{UserId}`) rather than the sensitive value itself.

### Where logging goes

- **Service** — yes. This is the layer with both business meaning and entity ids.
- **Repository** — no. A repository logging every query drowns the signal; EF
  Core already logs SQL at `Debug`.
- **Controller** — no, except for a one-line entry log if the project has that
  convention. Request logging belongs in middleware.
- **Helper** — never. Helpers are pure; a helper that logs is a service.

---

## Rule 5 — Split long methods along semantic seams

A method does **one** thing. When it does several, split it — but split by
meaning, not by line count.

### Split when any of these is true

- The method exceeds ~40 lines
- It has more than 2 levels of nesting
- You need a comment to label a section (`// validate input`, `// now send mail`)
- Its name needs "and" to be accurate (`ValidateAndSaveOrder`)
- You cannot describe what it does in one sentence

### Split along these seams

Most service methods decompose into the same four phases:

```
validate → load → transform/decide → persist
```

```csharp
// ✗ Before: one method, four responsibilities, 90 lines
public async Task<Result> ProcessOrderAsync(OrderRequest request) { /* 90 lines */ }

// ✓ After: each method covers exactly one concern
public async Task<Result> ProcessOrderAsync(OrderRequest request, CancellationToken ct = default)
{
    var validation = ValidateRequest(request);
    if (!validation.IsValid) return Result.Fail(validation.Errors);

    var customer = await _customerRepository.GetByIdAsync(request.CustomerId, ct);
    if (customer is null) return Result.Fail("Customer not found");

    var order = BuildOrder(request, customer);

    await using var transaction = await _context.Database.BeginTransactionAsync(ct);
    try
    {
        await PersistOrderAsync(order, ct);
        await transaction.CommitAsync(ct);
        return Result.Ok(order.Id);
    }
    catch
    {
        await transaction.RollbackAsync(ct);
        throw;
    }
}

private static ValidationResult ValidateRequest(OrderRequest request) { }
private static Order BuildOrder(OrderRequest request, Customer customer) { }
private async Task PersistOrderAsync(Order order, CancellationToken ct) { }
```

Each extracted method has a name that says what it does, and is testable alone.
`ValidateRequest` and `BuildOrder` are `static` — no dependencies, pure.

### Do NOT split like this

```csharp
// ✗ Arbitrary slicing — the names carry no meaning and the pieces
//   cannot be understood or tested independently
private void ProcessOrderPart1() { }
private void ProcessOrderPart2() { }
private void ProcessOrderStep3() { }
```

If an extracted method needs 5 parameters and out-variables to work, the seam is
wrong — you cut through the middle of one logic instead of between two. Put it
back and cut elsewhere.

### Test

Read the extracted method's name, then read its body. If the body does anything
the name does not promise, the split is wrong.

---

## Rule 6 — Commit messages describe the tasks completed

Every commit body lists what was actually done, one line per task.

```
<type>(<scope>): <short summary in imperative mood>

- <task 1 — what changed and where>
- <task 2>
- <task 3>

Implements .plans/YYYY-MM-DD-feature-name/
```

### Example

```
feat(order): add order cancellation endpoint

- Add IOrderRepository.UpdateStatusAsync and its EF Core implementation
- Add IInventoryRepository.RestoreStockAsync to return reserved stock
- Add OrderService.CancelOrderAsync wrapping both writes in one transaction
- Extract OrderValidationHelper.CanBeCancelled for the status-check rule
- Add POST /api/orders/{id}/cancel in OrdersController
- Add integration test covering cancel-then-restock in OrderServiceTests

Implements .plans/2026-08-09-order-cancel/
```

### Rules

- `<type>`: `feat` | `fix` | `refactor` | `test` | `chore` | `docs`
- `<scope>`: the module or feature area (`order`, `auth`, `payment`)
- Summary line ≤ 72 characters, imperative mood ("add", not "added")
- One bullet per completed task — describe **what changed**, not "updated files"
- Reference the plan folder on the last line when the work came from a plan
- Do not commit while any task in `tasks.md` is `[ ]`, `[~]`, or `[!]`

### Don't

```
✗ update code
✗ fix bug
✗ WIP
✗ - modified OrderService.cs        ← names the file, not the change
```

---

## Files involved

| Path | Role |
|---|---|
| `Controllers/` | HTTP endpoints — bind, validate, delegate, map response |
| `Services/` | Business logic and the transaction boundary |
| `Repositories/Interfaces/` | Repository contracts (`IOrderRepository`) |
| `Repositories/` | EF Core implementations — all DB access |
| `Helpers/` | Pure static utilities, no I/O |
| `Middleware/` | Correlation-id and global exception handling |
| `Data/AppDbContext.cs` | EF Core context and entity configuration |
| `Models/` or `Entities/` | Domain entities |
| `DTOs/` | Request/response contracts |

> Adjust these paths to the actual project layout — check `CLAUDE.md` →
> "Architecture" first. The layering rules hold regardless of folder names.

---

## Constraints and gotchas

- **Register new repositories in DI.** A new `IOrderRepository` is useless until
  `builder.Services.AddScoped<IOrderRepository, OrderRepository>();` is added in
  `Program.cs`. Forgetting this fails at runtime, not compile time.
- **`DbContext` is scoped, not thread-safe.** Never run two `await` DB calls
  concurrently on the same context (`Task.WhenAll` over repository calls will
  throw).
- **Migrations are a separate, explicit step.** Changing an entity requires
  `dotnet ef migrations add <Name>` — the plan must include it as its own task.
- **`AsNoTracking()` results cannot be updated.** Loading with it and then trying
  to save changes silently does nothing.
- **Nullable reference types**: match the project's `<Nullable>` setting. If
  enabled, return `Order?` from `GetByIdAsync` and handle null at the caller.
- **Interpolated strings silently break structured logging.** `$"..."` compiles,
  runs, and prints correctly — the fields are simply gone when you need to query
  them. Nothing warns you. Enable the `CA2254` analyzer to catch it at build.
- **`ILogger<T>` is injected per class**, with `T` being that class. The `T`
  becomes `SourceContext` in the log — using another class's logger makes the
  entry point to the wrong file.
- **Log scopes need a provider that supports them.** Plain console logging
  ignores `BeginScope` unless `IncludeScopes = true` is set. Serilog and
  Application Insights honour scopes by default.

---

## Step-by-step — adding a write operation

1. **Check `CLAUDE.md`** → "Architecture" and "Key files" for the actual layout
2. **Search for an existing repository method** before adding a new one
   (`grep -rn "Async" --include=*.cs Repositories/Interfaces/`)
3. **Add the method to the repository interface**, then implement it
4. **Register the repository in `Program.cs`** if it is new
5. **Add the service method** — wrap all writes in one transaction
6. **Add logging** — open a `BeginScope` with entity type/id/user/correlation,
   log `Information` after commit and `LogError(ex, …)` in the catch
7. **Extract to a helper** only if the logic now has 3+ callers
8. **Split the service method** if it passes ~40 lines or mixes phases
9. **Add the controller action** — no logic beyond bind → call → map, no
   `try/catch`
10. **Add or update the migration** if any entity changed
11. **Write the test** before marking the task `[x]`
12. **Commit** with the task list in the body

---

## Tests and verification

```bash
# Build — must be clean, warnings included
dotnet build --no-incremental

# Unit + integration tests
dotnet test

# Single test class while iterating
dotnet test --filter "FullyQualifiedName~OrderServiceTests"

# Verify migrations are consistent with the model
dotnet ef migrations has-pending-model-changes
```

**Pass:** `dotnet build` reports `0 Error(s)`, `dotnet test` reports
`Failed: 0`, and `has-pending-model-changes` reports no pending changes.

**Fail:** Any compile error, any failing test, or a pending model change that
has no migration. Stop, mark the task `[!]`, and report the exact output.

### Layering self-check before marking a task `[x]`

```bash
# No DbContext outside Repositories/ and Data/
grep -rn "AppDbContext" --include=*.cs Controllers/ Services/ Helpers/

# No IQueryable escaping the repository layer
grep -rn "IQueryable" --include=*.cs Repositories/Interfaces/

# Every service write path opens a transaction
grep -rn "SaveChangesAsync" --include=*.cs Services/

# No interpolated strings in logging calls — must return nothing
grep -rnE "Log(Information|Warning|Error|Debug|Critical)\(\\\$\"" --include=*.cs .

# Every LogError passes the exception object, not ex.Message
grep -rn "LogError" --include=*.cs . | grep -v "LogError(ex"

# No try/catch in controllers
grep -rn "catch" --include=*.cs Controllers/
```

The first two should return nothing — except `AppDbContext` in a service that
owns a transaction boundary, which is expected. Every hit from the third should
sit inside a `BeginTransactionAsync` block or be a documented single-write
operation. The last three must return nothing.
