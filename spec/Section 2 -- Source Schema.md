# Source Schema

A _source schema_ is a GraphQL schema that is part of a larger _composite
schema_. Source schemas use directives to express intent and requirements for
the composition process as well as to describe runtime behavior. The following
chapters describe the directives that are used to annotate a source schema.

## @lookup

```graphql
directive @lookup on FIELD_DEFINITION
```

The `@lookup` directive is used within a _source schema_ to specify output
fields that can be used by the _distributed GraphQL executor_ to resolve an
entity by a stable key.

The stable key is defined by the arguments of the field. Each lookup argument
must match a field on the return type of the lookup field.

Source schemas can provide multiple lookup fields for the same entity to resolve
the entity by different keys.

In this example, the source schema specifies that the `Product` entity can be
resolved with the `productById` field or the `productByName` field. Both lookup
fields are able to resolve the `Product` entity but do so with different keys.

```graphql example
type Query {
  version: Int # NOT a lookup field.
  productById(id: ID!): Product @lookup
  productByName(name: String!): Product @lookup
}

type Product {
  id: ID!
  name: String!
}
```

Lookup fields may return object, interface, or union types. In case a lookup
field returns an abstract type (interface type or union type), all possible
object types of the abstract return type are considered entities, and each must
have fields that correspond to every argument of the lookup field.

```graphql example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

type Clothing {
  id: ID!
  categoryId: Int
  name: String
  size: String
  price: Float
}
```

The following example shows an invalid lookup field because the `Clothing` type,
which is one of the possible object types of the abstract return type, does not
define all the fields required by the lookup field’s arguments.

```graphql counter-example
type Query {
  product(id: ID!, categoryId: Int): Product @lookup
}

union Product = Electronics | Clothing

type Electronics {
  id: ID!
  categoryId: Int
  name: String
  brand: String
  price: Float
}

# Clothing does not have a field that corresponds
# with the lookup field's argument signature.
type Clothing {
  id: ID!
  name: String
  size: String
  price: Float
}
```

Lookup fields must be accessible from the `Query` type. If a lookup field is not
defined directly on the `Query` type, it must be reachable by following a chain
of fields — starting from the `Query` root type — where none of the intermediate
fields have arguments. This ensures that lookup fields are accessible to the
executor.

```graphql example
type Query {
  lookups: Lookups!
}

type Lookups {
  productById(id: ID!): Product @lookup
}

type Product {
  id: ID!
}
```

## @internal

```graphql
directive @internal on OBJECT | FIELD_DEFINITION
```

The `@internal` directive is used in combination with lookup fields and allows
you to declare internal types and fields. Internal types and fields do not
appear in the final client-facing composite schema and do not participate in the
standard schema-merging process. This allows a source schema to define lookup
fields for resolving entities that should not be accessible through the
client-facing composite schema.

```graphql example
# Source Schema
type Query {
  productById(id: ID!): Product
  productBySku(sku: ID!): Product @internal
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Since internal types and fields do not participate in the standard
schema-merging process they do not collide with similar named fields or types on
other source schemas.

```graphql example
# Source Schema A
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field is internal and does not follow any field merging rules.
  productBySku(sku: ID!): Product @internal
}

