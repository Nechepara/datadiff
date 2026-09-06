# Development Brief — `Diff.Structure.dll`

**Product:** `Diff.Structure` — a .NET class library that compares the *structure* of same-named tables taken from two different MS SQL Server databases and returns the result as a JSON report.

**Target platform:** .NET 10, C# (latest language version). `async/await` is used everywhere an asynchronous path exists.

**Deliverable:** a single assembly `Diff.Structure.dll` plus a test project `Diff.Structure.Tests.dll`.

---

## 1. Purpose and scope

### 1.1 Purpose

Given two MS SQL Server connection strings (`left` and `right`), the library must:

1. Read the metadata of both databases (tables, fields, primary keys, foreign keys).
2. Determine which tables can be compared, which have to be skipped by the system, and which were excluded by the user.
3. Compare the structure of every comparable table pair.
4. Produce a single machine-readable result object that serializes to the JSON layout defined in **§5**.

A pair of same-named tables is considered **structurally equal** when all of the following hold:

- The two tables have the same set of fields, and every same-named field has the same **data type**, **size**, **nullable option**, **precision** and **scale**.
- The two tables have the same primary key — the same set of participating fields.
- The two tables have the same set of foreign keys — for every key the same referenced table and the same set of `field` → `referencedField` pairs.

If at least one of these conditions is violated, the violation must be reflected in the `difference.structure` section of the report.

### 1.2 In scope

- Tables, fields (columns), primary keys, foreign keys.
- Schema-qualified table identity (`schema.table`).
- User-driven exclusion of tables from the comparison.
- Cancellation through `CancellationToken` on every asynchronous operation.

### 1.3 Out of scope (v1)

The following objects are **not** compared and must not appear in the report: views, stored procedures, functions, triggers, indexes (other than the index backing a primary key), unique constraints, check constraints, default constraints, computed-column definitions, identity/seed settings, collations, filegroups, partitioning, extended properties, and table data. `ON DELETE` / `ON UPDATE` actions of a foreign key and the enabled/trusted state of a constraint are not compared either.

**Engine-generated shadow tables are silently ignored.** The history table of a system-versioned (temporal) table, the history table of an updatable ledger table, and the retained remains of a dropped ledger table or column are excluded from the run entirely: they never reach `tables.detected`, never appear in `tables.ignored` or `tables.unchecked`, and can never produce an entry under `difference`. The report gives no indication that such a table exists.

Two reasons make the exclusion safe rather than lossy. First, no information is lost: while versioning is on, SQL Server itself forbids altering a history table independently of its current table, so the history table's field set is a guaranteed mirror of one that *is* compared — reporting it would only duplicate every finding. The fact that a table is versioned at all remains observable through the generated columns on the current table (`ValidFrom` / `ValidTo`, or the four `ledger_*` columns), which are compared as ordinary fields; a table versioned on one side and not on the other therefore still surfaces, as those columns landing in `missings.<opposite side>.fields`. Second, without the exclusion the report would be wrong: a history table whose name the developer did not specify is named after the **object id** of its current table (`MSSQL_TemporalHistoryFor_<object_id>`, `MSSQL_LedgerHistoryFor_<object_id>`), and object ids are allocated per database, so two structurally identical databases yield two different names and thus a spurious entry in `tables.ignored` and `missings.*.tables`. The remains of dropped ledger objects are worse still — their names embed a GUID minted at drop time, they accumulate for the life of the database, and no single entry in `UserExcludedTables` can match both sides.

The exclusion also switches itself off at the right moment. `SET (SYSTEM_VERSIONING = OFF)` returns `temporal_type` to `0` on both tables, and the former history table — now an ordinary table that may genuinely drift — is compared again.

The `PERIOD FOR SYSTEM_TIME` declaration, the `SYSTEM_VERSIONING` / `LEDGER` state itself, and the auto-generated ledger view are likewise not compared; the view falls under the general exclusion of views above. What is compared is their observable column footprint, nothing more.

The library does **not** generate migration or synchronization scripts and **never** modifies either database — all access is strictly read-only.

---

## 2. Architecture

```
Diff.Structure (assembly)
├── TableStructureComparer            // orchestration + comparison logic
├── TableStructureComparisonOptions   // input
├── TableStructureComparisonResult    // output (serializes to the report JSON)
├── IMetadataProvider                 // metadata-reading contract
├── MsSqlMetadataProvider             // IMetadataProvider implementation for MS SQL Server
├── CollationInfo                     // a database's collation and its comparison flags
├── INameComparerResolver             // identifier-comparison rule contract
├── MsSqlNameComparerResolver         // collation -> StringComparer for MS SQL Server
├── QuotesUsage / Quotes / QuoteInfo  // identifier-quoting policy (§3.3)
├── NameIdentity / TableIdentity      // identifiers and their renderings
├── FieldMetadata / PrimaryKeyMetadata / ForeignKeyMetadata
└── ObjectExtensions                  // ToJson() extension — ObjectExtensions.cs

One public type per file, named after the type. `Quotes` and `QuoteInfo` are `partial`:
the engine-neutral half lives in Quotes.cs / QuoteInfo.cs, the MS SQL specifics
(the bracket pair and the "does this identifier need quoting" predicate) in
MsSqlQuotes.cs / MsSqlQuoteInfo.cs, so a second engine is added without touching
the neutral half.
```

**Separation of concerns (hard requirement).** `TableStructureComparer` never touches ADO.NET, never opens a connection and never executes SQL. All metadata retrieval is delegated to `IMetadataProvider`. This keeps the comparison logic pure, makes it unit-testable against a fake provider, and allows providers for other engines to be added later without touching the comparer.

**Dependencies.** `Microsoft.Data.SqlClient` (referenced only by `MsSqlMetadataProvider`) and `System.Text.Json`. No other third-party dependencies.

---

## 3. Public API

### 3.1 `TableStructureComparisonOptions`

```csharp
public sealed class TableStructureComparisonOptions
{
    /// <summary>Connection strings of the two databases being compared.</summary>
    public (string Left, string Right) ConnectionStrings { get; init; }

    /// <summary>Tables excluded from the comparison by the user, as "schema.table".
    /// Matched against the detected tables with the resolved comparer — see §6.1.</summary>
    public IReadOnlyList<string> UserExcludedTables { get; init; } = [];

    /// <summary>Quoting rules applied to every identifier the run produces.</summary>
    public QuoteInfo QuoteInfo { get; init; } = QuoteInfo.MsSql;

    /// <summary>How the report renders identifiers — see §3.3.1.</summary>
    public QuotesUsage QuotesUsage { get; init; } = QuotesUsage.DoNotUseIfPossible;
}
```

The options object carries no comparison rule of its own, and that is deliberate. `UserExcludedTables` is a plain `IReadOnlyList<string>` rather than a hashed set, because the rule by which those names are matched is not known until both databases have been read: it comes from their collation, through `INameComparerResolver` (§3.3.2, §6.1). A set hashed under one rule cannot answer questions asked under another, so the comparer builds its own lookup once the rule is known. Omitting the property excludes nothing.

Exclusions are written as `schema.table` strings. A caller may quote either part (`[dbo].[Order.Archive]`); every entry is normalized through `TableIdentity.Parse` and matched on the bare two-part name, so a dot inside a quoted identifier is not mistaken for the separator.

`QuoteInfo` and `QuotesUsage` carry the identifier-quoting policy for the whole run: the comparer builds every `NameIdentity` with them and never consults any ambient default. Both are `init`-only, so the policy cannot change while a comparison is in flight — a mutable setter would let one report mix two renderings. `QuoteInfo` defaults to `QuoteInfo.MsSql`, `QuotesUsage` to `DoNotUseIfPossible`, so a caller who does not care about quoting sets neither.

