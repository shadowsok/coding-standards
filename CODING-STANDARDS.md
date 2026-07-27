# Personal Coding Standards

**Author:** Joseph Perez  
**Purpose:** General-purpose coding standards for C#, SQL, and software development  
**Basis:** DISA STIGs (Application Security and Development V5R3), industry best practices, personal preferences

---

## CORE PRINCIPLES

1. **Readability over cleverness** — Code is read 10x more than it's written
2. **Explicit over implicit** — No magic, no shortcuts that obscure intent
3. **Testable by default** — Pure functions, dependency injection, avoid static state
4. **Security first** — STIGs compliance, input validation, least privilege
5. **Flexible within reason** — These are guidelines, not laws. Use judgment.

---

## C# STANDARDS

### Naming Conventions

**PascalCase:**
- Classes, interfaces, enums, methods, properties, events
```csharp
public class OrderService
public interface IOrderRepository
public enum OrderStatus
public void ProcessOrder()
public decimal TotalAmount { get; set; }
```

**camelCase:**
- Local variables, method parameters, private fields
- **EXCEPTION:** Use PascalCase if it''s type outlining (e.g., `Order order`)
```csharp
var totalAmount = 0m;
public void Process(decimal amount)
private readonly ILogger _logger;

// Type outlining OK:
Order order = await GetOrderAsync(orderId);
```

**Descriptive names, no abbreviations** (unless industry-standard):
- ✅ `customerId`, `orderAmount`, `orderRepository`
- ❌ `custId`, `ordAmt`, `ordRepo`

**Interface prefix:**
- All interfaces start with `I`: `IOrderRepository`, `IOrderService`

---

### Code Style

**Single-line blocks:**
- ✅ **Allowed** if short and contained
- ❌ **Multiple statements** must use braces
```csharp
// ✅ GOOD (single short statement)
if (amount > limit){ LogWarning(); }

// ✅ GOOD (multi-statement)
if (amount > limit){ 
    LogWarning(); 
    return false;
}

// ❌ BAD (multi-statement without braces)
if (amount > limit) LogWarning(); return false;
```

**Compound assignment:**
- ✅ **Allowed** for simple operations
```csharp
// ✅ GOOD
total += amount;
count++;

// ✅ Also fine if you prefer explicit
total = total + amount;
count = count + 1;
```

**Null handling:**
```csharp
// ✅ GOOD (explicit null check)
if (order == null) throw new ArgumentNullException(nameof(order));

// ✅ GOOD (null-coalescing)
var name = customer.Name ?? "Unknown";

// ✅ GOOD (null-conditional)
var firstItem = order?.Items?.FirstOrDefault();
```

---

### Security (DISA STIGs)

**Input Validation (APSC-DV-002560):**
```csharp
// ✅ Validate ALL inputs
public async Task<Order> GetOrderAsync(long orderId)
{
    if (orderId <= 0) throw new ArgumentException("Invalid order ID", nameof(orderId));
    // ...
}

// ✅ Use parameterized queries (EF Core does this automatically)
var order = await _context.Orders
    .FromSqlInterpolated($"EXEC GetOrderById {orderId}")
    .FirstOrDefaultAsync();
```

**Error Handling (APSC-DV-002330):**
```csharp
// ✅ Sanitize error messages
try
{
    await ProcessOrderAsync(orderId);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Order processing failed for order {OrderId}", orderId);
    throw new ApplicationException("Order processing failed"); // Don't leak internal details
}
```

**Logging (APSC-DV-002540):**
```csharp
// ✅ Log security events
_logger.LogWarning("Unauthorized access attempt for order {OrderId} by user {UserId}", 
    orderId, userId);

// ✅ Never log sensitive data
// ❌ _logger.LogDebug("Processing credit card {CardNumber}", cardNumber);
```

---

## SQL SERVER STANDARDS

### Security

**Always use parameterized queries:**
```sql
-- ✅ GOOD (stored procedure with parameters)
CREATE PROCEDURE GetOrderById
    @OrderId BIGINT
AS
BEGIN
    SELECT * FROM Orders WHERE OrderId = @OrderId;
END
```

