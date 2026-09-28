Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.
Please add the new Enum
```csharp
public enum ConflictResolutionAction
{
	Skip = 1, Map = 2
}
```
From this moment, the `ConflictResolution` has new read-only property `Id`. It is a GUID which should be automatically generated in the constructor. We also should add the new property `Action` of type `ConflictResolutionAction` (getter only). The value for `Action` should be calculated automatically using the following logic:
 - If at least one of the values from `Left`, `Right` is `null`, `Action` = `ConflictResolutionAction.Skip`
 - Otherwise `Action` = `ConflictResolutionAction.Map`
The set of the conflict resolutions should be present in the table structure comparison result JSON:
```json
{
	"conflictResolution": [
		{
			"id": "The generated GUID",
			"left": "The name of the object in the left database or `null`",
			"right": "The name of the object in the right database or `null`",
			"kind": "Two values are possible here: `table` or `field`",
			"action": "It is one of the values `skip` or `map`"
		}
	]
}
```
Each time when the system uses the specified conflict resolution to define a field name. This conflict resolution should be specified as object instead of the field name in the sections `differenece.structure.missings`, `difference.structure.inconsistencies` :
```json
{
	"field": { conflictResolution: "The GUID of the appropriate conflict resolution"},
	...
}
```
Each time when the system uses the specified conflict resolution to define a table name. This conflict resolution should be specified as object instead of the table name in the sections `differenece.structure.missings`, `difference.structure.inconsistencies` :
```json
{
	"table": { conflictResolution: "The GUID of the appropriate conflict resolution"},
	...
}
```
The same rule should be provided for the section `tables.ignored`:
```json
{
          "table": { conflictResolution: "The GUID of the appropriate conflict resolution"},
          "ignoredBy": ...,
          "reason": ...
}
```
In the previous requirements we ask to automatically ignore the table if it was not found in the other database. Starting from this moment, we should not ignore the table comparison in such cases. We will give user a possibility to define which tables to ignore always. The user will have to specify the appropriate record in the `ConflictResolution` to be able to ignore it. 
# Answers on the related questions
Question: **The partner of a one-sided exclusion is still skipped by the system.** The requirement that a table no longer be ignored automatically is read as applying to a table **found in one database only**, which is what it describes: such a table is now reported in `missings` and nowhere else. The other `system` case — a table whose partner the caller excluded on the other side ([§5.1](app://obsidian.md/index.html#5.1%20Rules%20that%20govern%20the%20report), _An exclusion takes the partner out with it_) — is kept, because it is not automatic in the sense the requirement objects to: it happens only as the consequence of a `Skip` entry the caller wrote, and dropping it would force the table into `missings` on a side that plainly holds it. Please confirm. The alternative is to compare that table against nothing as well — reporting it under `missings.<the excluding side>.tables`, which is false — or to require the caller to write `Ignore(table)` whenever both sides hold the table, which would make the one-sided `Ignore` fail the run in that case.
Answer: We should require the caller to write `Ignore(table)` whenever both sides hold the table, which would make the one-sided `Ignore` fail the run in that case. The `system` is not used anymore in the `ignoredBy`, so please remove it from the appropriate enum. Then please remove `ignoredBy` because it always set to `user` at the moment. 

Question: **`left` and `right` are written as an explicit `null`.** The requirement shows the empty side of a `Skip` element as `` The name of the object … or `null` `` , and this brief takes that literally: the property is present with the JSON value `null`, which is the one exception to the report's rule of never writing a null ([§4](app://obsidian.md/index.html#4.%20Result%20model)). The alternative — omitting the property, as everywhere else — keeps the rule uniform, at the price of elements whose shape varies. Please confirm which.
Answer: Fill free to ignore these properties in the JSON too if the value is `null`.

Question: **A table element whose two sides live in different schemas.** A `Map` may pair tables of two different schemas — `dbo.Orders` against `sales.Orders` — and a `table` element then has two owners where `owner` holds one. Until this is decided, `owner` carries the schema of the **left** side, in keeping with the report's rule that a paired object is rendered in the left database's spelling, and the right schema is still readable from `right`, which keeps its two-part form. Please confirm, or choose between the alternatives: forbid such a pairing, or write `owner` per side. A related point: now that the schema has a property of its own, `left` and `right` of a table element could drop the schema part (`"Orders"` rather than `"dbo.Orders"`); the brief keeps them two-part, matching every other table name in the report. Please confirm that too.
Answer: please write `owner` per side. The schema should stay in the table name as is.
# Improvements
Please add the new property `owner` to the elements of the `conflictResolution` section. It names the objects that own the objects specified in `left` and `right`, so that an element can be understood on its own, even when nothing in the report refers to it.
The `owner` is written per side, for both kinds of elements:
```json
{ "owner": { "left": "The owner of the object specified in `left`", "right": "The owner of the object specified in `right`" } }
```
 - For an element of `kind` = `table`, each side of `owner` is the name of the schema the table on that side belongs to, e.g. `dbo`.
 - For an element of `kind` = `field`, each side of `owner` is the name of the table the field on that side belongs to, e.g. `dbo.Orders`. It is always the table name taken from that side's database, even when the two tables are mapped by another conflict resolution (for example, `dbo.Orders` ↔ `dbo.CustomerOrders`): `owner.left` is `dbo.Orders` and `owner.right` is `dbo.CustomerOrders`.
 - A side of `owner` is present exactly when the same side is present in the element: an element of `action` = `skip` names one side only, so its `owner` has one side only too.
 - The names specified in `left` and `right` stay as they are: a table name keeps its schema (`dbo.Orders`), a field name is written without its table (`ModifiedOn`).

```json
{
	"conflictResolution": [
		{
			"id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
			"owner": { "left": "dbo", "right": "dbo" },
			"left": "dbo.Orders",
			"right": "dbo.CustomerOrders",
			"kind": "table",
			"action": "map"
		},
		{
			"id": "5b6e8f1a-2c3d-4e5f-8a9b-0c1d2e3f4a5b",
			"owner": { "left": "dbo", "right": "sales" },
			"left": "dbo.Invoices",
			"right": "sales.Invoices",
			"kind": "table",
			"action": "map"
		},
		{
			"id": "9a8b7c6d-5e4f-4a3b-9c2d-1e0f9a8b7c6d",
			"owner": { "left": "dbo" },
			"left": "dbo.__EFMigrationsHistory",
			"kind": "table",
			"action": "skip"
		},
		{
			"id": "3f2504e0-4f89-41d3-9a0c-0305e82c3301",
			"owner": { "left": "dbo.Orders", "right": "dbo.CustomerOrders" },
			"left": "ModifiedOn",
			"right": "Modified",
			"kind": "field",
			"action": "map"
		},
		{
			"id": "0f8fad5b-d9cb-469f-a165-70867728950e",
			"owner": { "left": "dbo.Tasks", "right": "dbo.Tasks" },
			"left": "ModifiedOn",
			"right": "Modified",
			"kind": "field",
			"action": "map"
		}
	]
}
```
The property is named `owner` rather than `table` because it is not always a table: for a table element it holds the names of the schemas.