### 3.2 `TableStructureComparer`

```csharp
public sealed class TableStructureComparer
{
    public TableStructureComparer(IMetadataProvider metadataProvider, INameComparerResolver nameComparerResolver);

    public Task<TableStructureComparisonResult> CompareTableStructureAsync(
        TableStructureComparisonOptions options,
        CancellationToken cancellationToken);
}
```

- The comparer takes two collaborators and owns neither concern itself: `IMetadataProvider` reads the schema (§3.3.1), `INameComparerResolver` decides how identifiers are compared (§3.3.2). Both are interfaces, so a unit test drives the whole algorithm with two fakes and no SQL Server.
- The constructor throws `ArgumentNullException` when either `metadataProvider` or `nameComparerResolver` is `null`.
- The class is stateless and therefore thread-safe: a single instance may serve concurrent calls, including runs against different databases with different collations.

### 3.3.1 `IMetadataProvider`

The interface declares the contract for every operation needed to read MS SQL metadata about tables, fields, primary keys and foreign keys. Metadata is fetched **per database in bulk**, not per table, so that one comparison costs a small fixed number of round trips regardless of how many tables the databases contain.

```csharp
public interface IMetadataProvider
{
    /// <summary>Returns the collation of the database — its name plus the properties
    /// a comparer is built from (§3.3.2).</summary>
    Task<CollationInfo> GetDatabaseCollationAsync(
        string connectionString,
        CancellationToken cancellationToken = default);

    /// <summary>Returns all user tables of the database.</summary>
    Task<IReadOnlyCollection<TableIdentity>> GetTablesAsync(
        string connectionString,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the fields of the specified tables.</summary>
    Task<IReadOnlyCollection<FieldMetadata>> GetFieldsAsync(
        string connectionString,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the primary keys of the specified tables (zero or one per table).</summary>
    Task<IReadOnlyCollection<PrimaryKeyMetadata>> GetPrimaryKeysAsync(
        string connectionString,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the foreign keys declared on the specified tables.</summary>
    Task<IReadOnlyCollection<ForeignKeyMetadata>> GetForeignKeysAsync(
        string connectionString,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);
}
```

Supporting metadata types (immutable records):

```csharp
//file QuotesUsage.cs
public enum QuotesUsage : byte { UseIfSpecifiedOrRequired = 0, Required = 1, DoNotUseIfPossible = 2 }
//file Quotes.cs
public partial record Quotes(char Open, char Close)
{
    public string Add(string identifier);
    public string Remove(string quotedIdentifier);
    public bool IsQuotedIdentifier(string identifier);
}
//file MsSqlQuotes.cs
public partial record Quotes
{
    public static Quotes MsSql { get; } = new Quotes('[', ']');
}
//file QuoteInfo.cs
public partial class QuoteInfo
{
    public Quotes Quotes { get; init; }
    public Func<string, bool> QuotesRequired { get; init; }
}
//file MsSqlQuoteInfo.cs
public partial class QuoteInfo
{
    public static QuoteInfo MsSql { get; } =
        new QuoteInfo { Quotes = Quotes.MsSql, QuotesRequired = MsSqlRequiresQuotedIdentifier };
    public static bool MsSqlRequiresQuotedIdentifier(string identifier);
}
//file NameIdentity.cs
public record NameIdentity
{
    public NameIdentity(
        string name,
        QuotesUsage quotesUsage,
        QuoteInfo quoteInfo)
    {
        this.QuoteInfo = quoteInfo;
        this.IsQuoted = this.QuoteInfo.Quotes.IsQuotedIdentifier(name);
        this.Name = this.IsQuoted ? this.QuoteInfo.Quotes.Remove(name) : name;
        this.QuotesUsage = quotesUsage;
    }

    public QuoteInfo QuoteInfo { get; }
    public QuotesUsage QuotesUsage { get; }
    public bool IsQuoted { get; }
    public string Name { get; }
    public string QuotedName => this.QuoteInfo.Quotes.Add(this.Name);

    public string PreferredName => this.QuotesUsage switch
    {
        QuotesUsage.Required => this.QuotedName,
        QuotesUsage.UseIfSpecifiedOrRequired => this.IsQuoted || this.QuoteInfo.QuotesRequired(this.Name)
            ? this.QuotedName
            : this.Name,
        QuotesUsage.DoNotUseIfPossible => this.QuoteInfo.QuotesRequired(this.Name)
            ? this.QuotedName
            : this.Name,
        _ => throw new ArgumentOutOfRangeException(nameof(this.QuotesUsage)),
    };
}
//file TableIdentity.cs
public sealed record TableIdentity
{
    public TableIdentity(
        string schema,
        string table,
        QuotesUsage quotesUsage,
        QuoteInfo quoteInfo)
    {
        this.Schema = new NameIdentity(schema, quotesUsage, quoteInfo);
        this.Table = new NameIdentity(table, quotesUsage, quoteInfo);
    }

    public NameIdentity Schema { get; }
    public NameIdentity Table { get; }

    public QuotesUsage QuotesUsage => this.Table.QuotesUsage;
    public QuoteInfo QuoteInfo => this.Table.QuoteInfo;

    public string Name => $"{Schema.Name}.{Table.Name}";
    public string QuotedName => $"{Schema.QuotedName}.{Table.QuotedName}";
    public string PreferredName => $"{Schema.PreferredName}.{Table.PreferredName}";

    /// <summary>Parses a two-part name: "schema.table" or "[schema].[table]".</summary>
    public static TableIdentity Parse(
        string tableWithSchema,
        QuotesUsage quotesUsage,
        QuoteInfo quoteInfo);

    /// <summary>Non-throwing counterpart of <see cref="Parse"/>.</summary>
    public static bool TryParse(string? tableWithSchema, QuotesUsage quotesUsage, QuoteInfo quoteInfo, out TableIdentity? result);
}
//file FieldMetadata.cs
public sealed record FieldMetadata
{
    public FieldMetadata(
        TableIdentity table,
        string name,
        string dataType,     // "nvarchar", "int", "decimal", ...
        int? size,           // length in characters/bytes; -1 = MAX; null when not applicable
        bool isNullable,
        byte? precision,     // null when not applicable
        byte? scale)         // null when not applicable
    {
        this.Table = table;
        this.Field = new NameIdentity(name, table.QuotesUsage, table.QuoteInfo);
        this.DataType = dataType;
        this.Size = size;
        this.IsNullable = isNullable;
        this.Precision = precision;
        this.Scale = scale;
    }

    public TableIdentity Table { get; }
    public NameIdentity Field { get; }
    public string DataType { get; }
    public int? Size { get; }
    public bool IsNullable { get; }
    public byte? Precision { get; }
    public byte? Scale { get; }
}
//file PrimaryKeyMetadata.cs
public sealed record PrimaryKeyMetadata
{
    public PrimaryKeyMetadata(
        TableIdentity table,
        string name,
        IReadOnlyList<string> fields)       // ordered by key ordinal
    {
        this.Table = table;
        this.Key = new NameIdentity(name, table.QuotesUsage, table.QuoteInfo);
        this.Fields = fields
            .Select(field => new NameIdentity(field, table.QuotesUsage, table.QuoteInfo))
            .ToImmutableArray();
    }

    public TableIdentity Table { get; }
    public NameIdentity Key { get; }
    public ImmutableArray<NameIdentity> Fields { get; }
}
//file ForeignKeyColumn.cs
public sealed record ForeignKeyColumn(string Field, string ReferencedField);
//file ForeignKeyMetadata.cs
public sealed record ForeignKeyMetadata
{
    public ForeignKeyMetadata(
        TableIdentity table,
        string name,
        TableIdentity referencedTable,
        IReadOnlyList<ForeignKeyColumn> columns)   // ordered by constraint column id
    {
        this.Table = table;
        this.Key = new NameIdentity(name, table.QuotesUsage, table.QuoteInfo);
        this.ReferencedTable = referencedTable;
        this.Columns = columns
            .Select(column =>
            {
                var Field = new NameIdentity(column.Field, table.QuotesUsage, table.QuoteInfo);
                var ReferencedField = new NameIdentity(column.ReferencedField, referencedTable.QuotesUsage, referencedTable.QuoteInfo);
                return (Field, ReferencedField);
            })
            .ToImmutableArray();
    }

    public TableIdentity Table { get; }
    public NameIdentity Key { get; }
    public TableIdentity ReferencedTable { get; }
    public ImmutableArray<(NameIdentity Field, NameIdentity ReferencedField)> Columns { get; }
}
```