# Source Schema B
type Query {
  productById(id: ID!): Product
  productBySku(sku: ID!, name: String!): Product @internal
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Internal fields can only be used by the distributed GraphQL executor as lookup
fields for entity resolution.

```graphql example
# Source Schema A
type Query {
  productById(id: ID!): Product @lookup
  lookups: InternalLookups! @internal
}

# all lookups within this internal type are hidden from the public API
# but can be used for entity resolution.
type InternalLookups @internal {
  productBySku(sku: ID!): Product @lookup
}

# Composite Schema
type Query {
  productById(id: ID!): Product
}
```

Since internal fields are not part of the standard schema-merging process, they
cannot be used as key fields or in requirements. This is because there is no
semantic equivalence of the field or type to another source schema.

```graphql counter-example
type Query {
  productById(id: ID!): Product @lookup
}

type Product {
  id: ID! @internal
}
```

In contrast to `@inaccessible`, the effect of `@internal` is local to its source
schema.

```graphql example
# Source Schema A
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field is internal and does not follow any field merging rules.
  productBySku(sku: ID!): Product @internal
}

# Source Schema B
type Query {
  # this field follows the standard field merging rules
  productById(id: ID!): Product

  # this field follows the standard field merging rules
  productBySku(sku: Int!): Product
}

# Composite Schema
type Product {
  productById(id: ID!): Product
  productBySku(sku: Int!): Product
}
```

## @inaccessible

```graphql
directive @inaccessible on FIELD_DEFINITION | OBJECT | INTERFACE | UNION | ARGUMENT_DEFINITION | SCALAR | ENUM | ENUM_VALUE | INPUT_OBJECT | INPUT_FIELD_DEFINITION
```

The `@inaccessible` directive is used to prevent specific type system members
from being accessible through the client-facing _composite schema_, even if they
are accessible in the underlying source schemas.

This directive is useful for restricting access to type system members that are
either irrelevant to the client-facing composite schema or sensitive in nature,
such as internal identifiers or fields intended only for backend use.

In the following example, the key field `sku` is inaccessible from the composite
schema. However, type system members marked as `@inaccessible` can still be used
by the distributed executor to fulfill requirements.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  note: String
}

type Query {
  productById(id: ID!): Product
  productBySku(sku: String!): Product @inaccessible
}
```

In contrast to the `@internal` directive, `@inaccessible` hides type system
members from the composite schema even if other source schemas on the same type
system member have no `@inaccessible` directive.

```graphql example
# Source Schema A
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String! @inaccessible
  note: String
}

# Source Schema B
type Product @key(fields: "sku") {
  sku: String!
  price: Float!
}

# Composite Schema
type Product {
  id: ID!
  note: String
  price: Float!
}
```

## @is

```graphql
directive @is(field: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@is` directive is utilized on lookup fields to describe how the arguments
can be mapped from the entity type that the lookup field resolves. The mapping
establishes semantic equivalence between disparate type system members across
source schemas and is used in cases where an argument does not directly align
with a field on the entity type.

In the following example, the directive specifies that the `id` argument on the
field `Query.personById` and the field `Person.id` on the return type of the
field are semantically the same.

Note: In cases where the lookup argument name aligns with the field name on the
return type, the `@is` directive can be omitted.

```graphql example
type Query {
  personById(productId: ID! @is(field: "id")): Person @lookup
}
```

The `@is` directive also allows referring to nested fields relative to `Person`.

```graphql example
type Query {
  personByAddressId(id: ID! @is(field: "address.id")): Person
}
```

The `@is` directive can be applied to multiple arguments within the same lookup
field, allowing each argument to be mapped individually to fields on the return
type.

```graphql example
type Query {
  personByAddressId(
    id: ID! @is(field: "address.id")
    kind: PersonKind @is(field: "kind")
  ): Person
}
```

The `@is` directive can also be used in combination with `@oneOf` to specify a
single lookup field that can resolve entities by multiple keys.

```graphql example
type Query {
  person(
    by: PersonByInput
      @is(field: "{ id } | { addressId: address.id } | { name }")
  ): Person
}

input PersonByInput @oneOf {
  id: ID
  addressId: ID
  name: String
}
```

**Arguments:**

- `field`: Represents a selection path map syntax.

## @require

```graphql
directive @require(field: FieldSelectionMap!) on ARGUMENT_DEFINITION
```

The `@require` directive is used to express data requirements with other source
schemas. Arguments annotated with the `@require` directive are removed from the
_composite schema_ and the value for these will be resolved by the _distributed
executor_.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    size: Int! @require(field: "dimension.size")
    weight: Int! @require(field: "dimension.weight")
  ): DeliveryEstimates
}
```

The above example would translate to the following in the _composite schema_.

```graphql example
type Product {
  id: ID!
  delivery(zip: String!): DeliveryEstimates
}
```

This can also be done by using input types. The selection path map specifies
which data is required and needs to be resolved from other source schemas. If
the input type is only used to express requirements it is removed from the
composite schema.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput!
      @require(field: "{ size: dimension.size, weight: dimension.weight }")
  ): DeliveryEstimates
}
```

If the input types do not match the output type structure the selection map
syntax can be used to specify how requirements translate to the input object.

```graphql example
type Product {
  id: ID!
  delivery(
    zip: String!
    dimension: ProductDimensionInput!
      @require(
        field: "{ productSize: dimension.size, productWeight: dimension.weight }"
      )
  ): DeliveryEstimates
}

