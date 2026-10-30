# Coding Standards -- Joseph Perez

**What this is:** My personal coding standard. The rules below are the same ones I use as the team standard on my current project (kept in sync on purpose, so there is one set of rules, not two). Sections that are project-specific say so; skip them on other work.  
**Basis:** DISA STIGs (Application Security and Development V5R3), C# and SQL Server best practices, 20+ years of production experience, and habits checked against my own code from 2008 onward.

---
## CORE PRINCIPLES

1. **Readability over cleverness** — Code is read 10x more than it's written
2. **Explicit over implicit** — No magic, no shortcuts that obscure intent
3. **Testable by default** — Pure functions, dependency injection, avoid static state
4. **Security first** — STIGs compliance, input validation, least privilege
5. **Port, don't rewrite** — when replacing proven legacy logic, port it faithfully and document every deviation
6. **Flexible within reason** — These are guidelines, not laws. Use judgment.

---

## C# STANDARDS

### Naming Conventions

**PascalCase:**
- Classes, interfaces, enums, methods, properties, events
```csharp
public class PermitRecalcService
public interface IDoseFactorRepo
public enum PathwayType
public void CalculateLiquidDose()
public decimal TotalDose { get; set; }
```

**camelCase:**
- Local variables, method parameters, private fields
- **EXCEPTION:** Use PascalCase if it's type outlining (e.g., `Permit permit`)
```csharp
decimal totalDose = 0m;
public void Calculate(decimal releaseAmount)
private readonly ILogger _logger;

// Type outlining OK:
Permit permit = await GetPermitAsync(permitId);
```

**Descriptive names, no abbreviations** (unless industry-standard):
- ✅ `dischargePointId`, `nuclideConcentration`, `doseFactorRepo`
- ❌ `dpId`, `nucConc`, `dfRepo`
- ✅ OK: `MPC` (Maximum Permissible Concentration — industry term)

**Interface prefix:**
- All interfaces start with `I`: `IDoseFactorRepo`, `IPermitRecalcService`

---

### Code Style

**LINQ in production code (tightened 2026-09-15):**
- ❌ **Avoid LINQ in new code** — classic loops and `List.Sort` read better and are easier to step through and validate. This now applies generally, not just to calculation code.
- ✅ EF Core query operators inside repository methods are fine (that is how EF works); the rule is about in-memory LINQ chains (`.Select`/`.Where`/`.OrderBy`/`.Single` on collections).
- ❌ **Never access `_context` directly** — use stored procedures or repository methods
```csharp
// ❌ BAD (direct context access in production)
var permit = await _context.Permits
    .Where(p => p.PermitId == permitId)
    .FirstOrDefaultAsync();

// ✅ GOOD (repository method wraps SQL function)
Permit permit = await _permitRepo.GetByIdAsync(permitId);

// In repository:
public async Task<Permit> GetByIdAsync(long permitId)
{
    return await _context.Permits
        .FromSqlInterpolated($"EXEC GetPermitById {permitId}")
        .FirstOrDefaultAsync();
}
```