Every identifier the library handles is a `NameIdentity`: the bare name plus the knowledge of whether the caller supplied it quoted and how it should be rendered back. Three renderings are available and they are **not** interchangeable — each has one job:

| Rendering | Content | Used for |
| --- | --- | --- |
| `Name` | the bare identifier, quotes stripped | **all comparison and matching** (§6.1) |
| `QuotedName` | always quoted | building SQL; never the report |
| `PreferredName` | quoted only when `QuotesUsage` demands it | **the names rendered in the report** (§5.1) |

`QuotesUsage` selects the third one's policy: `Required` always quotes; `DoNotUseIfPossible` (the default) quotes only when `QuoteInfo.QuotesRequired` says the identifier cannot survive unquoted; `UseIfSpecifiedOrRequired` does the same but also preserves quotes the caller already used, which is what `IsQuoted` records. `QuoteInfo.MsSql` supplies the MS SQL bracket pair together with `MsSqlRequiresQuotedIdentifier` as that predicate.

There is no ambient default and no process-wide current policy: `NameIdentity` and `TableIdentity` both require `quotesUsage` and `quoteInfo` explicitly, and the comparer feeds them from `TableStructureComparisonOptions` (§3.1). That is what keeps the comparer stateless and safe for concurrent runs — two comparisons may use different policies at the same time without interfering.

`TableIdentity` holds the two `NameIdentity` values and re-exposes their `QuotesUsage` and `QuoteInfo` — that is what lets `FieldMetadata`, `PrimaryKeyMetadata` and `ForeignKeyMetadata` build their own identifiers under the same policy as the table they belong to. A foreign key's `ReferencedField` is built from the **referenced** table's policy, not the declaring one, so a rendered name always follows the table it actually names.

`TableIdentity.Parse` is the string entry point into the pair — it lets a caller write an exclusion as `"dbo.Orders"` instead of spelling out both parts. Its grammar is fixed:

- The input is a **two-part** name. Surrounding whitespace is trimmed; the split is ordinal.
- Either part may be quoted with the pair taken from `QuoteInfo` — `[` and `]` for MS SQL — and a dot inside quotes is literal: `[dbo].[Order.Archive]` yields `Schema.Name == "dbo"` and `Table.Name == "Order.Archive"`, each with `IsQuoted == true`. A `]]` inside brackets is an escaped `]`. The quotes do not survive into `NameIdentity.Name`; they are re-applied on demand by `QuotedName` and `PreferredName`.
- An unquoted input must contain exactly one dot. `"dbo.Orders"` parses; `"dbo.Order.Archive"` throws, because a bare name with two dots is genuinely ambiguous and the method must not guess.
- Both `Parse` and `TryParse` take `quotesUsage` and `quoteInfo` explicitly and pass them to the two `NameIdentity` values; neither has a defaulted overload, so a caller always states the policy the resulting identity will be rendered under. A `null` input throws `ArgumentNullException`. An input with no dot, with an empty part, or with unbalanced quotes throws `FormatException`. No schema is inferred: `"Orders"` does not silently become `dbo.Orders`, since the effective default schema depends on the login and guessing it would exclude the wrong table.
- `Parse` performs no validation against either database — it is a pure string operation. A parsed name that matches nothing simply never reaches `tables.unchecked` (§5.1).
- `TryParse` applies exactly the same grammar but never throws: every rejection above — `null`, a missing dot, an empty part, unbalanced brackets, an ambiguous unquoted name — returns `false` with `result` set to `null`. It exists so that a caller validating user-supplied text does not have to drive control flow with exceptions.

**Contract rules for implementers**

- A method never returns `null`; an empty collection is returned when nothing is found.
- `PrimaryKeyMetadata.Fields` and `ForeignKeyMetadata.Columns` are **ordered sequences**, not sets: the provider passes the participating fields in key order (`key_ordinal` for a primary key, `constraint_column_id` for a foreign key), because the report renders them in that order (§5). Set semantics belong to the comparison, not to the metadata: §6.4 and §6.5 compare these sequences as sets under the comparer detected in §6.1. A provider must not pass a hash-set type here — doing so would both lose the key order and hard-code an equality rule that §6.1 reserves for the collation.
- The provider supplies every identifier as a plain `string`; the metadata records wrap it in a `NameIdentity` themselves, under the `QuotesUsage` and `QuoteInfo` of the owning `TableIdentity`. A provider therefore never constructs a `NameIdentity` and never chooses a quoting policy.
- The `tables` argument narrows the result set. Passing an empty collection returns an empty result without hitting the database.
- Every method honours `cancellationToken` and propagates `OperationCanceledException`.
- Implementations must be safe to call concurrently for the two connection strings.
### 3.3.2 `INameComparerResolver`

The rule by which two identifiers are judged equal is a property of the database, not of the library, so it is resolved through its own contract rather than hard-coded:

```csharp
//file CollationInfo.cs
public sealed record CollationInfo(
    string Name,           // "SQL_Latin1_General_CP1_CI_AS"
    int Lcid,              // COLLATIONPROPERTY(Name, 'LCID')            -> CultureInfo
    int CodePage,          // COLLATIONPROPERTY(Name, 'CodePage')        -> non-Unicode encoding
    int ComparisonStyle,   // COLLATIONPROPERTY(Name, 'ComparisonStyle') -> 0 for a binary collation
    int Version)           // COLLATIONPROPERTY(Name, 'Version')         -> 0, 90, 100, 140 ...
{
    // ComparisonStyle is a bit mask; a binary collation reports 0 and ignores nothing.
    public bool IsBinary       => this.ComparisonStyle == 0;
    public bool IgnoreCase     => (this.ComparisonStyle & 0x00001) != 0;
    public bool IgnoreAccent   => (this.ComparisonStyle & 0x00002) != 0;   // IgnoreNonSpace
    public bool IgnoreKanaType => (this.ComparisonStyle & 0x10000) != 0;
    public bool IgnoreWidth    => (this.ComparisonStyle & 0x20000) != 0;
}

//file INameComparerResolver.cs
public interface INameComparerResolver
{
    public StringComparer ResolveNameComparer(CollationInfo collation);
}
```

`CollationInfo` is what a database reports about its own collation, and it carries everything needed to reproduce that collation's rules in .NET:

| Member | Source | Role |
| --- | --- | --- |
| `Name` | `DATABASEPROPERTYEX(db, 'Collation')` | identity of the collation; §6.1 compares the two sides on this |
| `Lcid` | `COLLATIONPROPERTY(name, 'LCID')` | the culture whose linguistic rules apply — `CultureInfo.GetCultureInfo(Lcid)` |
| `ComparisonStyle` | `COLLATIONPROPERTY(name, 'ComparisonStyle')` | bit mask of what the collation ignores; `0` marks a binary collation |
| `CodePage` | `COLLATIONPROPERTY(name, 'CodePage')` | the non-Unicode code page; not used for comparison, carried for diagnostics |
| `Version` | `COLLATIONPROPERTY(name, 'Version')` | collation version (`0`, `90`, `100`, `140`…); two collations differing only in version sort differently |