type ProductDimension {
  size: Int!
  weight: Int!
}

input ProductDimensionInput {
  productSize: Int!
  productWeight: Int!
}
```

**Arguments:**

- `field`: Represents a selection path map syntax.

## @key

```graphql
directive @key(fields: FieldSelectionSet!) repeatable on OBJECT | INTERFACE
```

The `@key` directive is used to designate an entity's unique key, which
identifies how to uniquely reference an instance of an entity across different
source schemas.

```graphql example
type Product @key(fields: "id") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

Each occurrence of the `@key` directive on an object or interface type specifies
one distinct unique key for that entity. Keys allow the distributed GraphQL
executor to distinguish between different entities of the same type.

```graphql example
type Product @key(fields: "id") @key(fields: "sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

While multiple keys define separate ways to reference the same entity based on
different sets of fields, a composite key allows for uniquely identifying an
entity by using a combination of multiple fields.

```graphql example
type Product @key(fields: "id sku") {
  id: ID!
  sku: String!
  name: String!
  price: Float!
}
```

The directive is applicable to both OBJECT and INTERFACE types. This allows
entities that implement an interface to inherit the key(s) defined at the
interface level, ensuring consistent identification across different
implementations of that interface.

By applying the `@key` directive all referenced fields become sharable even if
the fields are not explicitly marked with `@shareable`.

```graphql example
# source schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# source schema B
type Product @key(fields: "id") {
  id: ID!
  name: String!
}
```

Fields must be explicitly marked as a key or annotated with the `@shareable`
directive to allow multiple source schemas to define them, ensuring that the
decision to serve a field from more than one source schema is intentional and
coordinated.

```graphql counter-example
# source schema A
type Product @key(fields: "id") {
  id: ID!
  price: Float!
}

# source schema B
type Product {
  id: ID!
  name: String!
}
```

**Arguments:**

- `fields`: Represents a field selection set syntax.

## @shareable

```graphql
directive @shareable repeatable on OBJECT | FIELD_DEFINITION
```

By default, only a single source schema is allowed to contribute a particular
field to an object type. This prevents source schemas from inadvertently
defining similarly named fields that are not semantically equivalent.

```graphql counter-example
# Schema A
type Product {
  name: String!
  description: String!
}

# Schema B
type Product {
  name: String!
  variation: ProductVariation!
}
```

Fields must be explicitly marked as `@shareable` to allow multiple source
schemas to define them, ensuring that the decision to serve a field from more
than one source schema is intentional and coordinated.

```graphql example
# Schema A
type Product {
  name: String! @shareable
  description: String!
}

# Schema B
type Product {
  name: String! @shareable
  variation: ProductVariation!
}
```

If multiple source schemas define the same sharable field, they are assumed to
be semantically equivalent, and the executor is free to choose between them as
it sees fit.

The `@shareable` directive can also be applied at the object-type level, having
the same effect as if `@shareable` were applied to each field of the type.

```graphql example
# Schema A
type Product @shareable {
  name: String!
  description: String!
}

# Schema B
type Product {
  name: String! @shareable
  variation: ProductVariation!
}
```

Key fields of an object-type are considered shareable by default and do not need
to be explicitly marked with `@shareable`.

```graphql example
# Schema A
type Product @key(fields: "id") {
  id: ID!
  name: String! @shareable
  description: String!
}

# Schema B
type Product @key(fields: "id") {
  id: ID!
  name: String! @shareable
  variation: ProductVariation!
}
```

## @provides

```graphql
directive @provides(fields: FieldSelectionSet!) on FIELD_DEFINITION
```

The `@provides` directive indicates that a field can provide certain subfields
of its return type from the same source schema, without requiring an additional
resolution step elsewhere.

```graphql example
type Review {
  id: ID!
  body: String!
  author: User @provides(fields: "email")
}

