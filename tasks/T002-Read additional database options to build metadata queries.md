Please analyze [[#New Requirements]] and adjust the related sections in the `brief.md`
# New Requirements
At the moment we have method `GetDatabaseCollationAsync` in the `IMetadataProvider` which reads collation info from database. Please replace this method with the new one:
```csharp
public Task<DatabaseOptions> GetDatabaseOptionsAsync(
        string connectionString,
        CancellationToken cancellationToken = default);
```
The new method read all the necessary database options including collation info read by the old method. The declaration for `DatabaseOptions` should look like:
```csharp
public class DatabaseOptions
{
	public CollationInfo Collation {get; init;}
}
```
Please change the signature for the rest methods of the `IMetadataProvider`:
```csharp
	Task<IReadOnlyCollection<TableIdentity>> GetTablesAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyCollection<FieldMetadata>> GetFieldsAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyCollection<PrimaryKeyMetadata>> GetPrimaryKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyCollection<ForeignKeyMetadata>> GetForeignKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken = default);
```
Please remove any references to the old method `GetDatabaseCollationAsync`. The brief should not contain any references to the old method and the old method should be mentioned nowhere. Instead, the `brief.md` should use the new method `GetDatabaseOptionsAsync`.
The implementation of `IMetadataProvider` for MS SQL Server `MsSqlMetadataProvider` should return the additional database options in the following class:
```csharp
public sealed class MsSqlDatabaseOptions: DatabaseOptions
{
	public bool SupportsLedgerTables {get; init;}
}
```
The query to define the new option can be like:
```sql
SELECT CAST(IIF(COL_LENGTH(N'sys.tables', N'ledger_type') IS NULL, 0, 1) as BIT) AS SupportsLedgerTables
```
Additionally, `MsSqlMetadataProvider` become dependent from the new abstractions:
```csharp
public sealed class MsSqlMetadataProvider: IMetadataProvider
{	
	private readonly MsSqlMetadataQueryBuilder _queryBuilder;
	
	public MsSqlMetadataProvider()
	{
		_queryBuilder = new MsSqlMetadataQueryBuilder();
	}

    public async Task<IReadOnlyCollection<TableIdentity>> GetTablesAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        CancellationToken cancellationToken)
    {
	    ...
	    var query = _queryBuilder.BuildTableMetadataQuery(databaseOptions);
	    ...
    }

    public async Task<IReadOnlyCollection<FieldMetadata>> GetFieldsAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken)
    {
	    ...
	    var query = _queryBuilder.BuildFieldMetadataQuery(databaseOptions);
	    ...
    }

    public async Task<IReadOnlyCollection<PrimaryKeyMetadata>> GetPrimaryKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken)
    {
	    ...
	    var query = _queryBuilder.BuildPrimaryKeyMetadataQuery(databaseOptions);
	    ...
    }
    
    public async Task<IReadOnlyCollection<ForeignKeyMetadata>> GetForeignKeysAsync(
        string connectionString,
        DatabaseOptions databaseOptions,
        IReadOnlyCollection<TableIdentity> tables,
        CancellationToken cancellationToken)
    {
	    ...
	    var query = _queryBuilder.BuildForeignKeyMetadataQuery(databaseOptions);
	    ...
    }
}
```
The new class `MsSqlMetadataQueryBuilder` has the following declaration:
```csharp
public sealed class MsSqlMetadataQueryBuilder
{
	public string BuildTableMetadataQuery(MsSqlDatabaseOptions options);
	public string BuildFieldMetadataQuery(MsSqlDatabaseOptions options);
	public string BuildPrimaryKeyMetadataQuery(MsSqlDatabaseOptions options);
	public string BuildForeignKeyMetadataQuery(MsSqlDatabaseOptions options);
}
```
`MsSqlMetadataProvider` should use the `MsSqlMetadataQueryBuilder` functionality to define the queries that reads metadata (except query that reads database options).
`MsSqlMetadataQueryBuilder` should take into account `SupportsLedgerTables` option and build the appropriate queries. If database supports ledger tables, all the queries returned by `MsSqlMetadataQueryBuilder` should filter out the information related to the ledger tables. Otherwise, the queries should not contain information related to the ledger tables.