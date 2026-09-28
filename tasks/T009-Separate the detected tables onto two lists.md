Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.
At the moment, the `tables.detected` is array of of the string values (the tables names found in the both databases). But we loose information about database from which the table came from. Please do the following adjustments to avoid this issue:
```json
{
	"detected": {
      "left": ["The distinct list of table names fetched from left database, `schema.table`"],
      "right": ["The distinct list of table names fetched from right database, `schema.table`"]
    }
}
```