The `ComparisonStyle` bits map one-to-one onto `System.Globalization.CompareOptions`, which is what makes the resolver a pure function rather than a table of special cases: `0x1` → `IgnoreCase`, `0x2` → `IgnoreNonSpace` (accent), `0x10000` → `IgnoreKanaType`, `0x20000` → `IgnoreWidth`. The record exposes them as named predicates so no caller repeats the masks.

Contract rules:

- The resolver is a **pure mapping** from a collation to a comparer, and its signature says so: no `Task`, no `CancellationToken`. It opens no connection and reads no metadata — the caller has already obtained the `CollationInfo` from `IMetadataProvider.GetDatabaseCollationAsync`. This is the one place in the library that is deliberately synchronous: there is nothing to await, and wrapping a table lookup in a `Task` would only add an allocation and hide that fact from the caller.
- The mapping must be **total and deterministic** for the collations it accepts: the same `CollationInfo` always yields an equivalent comparer, and an unsupported one throws rather than degrades (§3.4.2).
- Implementations never return `null`.
- Keeping this separate from `IMetadataProvider` is what lets a test pin the comparison rule without standing up any metadata at all, and lets a caller override the rule for a collation the built-in mapping handles poorly.

### 3.4.1 `MsSqlMetadataProvider`

An `IMetadataProvider` implementation on top of `Microsoft.Data.SqlClient` that reads the `sys.*` catalog views. One `SqlConnection` per call, opened with `OpenAsync(ct)` and read with `ExecuteReaderAsync(ct)` / `ReadAsync(ct)`. No synchronous ADO.NET calls anywhere.

Selection rule: user tables only — `sys.tables` with `type = 'U'` and `is_ms_shipped = 0`; system schemas (`sys`, `INFORMATION_SCHEMA`) are excluded. On top of that the provider drops the engine-generated shadow tables named in §1.3 — temporal history tables (`temporal_type = 1`), ledger history tables (`ledger_type = 1`) and the retained remains of dropped ledger tables (`is_dropped_ledger_table = 1`). The filter is applied in the provider, not in the comparer, and it is not configurable: these tables are absent from every collection `IMetadataProvider` returns, so no downstream code has to know they exist.

Two of those columns — `ledger_type` and `is_dropped_ledger_table` — exist only on SQL Server 2022 (major version 16) and later. Referencing a column that does not exist is a **parse-time** failure, not a runtime one, so the predicate cannot simply be written into a static statement and left to evaluate harmlessly on an older server. Each of the four metadata statements is therefore sent as a short batch that probes for the column, appends the two predicates only when it is there, and executes the composed text through `sys.sp_executesql`:

```sql
DECLARE @shadow nvarchar(200) = N' AND t.temporal_type <> 1';
IF COL_LENGTH(N'sys.tables', N'ledger_type') IS NOT NULL
    SET @shadow += N' AND t.ledger_type <> 1 AND t.is_dropped_ledger_table = 0';
EXEC sys.sp_executesql (@baseQuery + @shadow);
```

The batch is still **one** round trip, so §9's five-query budget is unchanged; the probe costs no extra call and the provider stays stateless, since nothing is carried between calls. `temporal_type` needs no probe — it has been present since SQL Server 2016, the library's minimum.

The probe is deliberately a **capability** check rather than a version check. Azure SQL Database supports ledger while still reporting `SERVERPROPERTY('ProductMajorVersion')` as `12`, so a version comparison would silently disable the filter on exactly the platform where ledger tables are most likely to be found.

Reference queries:

```sql
-- Database collation
DECLARE @collation sysname = CONVERT(sysname, DATABASEPROPERTYEX(DB_NAME(), 'Collation'));
SELECT @collation                                        AS Name,
       COLLATIONPROPERTY(@collation, 'LCID')             AS Lcid,
       COLLATIONPROPERTY(@collation, 'CodePage')         AS CodePage,
       COLLATIONPROPERTY(@collation, 'ComparisonStyle')  AS ComparisonStyle,
       COLLATIONPROPERTY(@collation, 'Version')          AS Version;

-- Tables
SELECT s.name AS SchemaName, t.name AS TableName
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE t.type = 'U' AND t.is_ms_shipped = 0
  AND t.temporal_type <> 1                 -- temporal history table
  AND t.ledger_type <> 1                   -- ledger history table       (2022+ only)
  AND t.is_dropped_ledger_table = 0;       -- remains of a dropped one   (2022+ only)

-- Fields
SELECT s.name, t.name, c.name, ty.name AS DataType,
       c.max_length, c.precision, c.scale, c.is_nullable
FROM sys.columns c
JOIN sys.tables   t  ON t.object_id     = c.object_id
JOIN sys.schemas  s  ON s.schema_id     = t.schema_id
JOIN sys.types    ty ON ty.user_type_id = c.user_type_id
WHERE t.type = 'U' AND t.is_ms_shipped = 0
  AND t.temporal_type <> 1
  AND t.ledger_type <> 1 AND t.is_dropped_ledger_table = 0;

-- Primary keys
SELECT s.name, t.name, kc.name AS KeyName, c.name AS ColumnName, ic.key_ordinal
FROM sys.key_constraints kc
JOIN sys.tables        t  ON t.object_id  = kc.parent_object_id
JOIN sys.schemas       s  ON s.schema_id  = t.schema_id
JOIN sys.index_columns ic ON ic.object_id = kc.parent_object_id
                         AND ic.index_id  = kc.unique_index_id
JOIN sys.columns       c  ON c.object_id  = ic.object_id
                         AND c.column_id  = ic.column_id
WHERE kc.type = 'PK' AND t.is_ms_shipped = 0
  AND t.temporal_type <> 1
  AND t.ledger_type <> 1 AND t.is_dropped_ledger_table = 0;

-- Foreign keys
SELECT s.name  AS SchemaName,    t.name  AS TableName,    fk.name AS KeyName,
       rs.name AS RefSchemaName, rt.name AS RefTableName,
       pc.name AS ColumnName,    rc.name AS RefColumnName,
       fkc.constraint_column_id
FROM sys.foreign_keys fk
JOIN sys.tables  t  ON t.object_id  = fk.parent_object_id
JOIN sys.schemas s  ON s.schema_id  = t.schema_id
JOIN sys.tables  rt ON rt.object_id = fk.referenced_object_id
JOIN sys.schemas rs ON rs.schema_id = rt.schema_id
JOIN sys.foreign_key_columns fkc ON fkc.constraint_object_id = fk.object_id
JOIN sys.columns pc ON pc.object_id = fkc.parent_object_id
                   AND pc.column_id = fkc.parent_column_id
JOIN sys.columns rc ON rc.object_id = fkc.referenced_object_id
                   AND rc.column_id = fkc.referenced_column_id
WHERE t.is_ms_shipped = 0
  AND t.temporal_type <> 1
  AND t.ledger_type <> 1 AND t.is_dropped_ledger_table = 0;
```

The shadow-table predicates are shown above in their SQL Server 2022 form; on an earlier server the two `ledger_*` lines are the ones the `COL_LENGTH` probe omits.

**Type normalization performed by the provider**, so that the comparer receives already-comparable values:

