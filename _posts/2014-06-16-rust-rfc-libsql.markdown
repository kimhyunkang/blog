---
layout: post
title: "Rust RFC: libsql &amp; libsql_macro"
date: 2014-06-16 11:41:43 +0900
comments: true
categories:
---

I'm trying to create a type-safe, implementation-neutral DSL to relational database systems. This is going be a fairly large project, and considering the importance of SQL libraries, I want as many criticism as possible in the earliest design phase. Please feel free to send me a feedback by the comment or by email (kimhyunkang at gmail.com).

## Design goal

The goal this library is trying to achieve is:

* A unified interface to access various RDBMS systems
* SQL-like DSL to generate SQL queries
* Type-safe database manipulation

The system would be separated into three subsystems.

* Compiler macro for rustc, which translates an SQL-like DSL into an internal representation.
* Database-neutral representation of SQL. To enforce type-safety, dynamic creation of this representation without `sql_macro` will be discouraged.
* Adapters for each different database systems. 

## Motivating Example

{% highlight rust %}
#![feature(plugin)]
#[phase(plugin)]
extern crate sql_macro;
extern crate sql;
extern crate sqladapter_sqlite3;

use sql::SqlTable;
use sqladapter_sqlite3::SqliteError;

#[sql_table(primary_key="id")]
pub struct Person {
  // TableRef class represents primary key to the table
  pub id: TableRef<Person>,
  pub name: String,
  pub address: Option<String>
}

fn create_person_table() -> Result<bool, SqliteError> {
  let mut connection = try!(sqladapter_sqlite3::open("person.sqlite3"));
  return connection.create_table::<Person>();
  // CREATE TABLE Person (
  //      id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
  //      name TEXT NOT NULL,
  //      address TEXT);
}
{% endhighlight %}

`sql_table` is a `deriving`-like macro which creates an impl for `SqlTable`. This impl will generate a table schema which matches the type of struct `Person`.

With the table ready, we could create interesting SQL-like DSL in type-safe manner.

Select query is done by macro
{% highlight rust %}
fn select_by_name(name: String) -> Result<Vec<Person>, SqliteError> {
  let mut connection = try!(sqladapter_sqlite3::open("person.sqlite3"));

  // {:s} denotes a binding variable with type String
  let query = sql!(SELECT * FROM Person WHERE name == {:s});

  // prepare statement
  let mut stmt = try!(connection.prepare_select(query));

  // bind values and execute the query
  let iter = try!(stmt.execute((name,)));

  return result::collect(iter)
}
{% endhighlight %}

`sql!` macro will create type-safe queries for `Person` table. By type-safe, I mean the macro catches many possible error cases in compile-time.

For example:

* Each field of `Person` must implement `SqlType` trait, which defines interface between Rust types and internal datatypes of each database.
* Select from table which does not implement `SqlTable` is not allowed.
* Selecting column names not defined in the `Person` table is not allowed.
* `String` field does not allow `NULL`, but `Option<String>` field allows `NULL`, which translates into `None` in the Rust code.
* `{:s}` enforces types of the binding. If you compare different types in the `WHERE` clause, or if you bind types other than String, the compiler throws type error.

Insert query is done without macro:

{% highlight rust %}
fn insert_person(name: String, address: Option<String>) -> Result<bool, SqliteError> {
  let mut connection = try!(sqladapter_sqlite3::open("person.sqlite3"));
  let person = Person {
    // null() represents that we don't know the id of the record yet
    id: TableRef::null(),
    name: name,
    address: address
  };

  connection.insert_table(person)
}
{% endhighlight %}

## Restrictions on DSL 

The DSL `sql!` looks like SQL, but will be different from standard SQL in some ways.