type User @key(fields: "id") {
  id: ID!
  email: String! @external
  name: String!
}

type Query {
  reviews: [Review!]
  users: [User!]
}
```

When a field annotated with `@provides` returns an object, interface or union
type that may also be contributed by other source schemas, this directive
declares which of that type’s subfields the current source schema can resolve
directly.

```graphql example
{
  reviews {
    body
    author {
      name
      email
    }
  }
}
```

If a client tries to fetch the same subfield (`User.email`) through a different
path (e.g., users query field), the source schema will not be able to resolve it
and will throw an error.

```graphql counter-example
{
  users {
    # The source schema does NOT provide email in this context,
    # and this field will fail at runtime.
    email
  }
}
```

The `@provides` directive may reference multiple fields or nested fields:

```graphql example
type Review {
  id: ID!
  product: Product @provides(fields: "sku variation { size }")
}

type Product @key(fields: "sku variation { id }") {
  sku: String! @external
  variation: ProductVariation!
  name: String!
}

type ProductVariation {
  id: String!
  size: String! @external
}
```

When a field annotated with the provides directive has an abstract return type
the fields syntax can leverage inline fragments to express fields that can be
resolved locally.

```graphql example
type Review {
  id: ID!
  # The @provides directive tells us that this source schema can supply different
  # fields depending on which concrete type of Product is returned.
  product: Product
    @provides(
      fields: """
      ... on Book { author }
      ... on Clothing { size }
      """
    )
}

interface Product @key(fields: "id") {
  id: ID!
}

type Book implements Product {
  id: ID!
  title: String!
  author: String! @external
}

type Clothing implements Product {
  id: ID!
  name: String!
  size: String! @external
}

type Query {
  reviews: [Review!]!
}
```

**Arguments:**

- `fields`: Represents a field selection set syntax describing the subfields of
  the returned type that can be provided by the current source schema.

## @external

```graphql
directive @external on FIELD_DEFINITION
```

The @external directive indicates that a field is recognized by the current
source schema but is not directly contributed (resolved) by it. Instead, this
schema references the field for specific composition purposes.

**Entity Keys**

When combined with one or more `@key` directives, an external field can serve as
an entity identifier (or part of a composite identifier).

```graphql example
type Query {
  productBySku(sku: String!): Product @lookup
  productByUpc(upc: String!): Product @lookup
}

type Product @key(fields: "sku") @key(fields: "upc") {
  sku: String! @external
  upc: String! @external
  name: String
}
```

**Field Resolution**

When another field in the same source schema uses `@provides` to declare that it
can resolve certain external fields in a single data-fetching step.

```graphql example
type Review {
  id: ID!
  text: String
  author: User @provides(fields: "email")
}

type User {
  id: ID!
  email: String! @external
}
```

When a field is marked `@external`, the composition process understands that the
field is provided by another source schema. The current source schema references
it only for entity identification (via `@key`) or for providing a field through
`@provides`. If no such usage exists, the presence of an `@external` field
produces a composition error.

## @override

```graphql
directive @override(from: String!) on FIELD_DEFINITION
```

The `@override` directive is used to migrate a field from one source schema to
another. When a field in the local schema is annotated with
`@override(from: "Catalog")`, it signals that the local schema overrides the
field previously contributed by the `Catalog` source schema. As a result, the
composite schema will source this field from the local schema, rather than from
the original source schema.

The following example shows how a field can be migrated from the `Catalog`
schema to the new `Payments` schema. By using `@override`, a field can be moved
to a new schema without requiring any change to the original `Catalog` schema.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}
```

Fields that are annotated can themselves be migrated.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float!
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}

# The new "Pricing" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Payments")
  tax: Float!
}
```

If the composition detects cyclic overrides it must throw a composition error.

```graphql example
# The original "Catalog" schema:
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Float! @override(from: "Pricing")
}

# The new "Payments" schema:
type Product @key(fields: "id") {
  id: ID! @external
  price: Float! @override(from: "Catalog")
  tax: Float!
}
```

**Arguments:**

- `from`: The name of the source schema that originally provided this field.