| Aspect | Rule |
| --- | --- |
| `DataType` | The system type name reported by `sys.types`, lower-case. |
| `Size` | `max_length` for length-bearing types (`char`, `varchar`, `binary`, `varbinary`, `nchar`, `nvarchar`). For the Unicode types (`nchar`, `nvarchar`) the byte length is divided by 2 so that the value is expressed **in characters**. `-1` means `MAX` and is passed through as `-1`. `null` for all other types. |
| `Precision` / `Scale` | Reported only for the types where they are meaningful (`decimal`, `numeric`, `time`, `datetime2`, `datetimeoffset`); `null` otherwise. |
| Aliases | `sysname` is normalized to `nvarchar(128)`. |
### 3.4.2 `MsSqlNameComparerResolver`
```csharp
public class MsSqlNameComparerResolver: INameComparerResolver
{
	/// <summary>
	/// The methods builds and returns the new string comparer
	/// which have exactly the same sorting and comparison rules 
	/// as the specified collation in the MS SQL Server database.
	/// </summary>
	public StringComparer ResolveNameComparer(CollationInfo collation);
}
```

It touches no database: the `CollationInfo` it receives already carries everything the mapping needs.

- A **binary** collation (`IsBinary`, i.e. `ComparisonStyle == 0`; the name ends in `_BIN` or `_BIN2`) compares by code point — `StringComparer.Ordinal`.
- Any other collation is **linguistic**. The resolver turns the ignore-flags into `CompareOptions` and asks the culture for a matching comparer:

```csharp
var options = CompareOptions.None
    | (collation.IgnoreCase     ? CompareOptions.IgnoreCase     : CompareOptions.None)
    | (collation.IgnoreAccent   ? CompareOptions.IgnoreNonSpace : CompareOptions.None)
    | (collation.IgnoreKanaType ? CompareOptions.IgnoreKanaType : CompareOptions.None)
    | (collation.IgnoreWidth    ? CompareOptions.IgnoreWidth    : CompareOptions.None);

return CultureInfo.GetCultureInfo(collation.Lcid)
                  .CompareInfo
                  .GetStringComparer(options);
```

- An `Lcid` that `CultureInfo.GetCultureInfo` does not recognise raises `CultureNotFoundException`, and a `ComparisonStyle` carrying bits outside the four listed above is not silently approximated either. Both are wrapped in a `NotSupportedException` that names the collation, so an unhandled collation surfaces as a clear failure rather than as a quietly wrong report.

**What this reproduces, and what it does not.** The four ignore-flags cover case, accent, kana and width, which is what decides whether two identifiers are the *same* identifier — the question this library actually asks. It does not reproduce SQL Server's *sort order* exactly: .NET orders through ICU, SQL Server through its own tables, and the legacy `SQL_*` collations use sort orders that predate Windows collations altogether. That is why the report's ordering does not depend on this comparer (§4), and why an exact ordering match is listed as an open question (§11).

---

## 4. Result model

`TableStructureComparisonResult` is a POCO tree that maps one-to-one onto the JSON of **§5**. Serialization requirements:

- `JsonNamingPolicy.CamelCase`.
- **All sections are always present.** Empty collections serialize as `[]`, empty objects as `{}` — a consumer must never have to null-check a section.
- **A null scalar is omitted, never emitted.** `size`, `precision` and `scale` are written only when they carry a value; when the underlying metadata has none, the property is absent from the JSON altogether rather than present with a `null` value (`JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`). The same applies to the `key` and `fields` members of a `missings.*.primaryKeys` entry when the primary key is absent in both databases (see Use Case 7). A consumer therefore tests for the *presence* of these properties, not for a null value.
- Actual JSON types: `size`, `precision`, `scale` — `number`, the property being absent when not applicable; `nullable` — `boolean`; every other property — `string`.
- Every list is sorted ascending by its natural key (table name, then field name, then key name) with `StringComparer.Ordinal`, so that two runs over the same pair of databases produce byte-identical JSON. Ordering deliberately does **not** use the comparer of §6.1: a linguistic comparer orders through ICU, whose tables differ between host platforms and runtime versions, so the same databases would serialize differently on two machines. Ordinal ordering is safe here precisely because equality is still decided by the resolved comparer — two names it considers equal can never both appear in one list.
- **No plaintext password ever reaches the result.** The `left` and `right` members of `TableStructureComparisonResult` hold the connection strings with the password value already replaced by the literal `***`, so the secret is absent from the object itself and not merely from its serialized form. The JSON carries exactly the same masked strings — serialization performs no further transformation. See §6.6.

Serialization is exposed as an extension method on `object`, declared in its own file **`ObjectExtensions.cs`**:

```csharp
// ObjectExtensions.cs
public static class ObjectExtensions
{
    /// <summary>Serializes the instance to JSON using the report's serialization settings.</summary>
    public static string ToJson(this object value, JsonSerializerOptions? options = null);
}
```

Requirements for the extension:

- When `options` is omitted, the method applies the library's default `JsonSerializerOptions` — the camelCase naming policy and `JsonIgnoreCondition.WhenWritingNull` listed above — so that `result.ToJson()` alone produces a report conforming to §5. When `options` is supplied it is used verbatim, and honouring the rules above becomes the caller's responsibility.
- The defaults are built once into a cached static `JsonSerializerOptions` instance; the object is never rebuilt per call.
- The body is a straight delegation — `JsonSerializer.Serialize(value, options)`. Although the parameter is declared as `object`, `System.Text.Json` resolves the value's runtime type and writes the full object graph, so no `value.GetType()` overload is required.
- `value` being `null` throws `ArgumentNullException`.
- The type is `public static` and lives in the library's root namespace, so `using Diff.Structure;` is all a consumer needs.

---

## 5. Report structure

```json
{
  "left": "The connection string to the first DB, with the password value replaced by ***.",
  "right": "The connection string to the second DB, with the password value replaced by ***.",
  "tables": {
    "detected": ["The distinct list of table names fetched from both databases, `schema.table`"],
    "ignored": {
      "left":  ["Tables fetched from the left DB that the system was unable to compare"],
      "right": ["Tables fetched from the right DB that the system was unable to compare"]
    },
    "unchecked": ["Tables excluded from the comparison by the user"]
  },
  "difference": {
    "structure": {
      "missings": {
        "left": {
          "tables": ["Tables that are missing in the LEFT database but present in the right one"],
          "fields": [
            { "field": "The field missing in the left table", "table": "schema.table" }
          ],
          "primaryKeys": [
            {
              "key": "The PK name found in the right DB; omitted when the PK is absent in both DBs",
              "table": "schema.table",
              "fields": ["field1", "field2"]
            }
          ],
          "foreignKeys": [
            {
              "key": "The FK name found in the right DB",
              "table": "schema.table",
              "referencedTable": "schema.table",
              "fields": [ { "field": "F", "referencedField": "RF" } ]
            }
          ]
        },
        "right": { }
      },
      "inconsistencies": {
        "fields": [
          {
            "field": "The field name",
            "table": "schema.table",
            "inconsistency": {
              "left":  { "type": "The data type name", "size": "number, omitted when not applicable", "nullable": "boolean", "precision": "number, omitted when not applicable", "scale": "number, omitted when not applicable" },
              "right": { "type": "The data type name", "size": "number, omitted when not applicable", "nullable": "boolean", "precision": "number, omitted when not applicable", "scale": "number, omitted when not applicable" }
            }
          }
        ],
        "primaryKeys": [
          {
            "table": "schema.table",
            "inconsistency": {
              "left":  { "key": "The PK name in the left DB",  "fields": ["field1", "field2"] },
              "right": { "key": "The PK name in the right DB", "fields": ["field1"] }
            }
          }
        ],
        "foreignKeys": [
          {
            "table": "schema.table",
            "referencedTable": "schema.table",
            "inconsistency": {
              "left":  { "key": "The FK name in the left DB",  "fields": [ { "field": "F", "referencedField": "RF" } ] },
              "right": { "key": "The FK name in the right DB", "fields": [ { "field": "F", "referencedField": "RF" } ] }
            }
          }
        ]
      }
    }
  }
}
```

