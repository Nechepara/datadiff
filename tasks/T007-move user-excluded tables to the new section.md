Please analyze the information specified bellow, and make the appropriate corrections in the `brief.md`.
Lets extend the JSON report section `tables.ignored` with additional information:
```json
{
	"tables": {
	    "ignored": {
	      "left":  [
		        { 
			      "table": "The table name taken from the left database",
			      "ignoredBy": "system", //only two values are possible: `system` and `user`
			      "reason": "The table was not found in the other database and was skipped by the system."
			    },
				{ 
			      "table": "The table name taken from the left database",
			      "ignoredBy": "user", //only two values are possible: `system` and `user`
			      "reason": "The table was skipped at the user's request."
			    },
			    ...
		    ],
	      "right": [
{ 
			      "table": "The table name taken from the left database",
			      "ignoredBy": "system", //only two values are possible: `system` and `user`
			      "reason": "The table was not found in the other database and was skipped by the system."
			    },
				{ 
			      "table": "The table name taken from the left database",
			      "ignoredBy": "user", //only two values are possible: `system` and `user`
			      "reason": "The table was skipped at the user's request."
			    },
			    ...
			]
	    }
  }
}
```
The property `table` contains the preferred table name.
The property `ignoredBy` answers on question: Who ignored the table? Two values are possible here: `system` or `user`.
The string property `reason` contains a user-friendly message which indicates the reason of ignoring.
The old section `tables.unchecked` does not exist anymore. All the values that were specified in the `tables.unchecked` should be specified in the `tables.ignored` section with `ignoredBy` = `user` and the appropriate `reason` = `The table was skipped at the user's request.`
The old section `tables.unchecked` should be mentioned nowhere in the brief.
# Resolving of edge-cases
## Case 1: The one side user-excluded table exists in the other database
If some table is excluded by the user from one side only, but the same table exists on the other side, and, if the table on other side is not paired to some other table (the other candidate for the table was not detected), the system should automatically exclude such table from the comparison and indicate the reason why the system excluded this table. The reason should differ from the reason specified above. For example, the table `dbo.Orders` was excluded from the comparison by the user from the left side only:
```json
{
	"tables": {
	    "ignored": {
	      "left":  [
		        { 
			      "table": "dbo.Orders",
			      "ignoredBy": "user",
			      "reason": "The table was skipped at the user's request."
			    }
		    ]
		}
	}
}
```
The table `dbo.Orders` exists on the other side too. The system did not find another candidate for this table and have to exclude such table from the comparison. Such case should be indicated in the JSON in the following way:
```json
{
	"tables": {
	    "ignored": {
	      "right":  [
		        { 
			      "table": "dbo.Orders",
			      "ignoredBy": "system",
			      "reason": "On the other side, the user excluded this table from comparison. The system did not find another candidate for the table. That's why it was skipped by the system during comparison."
			    }
		    ]
		}
	}
}
```
In this case, the table should not appear in the `missings` section because it actually exists.