**Least privilege:**
- Web app user: EXECUTE only on stored procedures
- No direct table access
- Use contained database users

**Input validation in SQL:**
```sql
-- ✅ Validate parameters
IF @OrderId IS NULL OR @OrderId <= 0
BEGIN
    RAISERROR(''Invalid order ID'', 16, 1);
    RETURN;
END
```

---

### Naming Conventions

**Tables:** PascalCase singular  
- `Order`, `Customer`, `OrderItem`

**Columns:** PascalCase  
- `OrderId`, `CustomerName`, `OrderDate`

**Primary keys:** `{TableName}Id`  
- `OrderId`, `CustomerId`

**Foreign keys:** `{ReferencedTable}Id`  
- `CustomerId` in `Order` table

**Indexes:** `IX_{Table}_{Columns}`  
- `IX_Order_CustomerId`, `IX_Order_OrderDate`

**Stored procedures:** Verb + Noun  
- `GetOrderById`, `CreateOrder`, `UpdateOrderStatus`

---

### Query Best Practices

**Use stored procedures for complex logic:**
```sql
-- ✅ Encapsulate business logic
CREATE PROCEDURE CalculateOrderTotal
    @OrderId BIGINT
AS
BEGIN
    SELECT 
        SUM(Quantity * UnitPrice) AS Total
    FROM OrderItems
    WHERE OrderId = @OrderId;
END
```

**Avoid SELECT *:**
```sql
-- ❌ BAD
SELECT * FROM Orders;

-- ✅ GOOD
SELECT OrderId, CustomerName, OrderDate, TotalAmount FROM Orders;
```

**Use CTEs for readability:**
```sql
-- ✅ GOOD (clear, testable)
WITH RecentOrders AS (
    SELECT OrderId, CustomerId, OrderDate
    FROM Orders
    WHERE OrderDate >= DATEADD(day, -30, GETDATE())
)
SELECT * FROM RecentOrders;
```

---

## TESTING STANDARDS

### Unit Tests

**Naming:** `MethodName_Scenario_ExpectedResult`
```csharp
[Fact]
public void CalculateTotal_WithMultipleItems_ReturnsCorrectSum()
{
    // Arrange
    var items = new[] { 10m, 20m, 30m };
    
    // Act
    var total = CalculateTotal(items);
    
    // Assert
    Assert.Equal(60m, total);
}
```

**Test structure:** Arrange, Act, Assert

**Mock external dependencies:**
```csharp
var mockRepo = new Mock<IOrderRepository>();
mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<long>()))
    .ReturnsAsync(new Order { OrderId = 1 });
```

---

## DOCUMENTATION

**XML comments for public APIs:**
```csharp
/// <summary>
/// Calculates the total amount for an order.
/// </summary>
/// <param name="orderId">The order ID.</param>
/// <returns>The total amount.</returns>
/// <exception cref="ArgumentException">Thrown when order ID is invalid.</exception>
public async Task<decimal> CalculateTotalAsync(long orderId)
```

**README.md in every repo:**
- What the project does
- How to run it locally
- How to run tests
- Deployment instructions

---

## GIT CONVENTIONS

**Branch naming:**
- `feature/description`
- `bugfix/description`
- `hotfix/description`

**Commit messages:**
- First line: imperative mood, <50 chars
- "Add feature" not "Added feature" or "Adds feature"
- Blank line, then detail if needed

**Example:**
```
Add order total calculation

- Implements stored procedure CalculateOrderTotal
- Adds unit tests for edge cases (null items, negative quantities)
- Updates OrderService to call new procedure
```

---

## TOOLS

**Linting:** EditorConfig + StyleCop  
**Static Analysis:** SonarQube (STIG compliance rules)  
**Testing:** xUnit, Moq, FluentAssertions  
**SQL:** SQL Server Data Tools (SSDT), Redgate SQL Prompt

---

## NOTES

- **This is a living document.** Update it as you learn and refine your preferences.
- **Guidelines, not laws.** Use judgment. Break rules if it makes the code clearer.
- **Team consensus matters.** When working with a team, align on shared standards.

---

**Last Updated:** 2026-07-27