### 5.1 Rules that govern the report

**Direction of the `missings` section.** `missings.left` lists objects that are **absent in the left database** but present in the right one. `missings.right` is the mirror image. `missings.right` has exactly the same shape as `missings.left`; it is shown empty above only for brevity.

**Shape of the `inconsistencies` nodes.** All three sections (`fields`, `primaryKeys`, `foreignKeys`) share one uniform shape: the properties common to both sides (`table`, `field`, `referencedTable`) live in the array element itself, and the two compared sides are nested in an `inconsistency` property with `left` and `right` nodes.

**Relationship between the table lists.** `tables.unchecked` is a subset of `tables.detected`; membership is decided with the comparer of §6.1, the same one that governs every other identifier comparison. Names listed in `UserExcludedTables` but not found in either database do not appear in the report. The comparison set is computed as `detected − unchecked − ignored.left − ignored.right`.

**A table found in only one database** lands in two sections at once: in `tables.ignored.<side where the table exists>`, explaining why it was not compared, and in `missings.<opposite side>.tables`, recording the structural difference. The fields and keys of such a table are **not** additionally listed as missing.

**Properties that may be absent.** The block above documents the full shape of the field descriptor. In an actual report `size`, `precision` and `scale` appear only when they carry a value — a `nvarchar` field emits `type`, `size` and `nullable` and nothing else, while a `decimal` field emits `type`, `nullable`, `precision` and `scale`. No property is ever written with a `null` value.

**Identifier form.** Every table name in the report consists of two parts — schema name and table name — separated by a `.`. Names are rendered from `PreferredName`, so an identifier that needs no quoting appears bare (`dbo.Orders`) while one that does appears quoted (`[dbo].[Order.Archive]`). Field and key names are rendered the same way. `QuotedName` is never written to the report, and the bare `Name` is used for comparison only.

---

## 6. Comparison algorithm

### 6.1 Collation check and comparer detection

Identifier comparison follows the collation of the databases being compared; there is no fixed rule. Before any structural work begins:

1. Read the `CollationInfo` of both databases through `IMetadataProvider.GetDatabaseCollationAsync`, concurrently for the two sides.
2. Compare the two collations. **If they differ, throw `CollationMismatchException` carrying both names and stop.** No further metadata is read and no report is produced: when the two databases disagree on identifier equality there is no defensible answer to whether `dbo.Orders` on one side denotes the same object as `dbo.orders` on the other. Equality is decided on the whole record, not on `Name` alone — two collations of the same name but different `Version` sort differently and are not interchangeable.
3. Hand the agreed `CollationInfo` to `INameComparerResolver.ResolveNameComparer` (§3.3.2). The call is synchronous because the mapping is pure arithmetic over the record — it touches no database and cannot block. For MS SQL it turns the collation's ignore-flags into `CompareOptions` and asks the culture named by `Lcid` for a comparer (§3.4.2).
4. The returned instance is the single `StringComparer` used for every schema, table, field and key name for the rest of the run. It is resolved once per call and passed down into §6.2–§6.6. No code path may fall back to a hard-coded comparer.

Consequences the implementation must respect:

- Under a case-sensitive collation `dbo.Table1` and `dbo.table1` are two **distinct** tables: they occupy two entries in `tables.detected` and are matched independently. Any dictionary keyed by object name must therefore be built with the detected comparer — a hard-coded case-insensitive dictionary would raise a duplicate-key error or silently drop one of them. The same applies to two columns of one table differing only in case.
- The exclusion list is matched under the same resolved comparer. `options.UserExcludedTables` arrives as a plain list carrying no comparison rule, so once step 3 has settled the rule the comparer normalizes each entry through `TableIdentity.Parse` and builds `new HashSet<string>(normalizedNames, resolvedComparer)` to match against. Quoted entries are matched by their bare two-part name, so `[dbo].[Order.Archive]` and `dbo.Orders` both work; duplicates collapse in that set; and under a case-sensitive collation a caller can list both `dbo.Orders` and `dbo.orders` and have each exclude its own table.
- The comparison follows the collation's case, accent, kana and width rules, because all four reach the comparer through `ComparisonStyle`. Under an accent-insensitive collation `Café` and `Cafe` are therefore one identifier, as they are on the server. What is **not** reproduced is the collation's exact sort order (§3.4.2) — which is why report ordering is kept independent of this comparer (§4).

### 6.2 Building the table sets

1. Fetch the tables of both databases concurrently (`Task.WhenAll` over the two `GetTablesAsync` calls).
2. `tables.detected` = the distinct union of the left and right table names, compared with the detected comparer.
3. `tables.unchecked` = `detected ∩ options.UserExcludedTables`, matched on the bare two-part name with the comparer of §6.1.
4. `tables.ignored.left` = tables present in the **left** database, not excluded by the user, and **absent in the right** one. `tables.ignored.right` is the mirror image.
5. Comparison set = `detected − unchecked − ignored.left − ignored.right`.
6. A table that exists only in the left database is reported in `missings.right.tables`; a table that exists only in the right database is reported in `missings.left.tables`.
7. A table listed in `tables.unchecked` is never reported as missing or inconsistent, even when it exists in one database only.
8. Fields, primary keys and foreign keys are then fetched for the comparison set only, again concurrently for the two databases.

### 6.3 Field comparison

Within each table of the comparison set, fields are matched **by name**, using the comparer detected in §6.1. Ordinal position is **not** compared.

- Field present on the right only → `missings.left.fields`.
- Field present on the left only → `missings.right.fields`.
- Field present on both sides → the tuple `(type, size, nullable, precision, scale)` is compared. If any component differs, an entry is added to `inconsistencies.fields` carrying the full descriptor of both sides. Type names are compared case-insensitively.

### 6.4 Primary key comparison

SQL Server allows at most one primary key per table, so only the outcome matters. The order of the fields inside the key is **not** taken into account — the participating fields are compared as a set under the detected comparer.

| Left PK | Right PK | Fields | Report |
| --- | --- | --- | --- |
| exists | exists | same set | nothing |
| exists | exists | different set | `inconsistencies.primaryKeys` — both key names (for reference only) and both field lists |
| exists | absent | — | `missings.right.primaryKeys` with the left key's `key`, `table`, `fields` |
| absent | exists | — | `missings.left.primaryKeys` with the right key's `key`, `table`, `fields` |
| absent | absent | — | an entry containing **only** `table` in *both* `missings.left.primaryKeys` and `missings.right.primaryKeys` |

Key names are carried into the report for reference only and never drive the decision: two primary keys with the same field set but different names produce no entry.

### 6.5 Foreign key comparison

Foreign keys of a table are grouped by the pair `(table, referencedTable)`. Within each group the left and right keys are matched in the following order:

1. **Match by composition.** A left key and a right key match when their sets of `field` / `referencedField` pairs are equal, ignoring order and comparing the field names with the detected comparer. Such a pair produces **no** report entry. When several candidates exist, the one whose name also matches is preferred.
2. **Match by name.** Among the keys still unmatched, keys whose names are equal under the detected comparer are paired. The pair is reported in `inconsistencies.foreignKeys`.
3. **Match by field intersection.** The keys still unmatched are paired greedily by the largest intersection of their field-pair sets (ties broken by ordinal key-name order for determinism). Each such pair is reported in `inconsistencies.foreignKeys`.
4. **Remainder.** A left key that found no pair is reported in `missings.right.foreignKeys`; a right key that found no pair is reported in `missings.left.foreignKeys` — in both cases with `key`, `table`, `referencedTable` and the full `fields` list.

