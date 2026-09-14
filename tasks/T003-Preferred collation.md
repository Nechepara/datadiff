At the moment we have a restriction - `when database collations are different, the program throws the exception and does not compare the DB structure`. We are going to change this requirement and add more flexibility. Please analyze the following [[# New Requirements]] and adjust the related sections in the `brief.md`.
# New Requirements
Class `TableStructureComparisonOptions` has new member:
```csharp
public CollationInfo? PreferredCollation {get; init;} = null;
```
The user can specify own database collation info in it. 
**`PreferredCollation` is not specified (null)**. The system should go by way that we have already had - define and compare database collations, if the collations are different -> throw exception, otherwise -> define name comparer and provide structure comparison.
**`PreferredCollation` is defined by user (not null)**. The system should not compare database collations. Never mind, whether the database collations are different or they are equal to each other. Instead, the system should use the value from `PreferredCollation` property to define the name comparer and provide the DB structure comparison.
# Answers on questions
Question: **A preferred collation leaves no trace in the report.** 
Answer: Please add to the JSON report the new section:
```json
{
	"collation": {
		"name": "SQL_Latin1_General_CP1_CI_AS",
		"lcid": 1033, // COLLATIONPROPERTY(Name, 'LCID')            -> CultureInfo
		"codePage": 1252, // COLLATIONPROPERTY(Name, 'CodePage')        -> non-Unicode encoding
		"comparisonStyle": 196609, // COLLATIONPROPERTY(Name, 'ComparisonStyle') -> 0 for a binary collation
		"version": 0 // COLLATIONPROPERTY(Name, 'Version')         -> 0, 90, 100, 140 ...
	}
}
```
Also, the collation info should be present in the `TableStructureComparisonResult` even if `PreferredCollation` is not set in the comparison options. In this case, it is the real collation read from both databases.

Question: **A preferred collation is not validated against the databases.**
Answer. Good catch. Please validate the value specified in the `PreferredCollation`. The specified value should be equal to the collation info read from the one of the databases at least. If this rule is violated, the system should throw `InvalidCollationException`.

Question: **Names that collide under a chosen collation.** Choosing the looser of the two databases' collations can make two distinct objects of the stricter database equal to each other — `dbo.Table1` and `dbo.table1` on a case-sensitive side, compared under the case-insensitive side's collation (§6.1). The case cannot arise on the default path and has no counterpart in the source requirements. The implementation will place every such name in `tables.ignored.<its side>`, list it once in `tables.detected`, and never report it as missing or inconsistent, on the grounds that `tables.ignored` is already the report's section for objects the system was unable to compare and that crashing on a duplicate key is not an option. Two alternatives exist — failing the run with a dedicated exception, or pairing one of the colliding names arbitrarily and ignoring the rest — and the second is unacceptable because it is not deterministic. Please confirm the chosen behaviour, and confirm that the same rule should apply to two fields or two keys of one table that collide the same way.
Answer: Before we start the comparison procedure, we should provide an additional analysis. During analysis, the system should detect all such cases with object naming and collect them into the collection of `CandidateInfo`:
```csharp
//please add new enum
public enum ObjectKind { Table, Field, PrimaryKey, ForeignKey }

//please add new class
public class CandidateInfo
{
	public ObjectKind Kind {get; init;}
	public HashSet<IOwnedIdentifier> Left {get; init;} //Names that collide under a chosen collation from the `left` database (for example 'dbo.Orders', 'dbo.orders')
	public HashSet<IOwnedIdentifier> Right {get; init;} //Names that collide under a chosen collation taken from the `right` database  (for example 'dbo.orders', 'dbo.Orders')
}
```
Please take into account the following changes:
```csharp
//please add new interface
public interface IOwnedIdentifier
{
	public string Name {get;}
	public IOwnedIdentifier? Owner {get;}
}
//please add new class
public class SchemaIdentity: NameIdentity, IOwnedIdentifier
{
	public IOwnedIdentifier? Owner {get;} = null;
}

//please adjust TableIdentity.cs with the following changes
public sealed record TableIdentity: IOwnedIdentifier
{
	//the constructor body is changed
    public TableIdentity(
        string schema,
        string table,
        QuotesUsage quotesUsage,
        QuoteInfo quoteInfo)
    {
        this.Schema = new SchemaIdentity(schema, quotesUsage, quoteInfo);
        this.Table = new NameIdentity(table, quotesUsage, quoteInfo);
    }

	//the property `Schema` has new signature
    public SchemaIdentity Schema { get; } 
    
    //implementation of `IOwnedIdentifier` interface
    public IOwnedIdentifier Owner => this.Schema;

	//other properties and methods are not changed
}

//please adjust FieldMetadata with the following changes
public sealed record FieldMetadata: IOwnedIdentifier
{
	//implementation of `IOwnedIdentifier` interface
    public IOwnedIdentifier Owner => this.Table;
    
    //other properties and methods are not changed
}

//please adjust PrimaryKeyMetadata with the following changes
public sealed record PrimaryKeyMetadata: IOwnedIdentifier
{
	//implementation of `IOwnedIdentifier` interface
    public IOwnedIdentifier Owner => this.Table;
    
    //other properties and methods are not changed
}

//please adjust ForeignKeyMetadata with the following changes
public sealed record ForeignKeyMetadata: IOwnedIdentifier
{
	//implementation of `IOwnedIdentifier` interface
    public IOwnedIdentifier Owner => this.Table;
    
    //other properties and methods are not changed
}
```
Additionally, please add to `TableStructureComparisonOptions` the following members:
```csharp
public HashSet<NameResolution> NameResolution {get;} = new();
```
The `NameResolution` has the following declaration:
```csharp
public record NameResolution(IOwnedIdentifier Left, IOwnedIdentifier Right, ObjectKind Kind);
```
The found candidate (with ambiguous name) should be added to the the collection of `CandidateInfo` only if the ambiguous name (from left or right database) has not been resolved yet in the  `TableStructureComparisonOptions.NameResolution`. If the ambiguous name has already been resolved (specified in `TableStructureComparisonOptions.NameResolution`), the system should use the resolved name from other side while comparing the database structure.
When the analysis is completed, in other words, all names for tables(fields, primary keys, foreign keys) have been analyzed:
 - if the candidate's collection is not empty, the system should throw `AmbiguousNameException` with the collection of candidates .
 - if the candidate's collection is empty, it means the system knows all the necessary name pairs and can start the database structure comparison.

Question:  **A collision with nothing to pair against.** A class may hold two names on one side and none on the other — a case-sensitive left database holding `dbo.Table1` and `dbo.table1` where the right database has neither. Both are missing from the right, so no pairing decision is actually needed, yet `tables.detected` would fold them into one entry and lose one, and `NameResolution` cannot express the answer because there is no right-hand name to name. The implementation will treat the class as a candidate and fail the run, leaving `UserExcludedTables` as the only way past it — safe, since nothing is silently dropped, but a dead end for a caller who wants both tables reported as missing. Please confirm, or say whether such a class should instead skip the analysis and have all its names listed individually in `detected` and `missings`.

Answer: **First**, the case for which we had at least one table in the one database and no tables detected in the other database - it is not a candidate with ambiguous name, it is just a case to ignore such table(s) by the system and do not compare them because we have nothing to compare on the other side. The right candidate  should have at least two objects on the one side (for example dbo.Table1 and dbo.table1) and at least one object on the other side (for example dbo.table1). Please take into account the following table:

| Names found in the one DB          | Names found in the other DB        | Action                                                                                                                                    |
| ---------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| dbo.Table1                         | nothing found                      | the found table should be ignored by the system automatically because we have nothing to compare                                          |
| dbo.Table1, dbo.table1             | nothing found                      | the found tables should be ignored by the system automatically because we have nothing to compare                                         |
| dbo.Table1, dbo.table1, dbo.TABLE1 | nothing found                      | the found tables should be ignored by the system automatically because we have nothing to compare                                         |
| dbo.Table1, dbo.table1             | dbo.table1                         | The candidate should be created, if the appropriate mapping have not been defined in the `TableStructureComparisonOptions.NameResolution` |
| dbo.Table1, dbo.table1             | dbo.table1, dbo.Table1             | The candidate should be created, if the appropriate mapping have not been defined in the `TableStructureComparisonOptions.NameResolution` |
| dbo.Table1, dbo.table1, dbo.TABLE1 | dbo.table1, dbo.Table1             | The candidate should be created, if the appropriate mapping have not been defined in the `TableStructureComparisonOptions.NameResolution` |
| dbo.Table1, dbo.table1             | dbo.table1, dbo.Table1, dbo.TABLE1 | The candidate should be created, if the appropriate mapping have not been defined in the `TableStructureComparisonOptions.NameResolution` |
**Second**. Before the system will union two table name lists (left and right) into `tables.detected` using the resolved comparer, we should provide the additional validation:
 - create the list create hash set based on the table names taken from the left database minus `UserExcludedTables` using the resolved comparer. If the count of items in the created hash set is less then count in the the original list, the system should detect the lost table names, and if the lost table names are not in `UserExcludedTables` throw the `ObjectLostException` with information about these tables and the collation used for resolving of the comparer.
 - create hash set based on the table names taken from the right database using the resolved comparer. If the count of items in the created hash set is less then count in the the original list, the system should detect the lost table names, and if the lost table names are not in `UserExcludedTables` throw the `ObjectLostException` with information about these tables and the collation used for resolving of the comparer.