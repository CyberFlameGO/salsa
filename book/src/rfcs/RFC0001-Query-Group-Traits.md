# Query group traits

## Metadata

* Author: nikomatsakis
* Date: 2019-01-15
* Introduced in: https://github.com/salsa-rs/salsa-rfcs/pull/1

## Motivation

- Support `dyn QueryGroup` for each query group trait as well as `impl QueryGroup`
  - `dyn QueryGroup` will be much more convenient, at the cost of runtime efficiency
- Don't require you to redeclare each query in the final database, just the query groups

## User's guide

### Declaring a query group

User's will declare query groups by decorating a trait with `salsa::query_group`:

```rust,ignore
#[salsa::query_group(MyGroupStorage)]
trait MyGroup {
  // Inputs are annotated with `#[salsa::input]`. For inputs, the final trait will include
  // a `set_my_input(&mut self, key: K1, value: V1)` method automatically added,
  // as well as possibly other mutation methods.
  #[salsa::input]
  fn my_input(&self, key: K1) -> V1;
  
  // "Derived" queries are just a getter.
  fn my_query(&self, key: K2) -> V2;
}
```

The `query_group` attribute is a procedural macro. It takes as
argument the name of the **storage struct** for the query group --
this is a struct, generated by the macro, which represents the query
group as a whole. It is attached to a trait definition which defines the
individual queries in the query group.

The macro generates three things that users interact with:

- the trait, here named `MyGroup`. This will be used when writing the definitions
  for the queries and other code that invokes them.
- the storage struct, here named `MyGroupStorage`. This will be used later when
  constructing the final database.
- query structs, named after each query but converted to camel-case
  and with the word query (e.g., `MyInputQuery` for `my_input`). These
  types are rarely needed, but are presently useful for things like
  invoking the GC. These types violate our rule that "things the user
  needs to name should be given names by the user", but we choose not
  to fully resolve this question in this RFC.
  
In addition, the macro generates a number of structs that users should
not have to be aware of. These are described in the "reference guide"
section.
  
#### Controlling query modes

Input queries, as described in the trait, are specified via the
`#[salsa::input]` attribute.

Derived queries can be customized by the following attributes,
attached to the getter method (e.g., `fn my_query(..)`):

- `#[salsa::invoke(foo::bar)]` specifies the path to the function to invoke
  when the query is called (default is `my_query`).
- `#[salsa::volatile]` specifies a "volatile" query, which is assumed to
  read untracked input and hence must be re-executed on every revision.
- `#[salsa::dependencies]` specifies a "dependencies-only" query, which is assumed to
  read untracked input and hence must be re-executed on every revision.

### Creating the database

Creating a salsa database works by using a `#[salsa::database(..)]`
attribute. The `..` content should be a list of paths leading to the
storage structs for each query group that the database will
implement. It is no longer necessary to list the individual
queries. In addition to the `salsa::database` query, the struct must
have access to a `salsa::Runtime` and implement the `salsa::Database`
trait. Hence the complete declaration looks roughly like so:

```rust,ignore
#[salsa::database(MyGroupStorage)]
struct MyDatabase {
  runtime: salsa::Runtime<MyDatabase>,
}

impl salsa::Database for MyDatabase {
  fn salsa_runtime(&self) -> salsa::Runtime<MyDatabase> {
    &self.runtime
  }
}  
```

This (procedural) macro generates various impls and types that cause
`MyDatabase` to implement all the traits for the query groups it
supports, and which customize the storage in the runtime to have all
the data needed. Users should not have to interact with these details,
and they are written out in the reference guide section.

## Reference guide

The goal here is not to give the *full* details of how to do the
lowering, but to describe the key concepts. Throughout the text, we
will refer to names (e.g., `MyGroup` or `MyGroupStorage`) that appear
in the example from the User's Guide -- this indicates that we use
whatever name the user provided.

### The `plumbing::QueryGroup` trait

The `QueryGroup` trait is a new trait added to the plumbing module. It
is implemented by the query group storage struct `MyGroupStorage`. Its
role is to link from that struct to the various bits of data that the
salsa runtime needs:

```rust,ignore
pub trait QueryGroup<DB: Database> {
    type GroupStorage;
    type GroupKey;
}
```