Step 2 preceding step 3 is what gives a name match priority over the similarity heuristic (see Use Case 8b).

**Key names are never a difference criterion.** They are used only to *pair* the keys of the two sides — as a tie-breaker in step 1 and as the pairing rule in step 2 — and are then reported for reference. A pair of keys whose field sets are equal produces no report entry no matter how their names differ. The source specification treats an explicit comparison by key name as an edge case and deliberately provides no option for it: where a name legitimately matters, it is already accounted for by steps 1 and 2 of the matching order.

### 6.6 Masking the password in the reported connection strings

The two connection strings the caller supplies in `TableStructureComparisonOptions` are used **only** to open connections — they are handed to `IMetadataProvider` unchanged. What the comparer copies into `TableStructureComparisonResult.Left` and `.Right` is a masked form:

1. Parse the original with `SqlConnectionStringBuilder`.
2. If a password is present — under the `Password` keyword or its `PWD` alias — replace its **value** with the literal `***` (`builder.Password = "***"`). The keyword itself is kept, so the report still shows that a password was supplied; only the secret is gone. The mask is the exact three-character string `***`, never a variable-length run derived from the real password. The builder normalizes the alias, so a caller's `PWD=…` is reported as `Password=***`. An empty password is treated as no password: the keyword is dropped, since there is no secret to mask.
3. Take the builder's `ConnectionString` as the reported value. Every other keyword — `Server`, `Database`, `User ID`, `Integrated Security`, `Encrypt`, application name, timeouts and the rest — is preserved untouched, so the report still identifies which databases were compared. Note that the builder emits **canonical** keyword names: `Server` comes back as `Data Source`, `Database` as `Initial Catalog`. The reported string is therefore equivalent to the caller's input but not textually identical to it, and consumers must not expect to match it character-for-character against the string they passed in.

The masked value is produced once, before the result object is populated, so the plaintext password is never assigned to a result field, never serialized, and never reachable through the returned object. Serialization writes these members through unchanged: the JSON contains the same `Password=***` form the result object holds.

A string that cannot be parsed by `SqlConnectionStringBuilder` is reported as the empty string rather than verbatim — the library must never fall back to echoing an unparsed string that might still carry a secret. This case is reachable in practice: a password containing a `;` makes the connection string unparseable unless the caller quoted the value (`Password="P@ss;w0rd!"`), and an unquoted one raises `ArgumentException` inside the builder. The comparison itself still succeeds — only the reported connection string is blanked.

A connection string that carries no password — integrated security, for instance — gains no `Password` keyword: there is nothing to mask, and the string passes through with only the builder's keyword normalization applied.

### 6.7 Determinism and identity handling

- Table, field and key names are compared with the comparer detected in §6.1, never with a hard-coded one, and always through `NameIdentity.Name` — the bare, unquoted form, which is independent of the quoting policy in `TableStructureComparisonOptions`. Comparing `QuotedName` or `PreferredName` would make `[Orders]` and `Orders` two different tables.
- The casing rendered in the report is the casing returned by the **left** database when the object exists there, otherwise the casing returned by the right database.
- Because a report name is the concatenation `schema + "." + table`, an identifier that itself contains a dot yields an ambiguous string. Such names are emitted verbatim, without quoting (see **§11**).

---

## 7. Acceptance scenarios

The following use cases are normative. Each must be covered by at least one automated test.

| # | Scenario | Expected report location |
| --- | --- | --- |
| 1 | `dbo.Orders` exists in one database only | `missings.<opposite side>.tables` **and** `tables.ignored.<side where it exists>` |
| 2 | Field `Sum` of `dbo.Orders` exists in one database only | `missings.<opposite side>.fields` → `{ "field": "Sum", "table": "dbo.Orders" }` |
| 3 | `PK_Orders` on `dbo.Orders` exists in one database only | `missings.<opposite side>.primaryKeys` → `{ "key": "PK_Orders", "table": "dbo.Orders", "fields": ["OrderId"] }` |
| 4 | `FK_Orders_Customers` on `dbo.Orders` → `dbo.Customers` exists in one database only | `missings.<opposite side>.foreignKeys` with `key`, `table`, `referencedTable`, `fields` |
| 5 | Field `Description` differs in type / size / nullable | `inconsistencies.fields` with the full left and right descriptors |
| 6 | `PK_Orders (OrderId, CustomerId)` versus `PKOrders_123 (OrderId)` | `inconsistencies.primaryKeys`. The key names are reported but did not cause the entry — the differing field set did. Identical field sets with different names produce **no** entry. |
| 7 | `dbo.Customers` has no primary key in either database | An entry containing only `table` in **both** `missings.left.primaryKeys` and `missings.right.primaryKeys` |
| 8a | `FK_Orders_Customers (CustomerId, DepartmentId)` on the left versus `FK_Ord_Cust_123 (CustomerId, CreatedOn)` on the right, no other candidates | `inconsistencies.foreignKeys` — a single entry pairing the two keys inside `inconsistency` |
| 8b | Same as 8a, but the right database *also* has `FK_Orders_Customers` on the same table and referenced table | The name match wins: `FK_Orders_Customers` ↔ `FK_Orders_Customers` goes to `inconsistencies.foreignKeys`, and the left-over right-side `FK_Ord_Cust_123` goes to `missings.left.foreignKeys` |

Additional required regression cases:

- Two identical databases produce an empty `difference.structure` with every section present.
- A user-excluded table never appears anywhere in `difference`.
- A primary key whose fields are declared in a different order on the two sides produces **no** entry.
- A foreign key whose field pairs are declared in a different order on the two sides produces **no** entry.
- On a case-sensitive collation, excluding `dbo.orders` leaves `dbo.Orders` compared and absent from `tables.unchecked`, while listing both spellings excludes both; on a case-insensitive collation the single entry excludes the table either way.
- On a case-sensitive collation, `dbo.Table1` and `dbo.table1` present in both databases are reported as two independent tables, and a `Description` column alongside a `description` column is likewise treated as two fields.
- Two databases whose collations differ raise `CollationMismatchException`, and no report is returned.
- A comparison run with a password in both connection strings produces a result whose `Left` and `Right` members, and whose serialized JSON, show the keyword with the masked value (`Password=***`) and nowhere contain the real password.

---

## 8. Error handling, cancellation, safety

- **Argument validation.** A `null` `options` value, or a `null`/empty connection string on either side, throws `ArgumentNullException` / `ArgumentException` synchronously, before any I/O starts.
- **Collation mismatch.** When the two databases do not share the same collation, `CollationMismatchException` is thrown (§6.1). It carries both collation names and reports which side had which. It is raised before any table metadata is read, so the call fails fast and returns no partial report.
- **Connectivity and permission failures.** A `SqlException` raised while reading metadata is wrapped in a dedicated `MetadataAccessException` that carries the affected side (`Left` / `Right`) and the original exception as `InnerException`. A partial report is never returned — a failure on either side fails the whole call.
- **Cancellation.** `cancellationToken` is threaded through every provider call and every ADO.NET call. Cancellation surfaces as `OperationCanceledException`; no partial result is produced.
- **Read-only guarantee.** The library issues `SELECT` statements against catalog views only. Callers are advised to use a login that has `VIEW DEFINITION` and no write permissions.
- **No credential leakage.** The result object and its JSON carry connection strings whose password is masked as `***` (§6.6), so a report may be logged, persisted or forwarded without exposing a secret. Exception messages produced by the library must not embed a raw connection string either; `MetadataAccessException` identifies the failing side by name, not by connection string.

---

## 9. Non-functional requirements