**Single-line blocks:**
- ✅ **Allowed** if short and contained (Joseph's preference)
- ❌ **Multiple statements** must use braces
```csharp
// ✅ GOOD (single short statement)
if (dose > limit){ LogWarning(); }

// ✅ GOOD (multi-statement)
if (dose > limit){ 
    LogWarning(); 
    return false;
}

// ❌ BAD (multi-statement without braces)
if (dose > limit) LogWarning(); return false;

// ✅ GOOD (full block if complex)
if (dose > limit)
{
    LogWarning();
    return false;
}
```

**Shorthand operators:**
- ✅ **Allowed** if simple and not complex (Joseph's preference)
- ❌ **Avoid** in calculation code if VB6 port needs explicit validation
```csharp
// ✅ GOOD (simple accumulation)
totalDose += dose;
counter++;

// ❌ BAD (calculation code - hard to validate against VB6)
dose += concentration * factor * dilution;

// ✅ GOOD (calculation code - explicit)
dose = dose + (concentration * factor * dilution);
```

**Explicit types (tightened 2026-09-15):**
- ✅ **Type-explicit everywhere in new code** — if it can have a written type, write the type. No `var`.
- Pre-existing lines you are not otherwise touching keep their style (do not churn old code for style alone).
```csharp
// ❌ BAD
var dose = CalculateDose();
var results = new List<DoseCalculationResult>();

// ✅ GOOD
decimal dose = CalculateDose();
List<DoseCalculationResult> results = new List<DoseCalculationResult>();
Permit permit = await _repo.GetPermitAsync(permitId);
```

**One statement per line:**
```csharp
// ❌ BAD
if (dose > limit) LogWarning(); return false;

// ✅ GOOD
if (dose > limit){ LogWarning(); return false; }

// ✅ ALSO GOOD
if (dose > limit)
{
    LogWarning();
    return false;
}
```

**No ternary returns, no expression-bodied logic (added 2026-09-15):**
- ❌ Ternary conditionals in return statements; expression-bodied members (`=>`) that contain logic
- ✅ Classic method bodies with if/return — readable and steppable
```csharp
// ❌ BAD
public string Pattern(bool useJson) => useJson ? "*.json" : "*.xlsx";

// ✅ GOOD
public string Pattern(bool useJson)
{
    if (useJson)
    {
        return "*.json";
    }
    return "*.xlsx";
}
```

**Explicit field initialization (added 2026-09-15):**
- Value-type fields get an explicit initial value even when it matches the language default — the intent is visible.
```csharp
// ❌ BAD
private readonly bool _enableJsonDefinitions;

// ✅ GOOD
private readonly bool _enableJsonDefinitions = false;
```

**Properties: backing-field style (added 2026-09-15):**
- Prefer a private backing field with an explicit property over auto-properties in classes that carry state.
```csharp
private bool _enableJsonDefinitions = false;

public bool EnableJsonDefinitions
{
    get { return _enableJsonDefinitions; }
    set { _enableJsonDefinitions = value; }
}
```

**Properties: data carriers are the exception (added 2026-10-30):**
- DTOs, message records, and options classes that only carry values may use auto-properties. The backing-field rule is for classes with behavior or state that changes.
```csharp
public class ProtocolDefinitionOptions
{
    public bool EnableJsonDefinitions { get; set; } = false;
}
```

**Named constants, no magic numbers or magic strings (added 2026-10-30):**
- Sizes, limits, timeouts, status values, and mode codes get a named `const`, an `enum`, or a settings entry. No inline status-string chains.
- A message or value that both the code and its tests depend on is defined once (a `public const` on the owning class) and referenced from both places.
```csharp
// BAD
if (status != "Collection In Process" && status != "Donor On Site") { return; }
buffer = new byte[215];

// GOOD
private const int MaxResponseLength = 215;
public const string UnsupportedFileMessage = "Only .xlsx definition files are accepted.";
if (status != CollectionStatus.InProcess && status != CollectionStatus.DonorOnSite) { return; }
```

**Options: bind configuration once (added 2026-10-30):**
- Read a settings section once at startup into a typed options class registered as a singleton. Services take the options class, not `IConfiguration`. One bind point means one place to validate and one place to log a bad value.

**Class organization (added 2026-09-15):**
- Group class members into labeled sections (Fields / Properties / Functions) with banner comments when a class has more than a couple of members.

**Signature stability (added 2026-09-15):**
- When extending an existing method or constructor, add new parameters with defaults so no existing call site breaks.
```csharp
// existing callers keep working
public InstrumentFileEventHostedService(IDbContextFactory<Db> factory, ILogger<T> logger,
    IOptions<ServiceOptions>? options = null)
```

**File encoding (added 2026-09-15):**
- When editing an existing file, match its existing encoding convention exactly (BOM or no BOM) — byte-check after tooling rewrites. New files follow the repository majority.

---

### Methods

**Keep methods short** (< 50 lines ideal, < 100 max):
- Extract helper methods if logic grows
- **Exception:** Ported VB6 methods can stay long if breaking them obscures the port
- **Split tripwire (added 2026-10-30):** a method with two or more `try` blocks, or two methods that are near copies of each other, gets split or shared before review.

**Rewrites: keep the replaced block until the replacement is proven (added 2026-10-30):**
- When rewriting existing logic, comment out the old block with a one-line reason instead of deleting it, so the reviewer can compare. Delete it in a follow-up commit once the new code is verified. Never leave it longer than the next release.

**Pure functions when possible:**
```csharp
// ✅ GOOD (pure — no side effects)
public decimal CalculateDose(decimal concentration, decimal factor)
{
    return concentration * factor;
}
```

**Dependency injection, no `new` for services:**
```csharp
// ❌ BAD
public class PermitService
{
    public void Recalc()
    {
        var repo = new DoseFactorRepo(); // hard dependency
    }
}

// ✅ GOOD
public class PermitService
{
    private readonly IDoseFactorRepo _repo;
    public PermitService(IDoseFactorRepo repo)
    {
        _repo = repo;
    }
}
```

**Return early, avoid deep nesting:**
```csharp
// ❌ BAD
public void Process(Permit permit)
{
    if (permit != null)
    {
        if (permit.Status == "Active")
        {
            if (permit.ReleaseAmount > 0)
            {
                // logic 10 levels deep
            }
        }
    }
}

// ✅ GOOD
public void Process(Permit permit)
{
    if (permit == null) return;
    if (permit.Status != "Active") return;
    if (permit.ReleaseAmount <= 0) return;

    // logic at top level
}
```

---

### Error Handling

**Validate inputs early:**
```csharp
public decimal CalculateDose(decimal concentration, decimal factor)
{
    if (concentration < 0)
        throw new ArgumentException("Concentration cannot be negative", nameof(concentration));
    if (factor < 0)
        throw new ArgumentException("Dose factor cannot be negative", nameof(factor));

    return concentration * factor;
}
```

**Use specific exceptions:**
- ✅ `ArgumentNullException`, `ArgumentException`, `InvalidOperationException`
- ❌ Generic `Exception`

**Log exceptions with context:**
```csharp
try
{
    decimal dose = CalculateDose(concentration, factor);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Dose calculation failed for permit {PermitId}, nuclide {NuclideId}", 
                     permitId, nuclideId);
    throw; // re-throw after logging
}
```

**Don't swallow exceptions:**
```csharp
// ❌ BAD
try { ... } catch { } // silent failure

// ✅ GOOD
try { ... } catch (Exception ex) { _logger.LogError(ex, "..."); throw; }
```

**Fail to the current acceptable behavior (added 2026-09-15):**
- Only add error handling where something can actually throw. If nothing can throw, no try/catch.
- Startup switches and similar non-critical lookups (a feature flag, a log level) may catch and return the safe default
  (the behavior the system has today) instead of rethrowing — but they must LOG the failure using
  the same logging the system uses in that context (Serilog static Log at startup, injected ILogger
  in services), and the deliberate fallback must be commented and pinned by a test.
```csharp
catch (Exception ex)
{
    // A bad config line must never crash startup or silently change behavior:
    // log it and default to the current behavior.
    Log.Warning(ex, "Could not read <setting>; defaulting to <current behavior>.");
    return false;
}
```
- Business logic keeps the standard rule: log with context and rethrow. Never silently continue past
  a failure that affects data correctness.

**Configuration content fails loudly (added 2026-10-30):**
- Data that drives behavior (device definitions, protocol descriptors, catalogs, mappings) is treated as code: a missing or invalid value throws with a clear message. No silent defaults, no partial loads.
- A loader never reports success when the read failed. Return or throw based on what actually happened.
- The safe-default rule above is for switches only. If a wrong value would change decoded data, it is content, and it must throw.
```csharp
// BAD
if (!definition.TryGetValue("Offset", out string offset)) { offset = "0"; }

// GOOD
if (!definition.TryGetValue("Offset", out string offset))
{
    throw new InvalidOperationException($"Definition '{definition.Name}' is missing required key 'Offset'.");
}
```

---

### Async/Await

**Async all the way** (don't block):
```csharp
// ❌ BAD
public void SavePermit(Permit permit)
{
    _context.SaveChangesAsync().Wait(); // blocks thread
}

// ✅ GOOD
public async Task SavePermitAsync(Permit permit)
{
    await _context.SaveChangesAsync();
}
```

**Suffix async methods with `Async`:**
- `CalculateDoseAsync`, `SavePermitAsync`, `LoadNuclidesAsync`

**Use `ConfigureAwait(false)` in libraries** (not controllers):
```csharp
await _repo.LoadDataAsync().ConfigureAwait(false);
```

---

### Comments

**Plain language, concise, no ticket numbers (added 2026-09-15):**
- Comments are written in plain language a new reader understands without context.
- No ticket/issue numbers in code comments or in shipped data files — tickets rot; code outlives them.
  Traceability lives in commit messages and PR titles (conventional commit + JIRA key), not in the code.

**Shortcuts are labeled (added 2026-10-30):**
- A known shortcut gets a `NOTE:` or `TODO:` that says what "better" looks like, so the next person knows the target. Still no ticket numbers.
```csharp
// TODO: Read the poll interval from the device definition instead of using the 5 second default.
```

**Public APIs in shared packages get XML doc comments (added 2026-10-30):**
- Public types and members in shared library packages carry `/// <summary>` (and `<param>` where it is not obvious). Application-internal code does not need them.

**Comment WHY, not WHAT:**
```csharp
// ❌ BAD
// Loop through nuclides
foreach (Nuclide nuclide in nuclides) { ... }

// ✅ GOOD
// RG 1.109 requires dose factors per nuclide-pathway pair
foreach (Nuclide nuclide in nuclides) { ... }
```

---

## SQL SERVER STANDARDS

### Security (STIGs Compliance)

**Use stored procedures or parameterized queries:**
```csharp
// ❌ BAD (SQL injection risk)
var sql = $"SELECT * FROM Permits WHERE PermitId = {permitId}";

// ✅ GOOD (stored procedure)
Permit permit = await _context.Permits
    .FromSqlInterpolated($"EXEC GetPermitById {permitId}")
    .FirstOrDefaultAsync();

// ✅ ALSO GOOD (parameterized)
Permit permit = await _context.Permits
    .FromSqlRaw("SELECT * FROM Permits WHERE PermitId = @p0", permitId)
    .FirstOrDefaultAsync();
```

**Least privilege:**
- Application accounts get EXECUTE on stored procedures only
- No direct table access (SELECT/INSERT/UPDATE/DELETE) in production
- Use contained database users (no server-level logins)

**Never log sensitive data:**
```csharp
// ❌ BAD
_logger.LogInfo($"User {username} logged in with password {password}");

// ✅ GOOD
_logger.LogInfo("User {Username} logged in", username);
```

---

### Data Operations

**NEVER delete rows — flag as deleted:**
```sql
-- ❌ BAD
DELETE FROM Permits WHERE PermitId = @PermitId

-- ✅ GOOD
UPDATE Permits 
SET Deleted = 1, 
    ModifiedBy = @UserId, 
    ModifiedDate = GETUTCDATE()
WHERE PermitId = @PermitId
```

**Always track who/when:**
- Every table must have: `CreatedBy`, `CreatedDate`, `ModifiedBy`, `ModifiedDate`
- Use `GETUTCDATE()` (not `GETDATE()`)
- Store user ID (not username — usernames can change)

```sql
CREATE TABLE Permits
(
    PermitId BIGINT PRIMARY KEY,
    -- ... other columns ...
    Deleted BIT NOT NULL DEFAULT 0,
    CreatedBy BIGINT NOT NULL,
    CreatedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    ModifiedBy BIGINT NOT NULL,
    ModifiedDate DATETIME2 NOT NULL DEFAULT GETUTCDATE(),
    UpdateCount BIGINT NOT NULL DEFAULT 0  -- Optimistic concurrency
)
```

**Document schema changes:**
```sql
-- Added 2026-07-27 by J. Perez (v-jperez)
-- Purpose: Track dose calculation method per site (SRS req 2.X)
ALTER TABLE StationParameters
ADD CalculationMethod NVARCHAR(50) NULL
```

---

### Stored Procedures

**Use stored procedures for production data access:**
```sql
CREATE PROCEDURE GetPermitById
    @PermitId BIGINT,
    @UserId BIGINT  -- Always pass user context
AS
BEGIN
    SET NOCOUNT ON;
    
    SELECT 
        PermitId,
        PermitNumber,
        Status,
        -- ... other columns ...
        CreatedBy,
        CreatedDate,
        ModifiedBy,
        ModifiedDate
    FROM Permits
    WHERE PermitId = @PermitId
      AND Deleted = 0  -- Never return deleted rows
END
```

**C# wrapper:**
```csharp
public async Task<Permit> GetByIdAsync(long permitId, long userId)
{
    return await _context.Permits
        .FromSqlInterpolated($"EXEC GetPermitById {permitId}, {userId}")
        .FirstOrDefaultAsync();
}
```

---

### Naming Conventions

**Tables:** PascalCase, singular (e.g., `Permit`, `DoseFactor`)  
**Columns:** PascalCase (e.g., `PermitId`, `ReleaseDate`)  
**Stored Procedures:** PascalCase verbs (e.g., `GetPermitById`, `SaveDoseResult`)  
**Indexes:** `IX_TableName_ColumnName` (e.g., `IX_Permit_Status`)  
**Primary Keys:** `PK_TableName` (e.g., `PK_Permit`)  
**Foreign Keys:** `FK_ChildTable_ParentTable_ColumnName` (e.g., `FK_Dose_Permit_PermitId`)

---

## TESTING STANDARDS

### xUnit Conventions

**Test method naming:**
```csharp
[Fact]
public void CalculateDose_WithValidInputs_ReturnsExpectedResult()
{
    // Arrange
    decimal concentration = 1.5m;
    decimal factor = 2.0m;
    
    // Act
    decimal result = _calcRepo.CalculateDose(concentration, factor);
    
    // Assert
    Assert.Equal(3.0m, result);
}
```

**Pattern:** `MethodName_Scenario_ExpectedBehavior`

**Coverage requirements:**
- 100% of calculation code (dose, decay, dilution formulas)
- 100% of critical paths (permit recalc, dose limits)
- 80%+ overall coverage

**Known-answer tests:**
```csharp
[Theory]
[InlineData(1.0, 2.0, 2.0)]  // Simple
[InlineData(0.0, 5.0, 0.0)]  // Zero concentration
[InlineData(10.0, 0.0, 0.0)] // Zero factor
public void CalculateDose_KnownAnswers_MatchesExpected(
    decimal concentration, decimal factor, decimal expected)
{
    decimal result = _calcRepo.CalculateDose(concentration, factor);
    Assert.Equal(expected, result, precision: 9); // 1e-9 tolerance
}
```

**Tolerance for floating-point:**
- Use `Assert.Equal(expected, actual, precision: 9)` for dose calculations
- Tolerance: 1e-9 for doses (matches IEEE 754 double precision)

**Every parser, decoder, and formula gets a harness (added 2026-10-30):**
- Replay real captured frames or known-answer data and compare results exactly. Sample data that a simulator generated proves nothing about the real device; use real captures when they exist and say so when they don't.

**Prove the test catches the bug (added 2026-10-30):**
- For a bug fix, run the new test against the old code once and confirm it fails. A test that passes on both versions is not testing the fix.

---

## DOCUMENTATION

### Per-Service `README.md`

Every service needs:
````markdown
# PermitRecalc

## What it does
Recalculates permit doses (liquid + gas) based on release data.

## How to run
```bash
docker-compose up permit-recalc
```

## Configuration
- `ConnectionStrings__AppDb`: SQL Server connection
- `StationParams__DenominatorType`: Dilution mode (1-4, default 3)

## Dependencies
- SQL Server (AppDb)
- RabbitMQ (optional, for event publishing)

## API Endpoints
- `POST /api/permits/{id}/recalc`: Trigger recalculation
````

### Architecture Decision Records (ADRs)

For big decisions, document in `docs/adr/`:
```markdown
# ADR-001: Use Stored Procedures for Data Access

## Context
Production SQL Server requires least-privilege security (STIGs compliance).

## Decision
All production data access uses stored procedures. Dev/test may use EF queries for speed.

## Consequences
- ✅ Pro: Security (app account has EXECUTE only, no table access)
- ✅ Pro: Performance (query plans cached)
- ❌ Con: More SQL code to maintain
```

---

## TOOLS & ENFORCEMENT

**IDE:** Visual Studio 2022 (primary), VS Code (scripts/config)

**Linters:**
- .editorconfig (spaces, braces, naming)
- Roslyn analyzers (StyleCop, Security Code Scan)

**CI/CD checks:**
- `dotnet build` (0 warnings)
- `dotnet test` (100% pass)
- `dotnet format --verify-no-changes` (style check)
- `dotnet list package --vulnerable` (security)

**Code review checklist:**
- [ ] Follows naming conventions
- [ ] Single-line blocks used appropriately (short + contained)
- [ ] Type-explicit locals in new code (no var); no LINQ chains in new code
- [ ] No ternary returns / expression-bodied logic; classic bodies
- [ ] Fields explicitly initialized; new parameters defaulted (no signature breaks)
- [ ] Deliberate safe-default catches are logged, commented, and test-pinned
- [ ] No ticket numbers in code comments or shipped data files
- [ ] File encoding matches the file's existing convention
- [ ] Input validation
- [ ] Tests included (xUnit)
- [ ] Comments explain WHY
- [ ] No secrets in code
- [ ] SQL uses stored procedures (production) or parameterized queries (dev)
- [ ] No DELETE statements (flag deleted instead)
- [ ] CreatedBy/ModifiedBy tracked
- [ ] No magic numbers or status strings; shared messages defined once
- [ ] Configuration content throws on missing/invalid values (only switches may default)
- [ ] Identifier spelling checked (class, method, JSON key, and enum names are expensive to rename later)
- [ ] Parsers/formulas have a replay harness; bug-fix tests fail on the old code
- [ ] Shortcuts labeled with NOTE/TODO; replaced blocks removed once verified

---

## REFERENCES

- **DISA STIGs:** Application Security and Development V5R3 (https://public.cyber.mil/stigs/)
- **C# Coding Conventions:** https://docs.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions
- **EF Core Best Practices:** https://docs.microsoft.com/en-us/ef/core/miscellaneous/
- **SQL Server Security:** https://docs.microsoft.com/en-us/sql/relational-databases/security/
- **RG 1.109:** NRC Regulatory Guide 1.109 (dose factor tables)
- **10 CFR 50 Appendix I:** Radiological effluent limits

---

**Last Updated:** 2026-10-30 (named constants, config content fails loudly, options bound once, data-carrier property exception, split tripwire, rewrite etiquette, labeled shortcuts, XML docs on shared packages, replay harness + fails-on-old-code tests, checklist items; earlier: 2026-09-15 type-explicit, no LINQ in new code, classic bodies, fail-to-current-state switches)  
**Owner:** Joseph Perez (personal copy of the team standard)  
**Review Cycle:** Update as team learns, quarterly review minimum  
**Flexibility:** Guidelines, not laws. Use judgment. If something doesn't make sense, discuss.

---

## APPENDIX: PERSONAL ADDITIONS

These apply to my own work everywhere. They are not project rules.

### Security notes with STIG references

- **Input validation (APSC-DV-002560):** validate every input at the boundary; reject, don't repair.
- **Error messages (APSC-DV-002330):** log the full exception internally, return a generic message outward. Never leak stack traces or internal names to a caller.
- **Logging (APSC-DV-002540):** log security events (failed access, permission denials) with user and resource IDs; never log secrets, tokens, or personal data.

### Null handling

```csharp
// GOOD: explicit guard at the boundary
if (order == null)
{
    throw new ArgumentNullException(nameof(order));
}

// GOOD: null-coalescing for display values only
string name = customer.Name ?? "Unknown";
```
Null-coalescing is for display defaults. Never use it to paper over missing configuration content (see "Configuration content fails loudly").

### SQL query habits

- Name every column; no `SELECT *` outside ad-hoc queries.
- Use CTEs when a query has more than one logical step.
- Keywords UPPERCASE, one clause per line, columns aligned.

### Git

- Branches: `feature/`, `fix/`, `hotfix/` + short description.
- Commits: conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `ci:`), plus the ticket key when the project uses one. Header under 100 characters.
- Squash noisy work-in-progress commits before review.

### Tools

EditorConfig, Roslyn analyzers / StyleCop, SonarQube, xUnit. A rule tool that fights this standard (for example a "loops should be LINQ" rule) gets tuned in config, not obeyed in code.