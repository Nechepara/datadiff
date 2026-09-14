Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.

Please rename `NameResolution` class to `ConflictResolution`.

Please rename `NameResolution` property to `ConflictResolution` in the `TableStructureComparisonOptions` class.
Please make properties `Left` and `Right` nullable in the `ConflictResolution` class.

How to interpret conflict resolution object:

| Left                                                 | Right                                              | Kind  | Interpretation                                                                                                                                                                                                                                   |
| ---------------------------------------------------- | -------------------------------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| refer to dbo.Table1                                  | `null`                                             | Table | The table `dbo.Table1` from the `Left` database should be excluded from the structure comparison. It is analog of `UserExcludedTables` which we will not use anymore.                                                                            |
| `null`                                               | refer to dbo.Table1                                | Table | The table `dbo.Table1` from the `Right` database should be excluded from the structure comparison. It is analog of `UserExcludedTables` which we will not use anymore.                                                                           |
| refer to dbo.Table2                                  | refer to dbo.table2                                | Table | The table `dbo.Table2` from the `Left` database should be compared with table `dbo.table2` from `Right` database. It is useful in case when the system did not detect the table pair for comparison because of the different database collations |
| refer to dbo.Orders                                  | refer to dbo.CustomerOrders                        | Table | The table `dbo.Orders` from the `Left` database should be compared with table `dbo.CustomerOrders` from `Right` database. It is useful in case when the table has been renamed in the one of the databases.                                      |
| refer to `ModifiedOn` field in the `dbo.Tasks` table | refer to `Modified` field in the `dbo.Tasks` table | Field | The field `ModifiedOn` (table `dbo.Tasks` from the left database) should be compared with field `Modified` (table `dbo.Tasks` from the right database) . Useful when the field has been renamed in the one of the databases.                     |
To find the appropriate `ConflictResolution` instance that matches to the specified object (table or field) taken from the `Left` database the system should compare the name of the specified object with the `ConflictResolution.Left.Name` using `StringComparer.Ordinal` ONLY.

To find the appropriate `ConflictResolution` instance that matches to the specified object (table or field) taken from the `Right` database the system should compare the name of the specified object with the `ConflictResolution.Right.Name` using `StringComparer.Ordinal` ONLY.

The unit test should be also present for all these cases.

Please remove `UserExcludedTables` property from the table structure comparison options. We should use the `ConflictResolution` property instead.  `UserExcludedTables` should not be mentioned in the `brief.md` at all.

# Answers on the related questions
Question: 1. **An exclusion written in the wrong case silently excludes nothing.** The matching rule is fixed by the requirement: a `ConflictResolution` side finds its object with `StringComparer.Ordinal` and nothing else ([§3.1.1](app://obsidian.md/index.html#3.1.1%20Conflicts:%20%60CandidateInfo%60%20and%20%60ConflictResolution%60)). That is the right rule for the mechanism's harder job — only an ordinal comparison can single out one of two names the run's own collation calls equal, which is what answering an `AmbiguousNameException` requires — but it costs something the string-based exclusion list this mechanism replaced used to give: a caller who writes `dbo.orders` against a database holding `dbo.Orders` now excludes nothing, on every collation, and is not told, since an entry matching nothing is not an error. The implementation follows the rule as written and mitigates it by documentation only: build entries from identities the library handed back, and check `tables.detected` for a table that should have been excluded. Please confirm, or say whether a matchless entry should instead be reported — either as a hard failure, or as a new report section listing entries that applied to nothing, which would make a typo visible without weakening the comparison.
Answer: It is the expected behavior. 
When the user specifies `dbo.orders` in the `ConflictResolution` in the `Left` property, but the real table name is `dbo.Orders` in the `Left` database, the system should not take into account such conflict resolution for that table.
When the user specifies `dbo.orders` in the `ConflictResolution` in the `Right` property, but the real table name is `dbo.Orders` in the `Right` database, the system should not take into account such conflict resolution for that table.
Please let me know if something is left unclear here.