- **Round trips.** Exactly five queries per database (collation, tables, fields, primary keys, foreign keys). No per-table queries. The collation query runs first and gates the other four; resolving the comparer from it costs no round trip at all, since `INameComparerResolver` is a pure mapping. The four metadata statements are sent as self-composing batches so that the shadow-table capability probe of §3.4.1 rides along inside them — the count of five stands.
- **Parallelism.** Left and right metadata are read concurrently.
- **Throughput target.** A pair of databases with 1,000 tables and 20,000 columns is compared in under 10 seconds over a LAN, excluding SQL Server response time.
- **Memory.** Metadata is held in memory for the duration of one comparison; the working set stays proportional to the metadata size, with no duplication of the raw reader output.
- **Thread safety.** `TableStructureComparer` and `MsSqlMetadataProvider` are stateless and safe for concurrent use.
- **Async discipline.** No `.Result`, no `.Wait()`, and no `Task.Run` wrappers around synchronous I/O anywhere in the library.

---

## 10. Testing requirements

1. **Unit tests of the comparer** against a fake `IMetadataProvider`, covering every row of the table in §7 plus the listed regression cases. These require no SQL Server instance and must run in CI.
2. **Golden-file tests** asserting the exact serialized JSON for a representative scenario, protecting the report contract against accidental change.
3. **Integration tests of `MsSqlMetadataProvider`** against a real MS SQL Server instance (LocalDB or a container), created from a pair of setup scripts that materialize the eight use cases.
4. **Cancellation tests** verifying that a token cancelled mid-comparison propagates `OperationCanceledException`.
5. **Comparer-resolution tests** — `MsSqlNameComparerResolver` over a table of `CollationInfo` values covering case-sensitive, case-insensitive, accent-insensitive, width-insensitive and binary collations, asserting for each which pairs of identifiers the returned comparer treats as equal; an unknown `Lcid` and an unknown `ComparisonStyle` bit raising `NotSupportedException`; two databases whose `CollationInfo` differs — by name and by `Version` alone — raising `CollationMismatchException` before any table metadata is read; a case-sensitive fixture holding two tables that differ only in case; and exclusion matching under both a case-sensitive and a case-insensitive comparer.
6. **`TableIdentity.Parse` / `TryParse` tests** covering each branch of the grammar: plain two-part names, bracket-quoted parts, a dot inside brackets, an escaped `]]`, a missing dot, an empty part, unbalanced brackets, an ambiguous unquoted name with two dots, and `null` — asserting that `Parse` throws where `TryParse` returns `false`.
7. **Credential-masking tests** covering SQL authentication, integrated security, a `PWD` alias, and a malformed connection string — asserting in each case that the real password appears neither in the result members nor in the serialized JSON, and that a supplied password is reported as `***` while integrated security gains no password keyword.
8. **Shadow-table filtering tests** for the exclusion of §1.3, run as integration tests against a real instance, since the filter lives in the provider's SQL and a fake `IMetadataProvider` cannot exercise it. The fixtures must cover:
   - **The auto-named case, which is the reason the filter exists.** Two databases each carrying a system-versioned `dbo.Orders` declared with `WITH (SYSTEM_VERSIONING = ON)` — no `HISTORY_TABLE` clause, so each server mints its own `MSSQL_TemporalHistoryFor_<object_id>`. The two names must differ (assert this in the fixture itself, otherwise the test is vacuous), and the comparison must still yield an **empty `difference.structure`** with every section present. Without the filter this case produces two spurious `tables.ignored` entries and two spurious `missings.*.tables` entries, so it is the regression this test guards.
   - **The explicitly named case.** `WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.OrdersHistory))` on both sides: `dbo.OrdersHistory` is absent from `tables.detected` even though its name is identical on the two sides and it would otherwise have compared cleanly. The filter keys on `temporal_type`, not on the name.
   - **The period columns are still compared.** `dbo.Orders` system-versioned on the left and a plain table with the same business columns on the right: `ValidFrom` and `ValidTo` appear in `missings.right.fields`. This is what makes the exclusion lossless, and it must hold whether or not the columns are declared `HIDDEN` — `sys.columns` reports them either way.
   - **The filter releases the table when versioning ends.** After `ALTER TABLE dbo.Orders SET (SYSTEM_VERSIONING = OFF)`, the former history table is an ordinary user table and must reappear in `tables.detected` and be compared normally.
   - **Ledger, on SQL Server 2022 and later.** An updatable ledger table's history table and the remains of a dropped ledger table are both absent from `tables.detected`; the four generated `ledger_*` columns of the ledger table itself are compared as ordinary fields. These cases are skipped — not failed — when the instance under test predates SQL Server 2022, gated on the same `COL_LENGTH` probe the provider uses, so the suite stays green on a 2016–2019 instance.
   - **The capability probe itself.** Against a pre-2022 instance, all four metadata statements execute without a parse error and the temporal cases above still pass; against a 2022+ instance the ledger predicates are in force. Assert on both instances that the run still costs exactly five queries per database (§9), so that the self-composing batch has not silently become a second round trip.

---

## 11. Assumptions and open questions

The points below are not fully determined by the source requirements. The stated resolution is what the implementation will follow unless the customer decides otherwise.

1. **Tie-breaking in step 3 of §6.5.** Pairing the remainder by the largest field-set intersection is prescribed by the specification, but the specification does not say how to break a tie when several candidates share the same intersection size. The implementation will pair them in ordinal key-name order, purely to keep the report deterministic. Please confirm this rule.
2. **Sort order is approximated, equality is not.** `CollationInfo` carries enough to reproduce *which identifiers are equal* under any collation, and the resolver does so. It does not reproduce SQL Server's exact *ordering*, and for the legacy `SQL_*` collations no .NET comparer can. The report therefore orders ordinally (§4), which is deterministic but not the server's order. Please confirm that report ordering need not match `ORDER BY` on the server.

---

## 12. Definition of done

- `Diff.Structure.dll` targets .NET 10 and builds with no warnings under `TreatWarningsAsErrors`.
- The public API matches §3 exactly, including the required names `TableStructureComparer`, `CompareTableStructureAsync`, `TableStructureComparisonOptions`, `TableStructureComparisonResult`, `IMetadataProvider`, `MsSqlMetadataProvider`, and the `ToJson` extension declared by `ObjectExtensions` in `ObjectExtensions.cs`.
- `TableStructureComparer` contains no ADO.NET references.
- Identifier comparison follows the comparer `INameComparerResolver` builds from the databases' `CollationInfo`, honouring its case, accent, kana and width rules; two sides whose collations differ fail the call with `CollationMismatchException` instead of producing a report.
- Neither `TableStructureComparisonResult` nor its serialized JSON contains a plaintext password in any supported authentication mode; a supplied password is reported as `***`.
- All eight use cases of §7, plus the listed regression cases, pass as automated tests.
- The serialized output matches the schema in §5, with every section always present.
- Public types and members carry XML documentation comments, and the assembly ships an XML documentation file.
- A `README.md` shows a minimal end-to-end usage sample:

```csharp
var leftConnectionString = "...";
var rightConnectionString = "...";

var provider = new MsSqlMetadataProvider();
var comparer = new TableStructureComparer(provider, new MsSqlNameComparerResolver());

var options = new TableStructureComparisonOptions
{
    ConnectionStrings = (leftConnectionString, rightConnectionString),
    UserExcludedTables = ["dbo.__EFMigrationsHistory"],
    QuoteInfo = QuoteInfo.MsSql,
    QuotesUsage = QuotesUsage.DoNotUseIfPossible,
};

TableStructureComparisonResult result =
    await comparer.CompareTableStructureAsync(options, cancellationToken);

string json = result.ToJson();
```