This trait is implemented by the **storage struct** (`MyGroupStorage`)
in our example. You can see there is a bit of confusing nameing going
on here -- what we call (for user's) the "storage struct" actually
does not wind up containing the true *storage* (that is, the hasmaps
and things salsa uses). Instead, it merely implements the `QueryGroup`
trait, which has associated types that lead us to structs we need:

- the **group storage** contains the hashmaps and things for all the queries in the group
- the **group key** is an enum with variants for each of the
  queries. It basically stores all the data needed to identify some
  particular *query value* from within the group -- that is, the name
  of the query, plus the keys used to invoke it.

As described further on, the `#[salsa::query_group]` macro is
responsible will generate an impl of this trait for the
`MyGroupStorage` struct, along with the group storage and group key
type definitions.

### The `plumbing::HasQueryGroup<G>` trait

The `HasQueryGroup<G>` struct a new trait added to the plumbing
module. It is implemented by the database struct `MyDatabase` for
every query group that `MyDatabase` supports. Its role is to offer
methods that move back and forth between the context of the *full
database* to the context of an *individual query group*:

```rust,ignore
pub trait HasQueryGroup<G>: Database
where
    G: QueryGroup<Self>,
{
    /// Access the group storage struct from the database.
    fn group_storage(db: &Self) -> &G::GroupStorage;

    /// "Upcast" a group key into a database key.
    fn database_key(group_key: G::GroupKey) -> Self::DatabaseKey;
}
```

Here the "database key" is an enum that contains variants for each
group. Its role is to take group key and puts it into the context of
the entire database.

### The `Query` trait

The query trait (pre-existing) is extended to include links to its
group, and methods to convert from the group storage to the query
storage, plus methods to convert from a query key up to the group key:

```rust,ignore
pub trait Query<DB: Database>: Debug + Default + Sized + 'static {
    /// Type that you you give as a parameter -- for queries with zero
    /// or more than one input, this will be a tuple.
    type Key: Clone + Debug + Hash + Eq;

    /// What value does the query return?
    type Value: Clone + Debug;

    /// Internal struct storing the values for the query.
    type Storage: plumbing::QueryStorageOps<DB, Self> + Send + Sync;

    /// Associate query group struct.
    type Group: plumbing::QueryGroup<
        DB,
        GroupStorage = Self::GroupStorage,
        GroupKey = Self::GroupKey,
    >;

    /// Generated struct that contains storage for all queries in a group.
    type GroupStorage;

    /// Type that identifies a particular query within the group + its key.
    type GroupKey;

    /// Extact storage for this query from the storage for its group.
    fn query_storage(group_storage: &Self::GroupStorage) -> &Self::Storage;

    /// Create group key for this query.
    fn group_key(key: Self::Key) -> Self::GroupKey;
}
```

### Converting to/from the context of the full database generically

Putting all the previous plumbing traits together, this means
that given:

- a database `DB` that implements `HasGroupStorage<G>`;
- a group struct `G` that implements `QueryGroup<DB>`; and,
- and a query struct `Q` that implements `Query<DB, Group = G>`

we can (generically) get the storage for the individual query
`Q` out from the database `db` via a two-step process:

```rust,ignore
let group_storage = HasGroupStorage::group_storage(db);
let query_storage = Query::query_storage(group_storage);
```

Similarly, we can convert from the key to an individual query
up to the "database key" in a two-step process:

```rust,ignore
let group_key = Query::group_key(key);
let db_key = HasGroupStorage::database_key(group_key);
```

### Lowering query groups

The role of the `#[salsa::query_group(MyGroupStorage)] trait MyGroup {
.. }` macro is primarily to generate the group storage struct and the
impl of `QueryGroup`.  That involves generating the following things:

- the query trait `MyGroup` itself, but with:
  - `salsa::foo` attributes stripped
  - `#[salsa::input]` methods expanded to include setters:
    - `fn set_my_input(&mut self, key: K1, value__: V1);`
    - `fn set_constant_my_input(&mut self, key: K1, value__: V1);`
- the query group storage struct `MyGroupStorage`
  - We also generate an impl of `QueryGroup<DB>` for `MyGroupStorage`,
    linking to the internal strorage struct and group key enum
- the individual query types
  - Ideally, we would use Rust hygiene to hide these struct, but as
    that is not currently possible they are given names based on the
    queries, but converted to camel-case (e.g., `MyInputQuery` and `MyQueryQuery`).
  - They implement the `salsa::Query` trait.
- the internal group storage struct
  - Ideally, we would use Rust hygiene to hide this struct, but as
    that is not currently possible it is entitled
    `MyGroupGroupStorage<DB>`. Note that it is generic with respect to
    the database `DB`. This is because the actual query storage
    requires sometimes storing database key's and hence we need to
    know the final database type.
  - It contains one field per query with a link to the storage information
    for that query:
    - `my_query: <MyQueryQuery as salsa::plumbing::Query<DB>>::Storage`
    - (the `MyQueryQuery` type is also generated, see the "individual query types" below)
  - The internal group storage struct offers a public, inherent method
    `for_each_query`:
    - `fn for_each_query(db: &DB, op: &mut dyn FnMut(...)`
    - this is invoked by the code geneated by `#[salsa::database]` when implementing the
      `for_each_query` method of the `plumbing::DatabaseOps` trait
- the group key
  - Again, ideally we would use hygiene to hide the name of this struct,
    but since we cannot, it is entitled `MyGroupGroupKey`
  - It is an enum which contains one variant per query with the value being the key:
    - `my_query(<MyQueryQuery as salsa::plumbing::Query<DB>>::Key)`
  - The group key enum offers a public, inherent method `maybe_changed_after`:
    - `fn maybe_changed_after<DB>(db: &DB, db_descriptor: &DB::DatabaseKey, revision: Revision)`
    - it is invoked when implementing `maybe_changed_after` for the database key

### Lowering database storage

The `#[salsa::database(MyGroup)]` attribute macro creates the links to the query groups.
It generates the following things:

- impl of `HasQueryGroup<MyGroup>` for `MyDatabase`
  - Naturally, there is one such impl for each query group.
- the database key enum
  - Ideally, we would use Rust hygiene to hide this enum, but currently
    it is called `__SalsaDatabaseKey`.
  - The database key is an enum with one variant per query group:
    - `MyGroupStorage(<MyGroupStorage as QueryGroup<MyDatabase>>::GroupKey)`
- the database storage struct
  - Ideally, we would use Rust hygiene to hide this enum, but currently
    it is called `__SalsaDatabaseStorage`.
  - The database storage struct contains one field per query group, storing
    its internal storage:
    - `my_group_storage: <MyGroupStorage as QueryGroup<MyDatabase>>::GroupStorage`
- impl of `plumbing::DatabaseStorageTypes` for `MyDatabase`
  - This is a plumbing trait that links to the database storage / database key types.
  - The `salsa::Runtime` uses it to determine what data to include. The query types
    use it to determine a database-key.
- impl of `plumbing::DatabaseOps` for `MyDatabase`
  - This contains a `for_each_query` method, which is implemented by invoking, in turn,
    the inherent methods defined on each query group storage struct.
- impl of `plumbing::DatabaseKey` for the database key enum
  - This contains a method `maybe_changed_after`. We implement this by
    matching to get a particular group key, and then invoking the
    inherent method on the group key struct.

## Alternatives

This proposal results from a fair amount of iteration. Compared to the
status quo, there is one primary downside. We also explain a few things here that
may not be obvious.

### Why include a group storage struct?

You might wonder why we need the `MyGroupStorage` struct at all. It is a touch of boilerplate,
but there are several advantages to it:

- You can't attach associated types to the trait itself. This is because the "type version"
  of the trait (`dyn MyGroup`) may not be available, since not all traits are dyn-capable.
- We try to keep to the principle that "any type that might be named
  externally from the macro is given its name by the user". In this
  case, the `[salsa::database]` attribute needed to name group storage
  structs.
  - In earlier versions, we tried to auto-generate these names, but
    this failed because sometimes users would want to `pub use` the
    query traits and hide their original paths.
  - (One exception to this principle today are the per-query structs.)
- We expect that we can use the `MyGroupStorage` to achieve more
  encapsulation in the future. While the struct must be public and
  named from the database, the *trait* (and query key/value types)
  actually does not have to be.

### Downside: Size of a database key

Database keys now wind up with two discriminants: one to identify the
group, and one to identify the query. That's a bit sad. This could be
overcome by using unsafe code: the idea would be that a group/database
key would be stored as the pair of an integer and a `union`. Each
group within a given database would be assigned a range of integer
values, and the unions would store the actual key values. We leave
such a change for future work.

## Future possibilities

Here are some ideas we might want to do later.

### No generics

We leave generic parameters on the query group trait etc for future work.

### Public / private

We'd like the ability to make more details from the query groups
private. This will require some tinkering.

### Inline query definitions

Instead of defining queries in separate functions, it might be nice to
have the option of defining query methods in the trait itself:

```rust,ignore
#[salsa::query_group(MyGroupStorage)]
trait MyGroup {
  #[salsa::input]
  fn my_input(&self, key: K1) -> V1;
  
  fn my_query(&self, key: K2) -> V2 {
      // define my-query right here!
  }
}
```

It's a bit tricky to figure out how to handle this, so that is left
for future work. Also, it would mean that the method body itself is
inside of a macro (the procedural macro) which can make IDE
integration harder.

### Non-query functions

It might be nice to be able to include functions in the trait that are
*not* queries, but rather helpers that compose queries. This should be
pretty easy, just need a suitable `#[salsa]` attribute.
