# Perfect CRUD [简体中文](README.zh_CN.md)

CRUD is an object-relational mapping (ORM) system for Swift 4+. CRUD takes Swift 4 `Codable` types and maps them to SQL database tables. CRUD can create tables based on `Codable` types and perform inserts and updates of objects in those tables. CRUD can also perform selects and joins of tables, all in a type-safe manner.

CRUD uses a simple, expressive, and type safe methodology for constructing queries as a series of operations. It is designed to be light-weight and has zero additional dependencies. It uses generics, KeyPaths and Codables to make sure any misusage is caught at compile time.

Database client library packages can add CRUD support by implementing a few protocols. Support is available for [SQLite](https://github.com/kjessup/Perfect-SQLite) and [Postgres](https://github.com/kjessup/Perfect-PostgreSQL).

## General Usage

This is a simple example to show how CRUD is used.

```swift
// CRUD can work with most Codable types.
struct PhoneNumber: Codable {
	let id: UUID
	let personId: UUID
	let planetCode: Int
	let number: String
}
struct Person: Codable {
	let id: UUID
	let firstName: String
	let lastName: String
	let phoneNumbers: [PhoneNumber]?
}
// CRUD usage begins by creating a database connection. The inputs for connecting to a database will differ depending on your client library.
// Create a `Database` object by providing a configuration. These examples will use SQLite for demonstration purposes.
let db = Database(configuration: try SQLiteDatabaseConfiguration(testDBName))
// Create the table if it hasn't been done already.
// Table creates are recursive by default, so "PhoneNumber" is also created here.
try db.create(Person.self, policy: .reconcileTable)
// Get a reference to the tables we will be inserting data into.
let personTable = db.table(Person.self)
let numbersTable = db.table(PhoneNumber.self)
// Add an index for personId, if it does not already exist.
try numbersTable.index(\.personId)
do {
	// Insert some sample data.
	let personId1 = UUID()
	let personId2 = UUID()
	try personTable.insert([
		Person(id: personId1, firstName: "Owen", lastName: "Lars", phoneNumbers: nil),
		Person(id: personId2, firstName: "Beru", lastName: "Lars", phoneNumbers: nil)])
	try numbersTable.insert([
		PhoneNumber(id: UUID(), personId: personId1, planetCode: 12, number: "555-555-1212"),
		PhoneNumber(id: UUID(), personId: personId1, planetCode: 15, number: "555-555-2222"),
		PhoneNumber(id: UUID(), personId: personId2, planetCode: 12, number: "555-555-1212")
	])
}
// Let's find all people with the last name of Lars which have a phone number on planet 12.
let query = try personTable
		.order(by: \.lastName)
	.join(\.phoneNumbers, on: \.id, equals: \.personId)
		.order(descending: \.planetCode)
	.where(\Person.lastName == .string("Lars") && \PhoneNumber.planetCode == .integer(12))
	.select()
// Loop through them and print the names.
for user in query {
	print("\(user.firstName) \(user.lastName)")
	// We joined PhoneNumbers, so we should have values here.
	guard let numbers = user.phoneNumbers else {
		continue
	}
	for number in numbers {
		print(number.number)
	}
}
```

## Operations

Activity in CRUD is accomplished by obtaining a database connection object and then chaining a series of operations on that database. Some operations execute immediately while others (select) are executed lazily. Each operation that is chained will return an object which can be further chained or executed.

Operations are grouped here according to the objects which implement them. Note that many of the type definitions shown below have been abbreviated for simplicity and some functions implemented in extensions have been moved in to keep things in a single block.

### Database

A Database object wraps and maintains a connection to a database. Database connectivity is specified by using a `DatabaseConfigurationProtocol` object. These will be specific to the database in question.

```swift
// postgres sample configuration
let db = Database(configuration: 
	try PostgresDatabaseConfiguration(database: postgresTestDBName, host: "localhost"))
// sqlite sample configuration
let db = Database(configuration: 
	try SQLiteDatabaseConfiguration(testDBName))
```

Database objects implement this set of logical functions:

```swift
public struct Database<C: DatabaseConfigurationProtocol>: DatabaseProtocol {
	public typealias Configuration = C
	public let configuration: Configuration
	public init(configuration c: Configuration)
	public func table<T: Codable>(_ form: T.Type) -> Table<T, Database<C>>
	public func transaction<T>(_ body: () throws -> T) throws -> T
	public func create<A: Codable>(_ type: A.Type, 
		primaryKey: PartialKeyPath<A>? = nil, 
		policy: TableCreatePolicy = .defaultPolicy) throws -> Create<A, Self>
}
```

The operations available on a Database object include `transaction`, `create`, and `table`. 

#### transaction

The `transaction` operation will execute the body between a set of "BEGIN" and "COMMIT" or "ROLLBACK" statements. If the body completes execution without throwing an error then the transaction will be committed, otherwise it is rolled-back.

```swift
public extension Database {
	func transaction<T>(_ body: () throws -> T) throws -> T
}
```

Example usage:

```swift
try db.transaction {
	... further operations
}
```

#### create

The `create` operation is given a Codable type. It will create a table corresponding to the type's structure. The table's primary key can be indicated as well as a "create policy" which determines some aspects of the operation. 

```swift
public extension DatabaseProtocol {
	func create<A: Codable>(
		_ type: A.Type, 
		primaryKey: PartialKeyPath<A>? = nil, 
		policy: TableCreatePolicy = .defaultPolicy) throws -> Create<A, Self>
}
```

Example usage:

```swift
try db.create(TestTable1.self, primaryKey: \.id, policy: .reconcileTable)
```

`TableCreatePolicy` consists of the following options:

* .shallow - If indicated, then joined type tables will not be automatically created. If not indicated, then any joined type tables will be automatically created.
* .dropTable - The database table will be dropped before it is created. This can be useful during development and testing, or for tables which contain ephemeral data which can be reset after a restart.
* .reconcileTable - If the database table already exists then any columns which differ between the type and the table will be either removed or added.

Calling create on a table which already exists is a harmless operation resulting in no changes unless the `.reconcileTable` or `.dropTable` policies are indicated. Existing tables will not be modified to match changes in the corresponding Codable type unless `.reconcileTable` is indicated.

#### table

The `table` operation returns a Table object based on the indicated Codable type. Table objects are used to perform further operations.

```swift
public protocol DatabaseProtocol {
	func table<T: Codable>(_ form: T.Type) -> Table<T, Self>
}
```

Example usage:

```swift
let table1 = db.table(TestTable1.self)
```

### Table

A Table object can be used to perform updates, inserts, deletes or selects. Tables can only be accessed through a database object by providing the Codable type which is to be mapped. A table object can only appear in an operation chain once, and it must be the first item.

Table objects are parameterized based on the Swift object type you provide when you retrieve the table. Tables indicate the over-all resulting type of any operation. This will be referred to as the *OverAllForm*.

Example usage:

```swift
// get a table object representing the TestTable1 struct
// any inserts, updates, or deletes will effect "TestTable1"
// any selects will produce a collection of TestTable1 objects.
let table1 = db.table(TestTable1.self)
```

In the example above, TestTable1 is the OverAllForm. Any destructive operations will effect the corresponding database table. Any selects will produce a collection of TestTable1 objects.

**Table** can follow: `Database`.

**Table** supports: `update`, `insert`, `delete`, `join`, `order`, `limit`, `where`, `select`, and `count`.

### Join

A `join` operation brings in a collection of additional objects which will be set on the resulting OverAllForm objects. The joined objects will be set as a property of the parent OverAllForm object. Joins are only useful when eventually performing a `select`. Joins are not currently supported in updates, inserts, or deletes (cascade deletes/recursive updates are not supported).

CRUD supports one-to-many as well as many-to-many joins using a pivot table.

```swift
public protocol JoinAble: TableProtocol {
	// standard join
	func join<NewType: Codable, KeyType: Equatable>(
		_ to: KeyPath<OverAllForm, [NewType]?>,
		on: KeyPath<OverAllForm, KeyType>,
		equals: KeyPath<NewType, KeyType>) throws -> Join<OverAllForm, Self, NewType, KeyType>
	// pivot join
	func join<NewType: Codable, Pivot: Codable, FirstKeyType: Equatable, SecondKeyType: Equatable>(
		_ to: KeyPath<OverAllForm, [NewType]?>,
		with: Pivot.Type,
		on: KeyPath<OverAllForm, FirstKeyType>,
		equals: KeyPath<Pivot, FirstKeyType>,
		and: KeyPath<NewType, SecondKeyType>,
		is: KeyPath<Pivot, SecondKeyType>) throws -> JoinPivot<OverAllForm, Self, NewType, Pivot, FirstKeyType, SecondKeyType>
}
```

A standard join requires three parameters: 

`to` - keypath to a property of the OverAllForm. This keypath should point to an Optional array of non-integral Codable types. This property will be set with the resulting objects.

`on` - keypath to a property of the OverAllForm which should be used as the primary key for the join (typically one would use the actual table primary key column).

`equals` - keypath to a property of the joined type which should be equal to the OverAllForm's `on` property. This would be the foreign key.

A pivot join requires six parameters: 

`to` - keypath to a property of the OverAllForm. This keypath should point to an Optional array of non-integral Codable types. This property will be set with the resulting objects.

`with` - The type of the pivot table.

`on` - keypath to a property of the OverAllForm which should be used as the primary key for the join (typically one would use the actual table primary key column).

`equals` - keypath to a property of the pivot type which should be equal to the OverAllForm's `on` property. This would be the foreign key.

`and` - keypath to a property of the child table type which should be used as the key for the join (typically one would use the actual table primary key column).

`is` - keypath to a property of the pivot type which should be equal to the child table's `and` property.

Example usage:

```swift
let table = db.table(TestTable1.self)
let join = try table.join(\.subTables, on: \.id, equals: \.parentId)
```

The example above joins TestTable2 on the TestTable1.subTables property, which is of type `[TestTable2]?`. When the query is executed, all objects from TestTable2 that have a TestTable2.parentId that matches a TestTable1.id will be included in the results.

Any joined type tables which are not explicitly included in a join will be set to nil for any resulting OverAllForm objects.

If a joined table is included in a join but there are no resulting joined objects, the OverAllForm's property will be set to an empty array.

**Join** can follow: `table`, `order`, `limit`, or another `join`.

**Join** supports: `join`, `where`, `order`, `limit`, `select`, `count`.

### Where

A `where` operation introduces a criteria which will be used to filter exactly which objects should be selected, updated, or deleted from the database. Where can only be used when performing a select/count, update, or delete. 

```swift
public protocol WhereAble: TableProtocol {
	func `where`(_ expr: Expression) -> Where<OverAllForm, Self>
}
```

Where operations are optional, but only one `where` can be included in an operation chain and it must be the penultimate operation in the chain.

Example usage:

```swift
let table = db.table(TestTable1.self)
// insert a new object and then find it
let newOne = TestTable1(id: 2000, name: "New One", integer: 40, double: nil, blob: nil, subTables: nil)
try table.insert(newOne)
// search for this one object by id
let query = table.where(\TestTable1.id == .integer(newOne.id))
guard let foundNewOne = try query.select().first else {
	...
}
```

The parameter given to the `where` operation is an `Expression` object. `Expression` is an enum defining the valid expression types. `Expression` is designed to let you use regular Swift syntax when specifying the expression given to CRUD. This expression object is eventually converted to SQL. CRUD uses statement parameter binding when generating SQL statement, so users need not worry about string quoting or binary data encoding.

Many of these expression types represent simple integral values such as `.string(String)` or `.null`. Others are binary or unary operators such as `AND`, or `==`. These would be expressed by using the regular Swift operators `&&` and `==`, respectively.

To illustrate, the two lines that follow are equivalent:

```swift
let query1 = table.where(\TestTable1.id == .integer(newOne.id))
let query2 = table.where(
	.equality(.keyPath(\TestTable1.id), .integer(newOne.id)))
```

The first query uses standard Swift syntax for the `where` clause. The second uses a more verbose approach.

CRUD `Expression` provides the following operator overloads:

```swift
public extension Expression {
	static func &&(lhs: Expression, rhs: Expression) -> Expression
	static func ||(lhs: Expression, rhs: Expression) -> Expression
	static func ==(lhs: Expression, rhs: Expression) -> Expression
	static func !=(lhs: Expression, rhs: Expression) -> Expression
	static func <(lhs: Expression, rhs: Expression) -> Expression
	static func <=(lhs: Expression, rhs: Expression) -> Expression
	static func >(lhs: Expression, rhs: Expression) -> Expression
	static func >=(lhs: Expression, rhs: Expression) -> Expression
	static prefix func !(rhs: Expression) -> Expression
}
```

It also provides versions of the above operators which accept an `Expression` parameter on either the left-hand or right-hand side of the operation, combined with a String, Int, or Double as the other argument.

Additional overloads accept keypaths as the left-hand parameter for the operation:

```swift
public extension Expression {
	static func == <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
	static func != <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
	static func > <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
	static func >= <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
	static func < <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
	static func <= <T: Codable>(lhs: PartialKeyPath<T>, rhs: Expression) -> Expression
}
```

**Where** can follow: `table`, `join`, `order`.

**Where** supports: `select`, `count`, `update` (when following `table`), `delete` (when following `table`).

### Order

An `order` operation introduces an ordering of the over-all resulting objects and/or of the objects selected for a particular join. An order operation should immediately follow either a `table` or a `join`.

```swift
public protocol OrderAble: TableProtocol {
	func order(by: PartialKeyPath<Form>...) -> Ordering<OverAllForm, Self>
	func order(descending by: PartialKeyPath<Form>...) -> Ordering<OverAllForm, Self>
}
```

Example usage:

```swift
let query = try db.table(TestTable1.self)
				.order(by: \.name)
			.join(\.subTables, on: \.id, equals: \.parentId)
				.order(by: \.id)
			.where(\TestTable2.name == .string("Me"))
```

When the above query is executed it will apply orderings to both the main list of returned objects and to their individual "subTables" collections.

**Order** can follow: `table`, `join`.

**Order** supports: `join`, `where`, `order`, `limit` `select`, `count`.

### Limit

A `limit` operation can follow a `table`, `join`, or `order` operation. Limit can both apply an upper bound on the number of resulting objects and impose a skip value. For example the first five found records may be skipped and the result set will begin at the sixth row.

```swift
public protocol LimitAble: TableProtocol {
	func limit(_ max: Int, skip: Int) -> Limit<OverAllForm, Self>
}
```

A limit applies only to the most recent `table` or `join`. A limit placed after a `table` limits the over-all number of results. A limit placed after a `join` limits the number of joined type objects returned.

Example usage:

```swift
let query = try db.table(TestTable1.self)
				.order(by: \.name)
				.limit(10, skip: 20)
			.join(\.subTables, on: \.id, equals: \.parentId)
				.order(by: \.id)
				.limit(1000)
			.where(\TestTable2.name == .string("Me"))
```

**Limit** can follow: `order`, `join`, `table`.

**Limit** supports: `join`, `where`, `order`, `select`, `count`.

### Update

An `update` operation can be used to replace values in the existing records which match the query. An update will almost always have a `where` operation in the chain, but it is not required. Providing no `where` operation in the chain will match all records. 

```swift
public protocol UpdateAble: TableProtocol {
	func update(_ instance: OverAllForm, setKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Update<OverAllForm, Self>
	func update(_ instance: OverAllForm, ignoreKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Update<OverAllForm, Self>
	func update(_ instance: OverAllForm) throws -> Update<OverAllForm, Self>
}
```

An update requires an instance of the OverAllForm. This instance provides the values which will be set in any records which match the query. The update can be performed with either a `setKeys` or `ignoreKeys` parameter, or with no additional parameter to indicate that all columns should be included in the update.

Example usage:

```swift
let table = db.table(TestTable1.self)
let newOne = TestTable1(id: 2000, name: "New One", integer: 40, double: nil, blob: nil, subTables: nil)
try db.transaction {
	try table.insert(newOne)
	let newOne2 = TestTable1(id: 2000, name: "New One Updated", integer: 41, double: nil, blob: nil, subTables: nil)
	try table
		.where(\TestTable1.id == .integer(newOne.id))
		.update(newOne2, setKeys: \.name)
}
let results = try table
	.where(\TestTable1.id == .integer(newOne.id))
	.select().map { $0 }

assert(results.count == 1)
assert(results[0].id == 2000)
assert(results[0].name == "New One Updated")
assert(results[0].integer == 40)
```

**Update** can follow: `table`, `where` (when `where` follows `table`).

**Update** supports: immediate execution.

### Insert

Insert is used to add new records to the database. One or more objects can be inserted at a time. Particular keys/columns can be added or excluded. An insert must immediately follow a `table`.

```swift
public extension Table {
	func insert(_ instances: [Form]) throws -> Insert<Form, Table<A,C>>
	func insert(_ instance: Form) throws -> Insert<Form, Table<A,C>>
	func insert(_ instances: [Form], setKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Insert<Form, Table<A,C>>
	func insert(_ instance: Form, setKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Insert<Form, Table<A,C>>
	func insert(_ instances: [Form], ignoreKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Insert<Form, Table<A,C>>
	func insert(_ instance: Form, ignoreKeys: PartialKeyPath<OverAllForm>, _ rest: PartialKeyPath<OverAllForm>...) throws -> Insert<Form, Table<A,C>>
}
```

Usage example:

```swift
let table = db.table(TestTable1.self)
let newOne = TestTable1(id: 2000, name: "New One", integer: 40, double: nil, blob: nil, subTables: nil)
let newTwo = TestTable1(id: 2001, name: "New One", integer: 40, double: nil, blob: nil, subTables: nil)
try table.insert([newOne, newTwo], setKeys: \.id, \.name)
```

**Insert** can follow: `table`.

**Insert** supports: immediate execution.

### Delete

A `delete` operation is used to remove records from the table which match the query. A delete will almost always have a `where` operation in the chain, but it is not required. Providing no `where` operation in the chain will delete all records.

```swift
public protocol DeleteAble: TableProtocol {
	func delete() throws -> Delete<OverAllForm, Self>
}
```

Example usage:

```swift
let table = db.table(TestTable1.self)
let newOne = TestTable1(id: 2000, name: "New One", integer: 40, double: nil, blob: nil, subTables: nil)
try table.insert(newOne)
let query = table.where(\TestTable1.id == .integer(newOne.id))
let j1 = try query.select().map { $0 }
assert(j1.count == 1)
try query.delete()
let j2 = try query.select().map { $0 }
assert(j2.count == 0)
```

**Delete** can follow: `table`, `where` (when `where` follows `table`).

**Delete** supports: immediate execution.

### Select & Count

Select returns an object which can be used to iterate over the resulting values.

```swift
public protocol SelectAble: TableProtocol {
	func select() throws -> Select<OverAllForm, Self>
	func count() throws -> Int
}
```

Count works similarly to `select` but it will execute the query immediately and simply return the number of resulting objects. Object data is not actually fetched.

Usage example:

```swift
let table = db.table(TestTable1.self)
let query = table.where(\TestTable1.blob == .null)
let values = try query.select().map { $0 }
let count = try query.count()
assert(count == values.count)
```

**Select** can follow: `where`, `order`, `limit`, `join`, `table`.

**Select** supports: iteration.

## Codable Types

Most `Codable` types can be used with CRUD, often, depending on your needs, with no modifications. All of a type's relevant properties will be mapped to columns in the database table. You can customize the column names by adding a `CodingKeys` property to your type. 

By default, the type name will be used as the table name. To customize the name used for a type's table, have the type implement the `TableNameProvider` protocol. This requires a `static let tableName: String` property.

CRUD supports the following property types:

* All Ints, Double, Float, Bool, String
* [UInt8], [Int8], Data
* Date, UUID

The actual storage in the database for each of these types will depend on the client library in use. For example, Postgres will have an actual "date" and "uuid" column types while in SQLite these will be stored as strings.

A type used with CRUD can also have one or more arrays of child, or joined types. These arrays can be populated by using a `join` operation in a query. Note that a table column will not be created for joined type properties.

The following example types illustrate valid CRUD `Codables` using `CodingKeys`, `TableNameProvider` and joined types.

```swift
struct TestTable1: Codable, TableNameProvider {
	enum CodingKeys: String, CodingKey {
		// specify custom column names for some properties
		case id, name, integer = "int", double = "doub", blob, subTables
	}
	// specify a custom table name
	static let tableName = "test_table_1"
	
	let id: Int
	let name: String?
	let integer: Int?
	let double: Double?
	let blob: [UInt8]?
	let subTables: [TestTable2]?
}

struct TestTable2: Codable {
	let id: UUID
	let parentId: Int
	let date: Date
	let name: String?
	let int: Int?
	let doub: Double?
	let blob: [UInt8]?
}
```

Joined types should be an Optional array of Codable objects. Above, the `TestTable1` struct has a joined type on its `subTables` property: `let subTables: [TestTable2]?`. Joined types will only be populated when the corresponding table is joined using the `join` operation.

### Identity

When CRUD creates the table corresponding to a type it attempts to determine what the primary key for the table will be. You can explicitly indicate which property is the primary key when you call the `create` operation. If you do not indicate the key then a property named "id" will be sought. If there is no "id" property then the table will be created without a primary key.
Note that a custom primary key name can be specified when creating tables "shallow" but not when recursively creating them. See the "Create" operation for more details.

## Error Handling

Any error which occurs during SQL generation, execution, or results fetching will produce a thrown Error object. 

CRUD will throw `CRUDDecoderError` or `CRUDEncoderError` for errors occurring during type encoding and decoding, respectively.

CRUD will throw `CRUDSQLGenError` for errors occurring during SQL statement generation.

CRUD will throw `CRUDSQLExeError` for errors occurring during SQL statement execution.

All of the CRUD errors are tied into the logging system. When they are thrown the error messages will appear in the log. Individual database client libraries may throw other errors when they occur.

## Logging

CRUD contains a built-in logging system which is designed to record errors which occur. It can also record individual SQL statements which are generated. CRUD logging is done asynchronously. You can flush all pending log messages by calling `CRUDLogging.flush()`.

Messages can be added to the log by calling `CRUDLogging.log(_ type: CRUDLogEventType, _ msg: String)`.

Example usage:

```swift
// log an informative message.
CRUDLogging.log(.info, "This is my message.")
```

`CRUDLogEventType` is one of: `.info`, `.warning`, `.error`, or `.query`.

You can control where log messages go by setting the `CRUDLogging.queryLogDestinations` and `CRUDLogging.errorLogDestinations` static properties. Handling for errors and queries can be set separately as SQL statement logging may be desirable during development but not in production.

```swift
public extension CRUDLogging {
	public static var queryLogDestinations: [CRUDLogDestination]
	public static var errorLogDestinations: [CRUDLogDestination]
}
```

Log destinations are defined as:

```swift
public enum CRUDLogDestination {
	case none
	case console
	case file(String)
	case custom((CRUDLogEvent) -> ())
}
```

Each message can go to multiple destinations. By default, both errors and queries are logged to the console.


