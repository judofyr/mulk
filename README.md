# The Mulk Data Model

The **Mul**ti-**k**ey data model is an approach for storing data.
This is sort-of a key-value data model combined with a graph.
The intention of this is to be used as a basis for a database which can have very flexible needs.

## Introduction

### Hierarchical key space

Every value in Mulk is represented by its *primary key* which has an hierarchical interpretation.
The keys are _vectors_ (e.g. consists of multiple items) and you logically group your data by using the same prefix.

**Example:** The following is typical example of modeling employees and departments.

* (department, 1, name) -> "Sales"
* (department, 1, location) -> "Sydney"
* (department, 2, name) -> "Engineering"
* (department, 2, location) -> "Berlin"
* (employee, 1, name) -> "Bob"
* (employee, 1, department_id) -> 1
* (employee, 2, name) -> "Alice"
* (employee, 2, department_id) -> 2

**In comparison:**

* Plain key-value stores (e.g. Redis, FoundationDB) often uses a similar structure, but hierarchy is  not native to the data model.
  Instead the "physical" key (used by the database) has to be a string and then you encode your "logical" key into a string.
* In the relational model (e.g. SQL) you have a native concept of "relation" and "attribute" (or "table" and "columns" as it's known in SQL).
  These are not special in any way in Mulk, but you can accomplish the same thing by using the key space.

In the example above we use the hierarchical property to represent "columns" in a SQL database, but we are not actually limited to that.
We can do something which you can't do at all in an SQL database;
we can structure the data in a purely hierarchical form:

* (department, 1, name) -> "Sales"
* (department, 1, location) -> "Sydney"
* (department, 1, employee, 1, name) -> "Bob"
* (department, 2, name) -> "Engineering"
* (department, 2, location) -> "Berlin"
* (department, 2, employee, 2, name) -> "Alice"

### Multi-key support

Hierarchical data models are nothing new, and their disadvantages are well-known:
You don't _always_ want the hierarchical structure.
Sometimes you're interested in employees regardless of which department their in.
The relational model solves this by saying you should _never_ have direct hierarchical structure, but instead construct it (by joining) when you need it.
In Mulk we take a different approach:
You can have your hierarchical structure **and** the flat structure at the same time. 

You can define a "link" at a given key.
Instead of storing a _value_ you store another key in the key space.
For instance, we could define `(department, 1, employee, 1)` to link to `(employee, 1)`:

* (department, 1, name) -> "Sales"
* (department, 1, location) -> "Sydney"
* (department, 2, name) -> "Engineering"
* (department, 2, location) -> "Berlin"
* (employee, 1, name) -> "Bob"
* (employee, 2, name) -> "Alice"
* (department, 1, employee, 1) => (employee, 1)
* (department, 2, employee, 2) => (employee, 2)

The critical part is that these links are _transparent_.
You never actually need to think about that these are links.
E.g. `(department, 1, employee, 1, name)` will contain `"Bob"`,
the exact same data as what is stored in `(employee, 1, name)`.
You're building up your key space as a graph.

Links can be recursive, and links can also point to "nothing".
A database using the Mulk data model might want to add support for extra constraints (e.g. "only allow links to other keys which exists" or "disallow deletes to keys which are linked to elsewhere"),
but these are separate from the data model itself.

### Multi-key for indexes

Imagine that you want to implement search for all the employees and department names/locations.
Using the multi-key support you can create a new part of the key space that links directly into the data:

* (search, "Alice") => (employee, 2, name)
* (search, "Berlin") => (department, 2, location)
* (search, "Bob") => (employee, 1, name)
* (search, "Engineering") => (department, 2, name)
* (search, "Sales") => (department, 1, name)
* (search, "Sydney") => (department, 1, location)

Notice how there's not a separate concept of "index" in Mulk.
Instead you just use the key space to link up your data.
In your database you might want to configure this part of the key space specially (notably: you want to ensure you can do range queries), 

## Advantages

### Hierarchical structures makes sense as well!

The whole premise of the relational model is that you shouldn't be "hiding" data in a hierarchical manner.
Instead you should make every piece of information available at the top-level.
For many cases this is very true:
Just because something _can_ be hierarchical doesn't mean that it's _intrinsically_ hierarchical.

However, there are such things as intrinsically hierarchical data.
This can be empirically proved by the popularity of document-based databases (e.g. MongoDB),
but let's highlight some use cases:

* Sometimes you just need a list of values.
  This is typically very clunky in relational model.
  To handle it "correctly" you should create a new table and use a join key.
  However, now this data is different from your other data:
  Some of the data are "columns" on the main table, and other type of data needs to be joined in.
* Once you get into the business of _scaling_ your database you need to look into sharding.
  Sharding is _all_ about hierarchy: You pick a dimension to shard on, and you want to make sure that every query only accesses data inside a single shard.
  Mulk allows you to put your sharding dimension directly inside your key and it becomes evident which shard you'll access.

### Transparent performance characteristics

SQL databases have a quite powerful feature:
When designing your data model you shouldn't have to worry about _how_ you're going to access your data as you can create separate _indexes_ later on.

This is however not _actually_ true in practice.
There's different ways of representing the "same" data model and your choice will have performance impact later on.
The simplest example is maybe a _nullable field_.
You can model this either by having nullable column on the main table, or you can create a separate table where the field is a non-nullable column and can be joined into the main table.
These have widely different performance characteristics regardless of how many indexes you add.

Mulk has a different philosophy, slightly inspired by NoSQL databases:
We want you to think about how the data is accessed and we want you to be aware of how it's _physically_ structured.
On a day-to-day basis you may use a query language which hides details, but at a fundamental level we believe that the _only_ way to build performant and scalable databases is to be aware of what's going on behind the scenes.
We _want_ you to be able to poke behind and understand how your data is being stored.

We also want our data model to be flexible enough so that we can use different data structures behind the scenes.
A database could implement the Mulk data model such that everything stored under `(data, ?)` is stored as a literal hash map (instead of e.g. a B-tree).
This works naturally in this data model.
The only restriction is that you wouldn't be able to do range-queries _inside_ this part of a key space, but other than that it would work great.
You could even have nested values _under_ this hash map as well.
A SQL database _could_ provide a functionality like `CREATE HASHMAP TABLE data(key TEXT, value TEXT)`
so that `SELECT value FROM data WHERE key = ?` would be a hash map lookup, but it would behave like a very foreign object compared to your rest of your database.

### … over FoundationDB

[FoundationDB](https://www.foundationdb.org/) encourages storing data in a similar way as described here, but they have a much simpler data model.
Their data model is based around **ordered key-values** where both keys and values are strings (or, technically _bytes_).

In the example above (with a search index) you might end up storing the following data:

* SET("department:1:name", "Sales")
* SET("department:1:location", "Sydney")
* SET("department:2:name", "Engineering")
* SET("department:2:location", "Berlin")
* SET("employee:1:name", "Bob")
* SET("employee:2:name", "Alice")
* SET("search:Alice", "employee:2:name")
* SET("search:Berlin", "department:2:location")
* SET("search:Bob", "employee:1:name")
* SET("search:Engineering", "department:2:name")
* SET("search:Sales", "department:1:name")
* SET("search:Sydney", "department:1:location")

This is an extremely simple data model, but it has some disadvantages though:

* The _whole_ FoundationDB database has to be ordered, even where you don't need it.
  There's no real value in being able to iterate the departments in order of their internal IDs ("1", "2").
  However, since all the keys are just _bytes_ FoundationDB has no way of knowing this.
* FoundationDB is highly scalable and will automatically shard your data into different servers.
  There's nothing in FoundationDB which links together "department:1:name" and "department:1:location" and there is chance for it to choose to store them in different places.
  This is suboptimal since they are practically speaking always fetched together.
* You could store `SET("department:1:employees:1", "employee:1")`, but if you're interested in "the name of all employees in department X" you'll have to.
* In practice, you always end up abstracting away the actual keys.
  You never deal with keys directly, but you use a library which turns your "logical" key into a "physical" key.
  This has a disadvantage because all tooling must then be aware of _how_ keys are encoded.
  If you're connecting to the same database from multiple languages you need to make sure they interpret keys in the same way.
  Any tooling won't be able to show you your data without also knowing 

In a sense, Mulk takes many of the concepts behind how you _use_ FoundationDB, but makes them native to the data model.
We believe that a database which has this built-in can take advantage of this in multiple ways:

* **Functionality:** Since the key space is more structured it's easier to add native features such as filtering ("find departments with name=X [by scanning]"), formatting ("give me this as a JSON object"), joining ("give me department with all employee names").
* **Debugging**: Your key space is _actually_ designed by a human and not a binary representation picked by a developer.
  Any tooling around it (clients, monitoring) will be able to present your actual data model.
* **Typed key space:** It's possible for the database to have a strictly typed key space.
* **Performance:** A user could configure which parts of the key space should be ordered/un-ordered and/or how data should be (preferred to be) stored together.
  You could maybe even customize the indexing strategy (B-tree vs LSM?).
