---
sidebar_position: 2
title: The schema file
description: >-
  The schema file defines the target data model with a GraphQL dialect.
---

# The schema file

The file `schema.graphql` defines the target db schema via the TypeORM entity classes generated by [`squid-substrate-typegen(1)`](https://github.com/subsquid/squid/tree/master/substrate-typegen) in `src/model/generated`. The generated entity classes are in turn used by `squid-typeorm-migration(1)` to generate the matching database migrations. The schema file uses a declarative GraphQL dialect enriched with custom directives.  

The schema file also defines the GraphQL API of the [OpenReader](https://github.com/subsquid/squid/tree/master/openreader) server to present the data. This page only describes the database schema part with the OpenReader part described elsewhere.

## Entities

Entities are defined by root-level GraphQL types decorated with `@entity`. The entity names and the properties are expected to be camelCased and are converted into snake_cased database tables and columns. The primary key column is always mapped to the entity field of a special `ID` type. Non-nullable fields are marked with an exclamation mark (`!`) and are nullable otherwise. 

The following [scalar types](https://graphql.org/learn/schema/#scalar-types) are supported by the `schema.graphql` dialect:

- `Boolean` (mapped to `bool`)
- `BigInt` (mapped to `numeric`)
- `DateTime` (mapped to `timestamptz`)
- `Bytes` (mapped to `bytea`)
- `JSON` (mapped to `jsonb`)
- `String` (mapped to `text`)
- `Int` (mapped to `int4`)
- Enums (mapped to `text`)
- User-defined scalars (non-entity types). Such properties are mapped as `jsonb` columns.

**Example** 
```graphql
type Scalar @entity {
  id: ID!
  boolean: Boolean
  string: String
  enum: Enum
  bigint: BigInt
  dateTime: DateTime
  bytes: Bytes
  json: JSON,
  deep: DeepScalar
}
        
type DeepScalar {
  bigint: BigInt
  dateTime: DateTime
  bytes: Bytes
  boolean: Boolean
}
        
enum Enum {
  A B C
}
```

## Arrays

Entity fields can be an array of any scalar type and are mapped to the corresponding Postgres array types. The array elements may be defined as nullable or non-nullable.

**Example**

```graphql
type Lists @entity {
  intArray: [Int!]!
  enumArray: [Enum!]
  bigintArray: [BigInt!]
  datetimeArray: [DateTime!]
  bytesArray: [Bytes!]
  listOfListOfInt: [[Int]]
  listOfJsonObjects: [Foo!]
}
        
enum Enum {
  A B C D E F
}
        
type Foo {
  foo: Int
  bar: Int
}
```

## Entity relations and inverse lookups

A relation between two entities is always assumed to be unidirectional. The owning entity is mapped to a database table holding a foreign key reference to the related entity. The non-owning entity may define a property decorated `@derivedFrom` for an inverse lookup on the owning field. In particular, the "many" side of the one-to-many relations is always the owning side.

Note that `@derivedFrom` has no effect on the corresponding table DDL but rather adds a property decorated with TypeORM `@OneToOne` or `@OneToMany` to the generated entity classes (and the GraphQL API generated from `schema.graphql`). All the foreign key reference columns are automatically indexed. 

The following examples illustrate this concept.

**One-to-one relation**

```graphql
type Account @entity {
  "Account address"
  id: ID!
  balance: BigInt!
  user: User @derivedFrom(field: "account")
}

type User @entity {
  id: ID!
  account: Account!
  username: String!
  creation: DateTime!
}
```

The `User` entity references a single `Account` and owns the relation (that is, the database schema will define the column `user.account_id` referencing to the `account` table).  

**Many-to-one/One-to-many relations**

```graphql
type Account @entity {
  "Account address"
  id: ID!
  transfersTo: [Transfer!] @derivedFrom(field: "to")
  transfersFrom: [Transfer!] @derivedFrom(field: "from")
}

type Transfer @entity {
  id: ID!
  to: Account!
  from: Account!
  amount: BigInt! 
}

```

Here `Tranfer` defines two foreign key references and `Account` defines the corresponding inverse lookup properties.

**Many-to-many relations**

Many-to-many entity relations should be modelled as two one-to-many relations with an explicitly defined join table. 
Here is an example:

```graphql
# an explicit join table 
type TradeToken @entity {
  id: ID! # This is required, even if useless
  trade: Trade!
  token: Token! 
}

type Token @entity {
  id: ID!
  symbol: String!
  trades: [TradeToken!]! @derivedFrom(field: "token")    
}

type Trade @entity {
  id: ID!
  tokens: [TradeToken!]! @derivedFrom(field: "trade")
}
```

## Indexes and unique constraints

To add an index to a column, the corresponding entity field must be decorated with `@index`. It is crucial to index the entity fields for which one expects filtering and ordering at the API level.

**Example**
```graphql
type Transfer @entity {
  id: ID!
  
  to: Account!
  amount: BigInt! @index
  fee: BigInt! 
}
```

Multi-column indices can be defined on the entity level, with the additional `unique` constraint. 

**Example**
```graphql
type Foo @entity @index(fields: ["baz", "bar"], unique: true) {
  id: ID!
  bar: Int!
  baz: [Enum!]
```
 
Similar to `@index` a field marked with `@unique` will have an additional unique constraint. 

**Example**
```graphql
type Extrinsic @entity {
  id: ID!
  hash: String! @unique
}
```


## Typed JSON

It is possible to define explicit types for JSON fields. The generated entity classes and the GraphQL API will respect the type definition of the field, enforcing the data integrity.

**Example**
```graphql
type Entity @entity {
  a: A
}

type A {
  a: String
  b: B
  c: JSON
}

type B {
  a: A
  b: String
  e: Entity
}
```

## Union types

One can leverage union types supported both by [Typescript](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#union-types) and [GraphQL](https://graphql.org/learn/schema/#union-types).  The union operator for `schema.graphql` supports only non-entity types, including typed JSON types described above. JSON types, however, are allowed to reference an entity type.

**Example**
```graphql
type User @entity {
  id: ID!
  login: String!
}
        
type Farmer {
  user: User!
  crop: Int
}

type Degen {
  user: User!
  bag: String
}
        
union Owner = Farmer | Degen
        
type NFT @entity {
  name: String!
  owner: Owner!
}
```