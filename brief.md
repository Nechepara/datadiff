# Development Brief — `Diff.Structure.dll`

**Product:** `Diff.Structure` — a .NET class library that compares the *structure* of same-named tables taken from two different MS SQL Server databases and returns the result as a JSON report.

**Target platform:** .NET 10, C# (latest language version). `async/await` is used everywhere an asynchronous path exists.

**Deliverable:** a single assembly `Diff.Structure.dll` plus a test project `Diff.Structure.Tests.dll`.

---

## 1. Purpose and scope

### 1.1 Purpose

Given two MS SQL Server connection strings (`left` and `right`), the library must:

1. Read the metadata of both databases (tables, fields, primary keys, foreign keys).
2. Determine which tables can be compared, which have to be skipped by the system, and which the caller excluded.
3. Compare the structure of every comparable table pair.
4. Produce a single machine-readable result object that serializes to the JSON layout defined in **[[#5. Report structure|§5]]**.

A pair of same-named tables is considered **structurally equal** when all of the following hold:

- The two tables have the same set of fields, and every same-named field has the same **data type**, **size**, **nullable option**, **precision** and **scale**.
- The two tables have the same primary key — the same set of participating fields.
- The two tables have the same set of foreign keys — for every key the same referenced table and the same set of `field` → `referencedField` pairs.

If at least one of these conditions is violated, the violation must be reflected in the `difference.structure` section of the report.

### 1.2 In scope

- Tables, fields (columns), primary keys, foreign keys.
- Schema-qualified table identity (`schema.table`).
- A caller-supplied collation that overrides the databases' own as the identifier-comparison rule.
- Caller-supplied conflict resolutions: one mechanism that excludes an object from the comparison, pairs two names an overriding collation can no longer tell apart, and pairs two objects the databases spell differently — a table or a field renamed on one side.
- Cancellation through `CancellationToken` on every asynchronous operation.

### 1.3 Out of scope (v1)

The following objects are **not** compared and must not appear in the report: views, stored procedures, functions, triggers, indexes (other than the index backing a primary key), unique constraints, check constraints, default constraints, computed-column definitions, identity/seed settings, collations, filegroups, partitioning, extended properties, and table data. `ON DELETE` / `ON UPDATE` actions of a foreign key and the enabled/trusted state of a constraint are not compared either.

**Engine-generated shadow tables are silently ignored.** The history table of a system-versioned (temporal) table, the history table of an updatable ledger table, and the retained remains of a dropped ledger table or column are excluded from the run entirely: they never reach `tables.detected`, never appear in `tables.ignored` or `tables.unchecked`, and can never produce an entry under `difference`. The report gives no indication that such a table exists.

Two reasons make the exclusion safe rather than lossy. First, no information is lost: while versioning is on, SQL Server itself forbids altering a history table independently of its current table, so the history table's field set is a guaranteed mirror of one that *is* compared — reporting it would only duplicate every finding. The fact that a table is versioned at all remains observable through the generated columns on the current table (`ValidFrom` / `ValidTo`, or the four `ledger_*` columns), which are compared as ordinary fields; a table versioned on one side and not on the other therefore still surfaces, as those columns landing in `missings.<opposite side>.fields`. Second, without the exclusion the report would be wrong: a history table whose name the developer did not specify is named after the **object id** of its current table (`MSSQL_TemporalHistoryFor_<object_id>`, `MSSQL_LedgerHistoryFor_<object_id>`), and object ids are allocated per database, so two structurally identical databases yield two different names and thus a spurious entry in `tables.ignored` and `missings.*.tables`. The remains of dropped ledger objects are worse still — their names embed a GUID minted at drop time, they accumulate for the life of the database, and a caller cannot write an exclusion for a name no one can predict.

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
├── MsSqlMetadataQueryBuilder         // composes the MS SQL metadata queries from the options
├── DatabaseOptions                   // engine-neutral database options (collation)
├── MsSqlDatabaseOptions              // + MS SQL capabilities (ledger tables)
├── CollationInfo                     // a collation and its comparison flags — read from a database or given by the caller
├── INameComparerResolver             // identifier-comparison rule contract
├── MsSqlNameComparerResolver         // collation -> StringComparer for MS SQL Server
├── QuotesUsage / Quotes / QuoteInfo  // identifier-quoting policy (§3.3)
├── INestedObject                     // an object and what owns it (§3.3.1)
├── IQuotedIdentifier / QuotedIdentifier
│                                     // an identifier and its three renderings (§3.3.1)
├── SchemaIdentity / TableIdentity /
│   FieldIdentity / PrimaryKeyIdentity /
│   ForeignKeyIdentity                // the named objects of a database
├── ObjectKind / CandidateInfo        // names the run's collation cannot tell apart (§3.1.1)
├── ConflictResolution                // the caller's exclusions and pairings — internal
│                                     // constructor, never built by the caller (§3.1.1)
├── From                              // which database an entry speaks about (§3.1.1)
├── ConflictResolutionExtensions      // Ignore() / Map() — the only way a caller
│                                     // produces a ConflictResolution (§3.1.1)
├── FieldMetadata / PrimaryKeyMetadata /
│   ForeignKeyMetadata / ForeignKeyColumn
│                                     // what those objects are made of
└── ObjectExtensions                  // ToJson() extension — ObjectExtensions.cs

One public type per file, named after the type. `Quotes` and `QuoteInfo` are `partial`:
the engine-neutral half lives in Quotes.cs / QuoteInfo.cs, the MS SQL specifics
(the bracket pair and the "does this identifier need quoting" predicate) in
MsSqlQuotes.cs / MsSqlQuoteInfo.cs, so a second engine is added without touching
the neutral half.
```

**Separation of concerns (hard requirement).** `TableStructureComparer` never touches ADO.NET, never opens a connection and never executes SQL. All metadata retrieval is delegated to `IMetadataProvider`. This keeps the comparison logic pure, makes it unit-testable against a fake provider, and allows providers for other engines to be added later without touching the comparer.

The comparer holds no SQL either, and never sees any: `MsSqlMetadataQueryBuilder` is created and used entirely inside `MsSqlMetadataProvider` ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]) and is invisible above it. The one thing the comparer carries between provider calls is the `DatabaseOptions` object of [[#3.3.1 `IMetadataProvider`|§3.3.1]] — an opaque description of a database, not a statement. It never inspects that object beyond `Collation` — which it reads on every run, to settle the comparison rule or to validate the caller's `PreferredCollation` against it ([[#3.1 `TableStructureComparisonOptions`|§3.1]], [[#6.1 Database options, collation settlement and comparer detection|§6.1]]) — and never downcasts it, so it stays engine-neutral even though the object it passes around may be an MS SQL one.

**A second split, inside the provider.** SQL text is composed by `MsSqlMetadataQueryBuilder` ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]) and executed by `MsSqlMetadataProvider` ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]). The two concerns are genuinely different: what to say to a database is a pure function of what that database supports, while how to say it is connections, parameters and readers. Separating them lets the first be unit-tested by asserting on a string, with no server of any version present — which is the whole reason the split exists, since the capability the queries branch on differs between SQL Server versions and the wrong branch fails at parse time rather than at run time ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]).

**Dependencies.** `Microsoft.Data.SqlClient` (referenced only by `MsSqlMetadataProvider`) and `System.Text.Json`. No other third-party dependencies.

---

## 3. Public API

### 3.1 `TableStructureComparisonOptions`

```csharp
public sealed class TableStructureComparisonOptions
{
    /// <summary>Connection strings of the two databases being compared.</summary>
    public (string Left, string Right) ConnectionStrings { get; init; }

    /// <summary>Quoting rules applied to every identifier the run produces.</summary>
    public QuoteInfo QuoteInfo { get; init; } = QuoteInfo.MsSql;

    /// <summary>How the report renders identifiers — see §3.3.1.</summary>
    public QuotesUsage QuotesUsage { get; init; } = QuotesUsage.DoNotUseIfPossible;

    /// <summary>The collation that decides identifier equality for this run, overriding the
    /// collations of the two databases. When <c>null</c> (the default) the run reads both
    /// databases' collations, requires them to be equal, and compares under that one; when
    /// set, the two are no longer required to agree, but the value must equal the collation
    /// of at least one of them or the run fails with <see cref="InvalidCollationException"/>
    /// — see §6.1.</summary>
    public CollationInfo? PreferredCollation { get; init; } = null;

    /// <summary>The caller's decisions about objects the library cannot pair on its own:
    /// a table excluded from the comparison, two names the run's collation cannot tell
    /// apart, or two objects the two databases spell differently because one of them was
    /// renamed. The property is <c>init</c>-only and defaults to an empty set; its contents
    /// are filled only through the <c>Ignore</c> and <c>Map</c> extension methods of
    /// <see cref="ConflictResolutionExtensions"/>, since <see cref="ConflictResolution"/>
    /// has no public constructor — see §3.1.1 and §6.2.1.</summary>
    public HashSet<ConflictResolution> ConflictResolution { get; init; } = new();
}
```

`PreferredCollation` is the one comparison rule the options may carry, and it is opt-in. Left `null` — the default, and what a caller who does not know the property exists gets — the run behaves exactly as [[#6.1 Database options, collation settlement and comparer detection|§6.1]] has always specified: both databases' collations are read, they must be equal, and the agreed one decides identifier equality. Set to a `CollationInfo`, it lifts the requirement that the two agree: their collations are no longer compared with each other, `CollationMismatchException` cannot be raised, and the supplied record is what `INameComparerResolver` ([[#3.3.2 `INameComparerResolver`|§3.3.2]]) turns into the run's comparer. That holds whether the databases disagree on collation or share one — a preferred collation equal to what both already use still governs, because the property's point is that the caller, not the servers, decides.

What the property does **not** grant is a free choice of rule. The supplied value must equal the collation of at least one of the two databases, or the run fails with `InvalidCollationException` ([[#6.1 Database options, collation settlement and comparer detection|§6.1]], [[#8. Error handling, cancellation, safety|§8]]). A preferred collation therefore chooses which of the two databases' rules governs the comparison when the two disagree — that, and nothing more. The library will compare a case-sensitive database against a case-insensitive one under either side's rule at the caller's direction; it will not compare them under a third rule neither server holds.

The override is a deliberate loosening of a guarantee rather than a convenience. Under the default the library can say that its notion of *same identifier* is the notion both servers hold; under a preferred collation it can only say that it is the caller's. Setting the property is the caller asserting that comparing two databases under one of their two rules is the answer they want — the case that motivates it is a legacy database compared against its replacement, where the collations genuinely differ and a structural diff is still worth having. The choice is not hidden from the reader of the report: the collation the run was performed under is written to its `collation` section ([[#4. Result model|§4]], [[#5. Report structure|§5]]), on every run and whatever its source, so a report is never ambiguous about the rule that produced it.

`ConflictResolution` is the one place where the caller overrides what the library would decide for itself, and it carries every such override — exclusion and pairing alike — in a single collection ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). There is no separate list of excluded table names: an exclusion is an entry whose opposite side is `null`, which is the same statement in the same vocabulary as a pairing, and both are matched against the databases the same way. Left empty — the default — nothing is excluded and nothing is paired by hand.

The caller never constructs an entry. `ConflictResolution`'s constructor is `internal`, and the collection is filled by the four extension methods of `ConflictResolutionExtensions` — `Ignore(table, from)`, `Ignore(table)`, `Map(leftTable, rightTable)` and `Map(leftField, rightField)` ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). They read as the four statements a caller actually wants to make, and between them they fix what the mechanism can express: a table is excluded from one side or from both, a table is paired with a table, a field is paired with a field, and nothing else is sayable.

Exclusions and pairings name objects rather than strings, so a table is written as a `TableIdentity` and a field as a `FieldIdentity`. `TableIdentity.Parse` is the string entry point ([[#3.3.1 `IMetadataProvider`|§3.3.1]]): a caller writes `TableIdentity.Parse("dbo.__EFMigrationsHistory", quotesUsage, quoteInfo)` and may quote either part (`[dbo].[Order.Archive]`), since the parser matches on the bare two-part name and does not mistake a dot inside a quoted identifier for the separator. A field is then built against the table it belongs to, `new FieldIdentity(table, "ModifiedOn")`, which is what gives a field entry the owner chain it needs to be unambiguous.

`QuoteInfo` and `QuotesUsage` carry the identifier-quoting policy for the whole run: the comparer builds every `SchemaIdentity` with them and every identifier below a schema inherits them through its owner, so nothing consults an ambient default. Both are `init`-only, so the policy cannot change while a comparison is in flight — a mutable setter would let one report mix two renderings. `QuoteInfo` defaults to `QuoteInfo.MsSql`, `QuotesUsage` to `DoNotUseIfPossible`, so a caller who does not care about quoting sets neither.

### 3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`

Three things the library cannot settle on its own are settled by the caller, and all three are the same statement: *this object is not to be compared*, *these two names are one object*, *these two differently-spelled objects are one object*. A collation chosen by the caller can be looser than the one a database uses, and then two names that are distinct objects on their own server become one name to this run; a table or a field may have been renamed on one side, so the two databases spell one object two ways; and a table may simply be of no interest — a migration-history table, a staging table — and have no business in the report at all. One type carries all three answers, and a closed set of four factory methods is the only way to produce it.

```csharp
//file ObjectKind.cs
public enum ObjectKind { Table, Field, PrimaryKey, ForeignKey }

//file CandidateInfo.cs
public class CandidateInfo
{
    /// <summary>What kind of object the colliding names denote.</summary>
    public ObjectKind Kind { get; init; }

    /// <summary>The objects in the LEFT database whose names the run's collation cannot tell
    /// apart, e.g. 'dbo.Orders' and 'dbo.orders'.</summary>
    public HashSet<INestedObject> Left { get; init; }

    /// <summary>The same class of names as found in the RIGHT database.</summary>
    public HashSet<INestedObject> Right { get; init; }
}

//file ConflictResolution.cs
public sealed record ConflictResolution
{
    /// <summary>Not part of the public surface. Entries are produced by the Ignore and Map
    /// extension methods below and by nothing else.</summary>
    internal ConflictResolution(INestedObject? left, INestedObject? right, ObjectKind kind);

    /// <summary>The object of the LEFT database this entry speaks about, or <c>null</c>
    /// when the entry excludes a table of the right database.</summary>
    public INestedObject? Left { get; }

    /// <summary>The object of the RIGHT database, or <c>null</c> when the entry excludes a
    /// table of the left one.</summary>
    public INestedObject? Right { get; }

    /// <summary>What kind of object both sides denote.</summary>
    public ObjectKind Kind { get; }
}

//file From.cs
/// <summary>Which of the two databases an entry speaks about.</summary>
public enum From
{
    Left = 1,
    Right = 2,
}

//file ConflictResolutionExtensions.cs
public static class ConflictResolutionExtensions
{
    /// <summary>Adds an entry excluding <paramref name="table"/> of the database named by
    /// <paramref name="from"/> from the comparison. Returns the same collection.</summary>
    public static ICollection<ConflictResolution> Ignore(
        this ICollection<ConflictResolution> resolution, TableIdentity table, From @from);

    /// <summary>Adds the two entries that exclude <paramref name="table"/> from both
    /// databases — the overload above, once per side. Returns the same collection.</summary>
    public static ICollection<ConflictResolution> Ignore(
        this ICollection<ConflictResolution> resolution, TableIdentity table);

    /// <summary>Adds an entry pairing a table of the left database with a table of the
    /// right one, however the two are spelled. Returns the same collection.</summary>
    public static ICollection<ConflictResolution> Map(
        this ICollection<ConflictResolution> resolution, TableIdentity left, TableIdentity right);

    /// <summary>Adds an entry pairing a field of the left database with a field of the right
    /// one; each side hangs from its own database's spelling of the owning table. Returns the
    /// same collection.</summary>
    public static ICollection<ConflictResolution> Map(
        this ICollection<ConflictResolution> resolution, FieldIdentity left, FieldIdentity right);
}
```

**Entries are built, never constructed.** The constructor is `internal`, the three properties are get-only, and the record is `sealed`, so a caller cannot reach an entry through `new`, through an object initializer, or through a `with` expression — the four extension methods above are the whole surface. `Ignore` produces a one-sided entry of `Kind = Table`; the two-argument overload produces the two of them, one per side, and is exactly `resolution.Ignore(table, From.Left).Ignore(table, From.Right)`. `Map` produces a two-sided entry, of `Kind = Table` for its table overload and `Kind = Field` for its field one. Each method **adds to the collection it is called on** and returns that same instance, so calls chain (`options.ConflictResolution.Ignore(audit).Map(leftOrders, rightOrders)`) and a caller who ignores the return value loses nothing.

The methods extend `ICollection<ConflictResolution>` rather than `IEnumerable<ConflictResolution>`, and the narrower receiver is the point: the operation they perform is an addition, and only `ICollection<>` declares one. A caller who reaches for a sequence that cannot be added to — the result of a LINQ operator, say — gets a compile error at the call site rather than a `NotSupportedException` at run time, and nothing a caller is meant to do is lost, since `TableStructureComparisonOptions.ConflictResolution` is a `HashSet<ConflictResolution>` and binds without the caller ever naming the concrete type.

`ICollection<>` still admits one shape that cannot be added to — an implementation whose `IsReadOnly` is `true`, such as an array or a `ReadOnlyCollection<>` — so each method checks that property and fails with `ArgumentException` rather than letting the underlying `Add` decide how to complain. A `null` receiver and a `null` identity fail with `ArgumentNullException`, and a `From` value that is neither `Left` nor `Right` with `ArgumentOutOfRangeException`. All four checks are pure, run before anything is added, and cost no round trip.

Adding the same statement twice is harmless: the collection is a `HashSet<ConflictResolution>` and the record's value equality makes the second `Ignore(orders, From.Left)` a no-op. Two entries that are *not* equal as records but claim the same object — the same table written once as a `TableIdentity` built under `QuotesUsage.Required` and once under `DoNotUseIfPossible`, say — both survive into the set and are caught by the same-object check below, as the contradiction they are.

**What can and cannot be said.** The four methods are the definition of the mechanism's reach, and two consequences follow directly:

- **Exclusion is for tables only.** There is no `Ignore` overload taking a `FieldIdentity`, a `PrimaryKeyIdentity` or a `ForeignKeyIdentity`, so a one-sided entry of any kind other than `Table` cannot exist. A caller who does not want a field compared excludes the table it belongs to, and the report says so in `tables.unchecked`; there is no way to take a member out of the comparison without leaving that trace ([[#5.1 Rules that govern the report|§5.1]]).
- **Pairing is for tables and fields only.** `Kind = PrimaryKey` and `Kind = ForeignKey` stay in `ObjectKind` because `CandidateInfo` reports collisions of those kinds ([[#6.2.1 Conflict analysis|§6.2.1]]), but no entry can carry them. A key collision is answered by excluding the table that holds the keys, which removes the class from the analysis altogether; it cannot be answered by pairing. The case is narrow — a table has at most one primary key, so only foreign keys can collide, and only under a `PreferredCollation` looser than the database's own.

**How an entry is read.** `Left` names an object in the left database, `Right` an object in the right one, and `Kind` says what kind of object both denote. Either side may be `null`, and which of them is decides what the entry means. Each row below names the call that produces it:

| `Left` | `Right` | `Kind` | Produced by | Interpretation |
| --- | --- | --- | --- | --- |
| refers to `dbo.Table1` | `null` | `Table` | `Ignore(table1, From.Left)` | The table `dbo.Table1` of the **left** database is excluded from the structure comparison. |
| `null` | refers to `dbo.Table1` | `Table` | `Ignore(table1, From.Right)` | The table `dbo.Table1` of the **right** database is excluded from the structure comparison. |
| *the two rows above, together* | | `Table` | `Ignore(table1)` | The table `dbo.Table1` is excluded from **both** databases. The one-argument overload is a shorthand for the two entries above, not a fifth shape of its own. |
| refers to `dbo.Table2` | refers to `dbo.table2` | `Table` | `Map(leftTable2, rightTable2)` | The left database's `dbo.Table2` is compared against the right database's `dbo.table2`. This is the answer to a collision the run's collation created — two spellings the library could not pair on its own ([[#6.2.1 Conflict analysis|§6.2.1]]). |
| refers to `dbo.Orders` | refers to `dbo.CustomerOrders` | `Table` | `Map(leftOrders, rightOrders)` | The left database's `dbo.Orders` is compared against the right database's `dbo.CustomerOrders`. This is the answer to a rename: one table under two names, which no comparison rule could ever pair. |
| refers to field `ModifiedOn` of `dbo.Tasks` | refers to field `Modified` of `dbo.Tasks` | `Field` | `Map(leftField, rightField)` | Field `ModifiedOn` of the left database's `dbo.Tasks` is compared against field `Modified` of the right database's `dbo.Tasks`. The same case one level down: a field renamed on one side. |
| `null` | `null` | any | — | Not a statement about anything, and unreachable: no factory method produces it. Rejected if it arrives anyway — see below. |

Those are all the shapes there are. A one-sided entry of `Kind = Field`, and an entry of `Kind = PrimaryKey` or `Kind = ForeignKey` in any shape, have no factory method and therefore no existence.

**Exclusion is one-sided by construction, and that is the point.** A table lives in one database, and the reason to exclude it — it is EF's migration history, it is a staging table, its two names folded into one ([[#6.2 Building the table sets|§6.2]] step 5) — is a fact about that database, not about the pair. `Ignore(table, From.Left)` says nothing about the right, and a caller who wants the table gone from both sides says `Ignore(table)`, which writes the two entries out rather than folding them into one. This is deliberate: under a case-sensitive collation the two databases may hold `dbo.Audit` and `dbo.audit`, and one entry that quietly matched both would be guessing. The shorthand does not change that — it produces two entries carrying the same spelling, which is the statement *this exact name, on either side*, and a caller whose two databases spell the table differently still calls `Ignore` twice, once per spelling.

**Pairing is two-sided, and it decides the pairing outright.** An entry with both sides set replaces whatever the run's comparer would have said about those two objects, for the whole run. The two objects are then one object: they contribute one entry to `tables.detected`, they are compared against each other, and neither can appear in `tables.ignored` or in `missings`. A forced pairing also **consumes** both objects — if the left database also holds a `dbo.CustomerOrders` of its own, that table has lost its partner to the entry and becomes one-sided like any other unmatched table.

**In use.** The four statements, written out. Each begins from a `SchemaIdentity`, which is the one place the quoting policy enters the graph ([[#3.3.1 `IMetadataProvider`|§3.3.1]]); `TableIdentity.Parse` is the shorthand for the same thing when the name is already a string.

```csharp
// 1.1 — exclude a table that exists in the left database only.
var options = new TableStructureComparisonOptions();
var dbo = new SchemaIdentity("dbo", QuotesUsage.DoNotUseIfPossible, QuoteInfo.MsSql);
var orders = new TableIdentity(dbo, "Orders");
options.ConflictResolution.Ignore(orders, From.Left);   // From.Right for the other side

// 1.2 — exclude a table both databases hold and neither should have compared.
options.ConflictResolution.Ignore(orders);              // the two entries of 1.1 at once

// 2 — the table is named differently on the two sides but is one table.
var leftOrders = new TableIdentity(dbo, "Orders");
var rightOrders = new TableIdentity(dbo, "CustomerOrders");
options.ConflictResolution.Map(leftOrders, rightOrders);

// 3 — a field of dbo.Orders was renamed on the right.
var leftField = new FieldIdentity(orders, "ModifiedOn");
var rightField = new FieldIdentity(orders, "Modified");
options.ConflictResolution.Map(leftField, rightField);
```

Sample 3 builds both `FieldIdentity` objects against one `TableIdentity` because that table is spelled the same in both databases. When it is not — when the field's table is itself paired by a `Map` of sample 2 — each side of the field entry hangs from its own database's table identity, `new FieldIdentity(leftOrders, "ModifiedOn")` against `new FieldIdentity(rightOrders, "Modified")`, for the reason the owner-chain paragraph below gives.

**An object is named by at most one entry per side.** Two entries claiming the same left object, or the same right one, are contradictory and the library does not choose between them: the run fails with `InvalidConflictResolutionException` ([[#8. Error handling, cancellation, safety|§8]]). Two sides are "the same object" here by the same three things everything else in this section uses — `Kind`, `Name` and the owner chain, compared ordinally — so the check is a pure pass over the snapshot and costs no round trip. Because entries are matched against the databases by exactly that rule, the pass is also complete: two entries can claim one object only by naming it identically, so there is no second, later check to make when the entries are applied. This check is the one shape of contradiction the factory methods cannot prevent — `Ignore(orders, From.Left)` alongside `Map(orders, customerOrders)` claims `dbo.Orders` twice on the left — so it is the one the validation genuinely earns its keep on.

The same pure pass still rejects an entry whose `Left` and `Right` are both `null`, and one whose non-null side carries a runtime type its `Kind` does not call for. Neither can now be produced through the public API: the factory methods take typed identities and never emit an empty entry. The checks are kept as internal invariants over the `internal` constructor rather than as caller-facing validation, and are asserted through it.

**What the sides actually hold.** The runtime type follows `Kind`: a `TableIdentity` for `Table` and a `FieldIdentity` for `Field` — the identity objects the run already had in hand, handed over rather than copied into some reduced form. The factory methods are what guarantee the correspondence: their parameters are those two types, so no entry can carry a side its `Kind` does not call for. A conflict is about a name and its place in the database, which is exactly what an identity carries; the descriptor records built on top of them ([[#3.3.1 `IMetadataProvider`|§3.3.1]]) hold nothing a pairing decision could use. A caller answering an `AmbiguousNameException` can therefore pass the candidates it was given straight to `Ignore` or `Map`, with no reconstruction and no chance of mistyping a name, and can read the full context of each collision — a field's table, that table's schema — by walking `Owner`. `CandidateInfo` hands back `INestedObject`, so a caller answering a table or field collision downcasts to the identity type the method takes; a candidate of `Kind = PrimaryKey` or `Kind = ForeignKey` has no such method and is answered by excluding its table instead.

The owner chain is what makes an entry precise enough to be useful: `Kind = Field` with `Name = "Description"` says nothing on its own, since a field name is unique only within its table, but an entry whose `Owner` is `dbo.Orders` pairs the fields of that table and leaves a `Description` / `description` collision in `dbo.Invoices` untouched. A caller may resolve the two tables differently, which a global reading could not express. When the two databases spell the owning table differently, each side of a field entry names **its own** database's spelling of that table — the left side hangs from the left table, the right side from the right one — and the table itself is paired by a `Table` entry of its own.

**How an entry finds its object.** An object of the left database is matched against `ConflictResolution.Left`, an object of the right database against `ConflictResolution.Right`, and the names are compared with **`StringComparer.Ordinal`, and nothing else**. The rule is fixed and admits no alternative: not the run's comparer, not a case-insensitive comparison, not the carried record's own equality.

`Kind` and the **owner chain** select *what* is offered to that comparison — an entry of `Kind = Field` is matched only against fields, and only against the fields of the table its `Owner` names — and every name along the chain is compared the same ordinal way. Without the chain a field entry could not work at all, since `FieldIdentity.Name` is unqualified and `Description` names a column in half the schema; `TableIdentity.Name` is already the two-part `schema.table`, so for a table the chain adds nothing the name does not already carry.

**A side therefore matches at most one object**, and no rule is needed for the case of several: a database cannot hold two objects of one kind whose qualified names are byte-identical, whatever its collation. This is what an ordinal rule buys that the run's comparer could not — under a case-insensitive `PreferredCollation` the comparer considers `dbo.Table1` and `dbo.table1` one name, while an entry still names exactly one of them, which is precisely what a caller answering an `AmbiguousNameException` needs to be able to say.

Two ways of matching are ruled out, and it is worth saying why:

- *Never the run's comparer.* It is the thing that could not tell colliding names apart in the first place; matching an entry with it would make the entry as ambiguous as the conflict it is meant to settle, and would leave a case-only collision unanswerable. It would also make an entry mean different things on different runs — the same set, applied to the same databases under two `PreferredCollation` values, would resolve differently. An ordinal rule makes the set a fixed statement, readable without knowing which collation a run will settle on.
- *Never the record equality of the carried object.* `ConflictResolution` holds an `INestedObject`, and its runtime type is an identity record whose equality also covers `QuoteInfo`, `QuotesUsage` and `IsQuoted`, and reaches through `Owner` into the whole chain — a `FieldIdentity` compares its `Table`, and that table its `SchemaIdentity`. A caller pairing two field names has no reason to reproduce the run's quoting policy, and an entry compared that way would silently fail to match. Only `Kind`, `Name` and the owner chain decide.

**An entry must spell its object exactly, and an entry that matches nothing is not an error.** Both halves are specified behaviour, and together they decide what happens to an entry written in the wrong case: it applies to nothing, and the run proceeds without complaint. Where the left database holds `dbo.Orders`, an entry whose `Left` names `dbo.orders` is **not taken into account for that table** — on every collation, including one that calls the two names equal — and the table is compared as though the entry had not been written. The right side behaves identically against `ConflictResolution.Right`. This is deliberate, not an oversight to be mitigated: the alternative is a looser comparison, and a looser comparison cannot single out one name of a colliding pair, which is the mechanism's harder job.

A caller keeps entries correct by not typing them. The candidates carried by an `AmbiguousNameException` **are** the objects they name ([[#6.2.1 Conflict analysis|§6.2.1]]), so entries built from them cannot be mis-spelled, and a `TableIdentity` obtained from `IMetadataProvider.GetTablesAsync` carries the server's own spelling. For entries whose identities were typed by hand, `tables.detected` in the report ([[#5.1 Rules that govern the report|§5.1]]) is where a mis-spelled exclusion shows up — as a table that was compared after all. The factory methods close off the *other* ways an entry used to go wrong — a `Kind` that contradicts its sides, a half-built entry, a shape the algorithm has no rule for — but they cannot check a name against a database they have not read, so spelling remains the caller's to get right.

The set is otherwise a standing instruction, not a per-run script: a caller may keep one set across many comparisons, and an entry naming a table neither database holds — or a pair of names that do not collide and were going to be compared against each other anyway — is simply never consulted. What an entry must never do is match on one side and not on the other: a two-sided entry whose `Left` matches an object and whose `Right` matches none is a pairing the caller asked for and the library cannot honour, and it fails the run with `InvalidConflictResolutionException` rather than degrading into a one-sided exclusion, which would quietly delete a table from the report.

**When a candidate arises anyway.** Two names of one kind that the run's comparer cannot tell apart, with something on the other side to pair with and no entry settling them, are a **candidate**: the library reports them and refuses to guess. That analysis, and the table of cases it follows, is [[#6.2.1 Conflict analysis|§6.2.1]]; `CandidateInfo` is what it hands back. Candidates cannot arise on the default path — when `PreferredCollation` is unset the collation judging the names is the collation both databases use, and a server does not permit two objects whose names it considers equal, so a caller who never sets that property never sees an `AmbiguousNameException`. That caller may still need `ConflictResolution`: `Ignore` and the `Map` of a renamed object have nothing to do with collation.

**The property is fixed, its contents are not, and the run is neither.** `ConflictResolution` is declared `init`-only over a `HashSet<ConflictResolution>`, so it joins `QuoteInfo` and `QuotesUsage` in being settled when the options object is built and never reassigned afterwards: no code can swap one caller's decisions for another's halfway through a run. What `init` does **not** freeze is the set itself, and that is deliberate — the factory methods add to the collection they are handed, so a caller must be able to reach the one the options object already holds. The two facts fit together: the property names one set for the lifetime of the options object, and `Ignore` and `Map` fill that set.

The set is therefore populated after the options object is built — `options.ConflictResolution.Ignore(migrations)` — rather than inside the object initializer. The initializer remains available for handing in a set of one's own, `ConflictResolution = existingSet`, which is how a caller reuses one standing collection across several comparisons; what it cannot do is carry a collection-initializer body, since that would need entries built with a public constructor and there is none. A chain of factory calls cannot be assigned to the property either, `Ignore` and `Map` returning `ICollection<ConflictResolution>` where the property's type is `HashSet<ConflictResolution>` — build the set first, then chain onto it, or do what every sample does and chain onto the property after construction.

Because the contents stay mutable, `CompareTableStructureAsync` takes its own snapshot of the set at the start of the call, validates that snapshot, and uses only it, so mutating the set while a comparison is in flight cannot change the rules mid-run. A caller must still not rely on doing so, and a caller who shares one set between two options objects — which `init` permits — is sharing live state, not a copy: the library never clones it, and the snapshot each run takes is what keeps that safe.

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

- The comparer takes two collaborators and owns neither concern itself: `IMetadataProvider` reads the schema ([[#3.3.1 `IMetadataProvider`|§3.3.1]]), `INameComparerResolver` decides how identifiers are compared ([[#3.3.2 `INameComparerResolver`|§3.3.2]]). Both are interfaces, so a unit test drives the whole algorithm with two fakes and no SQL Server.
- The constructor throws `ArgumentNullException` when either `metadataProvider` or `nameComparerResolver` is `null`.
- The comparer takes no options object of its own. It reads the two `DatabaseOptions` itself as the first step of a run ([[#6.1 Database options, collation settlement and comparer detection|§6.1]]) and threads each side's object through that side's subsequent provider calls; they are locals of the call, never fields, and never reach the constructor. Both are read on every run, whatever `TableStructureComparisonOptions.PreferredCollation` says: their `Collation` either settles the comparison rule or validates the caller's, and the same object carries the capabilities each side's metadata queries are built from ([[#3.3.1 `IMetadataProvider`|§3.3.1]]).
- The class is stateless and therefore thread-safe: a single instance may serve concurrent calls, including runs against different databases with different collations — and, since the options travel as arguments rather than as cached state, against databases of different server versions.

### 3.3.1 `IMetadataProvider`

The interface declares the contract for every operation needed to read MS SQL metadata about tables, fields, primary keys and foreign keys. Metadata is fetched **per database in bulk**, not per table, so that one comparison costs a small fixed number of round trips regardless of how many tables the databases contain.

```csharp
public interface IMetadataProvider
{
    /// <summary>Returns the options of the database — its collation (§3.3.2) plus whatever
    /// else this provider needs to know about the server before it reads any metadata.
    /// An engine-specific provider returns its own derived type; for MS SQL that is
    /// <see cref="MsSqlDatabaseOptions"/>.</summary>
    Task<DatabaseOptions> GetDatabaseOptionsAsync(
        string connectionString,
        CancellationToken cancellationToken = default);

    /// <summary>Returns all user tables of the database.</summary>
    Task<IReadOnlyCollection<TableIdentity>> GetTablesAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the fields of the specified tables.</summary>
    Task<IReadOnlyCollection<FieldMetadata>> GetFieldsAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the primary keys of the specified tables (zero or one per table).</summary>
    Task<IReadOnlyCollection<PrimaryKeyMetadata>> GetPrimaryKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    /// <summary>Returns the foreign keys declared on the specified tables.</summary>
    Task<IReadOnlyCollection<ForeignKeyMetadata>> GetForeignKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);
}
```

`GetDatabaseOptionsAsync` is the run's single preliminary read, and it answers two questions in one round trip: how the database compares identifiers, and what the database supports. The first is the collation, reached as `DatabaseOptions.Collation` — the only path to it, since the interface declares no separate collation-reading method. The second is whatever the implementation must know before it can decide what its metadata queries may say ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]).

Both answers are used on every run. When the caller sets `TableStructureComparisonOptions.PreferredCollation`, [[#6.1 Database options, collation settlement and comparer detection|§6.1]] takes the comparison rule from there rather than from the databases — but it still reads `Collation` on both sides, to check that the caller's value is one the databases actually use. A preferred collation therefore saves no round trip and skips no read ([[#9. Non-functional requirements|§9]]). A provider is told nothing about the caller's choice in any case: its contract, its return value and its behaviour are identical either way.

Every metadata method then takes that same object back as `databaseOptions`. It is not decoration: for MS SQL the statement a database gets depends on what that database supports, and this parameter is how the answer reaches the code that composes it. Reading the options once, up front, and passing them to each of the four calls means the capability is established in a single round trip and consulted four times for free — the alternative, re-reading it per call, would take a comparison from five queries per database to nine ([[#9. Non-functional requirements|§9]]).

Passing the options explicitly rather than letting a provider cache them is also what keeps implementations stateless and keeps a comparison honest: the object describes the database *as read at the start of this run*, so two runs against the same connection string are never silently served from one another's answer, and a provider need hold nothing — least of all a connection string — between calls.

**Database options.** What `GetDatabaseOptionsAsync` returns is an open, extensible carrier rather than a sealed record, because each engine has a different set of things worth knowing before it reads a catalog:

```csharp
//file DatabaseOptions.cs
public class DatabaseOptions
{
    /// <summary>The collation of the database — its name plus the properties
    /// a comparer is built from (§3.3.2).</summary>
    public CollationInfo Collation { get; init; }
}
//file MsSqlDatabaseOptions.cs
public sealed class MsSqlDatabaseOptions : DatabaseOptions
{
    /// <summary>True when sys.tables exposes the ledger columns — that is, when the
    /// server knows about ledger tables at all (SQL Server 2022+ / Azure SQL Database).</summary>
    public bool SupportsLedgerTables { get; init; }
}
```

- `DatabaseOptions` carries only what every engine has: the collation. It is a plain `class` — not a `record`, not `sealed` — precisely so that a provider can extend it. Equality is never taken on it, so record semantics would buy nothing. Its properties are `init`-only: the options describe a database as it was read and are never mutated afterwards.
- `MsSqlDatabaseOptions` adds the one MS SQL capability the metadata queries depend on. `SupportsLedgerTables` is a **capability** flag, not a version number, and deliberately not derived from `SERVERPROPERTY('ProductMajorVersion')`: Azure SQL Database supports ledger while still reporting major version `12`, so a version comparison would disable the ledger filter on exactly the platform where ledger tables are most likely to be found. It is answered by asking the catalog whether the column is there ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]).
- The base type is what travels through the engine-neutral parts of the library. `INameComparerResolver` and [[#6.1 Database options, collation settlement and comparer detection|§6.1]]'s mismatch check read `Collation` and nothing else; `TableStructureComparer` reads `Collation` and otherwise treats the object as opaque, passing it from one provider call to the next without ever downcasting. The downcast happens in exactly one place — the provider that produced the object ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]).
- Adding a second capability later means one more property here and one more branch in `MsSqlMetadataQueryBuilder`; neither `IMetadataProvider`, nor the comparer, nor any other engine's implementation changes. That is the point of putting the flag in the options object rather than in a method signature.

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
//file INestedObject.cs
public interface INestedObject
{
    /// <summary>The object's own name, unqualified for a schema, a field or a key,
    /// and the two-part "schema.table" for a table.</summary>
    public string Name { get; }

    /// <summary>The object this one belongs to — a table for a field or a key, a schema
    /// for a table, and <c>null</c> for a schema, which ends the chain.</summary>
    public INestedObject? Owner { get; }
}

//file IQuotedIdentifier.cs
public interface IQuotedIdentifier
{
    QuoteInfo QuoteInfo { get; }
    QuotesUsage QuotesUsage { get; }
    bool IsQuoted { get; }
    string Name { get; }

    string QuotedName => QuotedIdentifier.GetQuotedName(this.Name, this.QuoteInfo);

    string PreferredName => QuotedIdentifier.GetPreferredName(this.Name, this.QuoteInfo, this.QuotesUsage, this.IsQuoted);
}

//file QuotedIdentifier.cs
public static class QuotedIdentifier
{
    public static string GetQuotedName(string name, QuoteInfo quoteInfo) => quoteInfo.Quotes.Add(name);

    public static string GetPreferredName(string name, QuoteInfo quoteInfo, QuotesUsage quotesUsage, bool isQuoted) => quotesUsage switch
    {
        QuotesUsage.Required => GetQuotedName(name, quoteInfo),
        QuotesUsage.UseIfSpecifiedOrRequired => isQuoted || quoteInfo.QuotesRequired(name)
            ? GetQuotedName(name, quoteInfo)
            : name,
        QuotesUsage.DoNotUseIfPossible => quoteInfo.QuotesRequired(name)
            ? GetQuotedName(name, quoteInfo)
            : name,
        _ => throw new ArgumentOutOfRangeException(nameof(quotesUsage)),
    };
}

//file SchemaIdentity.cs
public record SchemaIdentity : IQuotedIdentifier, INestedObject
{
    public SchemaIdentity(string name, QuotesUsage quotesUsage, QuoteInfo quoteInfo)
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

    // The two interface defaults, restated as concrete members, because TableIdentity
    // composes them off a SchemaIdentity-typed reference — see the note below the table.
    public string QuotedName => QuotedIdentifier.GetQuotedName(this.Name, this.QuoteInfo);
    public string PreferredName => QuotedIdentifier.GetPreferredName(this.Name, this.QuoteInfo, this.QuotesUsage, this.IsQuoted);

    public INestedObject? Owner { get; } = null;
}

//file TableIdentity.cs
public sealed record TableIdentity : IQuotedIdentifier, INestedObject
{
    private readonly string _name;

    public TableIdentity(
        SchemaIdentity schema,
        string name)
    {
        this.Schema = schema;
        this.IsQuoted = schema.QuoteInfo.Quotes.IsQuotedIdentifier(name);
        this._name = this.IsQuoted ? schema.QuoteInfo.Quotes.Remove(name) : name;
    }

    public SchemaIdentity Schema { get; }
    public QuoteInfo QuoteInfo => this.Schema.QuoteInfo;
    public QuotesUsage QuotesUsage => this.Schema.QuotesUsage;
    public bool IsQuoted { get; }

    public INestedObject? Owner => this.Schema;

    public string Name => $"{Schema.Name}.{this._name}";
    public string QuotedName => $"{Schema.QuotedName}.{QuotedIdentifier.GetQuotedName(this._name, this.QuoteInfo)}";
    public string PreferredName => $"{Schema.PreferredName}.{QuotedIdentifier.GetPreferredName(this._name, this.QuoteInfo, this.QuotesUsage, this.IsQuoted)}";

    /// <summary>Parses a two-part name: "schema.table" or "[schema].[table]".</summary>
    public static TableIdentity Parse(
        string tableWithSchema,
        QuotesUsage quotesUsage,
        QuoteInfo quoteInfo);

    /// <summary>Non-throwing counterpart of <see cref="Parse"/>.</summary>
    public static bool TryParse(string? tableWithSchema, QuotesUsage quotesUsage, QuoteInfo quoteInfo, out TableIdentity? result);
}
//file FieldIdentity.cs
public record FieldIdentity : INestedObject, IQuotedIdentifier
{
    public FieldIdentity(
        TableIdentity table,
        string name)
    {
        this.Table = table;
        this.IsQuoted = table.QuoteInfo.Quotes.IsQuotedIdentifier(name);
        this.Name = this.IsQuoted ? table.QuoteInfo.Quotes.Remove(name) : name;
    }

    public TableIdentity Table { get; }
    public QuoteInfo QuoteInfo => this.Table.QuoteInfo;
    public QuotesUsage QuotesUsage => this.Table.QuotesUsage;
    public bool IsQuoted { get; }
    public string Name { get; }
    public INestedObject Owner => this.Table;
}
//file PrimaryKeyIdentity.cs
public record PrimaryKeyIdentity : INestedObject, IQuotedIdentifier
{
    public PrimaryKeyIdentity(
        TableIdentity table,
        string name)
    {
        this.Table = table;
        this.IsQuoted = table.QuoteInfo.Quotes.IsQuotedIdentifier(name);
        this.Name = this.IsQuoted ? table.QuoteInfo.Quotes.Remove(name) : name;
    }

    public TableIdentity Table { get; }
    public QuoteInfo QuoteInfo => this.Table.QuoteInfo;
    public QuotesUsage QuotesUsage => this.Table.QuotesUsage;
    public bool IsQuoted { get; }
    public string Name { get; }
    public INestedObject Owner => this.Table;
}
//file ForeignKeyIdentity.cs
public record ForeignKeyIdentity : INestedObject, IQuotedIdentifier
{
    public ForeignKeyIdentity(
        TableIdentity table,
        string name)
    {
        this.Table = table;
        this.IsQuoted = table.QuoteInfo.Quotes.IsQuotedIdentifier(name);
        this.Name = this.IsQuoted ? table.QuoteInfo.Quotes.Remove(name) : name;
    }

    public TableIdentity Table { get; }
    public QuoteInfo QuoteInfo => this.Table.QuoteInfo;
    public QuotesUsage QuotesUsage => this.Table.QuotesUsage;
    public bool IsQuoted { get; }
    public string Name { get; }
    public INestedObject Owner => this.Table;
}
//file FieldMetadata.cs
public sealed record FieldMetadata
{
    public FieldMetadata(
        FieldIdentity field,
        string dataType,     // "nvarchar", "int", "decimal", ...
        int? size,           // length in characters/bytes; -1 = MAX; null when not applicable
        bool isNullable,
        byte? precision,     // null when not applicable
        byte? scale)         // null when not applicable
    {
        this.Field = field;
        this.DataType = dataType;
        this.Size = size;
        this.IsNullable = isNullable;
        this.Precision = precision;
        this.Scale = scale;
    }

    public FieldIdentity Field { get; }
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
        PrimaryKeyIdentity primaryKey,
        IReadOnlyList<FieldIdentity> fields)       // ordered by key ordinal
    {
        this.PrimaryKey = primaryKey;
        this.Fields = fields;
    }

    public PrimaryKeyIdentity PrimaryKey { get; }
    public IReadOnlyList<FieldIdentity> Fields { get; }
}
//file ForeignKeyColumn.cs
public sealed record ForeignKeyColumn(FieldIdentity Field, FieldIdentity ReferencedField);
//file ForeignKeyMetadata.cs
public sealed record ForeignKeyMetadata
{
    public ForeignKeyMetadata(
        ForeignKeyIdentity foreignKey,
        TableIdentity referencedTable,
        IReadOnlyList<ForeignKeyColumn> columns)   // ordered by constraint column id
    {
        this.ForeignKey = foreignKey;
        this.ReferencedTable = referencedTable;
        this.Columns = columns;
    }

    public ForeignKeyIdentity ForeignKey { get; }
    public TableIdentity ReferencedTable { get; }
    public IReadOnlyList<ForeignKeyColumn> Columns { get; }
}
```


Every identifier the library handles is an `IQuotedIdentifier`: the bare name plus the knowledge of whether the caller supplied it quoted and how it should be rendered back. Three renderings are available and they are **not** interchangeable — each has one job:

| Rendering | Content | Used for |
| --- | --- | --- |
| `Name` | the bare identifier, quotes stripped | **all comparison and matching** ([[#6.1 Database options, collation settlement and comparer detection\|§6.1]]) |
| `QuotedName` | always quoted | building SQL; never the report |
| `PreferredName` | quoted only when `QuotesUsage` demands it | **the names rendered in the report** ([[#5.1 Rules that govern the report\|§5.1]]) |

`QuotesUsage` selects the third one's policy: `Required` always quotes; `DoNotUseIfPossible` (the default) quotes only when `QuoteInfo.QuotesRequired` says the identifier cannot survive unquoted; `UseIfSpecifiedOrRequired` does the same but also preserves quotes the caller already used, which is what `IsQuoted` records. `QuoteInfo.MsSql` supplies the MS SQL bracket pair together with `MsSqlRequiresQuotedIdentifier` as that predicate.

There is no ambient default and no process-wide current policy. The policy enters the object graph at exactly one point — the `SchemaIdentity` constructor, which requires `quotesUsage` and `quoteInfo` explicitly, as do `TableIdentity.Parse` and `TryParse` — and the comparer feeds it from `TableStructureComparisonOptions` ([[#3.1 `TableStructureComparisonOptions`|§3.1]]). That is what keeps the comparer stateless and safe for concurrent runs — two comparisons may use different policies at the same time without interfering.

Below the schema the policy is inherited, never restated: `TableIdentity` re-exposes its schema's `QuoteInfo` and `QuotesUsage`, and `FieldIdentity`, `PrimaryKeyIdentity` and `ForeignKeyIdentity` re-expose their table's. An identifier therefore cannot be rendered under a policy other than the one its owner was built with, because no constructor below `SchemaIdentity` takes one. A foreign key's `ReferencedField` is a `FieldIdentity` of the **referenced** table rather than of the declaring one, so a rendered name always follows the table it actually names.

`QuotedName` and `PreferredName` are **default implementations** on `IQuotedIdentifier`, delegating to the `QuotedIdentifier` helper so that every single-part identifier shares one definition and `TableIdentity` can compose its two-part one out of the same two functions. A default interface member is reachable only through an interface-typed reference, so a consumer holding a `FieldIdentity`, a `PrimaryKeyIdentity` or a `ForeignKeyIdentity` asks for a rendering as an `IQuotedIdentifier`. `SchemaIdentity` and `TableIdentity` are the two exceptions: they declare both renderings as ordinary members, because `TableIdentity` reads `Schema.QuotedName` and `Schema.PreferredName` off a `SchemaIdentity`-typed reference and a default member would not be visible there.

`TableIdentity.Parse` is the string entry point into the pair — it lets a caller write an exclusion as `"dbo.Orders"` instead of spelling out both parts. Its grammar is fixed:

- The input is a **two-part** name. Surrounding whitespace is trimmed; the split is ordinal.
- Either part may be quoted with the pair taken from `QuoteInfo` — `[` and `]` for MS SQL — and a dot inside quotes is literal: `[dbo].[Order.Archive]` yields `Schema.Name == "dbo"` and `Name == "dbo.Order.Archive"`, with `Schema.IsQuoted` and `IsQuoted` both `true`. A `]]` inside brackets is an escaped `]`. The quotes do not survive into `Name`; they are re-applied on demand by `QuotedName` and `PreferredName`.
- An unquoted input must contain exactly one dot. `"dbo.Orders"` parses; `"dbo.Order.Archive"` throws, because a bare name with two dots is genuinely ambiguous and the method must not guess.
- Both `Parse` and `TryParse` take `quotesUsage` and `quoteInfo` explicitly and use them to construct the `SchemaIdentity` the resulting `TableIdentity` hangs from; neither has a defaulted overload, so a caller always states the policy the resulting identity will be rendered under. A `null` input throws `ArgumentNullException`. An input with no dot, with an empty part, or with unbalanced quotes throws `FormatException`. No schema is inferred: `"Orders"` does not silently become `dbo.Orders`, since the effective default schema depends on the login and guessing it would exclude the wrong table.
- `Parse` performs no validation against either database — it is a pure string operation. A parsed name that matches nothing simply never reaches `tables.unchecked` ([[#5.1 Rules that govern the report|§5.1]]).
- `TryParse` applies exactly the same grammar but never throws: every rejection above — `null`, a missing dot, an empty part, unbalanced brackets, an ambiguous unquoted name — returns `false` with `result` set to `null`. It exists so that a caller validating user-supplied text does not have to drive control flow with exceptions.

**The ownership chain.** `INestedObject` gives every named object a `Name` and a link to what owns it, and the five implementing types form one chain: a `FieldIdentity`, a `PrimaryKeyIdentity` and a `ForeignKeyIdentity` are owned by their `TableIdentity`, a table by its `SchemaIdentity`, and a schema by nothing, which is where the walk ends. A consumer holding any one of them can therefore recover the object's full qualification without being told separately which table or schema it came from — which is exactly what `CandidateInfo` and `ConflictResolution` rely on ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]), since a field name means nothing without its table.

Four details of the declaration are worth stating, because the interface is small enough to look like it costs nothing:

- `Name` is unqualified for a schema, a field and a key, and two-part (`dbo.Orders`) for a table, matching what `TableIdentity.Name` already rendered. `INestedObject` exposes the bare name only — the one form a walk up the chain ever needs; the quoted and preferred renderings belong to `IQuotedIdentifier`, which every type in the chain also implements.
- Every identity type is a `record` rather than a `class`, because `CandidateInfo` keeps these objects in a `HashSet` and value equality over the name, its policy and its owner is the behaviour a set of identities should have. `SchemaIdentity` is the root of that value: it terminates the chain with a `null` `Owner`, and it is the one place in the graph where `QuotesUsage` and `QuoteInfo` are supplied rather than inherited.
- Identity and descriptor are separate types, and the chain lives on the identity. `FieldIdentity` names a column and knows its table; `FieldMetadata` says what that column is — type, size, nullability, precision, scale — and points at the identity. A name that did not know its table would leave a field candidate as ambiguous as it was before, which is why the interface is implemented where the owner is, and the descriptor is then free to grow without changing anything a `ConflictResolution` compares. The same split holds for both key kinds, through `PrimaryKeyIdentity` / `PrimaryKeyMetadata` and `ForeignKeyIdentity` / `ForeignKeyMetadata`.
- `TableIdentity` publishes only the two-part `Name` and keeps its own unqualified part private; the schema stays reachable as `Schema`, so no part of the pair is lost, but a consumer that needs the bare table name on its own does not get it from this type. Nothing in this brief needs it: matching uses the two-part `Name` ([[#6.7 Determinism and identity handling|§6.7]]) and the report renders `PreferredName` ([[#5.1 Rules that govern the report|§5.1]]).

**Contract rules for implementers**

- A method never returns `null`; an empty collection is returned when nothing is found.
- `PrimaryKeyMetadata.Fields` and `ForeignKeyMetadata.Columns` are **ordered sequences**, not sets: the provider passes the participating fields in key order (`key_ordinal` for a primary key, `constraint_column_id` for a foreign key), because the report renders them in that order ([[#5. Report structure|§5]]). Set semantics belong to the comparison, not to the metadata: [[#6.4 Primary key comparison|§6.4]] and [[#6.5 Foreign key comparison|§6.5]] compare these sequences as sets under the comparer detected in [[#6.1 Database options, collation settlement and comparer detection|§6.1]]. A provider must not pass a hash-set type here — doing so would both lose the key order and hard-code an equality rule that [[#6.1 Database options, collation settlement and comparer detection|§6.1]] reserves for the collation.
- The provider turns the raw catalog strings into identity records itself, against the `TableIdentity` it was handed for that table — `new FieldIdentity(table, name)` and the two key counterparts — so an identifier enters the graph already owned, and `FieldMetadata`, `PrimaryKeyMetadata` and `ForeignKeyMetadata` are handed identities rather than strings. The quoting policy is still never the provider's to choose: it arrives with that `TableIdentity` and is inherited, and no constructor below `SchemaIdentity` accepts a `QuotesUsage` or a `QuoteInfo`. A foreign key's `ReferencedField` is built against the referenced table's `TableIdentity`, which is what gives it that table's policy.
- The `tables` argument narrows the result set. Passing an empty collection returns an empty result without hitting the database.
- `GetDatabaseOptionsAsync` never returns `null`, and `DatabaseOptions.Collation` is never `null`. A provider with engine-specific options to report returns its own derived type; one with none may return a `DatabaseOptions` directly.
- The `databaseOptions` handed to a metadata method must be the object **this same provider** returned from `GetDatabaseOptionsAsync` for **this same** `connectionString`. Passing the other side's options is a caller error the provider cannot always detect, and [[#6.1 Database options, collation settlement and comparer detection|§6.1]] places the obligation on the comparer accordingly. A `null` throws `ArgumentNullException`; an instance of a type the provider does not recognise throws `ArgumentException` ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]).
- A provider must never paper over a missing or foreign options object by re-reading the database or by assuming default capabilities. A silently degraded query is exactly the failure this parameter exists to prevent.
- Implementations are **stateless** with respect to a database: nothing is cached between calls — not options, not connections, not query text — and in particular nothing is keyed by connection string. Everything a call needs arrives in its arguments.
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

`CollationInfo` is what a database reports about its own collation — and, when a caller sets `TableStructureComparisonOptions.PreferredCollation` ([[#3.1 `TableStructureComparisonOptions`|§3.1]]), what the caller states in its place. Either way it carries everything needed to reproduce that collation's rules in .NET:

| Member | Source | Role |
| --- | --- | --- |
| `Name` | `DATABASEPROPERTYEX(db, 'Collation')` | identity of the collation; [[#6.1 Database options, collation settlement and comparer detection\|§6.1]] compares the two sides on this when no `PreferredCollation` is set, and names it in `CollationMismatchException` |
| `Lcid` | `COLLATIONPROPERTY(name, 'LCID')` | the culture whose linguistic rules apply — `CultureInfo.GetCultureInfo(Lcid)` |
| `ComparisonStyle` | `COLLATIONPROPERTY(name, 'ComparisonStyle')` | bit mask of what the collation ignores; `0` marks a binary collation |
| `CodePage` | `COLLATIONPROPERTY(name, 'CodePage')` | the non-Unicode code page; not used for comparison, carried for diagnostics |
| `Version` | `COLLATIONPROPERTY(name, 'Version')` | collation version (`0`, `90`, `100`, `140`…); two collations differing only in version sort differently |

The `ComparisonStyle` bits map one-to-one onto `System.Globalization.CompareOptions`, which is what makes the resolver a pure function rather than a table of special cases: `0x1` → `IgnoreCase`, `0x2` → `IgnoreNonSpace` (accent), `0x10000` → `IgnoreKanaType`, `0x20000` → `IgnoreWidth`. The record exposes them as named predicates so no caller repeats the masks.

Contract rules:

- The resolver is a **pure mapping** from a collation to a comparer, and its signature says so: no `Task`, no `CancellationToken`. It opens no connection and reads no metadata — the caller has already obtained the `CollationInfo`, either as `DatabaseOptions.Collation` from `IMetadataProvider.GetDatabaseOptionsAsync` or straight from `TableStructureComparisonOptions.PreferredCollation`. This is the one place in the library that is deliberately synchronous: there is nothing to await, and wrapping a table lookup in a `Task` would only add an allocation and hide that fact from the caller.
- The mapping must be **total and deterministic** for the collations it accepts: the same `CollationInfo` always yields an equivalent comparer, and an unsupported one throws rather than degrades ([[#3.4.2 `MsSqlNameComparerResolver`|§3.4.2]]).
- Implementations never return `null`.
- The resolver is indifferent to where the record came from, and is given no way to find out. A record the implementation cannot honour throws for a caller-supplied value just as it does for a database-supplied one ([[#3.4.2 `MsSqlNameComparerResolver`|§3.4.2]]) — a preferred collation is never approximated, and never quietly replaced by the databases' own. Whether the record is one a database reported is not the resolver's question either: [[#6.1 Database options, collation settlement and comparer detection|§6.1]] has already settled it, and a `PreferredCollation` matching neither database never reaches this call.
- Keeping this separate from `IMetadataProvider` is what lets a test pin the comparison rule without standing up any metadata at all, and lets a caller override the rule for a collation the built-in mapping handles poorly.

### 3.4.1 `MsSqlMetadataProvider`

```csharp
public sealed class MsSqlMetadataProvider : IMetadataProvider
{
    private readonly MsSqlMetadataQueryBuilder _queryBuilder;

    public MsSqlMetadataProvider()
    {
        this._queryBuilder = new MsSqlMetadataQueryBuilder();
    }
}
```

An `IMetadataProvider` implementation on top of `Microsoft.Data.SqlClient` that reads the `sys.*` catalog views. One `SqlConnection` per call, opened with `OpenAsync(ct)` and read with `ExecuteReaderAsync(ct)` / `ReadAsync(ct)`. No synchronous ADO.NET calls anywhere.

The provider owns exactly one SQL literal of its own — the database-options statement below. Each of the four metadata methods asks `_queryBuilder` ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]) for its statement, passing the `databaseOptions` it was given, and then does what a provider is actually for: open the connection, bind the parameters, run the reader, shape the rows into records, normalize the types.

```csharp
var msSqlOptions = AsMsSqlOptions(databaseOptions);                     // the checked narrowing below
var query = this._queryBuilder.BuildTableMetadataQuery(msSqlOptions);   // and so on for the other three
```

The builder is created in the constructor rather than injected, which is what keeps the provider constructible as a bare `new MsSqlMetadataProvider()` ([[#12. Definition of done|§12]]). It is stateless and holds nothing, so one instance per provider costs nothing and the provider remains stateless with it. The consequence to be aware of is that the query text is fixed at compile time: it cannot be substituted from outside, and a unit test cannot stub it — the builder is instead tested directly ([[#10. Testing requirements|§10]]), which is cheap because it needs no database.

**The one downcast in the library.** The interface hands over a `DatabaseOptions`; the builder requires an `MsSqlDatabaseOptions`. Each of the four metadata methods therefore narrows the argument before using it, and the narrowing is checked:

- `null` throws `ArgumentNullException`.
- Any runtime type other than `MsSqlDatabaseOptions` throws `ArgumentException` naming both the expected and the actual type. This is what a provider/options mix-up looks like, and it must fail loudly: the provider does not fall back to re-reading the options itself, nor to assuming `SupportsLedgerTables = false`, because both would replace a visible composition error with a silently wrong or silently expensive run.
- In practice the object is the one this provider returned from its own `GetDatabaseOptionsAsync` for the same connection string, which is what [[#3.3.1 `IMetadataProvider`|§3.3.1]] requires of the caller and [[#6.1 Database options, collation settlement and comparer detection|§6.1]] of the comparer.

Selection rule: user tables only — `sys.tables` with `type = 'U'` and `is_ms_shipped = 0`; system schemas (`sys`, `INFORMATION_SCHEMA`) are excluded. On top of that the provider drops the engine-generated shadow tables named in [[#1.3 Out of scope (v1)|§1.3]] — temporal history tables (`temporal_type = 1`), ledger history tables (`ledger_type = 1`) and the retained remains of dropped ledger tables (`is_dropped_ledger_table = 1`). The filter lives in the metadata queries — composed by the builder ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]), executed by the provider — and never in the comparer; it is not configurable through `TableStructureComparisonOptions`. These tables are absent from every collection `IMetadataProvider` returns, so no downstream code has to know they exist.

Two of those columns — `ledger_type` and `is_dropped_ledger_table` — exist only on SQL Server 2022 (major version 16) and later, and referencing a column that does not exist is a **parse-time** failure in T-SQL, not a runtime one. A predicate left in a static statement "to evaluate harmlessly" on an older server therefore does not evaluate at all: the whole batch fails before it runs. Which of the two variants of a query a database gets is consequently a real decision, and [[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]] is where it is made — from `MsSqlDatabaseOptions.SupportsLedgerTables`, in C#, ahead of execution. No metadata statement probes the server about itself, composes its own text, or passes through `sys.sp_executesql`: each arrives from the builder as finished, static SQL, already correct for the server it is about to run on.

`temporal_type` needs no capability behind it — it has been present since SQL Server 2016, the library's minimum — so its predicate is unconditional in both variants.

**Database options.** The one statement the provider owns, and the one that cannot come from the builder, since it is what tells the builder which queries to produce. A single round trip answers both halves of the question — how this database compares identifiers, and what its `sys.tables` exposes:

```sql
DECLARE @collation sysname = CONVERT(sysname, DATABASEPROPERTYEX(DB_NAME(), 'Collation'));
SELECT @collation                                        AS Name,
       COLLATIONPROPERTY(@collation, 'LCID')             AS Lcid,
       COLLATIONPROPERTY(@collation, 'CodePage')         AS CodePage,
       COLLATIONPROPERTY(@collation, 'ComparisonStyle')  AS ComparisonStyle,
       COLLATIONPROPERTY(@collation, 'Version')          AS Version,
       CAST(IIF(COL_LENGTH(N'sys.tables', N'ledger_type') IS NULL, 0, 1) AS BIT)
                                                         AS SupportsLedgerTables;
```

`GetDatabaseOptionsAsync` returns an `MsSqlDatabaseOptions`: the first five columns become `Collation`, the sixth becomes `SupportsLedgerTables`. `COL_LENGTH` is a **capability** check rather than a version check, for the Azure SQL reason given in [[#3.3.1 `IMetadataProvider`|§3.3.1]]. Asking it here — once, as a column of a query that has to run anyway — is what makes the capability free: it rides along inside the one preliminary read, so the five-query budget of [[#9. Non-functional requirements|§9]] pays nothing for it.

**Reference queries.** The four below are the builder's, reproduced from [[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]] in their `SupportsLedgerTables = true` form. What the provider fixes about them is the reader contract — the columns each projects, in the order its reader expects; a builder may change a `WHERE` clause freely, but changing a projection breaks the provider.

```sql
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

On a database whose `SupportsLedgerTables` is `false`, the two `ledger_*` lines are absent from the emitted text entirely rather than merely inert — see [[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]] for why the distinction matters.

The `tables` argument narrows the three table-restricted queries exactly as before; the provider binds the restriction as a parameter and never splices a caller-supplied name into the statement as text. It is orthogonal to the ledger branch: both variants of each query accept it.

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

**What this reproduces, and what it does not.** The four ignore-flags cover case, accent, kana and width, which is what decides whether two identifiers are the *same* identifier — the question this library actually asks. It does not reproduce SQL Server's *sort order* exactly: .NET orders through ICU, SQL Server through its own tables, and the legacy `SQL_*` collations use sort orders that predate Windows collations altogether. That is why the report's ordering does not depend on this comparer ([[#4. Result model|§4]]), and why an exact ordering match is listed as an open question ([[#11. Assumptions and open questions|§11]]).

### 3.4.3 `MsSqlMetadataQueryBuilder`

```csharp
public sealed class MsSqlMetadataQueryBuilder
{
    public string BuildTableMetadataQuery(MsSqlDatabaseOptions options);
    public string BuildFieldMetadataQuery(MsSqlDatabaseOptions options);
    public string BuildPrimaryKeyMetadataQuery(MsSqlDatabaseOptions options);
    public string BuildForeignKeyMetadataQuery(MsSqlDatabaseOptions options);
}
```

The other half of the pair `MsSqlMetadataProvider` forms ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]): the provider knows *how to talk to* a database, the builder knows *what to say to it*. It touches no database — everything it needs to decide has already been read into the `MsSqlDatabaseOptions` it receives.

It is a concrete `sealed` class rather than an interface, and it takes `MsSqlDatabaseOptions` rather than the base type. Both follow from what it is: the MS SQL half of an MS SQL provider, useful only to that provider, with nothing for another engine to implement. The typed parameter is what makes the capability available without a runtime type test inside the builder — the one check the design does need happens once, at the provider's boundary ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]), where a wrong options object is a caller error worth naming. A builder for another engine would take that engine's options type and live beside this one, not behind a shared abstraction.

**The one decision it makes.** From `options.SupportsLedgerTables` it emits one of two variants of each query:

| `SupportsLedgerTables` | What the query contains |
| --- | --- |
| `true` | the ledger predicates — `t.ledger_type <> 1 AND t.is_dropped_ledger_table = 0` — so ledger history tables and the retained remains of dropped ledger tables are **filtered out** of the result |
| `false` | no mention of ledger at all: neither column is named anywhere in the returned text |

The `false` branch is a correctness requirement, not an optimization. Both columns are SQL Server 2022+ ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]), and naming a column that does not exist fails at parse time — so the two variants must differ in what the text *mentions*, not merely in what it *matches*. A predicate that would be false-by-construction on an old server is still fatal there, which is why "emit it and let it be ignored" is not an option and why the branch has to be taken in C#.

`t.temporal_type <> 1` is unconditional: present in all four queries and in both variants.

Reference queries, in the `SupportsLedgerTables = true` form. In the `false` form every line marked `-- ledger` is absent from the emitted text; nothing else differs, and the projection in particular is identical:

```sql
-- BuildTableMetadataQuery
SELECT s.name AS SchemaName, t.name AS TableName
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
WHERE t.type = 'U' AND t.is_ms_shipped = 0
  AND t.temporal_type <> 1                 -- temporal history table
  AND t.ledger_type <> 1                   -- ledger: ledger history table
  AND t.is_dropped_ledger_table = 0;       -- ledger: remains of a dropped one

-- BuildFieldMetadataQuery
SELECT s.name, t.name, c.name, ty.name AS DataType,
       c.max_length, c.precision, c.scale, c.is_nullable
FROM sys.columns c
JOIN sys.tables   t  ON t.object_id     = c.object_id
JOIN sys.schemas  s  ON s.schema_id     = t.schema_id
JOIN sys.types    ty ON ty.user_type_id = c.user_type_id
WHERE t.type = 'U' AND t.is_ms_shipped = 0
  AND t.temporal_type <> 1
  AND t.ledger_type <> 1                   -- ledger
  AND t.is_dropped_ledger_table = 0;       -- ledger

-- BuildPrimaryKeyMetadataQuery
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
  AND t.ledger_type <> 1                   -- ledger
  AND t.is_dropped_ledger_table = 0;       -- ledger

-- BuildForeignKeyMetadataQuery
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
  AND t.ledger_type <> 1                   -- ledger
  AND t.is_dropped_ledger_table = 0;       -- ledger
```

The shadow-table predicates apply to the declaring table `t` only. A foreign key's **referenced** table is deliberately not filtered by them, nor by the `tables` restriction of [[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]: a compared table may legitimately reference a table outside the comparison set, and dropping such a key would report a difference that does not exist.

Contract rules:

- The builder is a **pure function** of `options`: no `Task`, no `CancellationToken`, no connection string, no I/O. Given the same options it returns the same text, which is what lets a test assert on that text directly ([[#10. Testing requirements|§10]]).
- It never returns `null` or an empty string. A `null` `options` throws `ArgumentNullException`.
- Every returned statement is complete and executable as-is. The builder never embeds an identifier or a literal taken from user input; the `tables` restriction reaches the server as a bound parameter ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]), and the only thing that varies between the two variants is the ledger predicates.
- The projection of each query is the contract with the provider's readers ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]) and is identical in both variants.
- The builder is stateless and safe for concurrent use. The right SQL is computed on each call in accordance with the specified database options.
- Nothing is cached because nothing can be: the text depends on `options`, so there is no single value to build once. Any build-once scheme would need a key, and a wrongly-keyed one hands a 2022 statement to a 2019 server — the parse-time failure this whole branch exists to avoid. Composing it costs a branch on a `bool`, which is why there is nothing to amortize. Contrast the report's `JsonSerializerOptions` ([[#4. Result model|§4]]), whose configuration depends on nothing and is therefore built once and reused; the two rules point in opposite directions for that reason, not by oversight.
- Nothing requires the two sides of one comparison to receive the same text. A left database on SQL Server 2019 and a right one on 2022 report different capabilities and legitimately get different variants — whatever [[#6.1 Database options, collation settlement and comparer detection|§6.1]] does about the collation, it never compares the capabilities.

---

## 4. Result model

`TableStructureComparisonResult` is a POCO tree that maps one-to-one onto the JSON of **[[#5. Report structure|§5]]**. Serialization requirements:

- `JsonNamingPolicy.CamelCase`.
- **All sections are always present.** Empty collections serialize as `[]`, empty objects as `{}` — a consumer must never have to null-check a section.
- **The report names the collation it was produced under.** `TableStructureComparisonResult.Collation` is a `CollationInfo`, and it is the very record [[#6.1 Database options, collation settlement and comparer detection|§6.1]] resolved the run's comparer from: the collation both databases share when `PreferredCollation` was left unset, the caller's value when it was set. It is never `null`, and the `collation` section is never omitted — a consumer reading a report always knows which rule decided identifier equality, and two reports over the same pair of databases produced under different rules are told apart by this section alone. The result holds the resolved record itself rather than a copy or a name, so the report and the comparer cannot disagree about it.
- **A null scalar is omitted, never emitted.** `size`, `precision` and `scale` are written only when they carry a value; when the underlying metadata has none, the property is absent from the JSON altogether rather than present with a `null` value (`JsonSerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull`). The same applies to the `key` and `fields` members of a `missings.*.primaryKeys` entry when the primary key is absent in both databases (see Use Case 7). A consumer therefore tests for the *presence* of these properties, not for a null value.
- Actual JSON types: `size`, `precision`, `scale` — `number`, the property being absent when not applicable; `nullable` — `boolean`; the four numeric members of `collation` (`lcid`, `codePage`, `comparisonStyle`, `version`) — `number`, always present; every other property — `string`.
- Every list is sorted ascending by its natural key (table name, then field name, then key name) with `StringComparer.Ordinal`, so that two runs over the same pair of databases produce byte-identical JSON. Ordering deliberately does **not** use the comparer of [[#6.1 Database options, collation settlement and comparer detection|§6.1]]: a linguistic comparer orders through ICU, whose tables differ between host platforms and runtime versions, so the same databases would serialize differently on two machines. Ordinal ordering is safe here precisely because equality is still decided by the resolved comparer — two names it considers equal can never both appear in one list.
- **No plaintext password ever reaches the result.** The `left` and `right` members of `TableStructureComparisonResult` hold the connection strings with the password value already replaced by the literal `***`, so the secret is absent from the object itself and not merely from its serialized form. The JSON carries exactly the same masked strings — serialization performs no further transformation. See [[#6.6 Masking the password in the reported connection strings|§6.6]].

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

- When `options` is omitted, the method applies the library's default `JsonSerializerOptions` — the camelCase naming policy and `JsonIgnoreCondition.WhenWritingNull` listed above — so that `result.ToJson()` alone produces a report conforming to [[#5. Report structure|§5]]. When `options` is supplied it is used verbatim, and honouring the rules above becomes the caller's responsibility.
- The defaults are built once into a cached static `JsonSerializerOptions` instance; the object is never rebuilt per call. This is not a micro-optimization and not a stylistic preference: a `JsonSerializerOptions` instance caches the per-type serialization metadata it derives on first use — converters, property accessors, write order — and becomes effectively immutable afterwards. A fresh instance per call discards that cache and re-derives it through reflection every time, which costs orders of magnitude rather than percent. The rule therefore holds specifically because the configuration depends on nothing; where a value does depend on its input, it is composed per call instead ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]).
- The body is a straight delegation — `JsonSerializer.Serialize(value, options)`. Although the parameter is declared as `object`, `System.Text.Json` resolves the value's runtime type and writes the full object graph, so no `value.GetType()` overload is required.
- `value` being `null` throws `ArgumentNullException`.
- The type is `public static` and lives in the library's root namespace, so `using Diff.Structure;` is all a consumer needs.

---

## 5. Report structure

```json
{
  "left": "The connection string to the first DB, with the password value replaced by ***.",
  "right": "The connection string to the second DB, with the password value replaced by ***.",
  "collation": {
    "name": "SQL_Latin1_General_CP1_CI_AS",
    "lcid": 1033,
    "codePage": 1252,
    "comparisonStyle": 196609,
    "version": 0
  },
  "tables": {
    "detected": ["The distinct list of table names fetched from both databases, `schema.table`"],
    "ignored": {
      "left":  ["Tables fetched from the left DB that the system was unable to compare"],
      "right": ["Tables fetched from the right DB that the system was unable to compare"]
    },
    "unchecked": ["Tables the caller excluded from the comparison with ConflictResolution.Ignore"]
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

**Relationship between the table lists.** `tables.unchecked` is a subset of `tables.detected`, and holds the tables the caller excluded with `Ignore` ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). An entry that matched no table in the database it names contributes nothing and does not appear in the report. The comparison set is computed as `detected − unchecked − ignored.left − ignored.right`.

**Only tables have an `unchecked` list, and only tables can be excluded.** The asymmetry the section used to carry is gone: `ConflictResolutionExtensions.Ignore` takes a `TableIdentity` and nothing else ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]), so a field or a key cannot be taken out of the comparison in the first place and there is nothing for a member-level `unchecked` list to hold. Every exclusion the caller can express is therefore recorded in the report, and no object leaves the comparison without a trace. A caller who wants a particular field out of the report excludes the table that holds it, and `tables.unchecked` says so.

**A table found in only one database** lands in two sections at once: in `tables.ignored.<side where the table exists>`, explaining why it was not compared, and in `missings.<opposite side>.tables`, recording the structural difference. The fields and keys of such a table are **not** additionally listed as missing.

**Properties that may be absent.** The block above documents the full shape of the field descriptor. In an actual report `size`, `precision` and `scale` appear only when they carry a value — a `nvarchar` field emits `type`, `size` and `nullable` and nothing else, while a `decimal` field emits `type`, `nullable`, `precision` and `scale`. No property is ever written with a `null` value.

**The collation the report was produced under.** The `collation` section is always present and always complete: all five members are written on every report, whether the run took its collation from the two databases or from `TableStructureComparisonOptions.PreferredCollation` ([[#6.1 Database options, collation settlement and comparer detection|§6.1]]). The four numeric members are JSON numbers rather than strings, and a `0` among them is a value and not a missing one — `"version": 0` and the `"comparisonStyle": 0` of a binary collation are both written out, since the omit-when-null rule of [[#4. Result model|§4]] concerns nulls and not zeros. The section records the rule, not its provenance: nothing in the report says whether the collation was read from the databases or supplied by the caller, and nothing needs to — [[#6.1 Database options, collation settlement and comparer detection|§6.1]] admits a supplied collation only when it is also the collation of at least one of the two databases, so the section always names a collation that is genuinely in force on at least one side.

**Objects paired by a `ConflictResolution`.** When the caller has paired two differently-spelled names with `Map` — two spellings a collation folded together, or a table or field renamed on one side ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]], [[#6.2.1 Conflict analysis|§6.2.1]]) — the report renders the left database's spelling, or the right's when the object exists only on the right. So a `dbo.Orders` ↔ `dbo.CustomerOrders` pairing appears throughout the report as `dbo.Orders`, and a `ModifiedOn` ↔ `Modified` pairing as `ModifiedOn`, with every difference found between them attributed to that one name. A report therefore never shows both spellings of one paired object, and a reader who needs to know which name each server actually uses has the `ConflictResolution` set they supplied.

**Identifier form.** Every table name in the report consists of two parts — schema name and table name — separated by a `.`. Names are rendered from `PreferredName`, so an identifier that needs no quoting appears bare (`dbo.Orders`) while one that does appears quoted (`[dbo].[Order.Archive]`). Field and key names are rendered the same way. `QuotedName` is never written to the report, and the bare `Name` is used for comparison only.

---

## 6. Comparison algorithm

### 6.1 Database options, collation settlement and comparer detection

Identifier comparison follows a collation, never a rule fixed in the library: the collation the two databases share, or the one the caller named in `options.PreferredCollation` ([[#3.1 `TableStructureComparisonOptions`|§3.1]]). Before any structural work begins:

1. Read the `DatabaseOptions` of both databases through `IMetadataProvider.GetDatabaseOptionsAsync`, concurrently for the two sides. This is the run's only preliminary round trip per side, and it answers two independent questions at once: how the database compares identifiers (`Collation`, used by step 2 on both paths) and what the provider may say to it when reading metadata ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]], used from [[#6.2 Building the table sets|§6.2]] on). This read happens on **every** run, whether or not `options.PreferredCollation` is set, and step 2 needs its answer either way: without a preferred collation to settle the rule, with one to validate it.
2. Settle the `CollationInfo` this run compares identifiers under. `options.PreferredCollation` selects between two exclusive cases, and nothing else does:

   - **`PreferredCollation` is `null`** — the default, and the behaviour every earlier version of this brief described. Compare the two collations, `left.Collation` against `right.Collation`. **If they differ, throw `CollationMismatchException` carrying both names and stop.** No further metadata is read and no report is produced: when the two databases disagree on identifier equality, and the caller has expressed no preference, there is no defensible answer to whether `dbo.Orders` on one side denotes the same object as `dbo.orders` on the other. Equality is decided on the whole `CollationInfo` record, not on `Name` alone — two collations of the same name but different `Version` sort differently and are not interchangeable. If the two agree, the shared record is the run's collation. Nothing is validated on this path: the databases vouch for the collation by using it.
   - **`PreferredCollation` is not `null`.** The two databases' collations are **not compared with each other**: whether they agree or differ is of no consequence, and `CollationMismatchException` cannot be raised on this path. They are, however, each compared against the supplied value, because a preferred collation is admitted only if at least one of the two databases actually uses it. **If `options.PreferredCollation` equals neither `left.Collation` nor `right.Collation`, throw `InvalidCollationException` carrying the supplied collation and both databases', and stop.** Equality is taken on the whole `CollationInfo` record here too, so a value agreeing in `Name` but differing in `Version` is rejected like any other. Otherwise `options.PreferredCollation` is the run's collation — including when the two databases agree and it merely restates what both already use.

     The rule is what keeps the override honest. A preferred collation does not invent a comparison rule out of nothing; it **chooses which of the two databases' rules governs** when the two disagree, and that is the whole of the freedom it grants. The report can therefore always name a collation that is genuinely in force on at least one side ([[#5.1 Rules that govern the report|§5.1]]), and a typo in a hand-built `CollationInfo` fails the run instead of silently producing a report under a rule neither server holds.

   In both cases **only the collation is ever at issue.** Engine capabilities are never compared, and a difference in them is not an error: two servers of different versions may legitimately be compared, and [[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]] handles that by building each side's queries from that side's own options.
3. Hand the run's `CollationInfo` — the agreed one or the preferred one — to `INameComparerResolver.ResolveNameComparer` ([[#3.3.2 `INameComparerResolver`|§3.3.2]]). The resolver is handed the record and told nothing about where it came from; the two cases converge here and the rest of the algorithm cannot tell them apart. The call is synchronous because the mapping is pure arithmetic over the record — it touches no database and cannot block. For MS SQL it turns the collation's ignore-flags into `CompareOptions` and asks the culture named by `Lcid` for a comparer ([[#3.4.2 `MsSqlNameComparerResolver`|§3.4.2]]); a record it cannot honour fails the run with `NotSupportedException` ([[#8. Error handling, cancellation, safety|§8]]), and a caller-supplied one is treated no more leniently than a read one. This step runs after step 2's validation, so a `PreferredCollation` that is both unrecognised by the resolver and absent from the two databases fails as `InvalidCollationException` — the more specific diagnosis, and the one the caller can act on.
4. The returned instance is the single `StringComparer` used for every schema, table, field and key name for the rest of the run. It is resolved once per call and passed down into [[#6.2 Building the table sets|§6.2]]–[[#6.6 Masking the password in the reported connection strings|§6.6]]. No code path may fall back to a hard-coded comparer. The `CollationInfo` it was built from is carried to the result as `TableStructureComparisonResult.Collation` and written to the report's `collation` section ([[#4. Result model|§4]], [[#5. Report structure|§5]]) — the same record, so the rule the report names and the rule it was produced under are one object and cannot drift apart.
5. Keep both `DatabaseOptions` objects for the rest of the call. Every subsequent provider call takes the object belonging to **its own** side, and the comparer is what guarantees the pairing: nothing in the type system stops the left options from being handed to a call against the right connection string, and on two servers of different versions that mistake produces a statement the target cannot parse. Pair them at the point where the two sides are already paired — the `Task.WhenAll` of [[#6.2 Building the table sets|§6.2]] — rather than passing one shared object.

Both options objects, like the resolved comparer, are locals of the one `CompareTableStructureAsync` call. Nothing is cached across calls and nothing is stored on the comparer, which is what keeps it stateless and safe for concurrent runs ([[#3.2 `TableStructureComparer`|§3.2]]) — including two concurrent runs against the same pair of databases, one with a preferred collation and one without. The comparer reads `Collation` on both paths — to settle the rule in the default case, to validate the caller's in the preferred one — and treats the rest of the object as opaque either way: it never inspects a capability and never downcasts to `MsSqlDatabaseOptions`.

Consequences the implementation must respect:

- Under a case-sensitive collation `dbo.Table1` and `dbo.table1` are two **distinct** tables: they occupy two entries in `tables.detected` and are matched independently. Any dictionary keyed by object name must therefore be built with the detected comparer — a hard-coded case-insensitive dictionary would raise a duplicate-key error or silently drop one of them. The same applies to two columns of one table differing only in case.
- **The caller's conflict resolutions are the one thing this comparer does not govern.** Every side of a `ConflictResolution` entry is matched with `StringComparer.Ordinal` and never with the resolved comparer ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]), so the same set of entries names the same objects on every run, whatever collation the run settles on. An entry excluding `dbo.orders` therefore excludes `dbo.orders` and nothing else — not `dbo.Orders`, and not even on a case-insensitive run — and a caller wanting both spellings gone writes one entry per spelling. Matching on the bare two-part name is what makes `[dbo].[Order.Archive]` and `dbo.Orders` equally writable.
- The comparison follows the collation's case, accent, kana and width rules, because all four reach the comparer through `ComparisonStyle`. Under an accent-insensitive collation `Café` and `Cafe` are therefore one identifier, as they are on the server. What is **not** reproduced is the collation's exact sort order ([[#3.4.2 `MsSqlNameComparerResolver`|§3.4.2]]) — which is why report ordering is kept independent of this comparer ([[#4. Result model|§4]]).
- A preferred collation governs **everything** the resolved comparer governs, not merely the matching of table names: field names, key names and every dictionary built along the way ([[#6.2 Building the table sets|§6.2]]–[[#6.6 Masking the password in the reported connection strings|§6.6]]) use the one comparer of step 4. The single exception is the matching of `ConflictResolution` entries, which is ordinal by specification and is not this comparer's business at all. Otherwise there is no path on which one part of a run follows the databases' rule and another the caller's, and no code beyond step 2's validation may reach past the comparer to `DatabaseOptions.Collation` to get the "real" rule back.
- A preferred collation picks a side; it cannot invent a third rule. Because step 2 admits only a value one of the databases actually uses, the reachable outcomes are exactly two: the run proceeds under the left database's collation, or under the right's. A case-sensitive left and a case-insensitive right — a pair that fails outright without the property — compare under whichever of the two the caller names. Naming the right's, `dbo.Orders` on the left and `dbo.orders` on the right are one table whose structures are compared against each other; naming the left's, they are two tables, each landing in the other side's `missings.*.tables` — unless the caller pairs them with a two-sided `ConflictResolution` entry, which settles the question regardless of the collation. Neither reading is more correct than the other, which is precisely why the choice is the caller's.
- **Choosing the looser side can make two objects of the stricter side collide,** and the run must not crash on it. A case-sensitive database may legitimately hold both `dbo.Table1` and `dbo.table1`; run under a case-insensitive `PreferredCollation` taken from the other side, the comparer considers those two names equal, and any dictionary keyed by name would fail with a duplicate key. Colliding names are not an error in the databases, and the run does not guess: [[#6.2.1 Conflict analysis|§6.2.1]] detects the collision before anything is keyed by name and either asks the caller to settle it through `AmbiguousNameException`, when there is a name on the other side to pair with, or stops with `ObjectLostException` ([[#6.2 Building the table sets|§6.2]] step 5) when a database's own two names would be folded into one. The same applies to two fields of one table and to two keys of one table. This case cannot arise on the default path, where both databases share the collation that is judging them.
- A preferred collation changes nothing about how metadata is read. The four metadata queries are built from each side's own `DatabaseOptions` exactly as before ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]), the shadow-table filter is unaffected, and the server still sorts and filters under its own collation — the caller's collation applies to the comparison the library performs in memory, never to a `WHERE` clause sent to a database.

### 6.2 Building the table sets

0. Snapshot `options.ConflictResolution` and validate it ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]): no object claimed by two entries on the same side — the one contradiction the factory methods cannot prevent — and, as internal invariants the public API can no longer breach, no entry with both sides `null` and no side whose runtime type contradicts its `Kind`. A violation fails the call with `InvalidConflictResolutionException` ([[#8. Error handling, cancellation, safety|§8]]). This is a pure check over the caller's own objects and needs no database at all, so it happens at the very start of `CompareTableStructureAsync`, alongside the argument validation of [[#8. Error handling, cancellation, safety|§8]] and ahead of the reads of [[#6.1 Database options, collation settlement and comparer detection|§6.1]] — a contradictory set costs no round trip. The snapshot is what every step below reads, never the live set.
1. Fetch the tables of both databases concurrently (`Task.WhenAll` over the two `GetTablesAsync` calls), each passing that side's own `DatabaseOptions` from [[#6.1 Database options, collation settlement and comparer detection|§6.1]].
2. Apply the one-sided entries, every one of which is of `Kind = Table` — there is no other kind of one-sided entry ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). Match each such entry against its own side's table names with `StringComparer.Ordinal`, as [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] prescribes and never with the comparer of [[#6.1 Database options, collation settlement and comparer detection|§6.1]], and set the matched tables aside: they become `tables.unchecked`. This is a lookup per entry and needs no name-keyed dictionary of the tables themselves, which is why it can run before step 3.
3. Apply the two-sided entries of `Kind = Table` to what remains, matching each side the same ordinal way. Each entry claims one table on each side and binds them into a **forced pair**; both are removed from the pool the comparer would otherwise match by name. An entry that matches on one side but not on the other fails the call with `InvalidConflictResolutionException` ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]); an entry matching on neither side is ignored. Forced pairs are carried through the steps below as already-paired tables, never re-examined by name.
4. Run the **table stage** of the conflict analysis ([[#6.2.1 Conflict analysis|§6.2.1]]) over the table names still unpaired, before anything is keyed by name. Either it finds unresolved collisions and the run ends with `AmbiguousNameException`, or every remaining table name pairs with at most one name on the other side and the steps below are safe.
5. Guard against a name being lost in the union that follows. Take each side on its own and hash that side's still-unpaired table names with the resolved comparer; if the resulting set holds fewer items than the list it was built from, the comparer has folded two of that database's tables into one. Identify the folded names, and **throw `ObjectLostException` naming them, their side, and the collation the comparer was resolved from ([[#6.1 Database options, collation settlement and comparer detection|§6.1]])** — unless every one of them was already excluded at step 2 or bound into a forced pair at step 3, in which case nothing is lost that anyone asked to compare and the run continues. The check is run for the left side and for the right side independently, and it is the reason `tables.detected` can be formed at all: it converts a silent loss into a stop.
6. `tables.detected` = the distinct union of the left and right table names, the excluded ones included, compared with the detected comparer — a dedup across the two sides only, never within one, which is exactly what steps 4 and 5 have just established. A forced pair contributes **one** entry, rendered in the left database's spelling ([[#5.1 Rules that govern the report|§5.1]]).
7. `tables.ignored.left` = tables present in the **left** database, not excluded by the caller, not bound into a forced pair, and **absent in the right** one. `tables.ignored.right` is the mirror image. A table whose partner was taken by a forced pair falls in here like any other unmatched table.
8. Comparison set = `detected − unchecked − ignored.left − ignored.right`, and every forced pair is in it.
9. A table that exists only in the left database is reported in `missings.right.tables`; a table that exists only in the right database is reported in `missings.left.tables`. A forced pair is never reported as missing on either side.
10. A table listed in `tables.unchecked` is never reported as missing or inconsistent, even when it exists in one database only.
11. Fields, primary keys and foreign keys are then fetched for the comparison set only, again concurrently for the two databases, each call passing that side's `DatabaseOptions` alongside the comparison set. A forced pair contributes **its own side's** name to each side's request — the left database is asked about `dbo.Orders` and the right about `dbo.CustomerOrders` — which is the one place the two requests are not the same list. An empty comparison set short-circuits in the provider, which [[#3.3.1 `IMetadataProvider`|§3.3.1]] requires to return empty without touching the database.
12. Apply the caller's entries of `Kind = Field` inside each pair of tables in the comparison set, exactly as step 3 did for tables and under the same ordinal comparison: each binds a forced pair of fields that survives whatever the comparer would have said. There is nothing else to apply here — a member entry is always two-sided and always a field, since `Map(FieldIdentity, FieldIdentity)` is the only factory method that produces one ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]), so no member is ever removed from the comparison at this step and `tables.unchecked` needs no counterpart below the table level. An entry whose owner names a table outside the comparison set is ignored, as its table was never going to be compared. An entry that matches a field on one side and none on the other fails the call with `InvalidConflictResolutionException`, as at step 3.
13. Run the **member stage** of the analysis ([[#6.2.1 Conflict analysis|§6.2.1]]) over the fields, primary keys and foreign keys that remain unpaired. Only fields can have been pre-paired at step 12; a key class reaches the analysis exactly as the database reported it. As at step 4, the run either ends here with `AmbiguousNameException` or proceeds knowing every name pairs unambiguously. Only then does [[#6.3 Field comparison|§6.3]] begin.

### 6.2.1 Conflict analysis

The caller's conflict resolutions are applied first, at [[#6.2 Building the table sets|§6.2]] steps 2, 3 and 12; what this section covers is what is left over — the names the run's comparer must pair on its own, and the case where it cannot. A collation the caller chose can be looser than the one a database enforces, and then two objects that database holds as distinct collapse into one name ([[#6.1 Database options, collation settlement and comparer detection|§6.1]]). Before any structure is compared, the run establishes that no such collision is left undecided. The vocabulary — `CandidateInfo`, `ObjectKind`, `ConflictResolution` — is [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]].

**The rule.** For one kind of object, group every name still in play into classes under the comparer of [[#6.1 Database options, collation settlement and comparer detection|§6.1]]. Names already excluded or already bound into a forced pair are not in play and form no class. A class is a collision when neither of its sides is empty and at least one side holds more than one distinct name; a class whose opposite side is empty is not one, and its names are handled as ordinary one-sided findings. A collision becomes a `CandidateInfo` — `Kind` set, `Left` and `Right` carrying that class's names from each side — and when the analysis ends with a non-empty candidate collection, `AmbiguousNameException` is thrown carrying it. When it ends empty, every name pairs with at most one name on the other side and comparison begins.

| Names in one database | Names in the other | Outcome |
| --- | --- | --- |
| `dbo.Table1` | none | not a candidate — nothing to compare against, the table is ignored |
| `dbo.Table1`, `dbo.table1` | none | not a candidate — same reason, for both names |
| `dbo.Table1`, `dbo.table1`, `dbo.TABLE1` | none | not a candidate — same reason, for all three |
| `dbo.Table1`, `dbo.table1` | `dbo.table1` | **candidate**, unless `ConflictResolution` has already paired or excluded them |
| `dbo.Table1`, `dbo.table1` | `dbo.table1`, `dbo.Table1` | **candidate**, unless already paired or excluded |
| `dbo.Table1`, `dbo.table1`, `dbo.TABLE1` | `dbo.table1`, `dbo.Table1` | **candidate**, unless already paired or excluded |
| `dbo.Table1`, `dbo.table1` | `dbo.table1`, `dbo.Table1`, `dbo.TABLE1` | **candidate**, unless already paired or excluded |

The table is symmetric: which database holds the several names makes no difference.

Requiring both sides to be non-empty is what keeps the definition honest: a collision only needs an answer when there is something on the other side to pair with. A class whose opposite side is empty poses no pairing question at all — every one of its names is simply absent from the other database, which is an ordinary finding the report already has a place for ([[#6.2 Building the table sets|§6.2]]), not a conflict for the caller to settle.

**What clears a class.** A class disappears from this analysis when the caller's entries have emptied it of ambiguity — when the names bound into forced pairs or excluded at [[#6.2 Building the table sets|§6.2]] steps 2, 3 and 12 leave behind a class with at most one name on each side, or with nothing at all on one of them. A class reduced that far is not a collision.

Which remedies are available depends on the kind, because the factory methods of [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] are not symmetric across kinds:

- **A table class** can be cleared either way. `Map` pairs two of its names and `Ignore` removes one, and a caller answering an `AmbiguousNameException` about tables may use either or both — both readings are equally legitimate.
- **A field class** is cleared by `Map` alone, since a field cannot be excluded. Pairing is always enough: each `Map` takes one name off each side, so repeated pairing drives one side empty, and a class whose opposite side is empty is not a collision. A caller who does not want to pair them excludes the table instead, which removes the whole class with it.
- **A key class** — `PrimaryKey` or `ForeignKey` — has no entry that can touch it, since no factory method produces one. Its only remedy is `Ignore` on the table that holds the keys, which takes that table out of the comparison set and out of the member stage altogether. A primary-key class cannot arise at all, SQL Server permitting one primary key per table; a foreign-key class can, and only under a `PreferredCollation` looser than the database's own.

**The analysis does not stand alone.** It answers the collisions a caller *can* settle by pairing — those with a name on each side. A second, narrower guard covers what is left: [[#6.2 Building the table sets|§6.2]] step 5 checks, for each database separately, whether the resolved comparer would fold two of that database's own table names together, and stops the run with `ObjectLostException` when it would. The two are complementary and neither subsumes the other. `AmbiguousNameException` says *tell me which of these pairs with which*; `ObjectLostException` says *this comparison cannot represent your database at all, because two of its tables have become one name and the report has one slot for them*. Only the second can fire for a class with nothing on the opposite side, and its remedy is not a pairing — there is no second name to pair with — but a one-sided `ConflictResolution` entry per folded name, which the guard honours by design.

**Why it runs in two stages.** The analysis is one procedure but it cannot be one pass, because names nest: which fields belong to `dbo.Orders` is not a question that can be asked while it is still open whether `dbo.Orders` and `dbo.orders` are one table or two. The table stage therefore runs at [[#6.2 Building the table sets|§6.2]] step 4, on table names alone; the member stage runs at step 13, over the fields, primary keys and foreign keys of tables whose pairing the first stage settled. Each stage reports everything it found before stopping — all kinds of the member stage go into one exception together — so a caller pays at most two round trips of disambiguation, not one per collision. A caller who resolves the tables named by the first exception and runs again will see the second stage's candidates, if any, on that second run.

**What the analysis covers.** Only objects that would actually be compared. Tables excluded by a one-sided entry are removed before the table stage looks at what is left ([[#6.2 Building the table sets|§6.2]] step 2 decides membership by a lookup per entry and needs no name-keyed dictionary of its own, so it is safe to run first), and the member stage sees only the comparison set. Excluding a table is therefore a legitimate way to answer an `AmbiguousNameException` about it, and a collision inside a table nobody wants compared never stops a run.

**Which spelling reaches the report.** A pair holds two names — two spellings of one name after a collation folded them, or two genuinely different names after a rename — and the report has one slot. Every report entry renders the **left** database's spelling of a paired object, falling back to the right's when the object exists only on the right. This is arbitrary but fixed, and it is what keeps the report deterministic — the alternative, rendering whichever side the metadata happened to arrive from, would make two runs of the same comparison differ.

Consequences the implementation must respect:

- The analysis is the only thing standing between a looser collation and a duplicate-key exception. Every name-keyed structure in [[#6.2 Building the table sets|§6.2]]–[[#6.6 Masking the password in the reported connection strings|§6.6]] is built after the stage that clears its kind, and no dictionary is keyed by a name whose class has not been checked — which is the whole reason the table stage sits where it does rather than after `tables.detected` is materialized.
- A `ConflictResolution` entry that names objects with no conflict is not an error and has no effect. The set is a standing instruction, not a per-run script: a caller may keep one set across many comparisons, and entries whose names neither collide nor need pairing in a given pair of databases are simply never consulted. The exception is a two-sided entry that matches on one side only, which is a pairing the library cannot honour and fails the run ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]).
- Resolution is per kind **and per owner**. An entry of `Kind = Field` does not resolve a table collision of the same spelling, because `Kind` participates in the match; and an entry for a field of `dbo.Orders` does not resolve the same collision in `dbo.Invoices`, because the owner chain does. Two tables colliding on the same field spelling are two candidates and take two `Map` calls.
- The analysis and the entries do not share a comparison rule, and the asymmetry is what makes the mechanism work. A class is formed with the run's comparer, which is by definition unable to tell its members apart; an entry finds its object with `StringComparer.Ordinal`, which can. An entry can therefore name one member of a class the analysis could only report as a whole — and that, rather than any property of the collation, is why every collision with a name on each side is answerable.
- A candidate hands back the objects themselves — a `TableIdentity`, a `FieldIdentity`, a key identity — so the entries that answer it are built by passing what the exception carried to `Ignore` or `Map` rather than retyping it. The analysis reads only `Kind`, `Name` and `Owner` from either side, so an entry built from candidates and one built from identities the caller assembled behave identically as long as those three agree. A key identity is the one a candidate can hand back that no method accepts; the answer there is to `Ignore` its table.
- A forced pairing is not checked against the comparer and cannot be refused by it. Pairing `dbo.Orders` with `dbo.CustomerOrders` is a statement no comparison rule could have produced, and the run makes no attempt to second-guess it — which is exactly what makes the mechanism able to survive a rename.
- The analysis never reads the databases again. It works entirely over metadata already fetched, so neither stage adds a round trip and the five-per-database budget of [[#9. Non-functional requirements|§9]] is untouched.

### 6.3 Field comparison

Within each table of the comparison set, fields are matched **by name**, using the comparer detected in [[#6.1 Database options, collation settlement and comparer detection|§6.1]]. Ordinal position is **not** compared. Fields the caller bound into a forced pair with `Map` are already matched when this step begins and are not matched by name at all ([[#6.2 Building the table sets|§6.2]] step 12). There is no such thing as an excluded field: every field of a compared table is compared ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]).

- Field present on the right only → `missings.left.fields`.
- Field present on the left only → `missings.right.fields`.
- Field present on both sides → the tuple `(type, size, nullable, precision, scale)` is compared. If any component differs, an entry is added to `inconsistencies.fields` carrying the full descriptor of both sides. Type names are compared case-insensitively.

### 6.4 Primary key comparison

SQL Server allows at most one primary key per table, so only the outcome matters. The order of the fields inside the key is **not** taken into account — the participating fields are compared as a set under the detected comparer, and a field pair the caller forced with `Map` counts as one field on both sides. Primary keys themselves are never paired or excluded by the caller; there is no entry that could name one.

| Left PK | Right PK | Fields | Report |
| --- | --- | --- | --- |
| exists | exists | same set | nothing |
| exists | exists | different set | `inconsistencies.primaryKeys` — both key names (for reference only) and both field lists |
| exists | absent | — | `missings.right.primaryKeys` with the left key's `key`, `table`, `fields` |
| absent | exists | — | `missings.left.primaryKeys` with the right key's `key`, `table`, `fields` |
| absent | absent | — | an entry containing **only** `table` in *both* `missings.left.primaryKeys` and `missings.right.primaryKeys` |

Key names are carried into the report for reference only and never drive the decision: two primary keys with the same field set but different names produce no entry.

### 6.5 Foreign key comparison

Foreign keys of a table are grouped by the pair `(table, referencedTable)`. The caller has no say in this stage — no factory method produces an entry of `Kind = ForeignKey`, so every key of every compared table goes through the matching order below, and a caller who wants a table's keys left alone excludes the table itself ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). Field pairs the caller forced with `Map` are honoured inside the comparison, since a key's composition is read through its fields. Within each group the left and right keys are matched in the following order:

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

- Table, field and key names are compared with the comparer detected in [[#6.1 Database options, collation settlement and comparer detection|§6.1]], never with a hard-coded one, and always through `IQuotedIdentifier.Name` — the bare, unquoted form, which is independent of the quoting policy in `TableStructureComparisonOptions`. Comparing `QuotedName` or `PreferredName` would make `[Orders]` and `Orders` two different tables. Tables and fields the caller bound into a forced pair with `Map` are the one exception: their pairing is fixed by the entry and the comparer is never asked about them ([[#6.2.1 Conflict analysis|§6.2.1]]).
- The casing rendered in the report is the casing returned by the **left** database when the object exists there, otherwise the casing returned by the right database.
- Because a report name is the concatenation `schema + "." + table`, an identifier that itself contains a dot yields an ambiguous string. Such names are emitted verbatim, without quoting (see **[[#11. Assumptions and open questions|§11]]**).

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
- A table excluded with `Ignore` never appears anywhere in `difference`. Asserted for `Ignore(table, From.Left)`, for `Ignore(table, From.Right)`, and for `Ignore(table)` — the last putting the table in `tables.unchecked` whichever database holds it.
- A primary key whose fields are declared in a different order on the two sides produces **no** entry.
- A foreign key whose field pairs are declared in a different order on the two sides produces **no** entry.
- `Ignore` called with a `TableIdentity` spelling `dbo.orders` leaves `dbo.Orders` compared and absent from `tables.unchecked`, and does so under a case-sensitive and a case-insensitive collation alike, since entries are matched ordinally and never with the run's comparer ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). One entry per spelling excludes both. The case-insensitive half of this is the regression that guards the ordinal rule: a run whose collation calls the two names equal must still honour only the spelling the entry names.
- On a case-sensitive collation, `dbo.Table1` and `dbo.table1` present in both databases are reported as two independent tables, and a `Description` column alongside a `description` column is likewise treated as two fields.
- With `PreferredCollation` left unset, two databases whose collations differ raise `CollationMismatchException`, and no report is returned.
- With `PreferredCollation` set to either database's collation, those same two databases are compared without error, and identifier equality follows the supplied collation. The same holds when the two databases already share a collation and the supplied one restates it.
- A case-sensitive left database holding `dbo.Orders` and a case-insensitive right one holding `dbo.orders`: under a `PreferredCollation` equal to the right's collation the two are one table whose structures are compared against each other; under one equal to the left's they are two tables, each in the opposite side's `missings.*.tables`.
- `PreferredCollation` does **not** govern the matching of exclusions: with the two databases above, a left-side entry naming `dbo.orders` matches nothing in a left database holding only `dbo.Orders`, under either side's collation, so switching the property changes the comparison but never which entries applied.
- A `PreferredCollation` equal to neither database's collation — including one agreeing in `Name` but differing in `Version` — fails the run with `InvalidCollationException` before any table metadata is read.
- A `PreferredCollation` whose `Lcid` is unknown, and which matches neither database, fails with `InvalidCollationException` rather than `NotSupportedException`: the validation of [[#6.1 Database options, collation settlement and comparer detection|§6.1]] step 2 precedes the resolution of step 3.
- A case-sensitive database holding `dbo.Table1` and `dbo.table1`, compared under a `PreferredCollation` equal to the other, case-insensitive database's collation, fails with `AmbiguousNameException` whose single `CandidateInfo` has `Kind = Table` and carries both names in the set for that side.
- The same pair, run again after a `Map` of the two, compares without error, and the report renders the left database's spelling of the paired table; run instead after an `Ignore` of one of the two colliding names, it also compares without error, with that name — and only that name — in `tables.unchecked`. Both assert the point of the ordinal rule: the entry singles out one member of a class whose members the run's own comparer cannot distinguish.
- A class with names on one side only produces no candidate: a case-sensitive left database holding `dbo.Table1` and `dbo.table1` where the right database holds neither raises no `AmbiguousNameException`, for one name, two, or three.
- That same fixture raises `ObjectLostException` at [[#6.2 Building the table sets|§6.2]] step 5, naming both table names, the left side, and the run's collation — and raises nothing once both names are excluded, one `Ignore(name, From.Left)` per spelling.
- The candidate table of [[#6.2.1 Conflict analysis|§6.2.1]] is exercised row by row, in both directions, so that which database holds the several names makes no difference to the outcome.
- Two fields of one compared table colliding the same way produce a `CandidateInfo` with `Kind = Field` — and do so only once the table stage has found nothing, so the two exceptions are never raised from one run.
- A collision inside an excluded table raises nothing: the table never reaches the analysis.
- A `ConflictResolution` entry naming objects that neither collide nor need pairing is ignored, and a run carrying only such entries behaves as if the set were empty.
- **The property is `init`-only and its set is used as given.** A set built before the options object and handed in through the initializer (`ConflictResolution = existingSet`) governs the run exactly as one filled afterwards does; the library neither copies it nor replaces it, and the same set reused across two consecutive comparisons applies to both. The property has no setter, asserted at build level alongside the bullet below.
- **The construction surface is closed.** A compile-time assertion — an analyzer test, or a test project that fails to build — pins that `new ConflictResolution(...)` is unavailable outside the assembly, that no public member of any type returns a `ConflictResolution` a caller could clone, and that `with` cannot rewrite a side. The four factory methods are the whole surface, and this is the test that keeps it that way.
- **A foreign-key collision has exactly one remedy.** Two foreign keys of one table differing only in case, on a database whose own collation tells them apart, compared under a looser `PreferredCollation`: the run fails with `AmbiguousNameException` carrying a `CandidateInfo` of `Kind = ForeignKey`, and passes once the table holding them is excluded with `Ignore`. There is no entry that resolves the class itself, and the test asserts that excluding the table is what clears it.

Every row of the interpretation table in [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] is a required case of its own, and each is covered against a fake `IMetadataProvider`:

- **Left set, `Right` `null`, `Kind = Table`.** Both databases hold `dbo.Table1`; an entry naming the left one excludes it. `dbo.Table1` appears in `tables.detected` and in `tables.unchecked`, in neither `tables.ignored` nor `missings`, and nowhere under `difference` — even though the two copies differ structurally, which the same fixture without the entry reports.
- **`Left` `null`, `Right` set, `Kind = Table`.** The mirror image, entry by entry: the right-side entry excludes the right database's `dbo.Table1` with the same outcome, so which side carries the exclusion makes no difference to the report.
- **Both sides set, differing only in case, `Kind = Table`.** A left `dbo.Table2` and a right `dbo.table2` under a case-sensitive run — where the comparer pairs neither with the other and both would otherwise land in `missings` — are compared against each other because of the entry, and the report names the pair `dbo.Table2`. The same shape of entry is what settles a collision under a looser `PreferredCollation`, covered in the ambiguity group below, which is the second reason this row exists.
- **Both sides set, genuinely different names, `Kind = Table`.** A left `dbo.Orders` paired with a right `dbo.CustomerOrders`: the two are compared, the report names the pair `dbo.Orders`, neither name appears in `tables.ignored` or `missings.*.tables`, and a field present only on the right of the pair surfaces in `missings.left.fields` against `dbo.Orders`. A variant where the left database *also* holds its own `dbo.CustomerOrders` asserts the consuming rule: that table is left without a partner and lands in `tables.ignored.left` and `missings.right.tables`.
- **Both sides set, `Kind = Field`.** `ModifiedOn` of the left database's `dbo.Tasks` paired with `Modified` of the right database's `dbo.Tasks`: the two fields are compared as one, the report names the pair `ModifiedOn`, neither name appears in `missings.*.fields`, and a difference in their data types is reported in `inconsistencies.fields` under `ModifiedOn`. Without the entry the same fixture yields one entry in each side's `missings.*.fields`, which is what makes the entry's effect visible.
- **A field pairing inside a paired table.** The two cases above combined — `dbo.Orders` ↔ `dbo.CustomerOrders` with a field entry whose left side hangs from `dbo.Orders` and whose right side hangs from `dbo.CustomerOrders` — compares cleanly, proving that each side of a member entry names its own database's spelling of the owning table.
- **The two-sided `Ignore`.** `Ignore(table)` adds exactly two entries and is indistinguishable in its effect from `Ignore(table, From.Left)` followed by `Ignore(table, From.Right)`; called twice it still adds two, the set dedups the repeats, and the run is unaffected.
- **The factory methods add and chain.** Each method adds to the collection it was called on — asserted on `options.ConflictResolution.Count` — and returns that same instance, so a chained `Ignore(a).Map(b, c)` leaves both statements in the one set. A read-only `ICollection<ConflictResolution>` receiver throws `ArgumentException`; a `null` receiver or identity throws `ArgumentNullException`; a `From` value outside `Left` and `Right` throws `ArgumentOutOfRangeException`.
- **Two entries claiming the same object on one side**, and a two-sided entry matching on one side only, each fail the call with `InvalidConflictResolutionException`; the first does so before any metadata is read. The fixture for the first is a reachable one — `Ignore(orders, From.Left)` alongside `Map(orders, customerOrders)` — since the factory methods rule out the other shapes.
- **The invariants the API can no longer breach.** An entry with both sides `null` and one whose runtime type contradicts its `Kind` are still rejected, asserted through the `internal` constructor from inside the assembly (`InternalsVisibleTo`), and are marked in the suite as unreachable through the public surface.
- **The matching rule is asserted directly.** An entry naming `dbo.Orders` matches `dbo.Orders` and nothing else; entries naming `dbo.orders`, `DBO.ORDERS` and `dbo.Ordrs` each match nothing and are ignored without error, asserted under a case-insensitive collation as well as a case-sensitive one so that the run's comparer is shown to play no part. A field entry naming `ModifiedOn` matches that field in the table its `Owner` names and leaves the identically-spelled field of another table untouched, and one naming `modifiedon` matches neither. An entry built with a different `QuotesUsage` or `QuoteInfo` than the run's still matches, since only `Kind`, `Name` and the owner chain are compared.
- **A wrong-case entry is ignored, on both sides, and this is the specified outcome rather than a tolerated defect.** The left database holds `dbo.Orders` and an entry's `Left` names `dbo.orders`: the entry is not taken into account for that table, `dbo.Orders` is compared as though no entry existed, it is absent from `tables.unchecked`, and the call neither throws nor reports the entry anywhere. The mirror case — the right database holding `dbo.Orders` against an entry's `Right` naming `dbo.orders` — is asserted the same way. Both are run twice, under a case-sensitive collation and a case-insensitive one, to pin that the outcome does not depend on the collation.
- An entry whose `Kind` does not match the colliding objects' kind does not resolve them, and the run still fails with `AmbiguousNameException`.
- With `PreferredCollation` unset, no fixture raises `AmbiguousNameException`.
- The `collation` section of the report names the run's collation in every case above: the shared one when `PreferredCollation` is unset, the supplied one when it is set, with all five members written and numbers serialized as numbers.
- A comparison run with a password in both connection strings produces a result whose `Left` and `Right` members, and whose serialized JSON, show the keyword with the masked value (`Password=***`) and nowhere contain the real password.

---

## 8. Error handling, cancellation, safety

- **Argument validation.** A `null` `options` value, or a `null`/empty connection string on either side, throws `ArgumentNullException` / `ArgumentException` synchronously, before any I/O starts.
- **Collation mismatch.** When `options.PreferredCollation` is `null` and the two databases do not share the same collation, `CollationMismatchException` is thrown ([[#6.1 Database options, collation settlement and comparer detection|§6.1]]). It carries both collation names and reports which side had which. It is raised before any table metadata is read — though after `GetDatabaseOptionsAsync` has answered for both sides, since that call is what supplies the collations — so the call fails fast and returns no partial report. When `PreferredCollation` is set the check does not run at all and this exception cannot be raised, whatever the two databases report. A difference in engine **capabilities** is not a mismatch and never fails the call in either case.
- **Invalid preferred collation.** When `options.PreferredCollation` is set and equals neither `left.Collation` nor `right.Collation`, `InvalidCollationException` is thrown ([[#6.1 Database options, collation settlement and comparer detection|§6.1]]). It carries the supplied collation and both databases', so the caller can see what was asked for and what was on offer. Like the mismatch above it is raised after `GetDatabaseOptionsAsync` has answered for both sides and before any table metadata is read, and it returns no partial report. Equality is taken on the whole `CollationInfo` record, so a value differing only in `Version` is rejected — the diagnostic value of the exception depends on saying so in its message rather than letting the caller compare two names that look identical.
- **Unsupported collation.** A `CollationInfo` the resolver cannot honour — an `Lcid` unknown to `CultureInfo`, a `ComparisonStyle` carrying an unrecognised bit — fails the call with `NotSupportedException` naming the collation ([[#3.4.2 `MsSqlNameComparerResolver`|§3.4.2]]). A caller-supplied `PreferredCollation` reaches this check only after passing the validation above, so in practice it fails here only when the database it matches uses a collation the resolver itself cannot honour — in which case the run would have failed the same way without the property. A bad value is never approximated, and never silently replaced by the databases' own collation. Like the failures above, it surfaces at [[#6.1 Database options, collation settlement and comparer detection|§6.1]] before any table metadata is read.
- **Wrong or missing options object.** `MsSqlMetadataProvider` throws `ArgumentException` when a metadata call receives a `DatabaseOptions` that is not an `MsSqlDatabaseOptions`, and `ArgumentNullException` when it receives none ([[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]). This is a composition error rather than a data error: it surfaces before any query is built or executed, and must not be caught and degraded into a query built on assumed capabilities.
- **A malformed call to a factory method.** `ConflictResolutionExtensions.Ignore` and `.Map` validate their arguments before adding anything ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]): a `null` receiver or a `null` identity throws `ArgumentNullException`, a `From` value that is neither `Left` nor `Right` throws `ArgumentOutOfRangeException`, and a receiver whose `IsReadOnly` is `true` throws `ArgumentException`. These are thrown by the call that builds the entry, long before `CompareTableStructureAsync`, and are the reason most of what `InvalidConflictResolutionException` used to cover can no longer arise. A receiver that cannot be added to *at all* is not among them: `ICollection<ConflictResolution>` is the declared parameter type, so that mistake is a compile error rather than an exception ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]).
- **A conflict resolution the library cannot honour.** `InvalidConflictResolutionException` is thrown, carrying the offending entries, when the snapshot of `options.ConflictResolution` contradicts itself or contradicts the databases ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]): two entries claiming the same object on the same side, or a two-sided entry that matches an object on one side and none on the other. The first is decided by validating the snapshot alone and is raised at [[#6.2 Building the table sets|§6.2]] step 0, which runs before any I/O; the second needs the metadata and is raised as soon as the entry is applied — step 3 for a table, step 12 for a field. Two further shapes — an entry with both sides `null`, and one whose non-null side carries a runtime type its `Kind` does not call for — are still rejected by that same step-0 pass, but no caller can produce them: the factory methods take typed identities and emit no empty entry, so these are internal invariants rather than reachable failures. An entry matching nothing on either side is not an error and raises nothing — including one whose spelling differs from the database's only in case, which applies to nothing and is passed over in silence ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]). No partial report is produced in any case.
- **Ambiguous names.** When the conflict analysis of [[#6.2.1 Conflict analysis|§6.2.1]] ends with a non-empty candidate collection, `AmbiguousNameException` is thrown carrying it as an `IReadOnlyCollection<CandidateInfo>`, so the caller can see every collision at once and answer them in one edit to `TableStructureComparisonOptions.ConflictResolution` — with `Map` for a table or field class, with `Ignore` for a table class, and with `Ignore` of the owning table for a key class, which is the only remedy a key class has ([[#6.2.1 Conflict analysis|§6.2.1]]). No report is produced and no structure is compared. The exception can be raised at either of the analysis's two stages ([[#6.2.1 Conflict analysis|§6.2.1]]); reaching the second means the first found nothing left unresolved. It cannot be raised at all when `PreferredCollation` is unset, since a database does not hold two objects whose names its own collation calls equal.
- **A table lost to the comparison rule.** When the resolved comparer folds two table names of one and the same database into a single name, `ObjectLostException` is thrown ([[#6.2 Building the table sets|§6.2]] step 5). It carries the folded names, the side they came from, and the `CollationInfo` the comparer was resolved from, since that collation is the cause and the only thing the caller can change. It is not raised when every folded name was already excluded with `Ignore` or bound into a forced pair with `Map` — either is the caller stating that the outcome is intended. Like `AmbiguousNameException` it is unreachable while `PreferredCollation` is unset, and it is raised before `tables.detected` exists, so no partial report is produced.
- **Connectivity and permission failures.** A `SqlException` raised while reading metadata is wrapped in a dedicated `MetadataAccessException` that carries the affected side (`Left` / `Right`) and the original exception as `InnerException`. A partial report is never returned — a failure on either side fails the whole call.
- **Cancellation.** `cancellationToken` is threaded through every provider call and every ADO.NET call. Cancellation surfaces as `OperationCanceledException`; no partial result is produced.
- **Read-only guarantee.** The library issues `SELECT` statements against catalog views only. Callers are advised to use a login that has `VIEW DEFINITION` and no write permissions.
- **No credential leakage.** The result object and its JSON carry connection strings whose password is masked as `***` ([[#6.6 Masking the password in the reported connection strings|§6.6]]), so a report may be logged, persisted or forwarded without exposing a secret. Exception messages produced by the library must not embed a raw connection string either; `MetadataAccessException` identifies the failing side by name, not by connection string.

---

## 9. Non-functional requirements

- **Round trips.** Exactly five queries per database (database options, tables, fields, primary keys, foreign keys). No per-table queries. The options query runs first and gates the other four: it supplies both the collation the comparer is resolved from and the capability the four statements are built from. Neither resolving the comparer nor building a query costs a round trip — `INameComparerResolver` and `MsSqlMetadataQueryBuilder` are both pure, in-process mappings. The ledger capability rides along as a column of the options query rather than being probed by the metadata statements themselves, which is what lets those four be plain static SQL and still keeps the count at five. Passing the options into each metadata call, rather than having the provider re-read them, is what stops that count from becoming nine. A `PreferredCollation` does not lower the count either: the options query still runs on both sides, and its collation columns are still read — the capability has no other source, and [[#6.1 Database options, collation settlement and comparer detection|§6.1]] validates the caller's collation against the two it returns ([[#3.3.1 `IMetadataProvider`|§3.3.1]]).
- **Parallelism.** Left and right metadata are read concurrently.
- **Throughput target.** A pair of databases with 1,000 tables and 20,000 columns is compared in under 10 seconds over a LAN, excluding SQL Server response time.
- **Memory.** Metadata is held in memory for the duration of one comparison; the working set stays proportional to the metadata size, with no duplication of the raw reader output.
- **Thread safety.** `TableStructureComparer` and `MsSqlMetadataProvider` are stateless and safe for concurrent use.
- **Async discipline.** No `.Result`, no `.Wait()`, and no `Task.Run` wrappers around synchronous I/O anywhere in the library.

---

## 10. Testing requirements

1. **Unit tests of the comparer** against a fake `IMetadataProvider`, covering every row of the table in [[#7. Acceptance scenarios|§7]] plus the listed regression cases. These require no SQL Server instance and must run in CI. One of them covers the pairing obligation of [[#6.1 Database options, collation settlement and comparer detection|§6.1]] step 5: with a fake returning a distinguishable options object per side, assert that every metadata call received the object belonging to its own connection string. Another covers both branches of [[#6.1 Database options, collation settlement and comparer detection|§6.1]] step 2 against a fake that reports whatever collations the test asks for: with `PreferredCollation` left `null`, differing collations throw `CollationMismatchException` and equal ones resolve the comparer from the shared record; with `PreferredCollation` set to a value one side reports, the two collations are no longer compared with each other and the comparer is resolved from the supplied record — asserted for a differing pair and for an equal pair; and with it set to a value neither side reports, `InvalidCollationException` is raised before any metadata call. A spy `INameComparerResolver` recording the `CollationInfo` it was handed is what turns "resolved from the supplied record" into an assertion rather than an inference.
2. **Golden-file tests** asserting the exact serialized JSON for a representative scenario, protecting the report contract against accidental change. At least one golden file is produced under a `PreferredCollation` and one without, so that the `collation` section is pinned on both paths; both assert that its four numeric members are JSON numbers rather than quoted strings, and a binary-collation case pins `"comparisonStyle": 0` as written rather than omitted.
3. **Integration tests of `MsSqlMetadataProvider`** against a real MS SQL Server instance (LocalDB or a container), created from a pair of setup scripts that materialize the eight use cases. `GetDatabaseOptionsAsync` gets its own case: the returned object is an `MsSqlDatabaseOptions`, its `Collation` matches `DATABASEPROPERTYEX(DB_NAME(), 'Collation')`, and its `SupportsLedgerTables` matches whether the instance is SQL Server 2022 or later — or Azure SQL Database, which reports `true` at major version `12` and is the case a version check would get wrong. Two argument cases go with it: a metadata call handed a plain `DatabaseOptions` throws `ArgumentException`, and one handed `null` throws `ArgumentNullException`, both before a connection is opened.
4. **Conflict-resolution tests** against a fake `IMetadataProvider`, which is where this logic is cheapest to exercise — the fake reports whatever names the test asks for, with no case-sensitive server to stand up. Three groups:

   - **Every row of the interpretation table of [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] gets its own test**, as enumerated in [[#7. Acceptance scenarios|§7]], and each is written through the factory method that produces it: `Ignore(table, From.Left)`, `Ignore(table, From.Right)`, `Ignore(table)`, `Map(leftTable2, rightTable2)` for a pairing that differs only in case, `Map(leftOrders, rightOrders)` for two genuinely different names, and `Map(leftField, rightField)` for a field pairing (`ModifiedOn` ↔ `Modified` in `dbo.Tasks`). Each is asserted twice — once on the report the entry produces, and once on the report the same fixture produces *without* the entry, so that the test fails if the entry is quietly ignored. Each is also run in both directions, left entry and right entry, since no rule in this section is meant to be asymmetric. Alongside them: a field pairing inside a table that is itself paired, the rule that a forced pair consumes both objects, and the two reachable shapes that raise `InvalidConflictResolutionException` — two entries claiming one object, and a two-sided entry matching on one side only.
   - **The factory methods themselves**, tested directly against a bare `HashSet<ConflictResolution>` with no comparison run at all: each method adds the entries the table of [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] says it does, with the `Left`, `Right` and `Kind` the row calls for; `Ignore(table)` adds two and equals the two single-sided calls; every method returns its receiver so calls chain; a repeat of one statement is absorbed by the set; two identities naming one table but built under different quoting policies both land in the set and are caught later by the same-object check; and the argument failures of [[#8. Error handling, cancellation, safety|§8]] — `null` receiver, `null` identity, an out-of-range `From`, and a read-only `ICollection<ConflictResolution>` such as an array — each throw before anything is added. A separate build-level assertion pins that the constructor is unreachable from outside the assembly and that no `with` expression can rewrite a side ([[#7. Acceptance scenarios|§7]]).
   - **Ambiguity.** A table collision raising `AmbiguousNameException` with one `CandidateInfo` of `Kind = Table` carrying both spellings on the colliding side and the matching name on the other; the same fixture passing once the pairing is supplied, with the report rendering the left spelling, and passing again when the colliding names are excluded instead; a field collision surfacing only after the tables are unambiguous; the ordinal match, so that an entry built with a different `QuotesUsage` than the run's — or, for a field, with a `FieldIdentity` built under a different `QuoteInfo` — still resolves its class, while one differing in `Name` by so much as case does not; the independence of that match from the collation, asserted by running one fixture twice under two `PreferredCollation` values and showing the same entries applied to the same objects both times, and by an entry naming `dbo.orders` excluding nothing on a case-insensitive run whose comparer would have called that name equal to `dbo.Orders`; the wrong-case entry being a silent no-op on either side, asserted as the specified outcome — no exception, no report entry, the object compared as though the entry were absent; `Kind` and the owner both participating in the match, asserted by a fixture where the same field spelling collides in two tables and one entry resolves only its own; an entry that names nothing being ignored; a collision confined to an excluded table raising nothing; a foreign-key collision, which no entry can resolve, cleared only by excluding the table that holds the keys; and no fixture raising `AmbiguousNameException` while `PreferredCollation` is unset. Every row of the candidate table of [[#6.2.1 Conflict analysis|§6.2.1]] gets a case, run in both directions.
   - **The lost-table guard**, covering [[#6.2 Building the table sets|§6.2]] step 5: two table names of one database folding into one raises `ObjectLostException` carrying both names, the side and the collation; the same fixture raises nothing when both names are excluded, one `Ignore` per spelling; a fold on the right side is caught as readily as one on the left; and a database whose names do not fold raises nothing, so the guard costs a passing run no behaviour.

   One further test asserts the two-stage contract directly: a fixture colliding in both tables and fields raises twice — tables first, fields only on the run that resolves them.
5. **Cancellation tests** verifying that a token cancelled mid-comparison propagates `OperationCanceledException`.
6. **Comparer-resolution tests** — `MsSqlNameComparerResolver` over a table of `CollationInfo` values covering case-sensitive, case-insensitive, accent-insensitive, width-insensitive and binary collations, asserting for each which pairs of identifiers the returned comparer treats as equal; an unknown `Lcid` and an unknown `ComparisonStyle` bit raising `NotSupportedException`; two databases whose `CollationInfo` differs — by name and by `Version` alone — raising `CollationMismatchException` before any table metadata is read while `PreferredCollation` is `null`, and those same two fixtures raising nothing and comparing cleanly once a `PreferredCollation` equal to one of the two sides is supplied; a `PreferredCollation` matching neither side — by name, and by `Version` alone — raising `InvalidCollationException` before any table metadata is read, and doing so in preference to `NotSupportedException` when its `Lcid` is also unknown; and a case-sensitive fixture holding two tables that differ only in case. Exclusion matching is **not** among these: it is ordinal, does not depend on the resolved comparer, and is covered with the other conflict-resolution rules in item 4.
7. **`MsSqlMetadataQueryBuilder` tests** — plain unit tests over returned strings, needing no database at all. This is what makes the ledger branch testable in CI: the only alternative to asserting on the text is standing up two differently-versioned servers and watching for a parse error. For each of the four methods assert: with `SupportsLedgerTables = true` the text contains both `ledger_type` and `is_dropped_ledger_table`; with `false` it contains **neither identifier anywhere** — a substring assertion over the whole statement, not an assertion about predicate shape, since the point is that the column is never *named* ([[#3.4.3 `MsSqlMetadataQueryBuilder`|§3.4.3]]); `temporal_type <> 1` is present in both; the projection is identical between the two variants; the same options yield the same string across calls; and `null` throws `ArgumentNullException`.
8. **Ownership-chain and rendering tests** — `INestedObject` over each implementing type: a `SchemaIdentity` reporting a `null` `Owner`, a `TableIdentity` reporting its schema and a two-part `Name`, and `FieldIdentity`, `PrimaryKeyIdentity` and `ForeignKeyIdentity` each reporting its table and its own unqualified `Name`; walking `Owner` from a field up to its schema in two steps. Alongside them, `IQuotedIdentifier` over the same types: each of the three `QuotesUsage` values against a name that needs quoting and one that does not, asserted on `Name`, `QuotedName` and `PreferredName`; a `TableIdentity` rendering both parts, with the schema quoted and the table bare and the other way round; an identifier built from a quoted string reporting `IsQuoted` and stripping the quotes out of `Name`; and every identity below a schema reporting the `QuoteInfo` and `QuotesUsage` of its owner rather than one of its own.
9. **`TableIdentity.Parse` / `TryParse` tests** covering each branch of the grammar: plain two-part names, bracket-quoted parts, a dot inside brackets, an escaped `]]`, a missing dot, an empty part, unbalanced brackets, an ambiguous unquoted name with two dots, and `null` — asserting that `Parse` throws where `TryParse` returns `false`.
10. **Credential-masking tests** covering SQL authentication, integrated security, a `PWD` alias, and a malformed connection string — asserting in each case that the real password appears neither in the result members nor in the serialized JSON, and that a supplied password is reported as `***` while integrated security gains no password keyword.
11. **Shadow-table filtering tests** for the exclusion of [[#1.3 Out of scope (v1)|§1.3]], run as integration tests against a real instance, since the filter lives in the provider's SQL and a fake `IMetadataProvider` cannot exercise it. The fixtures must cover:
   - **The auto-named case, which is the reason the filter exists.** Two databases each carrying a system-versioned `dbo.Orders` declared with `WITH (SYSTEM_VERSIONING = ON)` — no `HISTORY_TABLE` clause, so each server mints its own `MSSQL_TemporalHistoryFor_<object_id>`. The two names must differ (assert this in the fixture itself, otherwise the test is vacuous), and the comparison must still yield an **empty `difference.structure`** with every section present. Without the filter this case produces two spurious `tables.ignored` entries and two spurious `missings.*.tables` entries, so it is the regression this test guards.
   - **The explicitly named case.** `WITH (SYSTEM_VERSIONING = ON (HISTORY_TABLE = dbo.OrdersHistory))` on both sides: `dbo.OrdersHistory` is absent from `tables.detected` even though its name is identical on the two sides and it would otherwise have compared cleanly. The filter keys on `temporal_type`, not on the name.
   - **The period columns are still compared.** `dbo.Orders` system-versioned on the left and a plain table with the same business columns on the right: `ValidFrom` and `ValidTo` appear in `missings.right.fields`. This is what makes the exclusion lossless, and it must hold whether or not the columns are declared `HIDDEN` — `sys.columns` reports them either way.
   - **The filter releases the table when versioning ends.** After `ALTER TABLE dbo.Orders SET (SYSTEM_VERSIONING = OFF)`, the former history table is an ordinary user table and must reappear in `tables.detected` and be compared normally.
   - **Ledger, on SQL Server 2022 and later.** An updatable ledger table's history table and the remains of a dropped ledger table are both absent from `tables.detected`; the four generated `ledger_*` columns of the ledger table itself are compared as ordinary fields. These cases are skipped — not failed — when the instance under test predates SQL Server 2022, gated on the same `COL_LENGTH` probe the provider uses, so the suite stays green on a 2016–2019 instance.
   - **The capability read itself.** Against a pre-2022 instance, `SupportsLedgerTables` is `false`, all four metadata statements execute without a parse error, and the temporal cases above still pass; against a 2022+ instance it is `true` and the ledger predicates are in force. Assert on both instances that the run still costs exactly five queries per database ([[#9. Non-functional requirements|§9]]), so that reading the capability up front has not added a round trip of its own.
   - **Mixed versions across the two sides.** A pre-2022 database on one side and a 2022+ database on the other, structurally identical, must compare cleanly: the run does not fail, each side receives the query variant its own `SupportsLedgerTables` calls for, and `difference.structure` is empty. This is the case that catches a shared options object being passed to both sides. Skipped when the suite has only one server version available.

---

## 11. Assumptions and open questions

The points below are not fully determined by the source requirements. The stated resolution is what the implementation will follow unless the customer decides otherwise.

1. **Tie-breaking in step 3 of [[#6.5 Foreign key comparison|§6.5]].** Pairing the remainder by the largest field-set intersection is prescribed by the specification, but the specification does not say how to break a tie when several candidates share the same intersection size. The implementation will pair them in ordinal key-name order, purely to keep the report deterministic. Please confirm this rule.
2. **Sort order is approximated, equality is not.** `CollationInfo` carries enough to reproduce *which identifiers are equal* under any collation, and the resolver does so. It does not reproduce SQL Server's exact *ordering*, and for the legacy `SQL_*` collations no .NET comparer can. The report therefore orders ordinally ([[#4. Result model|§4]]), which is deterministic but not the server's order. Please confirm that report ordering need not match `ORDER BY` on the server.
3. **The one-sided class is excused by one rule and caught by the other.** The candidate table of [[#6.2.1 Conflict analysis|§6.2.1]] says that `dbo.Table1` and `dbo.table1` on one side with nothing on the other are "ignored by the system automatically because we have nothing to compare". The `ObjectLostException` guard says that two names of one database folding into one stops the run unless both were excluded. Both rules see exactly these names, and they disagree: taken together, the second wins and the case ends in `ObjectLostException` rather than in automatic ignoring, which is how [[#6.2 Building the table sets|§6.2]] step 5 is currently written. The recommendation is to keep the guard as the deciding rule — it is the stronger guarantee, and a caller who does want the tables ignored says so with one `ConflictResolution` entry per name. The alternative is to exempt a one-sided class from the guard and give each of its names its own entry in `tables.detected`, `tables.ignored.<side>` and `missings.<opposite>.tables`, which honours the table literally but makes `detected` something other than a comparer-based union. Please confirm which.
4. **The guard covers tables only.** [[#6.2 Building the table sets|§6.2]] step 5 is specified for table names, as requested. Two fields of one table, or two keys of one table, can fold in exactly the same way, and nothing currently stops that — the member stage of [[#6.2.1 Conflict analysis|§6.2.1]] sees them only when the opposite side is non-empty. The implementation will leave fields and keys unguarded, matching the requirement as written. Please confirm, or approve extending the same check to the member stage.

**Settled since an earlier draft.**

- *How a `ConflictResolution` entry finds its object.* The comparison is `StringComparer.Ordinal` and nothing else, and an entry whose spelling does not match the database's — `dbo.orders` against a `dbo.Orders` the database actually holds — applies to nothing, on every collation, without an error and without a report entry of its own. That is confirmed expected behaviour and is specified in [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]], not a trade still to be decided.
- *What the factory methods extend.* The receiver is `ICollection<ConflictResolution>`, not `IEnumerable<ConflictResolution>`: the methods add, and only `ICollection<>` declares an addition, so a sequence that cannot be added to is refused by the compiler instead of at run time. `TableStructureComparisonOptions.ConflictResolution` is a `HashSet<ConflictResolution>` and binds to it directly, so the narrower receiver costs a caller nothing ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]).
- *Whether a field or a key may be excluded.* It was open here whether a member excluded by a one-sided entry — invisible in the report, since `tables.unchecked` has no counterpart below the table level — was wanted, should be rejected, or should get a report section of its own. It is answered by the construction rules of [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]: member exclusion is **rejected outright**, and rejected at the strongest point available — a one-sided entry of any kind other than `Table` has no factory method and so cannot be built. No new report section is needed, because there is nothing left for it to record: every exclusion a caller can express names a table, and every table excluded appears in `tables.unchecked`. A caller who wants a member left out of the report excludes the table that holds it. The same rule settles what the library does about a `PrimaryKey` or `ForeignKey` conflict: nothing directly — no entry of either kind exists — and a key collision is answered by excluding its table ([[#6.2.1 Conflict analysis|§6.2.1]]).

---

## 12. Definition of done

- `Diff.Structure.dll` targets .NET 10 and builds with no warnings under `TreatWarningsAsErrors`.
- The public API matches [[#3. Public API|§3]] exactly, including the required names `TableStructureComparer`, `CompareTableStructureAsync`, `TableStructureComparisonOptions`, `TableStructureComparisonResult`, `IMetadataProvider`, `GetDatabaseOptionsAsync`, `DatabaseOptions`, `MsSqlDatabaseOptions`, `MsSqlMetadataProvider`, `MsSqlMetadataQueryBuilder`, `ConflictResolutionExtensions` with its `Ignore` and `Map` overloads, `From`, and the `ToJson` extension declared by `ObjectExtensions` in `ObjectExtensions.cs`. `IMetadataProvider` declares exactly the five methods of [[#3.3.1 `IMetadataProvider`|§3.3.1]] and no others; a collation a database reported is reachable only as `DatabaseOptions.Collation`, and no member of any type returns a `CollationInfo` straight from a database. `TableStructureComparisonOptions` carries `PreferredCollation`, an optional `CollationInfo?` defaulting to `null`; being an input, it does not breach that rule, and `TableStructureComparisonResult.Collation` reports the record the run resolved rather than reading one from a database.
- `TableStructureComparer` contains no ADO.NET references and no SQL of any kind. It never names `MsSqlMetadataQueryBuilder`, never names `MsSqlDatabaseOptions`, and never downcasts a `DatabaseOptions`.
- `MsSqlMetadataProvider` contains no dynamic SQL: it composes no statement text and calls no `sys.sp_executesql`. Its only SQL literal is the database-options statement of [[#3.4.1 `MsSqlMetadataProvider`|§3.4.1]]; the other four statements come from `MsSqlMetadataQueryBuilder`.
- `MsSqlMetadataQueryBuilder` names neither `ledger_type` nor `is_dropped_ledger_table` in any string it returns when `SupportsLedgerTables` is `false`, and names both when it is `true`, verified by unit tests that require no database.
- Identifier comparison follows the comparer `INameComparerResolver` builds from a single `CollationInfo` — the one both databases report, or the one the caller set as `PreferredCollation` — honouring its case, accent, kana and width rules.
- With `PreferredCollation` unset, two sides whose collations differ fail the call with `CollationMismatchException` instead of producing a report; a run that leaves the property unset behaves in every respect as the previous version of this brief specified, since the override adds a path rather than altering the existing one.
- With `PreferredCollation` set, the two databases' collations are never compared with each other and `CollationMismatchException` is unreachable; the supplied value is instead required to equal one of them, failing the call with `InvalidCollationException` when it equals neither. The whole run — table matching, fields, keys — then uses the single comparer resolved from that record; the matching of `ConflictResolution` entries is the one thing it does not touch, being ordinal by specification ([[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]]).
- No table set is built until the guard of [[#6.2 Building the table sets|§6.2]] step 5 has passed on both sides: a comparer that folds two of one database's table names together fails the call with `ObjectLostException` naming them and the run's collation, unless every folded name was excluded or paired by the caller.
- No comparison begins until the conflict analysis of [[#6.2.1 Conflict analysis|§6.2.1]] has passed for the kind of name it is about to use: a collision the caller has neither paired nor excluded through `ConflictResolution` fails the call with `AmbiguousNameException` carrying every candidate found, and no name-keyed dictionary is built over an unchecked class. A run that leaves `PreferredCollation` unset can raise neither.
- The public API carries `ObjectKind`, `CandidateInfo`, `ConflictResolution`, `From`, `ConflictResolutionExtensions`, `AmbiguousNameException` and `InvalidConflictResolutionException` as declared in [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]], and `TableStructureComparisonOptions.ConflictResolution` is an `init`-only `HashSet<ConflictResolution>` that defaults to an empty set, is never reassigned after the options object is built, has contents that stay mutable so the factory methods can fill them, and is validated and snapshotted at the start of a call, and the **only** way a caller excludes an object or pairs two objects — there is no separate list of excluded table names.
- `ConflictResolution` has **no public constructor**: the constructor is `internal`, the record is `sealed`, and `Left`, `Right` and `Kind` are get-only, so no `new`, no object initializer and no `with` expression reaches an entry from outside the assembly. The four extension methods of `ConflictResolutionExtensions`, each extending `ICollection<ConflictResolution>` — `Ignore(TableIdentity, From)`, `Ignore(TableIdentity)`, `Map(TableIdentity, TableIdentity)` and `Map(FieldIdentity, FieldIdentity)` — are the whole construction surface, each adds to the collection it is called on and returns that same instance, and a build-level test asserts that the surface has not widened.
- Only what those four methods can say is sayable. A one-sided entry is always of `Kind = Table`; a two-sided entry is of `Kind = Table` or `Kind = Field`; `Kind = PrimaryKey` and `Kind = ForeignKey` exist in `ObjectKind` for `CandidateInfo` alone and no entry carries them. A field, a primary key or a foreign key is therefore never excluded from a comparison, and a key collision is answered by excluding the table that holds it ([[#6.2.1 Conflict analysis|§6.2.1]]).
- `ConflictResolution.Left` and `ConflictResolution.Right` are both `INestedObject?`. Every shape of the interpretation table in [[#3.1.1 Conflicts: `CandidateInfo` and `ConflictResolution`|§3.1.1]] behaves as tabulated — a one-sided entry excludes a table, a two-sided entry pairs and consumes both objects, whether the two names differ by case or are two different names after a rename, and whether they name tables or fields — and each has an automated test of its own, written through the factory method its row names. An entry whose spelling does not match the database's, case included, applies to nothing: it raises no exception, appears nowhere in the report, and leaves its object compared as though the entry were absent.
- `INestedObject` is implemented by `SchemaIdentity`, `TableIdentity`, `FieldIdentity`, `PrimaryKeyIdentity` and `ForeignKeyIdentity`, and `Owner` walks from any of them to a schema in at most two steps. `TableIdentity.Schema` is a `SchemaIdentity`, and a `ConflictResolution` side is matched by `Kind`, `Name` and the owner chain, with every name compared using `StringComparer.Ordinal` and nothing else — never with the run's comparer, never case-insensitively, and never by the carried record's own equality.
- `IQuotedIdentifier` is implemented by those same five types, and `QuotedName` and `PreferredName` are computed only by `QuotedIdentifier.GetQuotedName` and `QuotedIdentifier.GetPreferredName` — no type re-implements the quoting rules. `QuotesUsage` and `QuoteInfo` are accepted by exactly two entry points, the `SchemaIdentity` constructor and `TableIdentity.Parse` / `TryParse`; every other identity takes them from its owner, and `FieldMetadata`, `PrimaryKeyMetadata`, `ForeignKeyMetadata` and `ForeignKeyColumn` carry identities rather than strings.
- `TableStructureComparisonResult` carries a non-null `Collation`, and every report serializes a complete `collation` section — five members, the four numeric ones as JSON numbers — naming the collation the run was performed under, on both paths and including when a member's value is `0`.
- Neither `TableStructureComparisonResult` nor its serialized JSON contains a plaintext password in any supported authentication mode; a supplied password is reported as `***`.
- All eight use cases of [[#7. Acceptance scenarios|§7]], plus the listed regression cases, pass as automated tests.
- The serialized output matches the schema in [[#5. Report structure|§5]], with every section always present.
- Public types and members carry XML documentation comments, and the assembly ships an XML documentation file.
- A `README.md` shows a minimal end-to-end usage sample:

```csharp
var leftConnectionString = "...";
var rightConnectionString = "...";

var provider = new MsSqlMetadataProvider();
var comparer = new TableStructureComparer(provider, new MsSqlNameComparerResolver());

var quoteInfo = QuoteInfo.MsSql;
var quotesUsage = QuotesUsage.DoNotUseIfPossible;

// The caller's decisions about objects the library cannot pair on its own (§3.1.1).
var migrations = TableIdentity.Parse("dbo.__EFMigrationsHistory", quotesUsage, quoteInfo);
var leftOrders = TableIdentity.Parse("dbo.Orders", quotesUsage, quoteInfo);
var rightOrders = TableIdentity.Parse("dbo.CustomerOrders", quotesUsage, quoteInfo);
// The table is spelled the same on both sides here, but each side of a field entry
// still hangs from its own database's table identity.
var leftTasks = TableIdentity.Parse("dbo.Tasks", quotesUsage, quoteInfo);
var rightTasks = TableIdentity.Parse("dbo.Tasks", quotesUsage, quoteInfo);

var options = new TableStructureComparisonOptions
{
    ConnectionStrings = (leftConnectionString, rightConnectionString),
    QuoteInfo = quoteInfo,
    QuotesUsage = quotesUsage,

    // Optional. Left out, the two databases must share a collation and that collation
    // decides identifier equality. Set, the two no longer have to agree — but the value
    // must be the collation of one of them, or the call throws InvalidCollationException:
    // PreferredCollation = new CollationInfo("Latin1_General_CI_AS", 1033, 1252, 196609, 0),
};

// ConflictResolution has no public constructor: entries are produced by Ignore and Map,
// which add to the collection they are called on and return it, so calls chain (§3.1.1).
options.ConflictResolution

    // Exclude dbo.__EFMigrationsHistory from both databases — two entries, one per side.
    // Ignore(migrations, From.Left) would exclude it from the left database only.
    .Ignore(migrations)

    // The table was renamed on the right: compare the two as one table.
    .Map(leftOrders, rightOrders)

    // A field of dbo.Tasks was renamed on the right: compare the two as one field.
    .Map(new FieldIdentity(leftTasks, "ModifiedOn"), new FieldIdentity(rightTasks, "Modified"));

// Answering an AmbiguousNameException goes through the same two methods — its candidates
// carry the identities to pass to them (§3.1.1, §6.2.1). A candidate of Kind = PrimaryKey
// or ForeignKey has no Map of its own: Ignore the table that holds the keys.

TableStructureComparisonResult result =
    await comparer.CompareTableStructureAsync(options, cancellationToken);

// The collation the comparison was performed under, whatever its source.
CollationInfo collation = result.Collation;

string json = result.ToJson();
```
