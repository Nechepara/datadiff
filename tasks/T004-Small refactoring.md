Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.

Please rename `IOwnedIdentifier` to `INestedObject`.

Please add the following interface and class to the brief:
```csharp
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
```
The specified interface replaces the existing class `NameIdentity`. So please replace all occurrences `NameIdentity` with `IQuotedIdentifier`.
Please adjust the following existing classes:
```csharp
//file SchemaIdentity.cs
public record SchemaIdentity : IQuotedIdentifier, INestedObject
{
    public SchemaIdentity(string name, QuotesUsage quotesUsage, QuoteInfo quoteInfo)
        : base(name, quotesUsage, quoteInfo)
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
        
    public INestedObject? Owner { get; } = this.Schema;

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
```

Then please add the following new classes:
```csharp
public record FieldIdentity: INestedObject, IQuotedIdentifier
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
	public string Name {get;}
	public INestedObject Owner => this.Table;
}

public record PrimaryKeyIdentity: INestedObject, IQuotedIdentifier
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
	public string Name {get;}
	public INestedObject Owner => this.Table;
}

public record ForeignKeyIdentity: INestedObject, IQuotedIdentifier
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
	public string Name {get;}
	public INestedObject Owner => this.Table;
}
```

Please make corrections or the following existing classes:
```csharp
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