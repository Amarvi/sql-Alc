# Demo Project

SQLAlchemy

https://docs.sqlalchemy.org/en/14/tutorial/

￼

Tutorial Overview
The tutorial will present both concepts in the natural order that they should be learned, first with a mostly-Core-centric approach and then spanning out into more ORM-centric concepts.
The major sections of this tutorial are as follows:

* Establishing Connectivity - the Engine - all SQLAlchemy applications start with an Engine object; here’s how to create one.
* Working with Transactions and the DBAPI - the usage API of the Engine and its related objects Connection and Result are presented here. This content is Core-centric however ORM users will want to be familiar with at least the Result object.
* Working with Database Metadata - SQLAlchemy’s SQL abstractions as well as the ORM rely upon a system of defining database schema constructs as Python objects. This section introduces how to do that from both a Core and an ORM perspective.
* Working with Data - here we learn how to create, select, update and delete data in the database. The so-called CRUD operations here are given in terms of SQLAlchemy Core with links out towards their ORM counterparts. The SELECT operation that is introduced in detail at Selecting Rows with Core or ORM applies equally well to Core and ORM.
* Data Manipulation with the ORM covers the persistence framework of the ORM; basically the ORM-centric ways to insert, update and delete, as well as how to handle transactions.
* Working with Related Objects introduces the concept of the relationship() construct and provides a brief overview of how it’s used, with links to deeper documentation.
* Further Reading lists a series of major top-level documentation sections which fully document the concepts introduced in this tutorial.

A Note on the Future
This tutorial describes a new API that’s released in SQLAlchemy 1.4 known as 2.0 style. The purpose of the 2.0-style API is to provide forwards compatibility with SQLAlchemy 2.0, which is planned as the next generation of SQLAlchemy.
In order to provide the full 2.0 API, a new flag called future will be used, which will be seen as the tutorial describes the Engine and Session objects. These flags fully enable 2.0-compatibility mode and allow the code in the tutorial to proceed fully. When using the future flag with the create_engine() function, the object returned is a subclass of sqlalchemy.engine.Engine described as sqlalchemy.future.Engine. This tutorial will be referring to sqlalchemy.future.Engine.

from sqlalchemy import create_engine
engine = create_engine("sqlite+pysqlite:///:memory:", echo=True, future=True)

The main argument to create_engine is a string URL, above passed as the string "sqlite+pysqlite:///:memory:". This string indicates to the Engine three important facts:
1. What kind of database are we communicating with? This is the sqlite portion above, which links in SQLAlchemy to an object known as the dialect.
2. What DBAPI are we using? The Python DBAPI is a third party driver that SQLAlchemy uses to interact with a particular database. In this case, we’re using the name pysqlite, which in modern Python use is the sqlite3 standard library interface for SQLite. If omitted, SQLAlchemy will use a default DBAPI for the particular database selected.
3. How do we locate the database? In this case, our URL includes the phrase /:memory:, which is an indicator to the sqlite3 module that we will be using an in-memory-only database. This kind of database is perfect for experimenting as it does not require any server nor does it need to create new files.

Lazy Connecting
The Engine, when first returned by create_engine(), has not actually tried to connect to the database yet; that happens only the first time it is asked to perform a task against the database. This is a software design pattern known as lazy initialization.

We have also specified a parameter create_engine.echo, which will instruct the Engine to log all of the SQL it emits to a Python logger that will write to standard out. This flag is a shorthand way of setting up Python logging more formally and is useful for experimentation in scripts. Many of the SQL examples will include this SQL logging output beneath a [SQL] link that when clicked, will reveal the full SQL interaction.

Working with Transactions and the DBAPI
With the Engine object ready to go, we may now proceed to dive into the basic operation of an Engine and its primary interactive endpoints, the Connection and Result. We will additionally introduce the ORM’s facade for these objects, known as the Session.

As we have yet to introduce the SQLAlchemy Expression Language that is the primary feature of SQLAlchemy, we will make use of one simple construct within this package called the text() construct, which allows us to write SQL statements as textual SQL. Rest assured that textual SQL in day-to-day SQLAlchemy use is by far the exception rather than the rule for most tasks, even though it always remains fully available.

from sqlalchemy import text
with engine.connect() as conn:
    result = conn.execute(text("select 'hello world'"))
    print(result.all())


Tip
“autocommit” mode is available for special cases. The section Setting Transaction Isolation Levels including DBAPI Autocommit discusses this.

with engine.connect() as conn:
    conn.execute(text("CREATE TABLE some_table (x int, y int)"))
    conn.execute(
        text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
        [{"x": 1, "y": 1}, {"x": 2, "y": 4}]
    )
    conn.commit()

