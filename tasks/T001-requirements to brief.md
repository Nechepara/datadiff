# Важно
Этот документ является отправной точкой для создания `brief.md` и ничего более. Этот документ не является иструкцией к исполнению для чего либо другого кроме `brief.md` и не дожен быть задействован никогда и нигде в других процесах кроме внесения корректировок в `brief.md`.
# Задание
Напиши бриф на английском языке на разработку библиотеки `Diff.Structure.dll`, которая будет формировать и возвращать отчет в виде JSON с результатами сравнения структуры одноименных таблиц, взятых из двух разных баз данных MS SQL Server. Для каждой из одноименных таблиц должны совпадать:

- поля и их типы данных (включая data type size, nullable option, precision and scale)
- первичные ключи
- внешние (вторичные) ключи

Если хотя бы одно из условий не выполняется, то это должно быть отражено в отчете в секции `difference.structure`. Описание структуры отчета можно найти в разделе [[#Описание структуры отчета]]

Отчет также содержит:

- строку соединения с первой базой данных, в которой значение пароля заменено на `***` (см. секцию `left`)
- строку соединения со второй базой данных, в которой значение пароля заменено на `***` (см. секцию `right`)
- уникальный список таблиц, найденных в обеих базах данных (см. секцию `tables.detected`)
- список таблиц, которые были исключены системой и не были сравнены по причине невозможности сравнить их структуру. Например, таблица отсутствует в одной базе данных (см. секцию `tables.ignored`)
- список таблиц, которые были исключены пользователем из списка сравнения (см. секцию `tables.unchecked`)

Сравнение идентификаторов таблиц, ключей и полей должно производиться в соответствии с collation базы данных. Строковый компаратор, которым сравниваются имена таблиц, ключей и полей, определяется автоматически по collation левой и правой баз данных. Прежде всего программа обязана убедиться, что collation левой базы данных совпадает с collation правой базы данных. Если эти collation не совпадают, программа должна выбросить соответствующее исключение и прервать процедуру автоопределения компаратора. Список таблиц, исключенных пользователем (`UserExcludedTables`), также должен сравниваться с помощью определенного таким образом компаратора.

При сравнении первичных и внешних ключей:

- имена ключей не должны учитываться, т.к. они могут быть сгенерированы автоматически. Исключение составляют случаи, когда явно запрашивается сравнение по имени ключа (никакого дополнительного свойства в `TableStructureComparisonOptions` в этом случае не требуется, т.к. это будут edge cases)
- порядок следования полей в ключе не учитывается: состав полей сравнивается как множество

Сопоставление внешних ключей одной и той же таблицы выполняется в пределах пары `(table, referencedTable)` в следующем порядке:

1. **Совпадение по составу.** Левый и правый ключ считаются совпавшими, если множества пар `field`/`referencedField` у них равны (без учета порядка и регистра). Такая пара в отчет не попадает. Если кандидатов несколько, предпочтение отдается тому, у которого совпадает еще и имя.
2. **Совпадение по имени.** Среди оставшихся несопоставленных ключей в пару объединяются ключи с одинаковым именем. Такая пара попадает в `difference.structure.inconsistencies.foreignKeys`.
3. **Совпадение по пересечению полей.** Оставшиеся ключи объединяются в пары по максимальному пересечению множеств полей. Такая пара также попадает в `inconsistencies.foreignKeys`.
4. **Остаток.** Ключи, которым не нашлось пары, попадают в `difference.structure.missings` (см. описание направления секции `missings` в разделе [[#Пояснения к структуре отчета]]).

Библиотека должна содержать в себе класс `TableStructureComparer`, который будет формировать ожидаемый отчет в виде JSON. Отчет должен быть построен и возвращен методом `CompareTableStructureAsync`, который принимает на вход экземпляр `TableStructureComparisonOptions` и экземпляр `CancellationToken`:
```csharp
public enum QuotesUsage : byte { UseIfSpecifiedOrRequired = 0, Required = 1, DoNotUseIfPossible = 2 }

public partial class QuoteInfo
{
	public Quotes Quotes { get; init; }
	public Func<string, bool> QuotesRequired { get; init; }

	public static QuoteInfo MsSql { get; }
}

public sealed class TableStructureComparisonOptions
{
	public (string Left, string Right) ConnectionStrings {get; init;}
	public IReadOnlyList<string> UserExcludedTables { get; init; } = [];
	public QuoteInfo QuoteInfo { get; init; } = QuoteInfo.MsSql;
	public QuotesUsage QuotesUsage { get; init; } = QuotesUsage.DoNotUseIfPossible;
}
```
Имена исключенных пользователем таблиц задаются строками вида `schema.table`. Любая из двух частей может быть взята в кавычки (`[dbo].[Order.Archive]`), поэтому точка внутри идентификатора не путается с разделителем.

Свойства `QuoteInfo` и `QuotesUsage` задают правила квотирования идентификаторов на весь запуск: `QuoteInfo` описывает пару символов кавычек и предикат «требует ли идентификатор кавычек», а `QuotesUsage` определяет, как имена выводятся в отчет — `Required` всегда в кавычках, `DoNotUseIfPossible` только когда без них нельзя, `UseIfSpecifiedOrRequired` дополнительно сохраняет кавычки, поставленные пользователем.
Метод `CompareTableStructureAsync` возвращает результат в виде экземпляра `Task<TableStructureComparisonResult>`. Тип `TableStructureComparisonResult` представляет структуру JSON, описанную в разделе [[#Описание структуры отчета]].
```csharp
public sealed class TableStructureComparer
{
	public TableStructureComparer(
		IMetadataProvider metadataProvider,
		INameComparerResolver nameComparerResolver);

	public Task<TableStructureComparisonResult> CompareTableStructureAsync(
		TableStructureComparisonOptions options,
		CancellationToken cancellationToken);
}
```
Конструктор класса `TableStructureComparer` принимает два входящих параметра: экземпляр `IMetadataProvider` и экземпляр `INameComparerResolver`.

`INameComparerResolver` отвечает за определение строкового компаратора по collation базы данных. Он вызывается по одному разу для каждой из двух баз, и `TableStructureComparer` сравнивает два полученных результата между собой. Реализация для MS SQL должна быть предоставлена в классе `MsSqlNameComparerResolver`, который читает collation через `IMetadataProvider` и отображает ее на `StringComparer`:
```csharp
public interface INameComparerResolver
{
	public Task<StringComparer> ResolveNameComparerAsync(
		string connectionString,
		CancellationToken cancellationToken);
}
```
`IMetadataProvider` объявляет контракт чтения метаданных:
```csharp
public interface IMetadataProvider
{
//please add the necessary methods. use async/await approach if possible
}
```
Интерфейс `IMetadataProvider` объявляет контракт для всех необходимых операций по чтению метаданных MS SQL о таблицах, полях, первичных и внешних ключах. Реализация данного интерфейса должна быть предоставлена в классе `MsSqlMetadataProvider`. `TableStructureComparer` сам напрямую не читает метаданные, т.к. эта функция полностью возложена на `MsSqlMetadataProvider`.
Библиотека будет разработана на языке C# с использованием возможностей .NET 10. Везде где это возможно по коду мы должны использовать `async/await`.
Бриф должен быть сохранен в файле brief.md.
# Описание структуры отчета
```json
{
	"left": "The connection string to the first DB. The password value is replaced by ***.",
	"right": "The connection string to the second DB. The password value is replaced by ***.",
	"tables": {
		"detected": ["A string array that contains the distinct list of the table names fetched from both databases. Each name consists of two parts: schema name and table name separated by `.`"],
		"ignored": {
			"left": ["A string array that contains the list of the table names fetched from the left DB and ignored by the system. These tables are not compared because the system is unable to compare them. Each name consists of two parts: schema name and table name separated by `.`"],
			"right": ["A string array that contains the list of the table names fetched from the right DB and ignored by the system. These tables are not compared because the system is unable to compare them. Each name consists of two parts: schema name and table name separated by `.`"]
		},
		"unchecked": ["A string array that contains the distinct list of the table names fetched from both databases and unchecked by user. These tables are not compared because the user excluded them from comparison. Each name consists of two parts: schema name and table name separated by `.`"]
	},
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"tables": ["A string array that contains the list of the names of the tables that are missing in the LEFT DB but present in the right one. Each name consists of two parts: schema name and table name separated by `.`"],
					"fields": [
						{
							"field": "The name of the field which is missing in the specified table.",
							"table": "The name of the table (with schema) in which the field was found in the other DB."
						}
					],
					"primaryKeys": [
						{
							"key": "The name of the primary key that was found in other DB. Omitted when the primary key is absent in both databases.",
							"table": "The name of the table (with schema) for which the specified primary key was created.",
							"fields": [
								"The name of the field #1 included to the specified primary key",
								"The name of the field #2 included to the specified primary key",
								"...and so on. The whole `fields` property is omitted when the primary key is absent in both databases."
							]
						}
					],
					"foreignKeys": [
						{
							"key": "The name of the foreign key that was found in other DB.",
							"table": "The name of the table (with schema) for which the specified foreign key was created.",
							"referencedTable": "The name of the table (with schema) to which the specified foreign key references.",
							"fields": [{
								"field": "The name of the field included to the specified foreign key",
								"referencedField": "The name of the field in the referenced table to which the field specified in `field` property references."
							}]
						}
					]
				},
				"right": {}
			},
			"inconsistencies": {
				"fields": [{
					"field": "The name of the field for which the data type inconsistency was found.",
					"table": "The name of the table (with schema) in which the specified field is created.",
					"inconsistency": {
						"left": {
							"type": "The name of the field data type.",
							"size": "The data type size. The property is omitted if not specified. JSON type: number.",
							"nullable": "The nullable option. JSON type: boolean.",
							"precision": "The data type precision. The property is omitted if not specified. JSON type: number.",
							"scale": "The data type scale. The property is omitted if not specified. JSON type: number."
						},
						"right": {
							"type": "The name of the field data type.",
							"size": "The data type size. The property is omitted if not specified. JSON type: number.",
							"nullable": "The nullable option. JSON type: boolean.",
							"precision": "The data type precision. The property is omitted if not specified. JSON type: number.",
							"scale": "The data type scale. The property is omitted if not specified. JSON type: number."
						}
					}
				}],
				"primaryKeys": [
					{
						"table": "The name of the table (with schema) for which the primary key was created.",
						"inconsistency": {
							"left": {
								"key": "The name of the primary key that was found in the left DB.",
								"fields": [
									"The name of the field #1 included to the specified primary key",
									"The name of the field #2 included to the specified primary key",
									"...and so on"
								]
							},
							"right": {
								"key": "The name of the primary key that was found in the right DB.",
								"fields": [
									"The name of the field #1 included to the specified primary key",
									"The name of the field #2 included to the specified primary key",
									"...and so on"
								]
							}
						}
					}
				],
				"foreignKeys": [
					{
						"table": "The name of the table (with schema) for which the foreign key was created in both databases.",
						"referencedTable": "The name of the table (with schema) to which the foreign key references in both databases.",
						"inconsistency": {
							"left": {
								"key": "The name of the foreign key that was found in the left DB.",
								"fields": [{
									"field": "The name of the field included to the specified foreign key",
									"referencedField": "The name of the field to which the specified field references."
								}]
							},
							"right": {
								"key": "The name of the foreign key that was found in the right DB.",
								"fields": [{
									"field": "The name of the field included to the specified foreign key",
									"referencedField": "The name of the field to which the specified field references."
								}]
							}
						}
					}
				]
			}
		}
	}
}
```
## Пояснения к структуре отчета

**Направление секции `missings`.** `missings.left` содержит объекты, которых **нет в левой** базе данных, но которые присутствуют в правой. `missings.right` — зеркально: объекты, которых нет в правой базе данных, но которые есть в левой. Узел `missings.right` имеет ту же структуру, что и `missings.left`, и в описании выше приведен пустым только для краткости.

**Соотношение списков таблиц.** `tables.unchecked` является подмножеством `tables.detected`. Имена, указанные пользователем в `UserExcludedTables`, но не найденные ни в одной базе данных, в отчет не попадают. Сравниваемое множество таблиц вычисляется как `detected − unchecked − ignored.left − ignored.right`.

**Таблица, найденная только в одной базе данных**, попадает одновременно в две секции: в `tables.ignored.<сторона, где таблица есть>` — как объяснение, почему она не сравнивалась, и в `missings.<противоположная сторона>.tables` — как зафиксированное структурное различие. Поля и ключи такой таблицы отдельно как отсутствующие не перечисляются.

**Отсутствие первичного ключа в обеих базах данных.** В элементе `missings.*.primaryKeys` свойства `key` и `fields` являются необязательными: если первичный ключ отсутствует в обеих базах данных, элемент содержит только свойство `table` (см. [[#Use Case 7: Primary key is missing in both databases]]).

**Типы значений.** В блоке выше значения свойств заменены их описанием. Фактические типы: `size`, `precision`, `scale` — `number`, причем свойство отсутствует в JSON, если значение не задано (ни одно свойство никогда не выводится со значением `null`); `nullable` — `boolean`; все остальные свойства — `string`. Размер для типов `nchar`/`nvarchar` указывается в символах, а не в байтах; значение `MAX` передается как `-1`.

**Форма узлов в `inconsistencies`.** Все три секции (`fields`, `primaryKeys`, `foreignKeys`) имеют единообразную форму: свойства, общие для обеих сторон (`table`, `field`, `referencedTable`), находятся в элементе массива, а сравниваемые стороны вложены в свойство `inconsistency` с узлами `left` и `right`.

**Пустые секции.** Все секции отчета присутствуют всегда. Пустые коллекции сериализуются как `[]`, пустые объекты — как `{}`.
# Use Case 1: Table is missing
Если в правой базе данных присутствует таблица `dbo.Orders`, а в левой ее нет, то такая таблица должна быть отражена в отчете в разделе `difference.structure.missings.left.tables`. Если таблица есть в левой базе данных, но отсутствует в правой, то она отражается в разделе `difference.structure.missings.right.tables`:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": { "tables": [ "dbo.Orders" ] }
			}
		}
	}
}
```
# Use Case 2: Field is missing
Если программа находит поле `Sum`, которое есть в таблице `dbo.Orders` в правой базе данных, однако отсутствует в этой же таблице в левой базе данных, то такое поле должно быть отражено в отчете в разделе `difference.structure.missings.left.fields`. В обратной ситуации поле отражается в разделе `difference.structure.missings.right.fields`:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"fields": [ { "field": "Sum", "table": "dbo.Orders" } ]
				}
			}
		}
	}
}
```
# Use Case 3: Primary key is missing
Если для одной и той же таблицы `dbo.Orders` в правой базе данных создан первичный ключ `PK_Orders`, а в левой базе данных такого первичного ключа нет, то такой первичный ключ должен быть указан в отчете в разделе `difference.structure.missings.left.primaryKeys`. В обратной ситуации ключ указывается в разделе `difference.structure.missings.right.primaryKeys`:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"primaryKeys": [
						{
							"key": "PK_Orders",
							"table": "dbo.Orders",
							"fields": ["OrderId"]
						}
					]
				}
			}
		}
	}
}
```
# Use Case 4: Foreign key is missing
Если для одной и той же таблицы `dbo.Orders` в правой базе данных создан внешний ключ `FK_Orders_Customers`, который ссылается на таблицу `dbo.Customers` и на поле `CustomerId` по полю `CustomerId`, а в левой базе данных такого внешнего ключа нет, то такой ключ должен быть указан в отчете в разделе `difference.structure.missings.left.foreignKeys`. В обратной ситуации ключ указывается в разделе `difference.structure.missings.right.foreignKeys`:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"foreignKeys": [
						{
							"key": "FK_Orders_Customers",
							"table": "dbo.Orders",
							"referencedTable": "dbo.Customers",
							"fields": [{
								"field": "CustomerId",
								"referencedField": "CustomerId"
							}]
						}
					]
				}
			}
		}
	}
}
```
# Use Case 5: Field type inconsistency
Если для одной и той же таблицы `dbo.Orders` программа находит поле `Description`, однако data type (or size or nullable option) для этого поля отличаются в базах данных, то такое поле должно быть указано в отчете в разделе `difference.structure.inconsistencies.fields`:
```json
{
	"difference": {
		"structure": {
			"inconsistencies": {
				"fields": [{
					"field": "Description",
					"table": "dbo.Orders",
					"inconsistency": {
						"left": {
							"type": "nvarchar",
							"size": 250,
							"nullable": true
						},
						"right": {
							"type": "varchar",
							"size": 500,
							"nullable": false
						}
					}
				}]
			}
		}
	}
}
```
# Use Case 6: Primary key inconsistency
Если структура первичного ключа в таблице `dbo.Orders` не соответствует структуре первичного ключа в одноименной таблице, взятой из другой базы данных, то информация об этом первичном ключе должна быть отражена в отчете в разделе `difference.structure.inconsistencies.primaryKeys`:
Обратите внимание: имена ключей (`PK_Orders` и `PKOrders_123`) в сравнении не участвуют и приводятся в отчете только справочно. Причиной попадания ключа в отчет является отличающийся состав полей: `OrderId, CustomerId` слева против `OrderId` справа. Если бы состав полей совпадал, а отличались только имена, то такой первичный ключ в отчет не попадает. Порядок следования полей в ключе при сравнении не учитывается: в массиве `fields` поля приводятся в порядке ключа, но сравниваются как множество.
```json
{
	"difference": {
		"structure": {
			"inconsistencies": {
				"primaryKeys": [{
					"table": "dbo.Orders",
					"inconsistency": {
						"left": {
							"key": "PK_Orders",
							"fields": [ "OrderId", "CustomerId" ]
						},
						"right": {
							"key": "PKOrders_123",
							"fields": [ "OrderId" ]
						}
					}
				}]
			}
		}
	}
}
```
# Use Case 7: Primary key is missing in both databases
Если для одной и той же таблицы `dbo.Customers` отсутствует первичный ключ в обеих базах данных, то это должно быть отражено в отчете в разделах `difference.structure.missings.left.primaryKeys` и `difference.structure.missings.right.primaryKeys`. Свойства `key` и `fields` в этом случае не указываются:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"primaryKeys": [
						{
							"table": "dbo.Customers"
						}
					]
				},
				"right": {
					"primaryKeys": [
						{
							"table": "dbo.Customers"
						}
					]
				}
			}
		}
	}
}
```
# Use Case 8: Foreign key inconsistency
Пусть в первой базе данных существует таблица `dbo.Orders`, для которой определен внешний ключ `FK_Orders_Customers`, ссылающийся на таблицу `dbo.Customers` по следующим полям:

| Field        | Referenced Field |
| ------------ | ---------------- |
| CustomerId   | CustomerId       |
| DepartmentId | DepartmentId     |

Также этот внешний ключ не совпадает ни с одним внешним ключом этой же таблицы, но из второй базы данных.

Пусть во второй базе данных существует таблица `dbo.Orders`, для которой определен внешний ключ `FK_Ord_Cust_123`, ссылающийся на таблицу `dbo.Customers` по следующим полям:

| Field      | Referenced Field |
| ---------- | ---------------- |
| CustomerId | CustomerId       |
| CreatedOn  | CreatedOn        |

Также этот внешний ключ не совпадает ни с одним внешним ключом этой же таблицы, но из первой базы данных.

Если такая ситуация обнаруживается, то это должно быть отражено в отчете в разделе `difference.structure.inconsistencies.foreignKeys`:
```json
{
	"difference": {
		"structure": {
			"inconsistencies": {
				"foreignKeys": [
					{
						"table": "dbo.Orders",
						"referencedTable": "dbo.Customers",
						"inconsistency": {
							"left": {
								"key": "FK_Orders_Customers",
								"fields": [
									{
										"field": "CustomerId",
										"referencedField": "CustomerId"
									},
									{
										"field": "DepartmentId",
										"referencedField": "DepartmentId"
									}
								]
							},
							"right": {
								"key": "FK_Ord_Cust_123",
								"fields": [
									{
										"field": "CustomerId",
										"referencedField": "CustomerId"
									},
									{
										"field": "CreatedOn",
										"referencedField": "CreatedOn"
									}
								]
							}
						}
					}
				]
			}
		}
	}
}
```
Если во второй базе данных существует также внешний ключ `FK_Orders_Customers` на таблице `dbo.Orders`, который ссылается на таблицу `dbo.Customers`, и этот внешний ключ не совпадает по составу полей ни с одним внешним ключом этой же таблицы, но из первой базы данных, то этот внешний ключ имеет более высокий приоритет перед `FK_Ord_Cust_123`, т.к. его имя совпадает с именем внешнего ключа из первой базы данных (правило 2 порядка сопоставления, см. раздел [[#Задание]]). В таком случае разница должна быть отражена в отчете следующим образом:
```json
{
	"difference": {
		"structure": {
			"missings": {
				"left": {
					"foreignKeys": [
						{
							"key": "FK_Ord_Cust_123",
							"table": "dbo.Orders",
							"referencedTable": "dbo.Customers",
							"fields": [
								{
									"field": "CustomerId",
									"referencedField": "CustomerId"
								},
								{
									"field": "CreatedOn",
									"referencedField": "CreatedOn"
								}
							]
						}
					]
				}
			},
			"inconsistencies": {
				"foreignKeys": [
					{
						"table": "dbo.Orders",
						"referencedTable": "dbo.Customers",
						"inconsistency": {
							"left": {
								"key": "FK_Orders_Customers",
								"fields": [
									{
										"field": "CustomerId",
										"referencedField": "CustomerId"
									},
									{
										"field": "DepartmentId",
										"referencedField": "DepartmentId"
									}
								]
							},
							"right": {
								"key": "FK_Orders_Customers",
								"fields": [
									{
										"field": "CustomerId",
										"referencedField": "CustomerId"
									}
								]
							}
						}
					}
				]
			}
		}
	}
}
```