* To support unified interface over various RDBMSs, `sql!` macro would support a subset of standard SQL. I'm thinking about two DDL queries, `CREATE` and `DROP`, and four DML queries, `SELECT`, `INSERT`, `UPDATE` and `DELETE`.
* I'm going to disallow any implicit type conversion. Even `Option<T>` <-> `T` won't be implicit.
* When selecting from multiple tables, column reference would always require table names. For example, `SELECT a, b FROM Table1, Table2` will be disallowed even if column names are not ambiguous.
* Types of binding variables will be explicitly annotated. `{:s}` denotes `String`, `{:s?}` denotes `Option<String>`, and so on...

## Possible internal representation

The internal representation is going to change time to time, as it is not intended to directly created by users. Currently I'm planning to generate something like this.

{% highlight rust %}
  let query = sql!(SELECT id, address FROM Person WHERE name == {:s});

  // Above query would roughly translate into below

  let query:SelectQuery<(TableRef<Person>, Option<String>), (String,)> = SelectTableQuery {
    column_type: None::<Person>.map(|person| {
      (person.id, person.address)
    }),
    bind_type: None::<(Person, String)>.map(|(person, bindvar0)| {
      let _where_clause:bool = person.name == bindvar0;
      (bindvar0,)
    }),
  
    columns: SelectColumns(vec![ColumnExpr("id"), ColumnExpr("address")]),
    from_clause: TableList(vec!["Person"]),
    where_clause: BinaryExpr(ColumnExpr("name"), SqlOpEq, BindExpr)
  };
{% endhighlight %}

The internal representation has type `SelectQuery<ColumnType, BindType>`. `ColumnType` is `SqlTable` or tuple of `SqlType`s. `BindType` is `()` (when there's nothing to bind) or tuple of `SqlType`s. Both `ColumnType` and `BindType` is inferred by the compiler.

`column_type` and `bind_type` part is for the type checker, and the closures wrapped by `None.map()` are never actually run. `column_type` has type `Option<ColumnType>` and `bind_type` has type `Option<BindType>`.

`columns`, `from_clause` and `where_clause` are for query generators, and this translates to different SQLs for each different database. 

## Transaction control

`Connection::transaction()` method receives a closure, and wraps them with `BEGIN TRANSACTION` and `END TRANSACTION`. Transaction receives a closure with type `f: |Transaction| -> Result<(), SqlError>`. If the closure returns `Ok`, the transaction successfully commits. If it's not successful, the transaction rollbacks. The transaction object also exposes `rollback(self)` method which rolls back transaction even if closure returns `Ok`. `Transaction::rollback()` method receives self by value, which prevents using the transaction object again.

## Proposed design of database adapter

Each database adapters must implement 3 (or more) layers of traits.

### Connection
`Connection` represents database connection. Access to the connection would require `&mut` access, which disallows shared access between multiple tasks, though I'm not entirely sure about this decision.

### Statement
`Connection::prepare_*` method returns Statement. Statement represents prepared SQL statements. You can `execute` the statement to fetch results from the query.

### Row
Row represents each row returned by `execute` method. Queries other than `SELECT` returns number of successful operations instead of Row.

Additionally, each adapter library must implement QueryProfile, which represents policy of each database to translate internal representation to actual SQL. The design of policy object would roughly follow that of SQLAlchemy.

## Things which need more thoughts

* Type-safe `CREATE INDEX` query
* Collate handling / unicode handling
* Extensibility of the type system
* Limited ability to generate queries in runtime
* Callback handling
* Generic connection pool to share connection between tasks (Implementation in [sfacker/rust-progres](https://github.com/sfackler/rust-postgres) looks good)

## Possible Q&As

### Is this an ORM for Rust?

Though the design resembles an ORM, the design is intended to provide type safety to database access, not providing full-featured ORM. You can create an ORM over this if you want to, but that's not going to be me.

### Why not generate the entire SQL in the syntax phase?

If we know the type of database beforehand, we can generate entire SQL in the syntax phase and save it as a static string. But I believe that would seriously harm the expressiveness of the DSL, and the ability to switch between different databases.