BEGIN (implicit)
CREATE TABLE some_table (x int, y int)
[...] ()
<sqlalchemy.engine.cursor.CursorResult object at 0x...>
INSERT INTO some_table (x, y) VALUES (?, ?)
[...] ((1, 1), (2, 4))
<sqlalchemy.engine.cursor.CursorResult object at 0x...>
COMMIT

Above, we emitted two SQL statements that are generally transactional, a “CREATE TABLE” statement [1] and an “INSERT” statement that’s parameterized (the parameterization syntax above is discussed a few sections below in Sending Multiple Parameters). As we want the work we’ve done to be committed within our block, we invoke the Connection.commit() method which commits the transaction. After we call this method inside the block, we can continue to run more SQL statements and if we choose we may call Connection.commit() again for subsequent statements. SQLAlchemy refers to this style as commit as you go.
There is also another style of committing data, which is that we can declare our “connect” block to be a transaction block up front. For this mode of operation, we use the Engine.begin() method to acquire the connection, rather than the Engine.connect() method. This method will both manage the scope of the Connection and also enclose everything inside of a transaction with COMMIT at the end, assuming a successful block, or ROLLBACK in case of exception raise. This style may be referred towards as begin once:


with engine.begin() as conn:
    conn.execute(
        text("INSERT INTO some_table (x, y) VALUES (:x, :y)"),
        [{"x": 6, "y": 8}, {"x": 9, "y": 10}]
    )

BEGIN (implicit)
INSERT INTO some_table (x, y) VALUES (?, ?)
[...] ((6, 8), (9, 10))
<sqlalchemy.engine.cursor.CursorResult object at 0x...>
COMMIT

“Begin once” style is often preferred as it is more succinct and indicates the intention of the entire block up front. However, within this tutorial we will normally use “commit as you go” style as it is more flexible for demonstration purposes.

DDL refers to the subset of SQL that instructs the database to create, modify, or remove schema-level constructs such as tables. DDL such as “CREATE TABLE” is recommended to be within a transaction block that ends with COMMIT, as many databases uses transactional DDL such that the schema changes don’t take place until the transaction is committed. However, as we’ll see later, we usually let SQLAlchemy run DDL sequences for us as part of a higher level operation where we don’t generally need to worry about the COMMIT.

Basics of Statement Execution¶
We have seen a few examples that run SQL statements against a database, making use of a method called Connection.execute(), in conjunction with an object called text(), and returning an object called Result. In this section we’ll illustrate more closely the mechanics and interactions of these components.


with engine.connect() as conn:
    result = conn.execute(text("SELECT x, y FROM some_table"))
    for row in result:
        print(f"x: {row.x}  y: {row.y}")


BEGIN (implicit)
SELECT x, y FROM some_table
[...] ()
x: 1  y: 1
x: 2  y: 4
x: 6  y: 8
x: 9  y: 10
ROLLBACK

Fetching Rows
We’ll first illustrate the Result object more closely by making use of the rows we’ve inserted previously, running a textual SELECT statement on the table we’ve created:

Result has lots of methods for fetching and transforming rows, such as the Result.all() method illustrated previously, which returns a list of all Row objects. It also implements the Python iterator interface so that we can iterate over the collection of Row objects directly.
The Row objects themselves are intended to act like Python named tuples. Below we illustrate a variety of ways to access rows.
* Tuple Assignment - This is the most Python-idiomatic style, which is to assign variables to each row positionally as they are received: result = conn.execute(text("select x, y from some_table"))
* 
* for x, y in result:
*     # ...   
* Integer Index - Tuples are Python sequences, so regular integer access is available too: result = conn.execute(text("select x, y from some_table"))
* 
*   for row in result:
*       x = row[0]   
* Attribute Name - As these are Python named tuples, the tuples have dynamic attribute names matching the names of each column. These names are normally the names that the SQL statement assigns to the columns in each row. While they are usually fairly predictable and can also be controlled by labels, in less defined cases they may be subject to database-specific behaviors: result = conn.execute(text("select x, y from some_table"))
* 
* for row in result:
*     y = row.y
* 
*     # illustrate use with Python f-strings
*     print(f"Row: {row.x} {y}")   
* Mapping Access - To receive rows as Python mapping objects, which is essentially a read-only version of Python’s interface to the common dict object, the Result may be transformed into a MappingResult object using the Result.mappings() modifier; this is a result object that yields dictionary-like RowMapping objects rather than Row objects: result = conn.execute(text("select x, y from some_table"))
* 
* for dict_row in result.mappings():
*     x = dict_row['x']
*     y = dict_row['y']  

