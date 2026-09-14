Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.
We are going to resolve the question 5 from [[brief#11. Assumptions and open questions|§11]]. Please take into account the following requirements.
# New Requirements
The constructor of the `ConflictResolution` should be `internal` only. The user have no possibility to use it directly. The system allows to create the conflict resolutions using the following extension methods only:
```csharp
public enum From { Left = 1, Right = 2 }

public static class ConflictResolutionExtensions
{
	//adds a new instance of `CoflictResolution` to the `resolution` which provides ignoring of the specified table taken from the database specified in `from` parameter
	public static ICollection<ConflictResolution> Ignore(this ICollection<ConflictResolution> resolution, TableIdentity table, From @from);
	//adds a couple of instances of `CoflictResolution` to the `resolution` which provides ignoring of the specified table for both databases
	public static ICollection<ConflictResolution> Ignore(this ICollection<ConflictResolution> resolution, TableIdentity table)
	{
		resolution.Ignore(table, From.Left);
		resolution.Ignore(table, From.Right);
	}
	//adds a new instance of `CoflictResolution` to the `resolution` which provides mapping for the tables with different names but taken from the different databases
	public static ICollection<ConflictResolution> Map(this ICollection<ConflictResolution> resolution, TableIdentity left, TableIdentity right);
	//adds a new instance of `CoflictResolution` to the `resolution` which provides mapping for the fields of the same table but with different names. The tables taken from the different databases
	public static ICollection<ConflictResolution> Map(this ICollection<ConflictResolution> resolution, FieldIdentity left, FieldIdentity right);
}
```
Also please make the property`ConflictResolution` of the ``TableStructureComparisonOptions` immutable with `init` section in the property declaration.

Please take into account the following code samples.
# Samples
## Sample 1.1: Force table ignoring (excluding the table from comparison for the one of the databases)
Lets the table `dbo.Orders` exists in the `Left` database only. The following sample shows how to ignore the table`dbo.Orders` which exists in the `Left` database only.
```csharp
var options = new TableStructureComparisonOptions();
var dbo = new SchemaIdentity("dbo", QuotesUsage.DoNotUseIfPossible, QuoteInfo.MsSql);
var orders = new TableIdentity(dbo, "Orders");
options.ConflictResolution.Ignore(orders, From.Left); //use From.Right to exclude the specified table for the `Right` database
```
## Sample 1.2: Force table ignoring (excluding the table from comparison for both databases)
Lets the table `dbo.Orders` exists in the both databases, but should be excluded from comparison. The following sample shows how to exclude the table`dbo.Orders` which exists in both databases.
```csharp
var options = new TableStructureComparisonOptions();
var dbo = new SchemaIdentity("dbo", QuotesUsage.DoNotUseIfPossible, QuoteInfo.MsSql);
var orders = new TableIdentity(dbo, "Orders");
options.ConflictResolution.Ignore(orders);
```
## Sample 2: The table has different names in the databases but should be compared
If the same table has different names in the databases or the database collation is different in the specified databases, the user should have a possibility to resolve such collisions in the following way:
```csharp
//The following sample shows how to map the table `dbo.Orders` from the `Left` database to the table `dbo.CustomerOrders` from the `Right` database.
var options = new TableStructureComparisonOptions();
var dbo = new SchemaIdentity("dbo", QuotesUsage.DoNotUseIfPossible, QuoteInfo.MsSql);
var leftOrders = new TableIdentity(dbo, "Orders");
var rightOrders = new TableIdentity(dbo, "CustomerOrders");
options.ConflictResolution.Map(leftOrders, rightOrders);
```
## Sample 3: The field has been renamed
If the same field has different names in the databases, but should be compared, the user should have a possibility to compare the field metadata (with different field names) by resolving the collision in the following way:
```csharp
//the following sample shows how to map the field `ModifiedOn` from dbo.Orders (`Left` database) to the field `Modified` from dbo.Orders (`Right` database) 
var options = new TableStructureComparisonOptions();
var dbo = new SchemaIdentity("dbo", QuotesUsage.DoNotUseIfPossible, QuoteInfo.MsSql);
var orders = new TableIdentity(dbo, "Orders");
var leftField = new FieldIdentity(orders, "ModifiedOn");
var rightField = new FieldIdentity(orders, "Modified");
options.ConflictResolution.Map(leftField, rightField);
```

# Feedback
Please let me know if something related to the question 5 from [[brief#11. Assumptions and open questions|§11]]  left unclear. 