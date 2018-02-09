---
rfc:
start_date: 2017-11-17
status: draft
---

# Schema Evolution: Add field operation

## Summary

This RFC proposes a way to add fields to a Register, and defines the meaning
of this operation. Thus, it doesn’t attempt to cover other field operations
such as field subtraction or field transformation (e.g.  change the datatype
of a field).

Note that you can find the [glossary](#glossary) of concepts used throughout
this RFC at the end of it.

Note: The pseudocode in this RFC resembles [Elm](http://elm-lang.org) but it
is not intended to be valid.


## Motivation

Currently, the Registers Data Model has no way to evolve the data structure
described in the schema and used in data items. This means a Register has to
be designed upfront with the right fields forever given it is an append-only
log. We need a way to be able to evolve the schema definition and a way to
reflect such evolution on existing and new data.


## Explanation

The proposal is to allow adding new field definitions to the schema. As a
result new items added to the Register will be validated against the new
schema. To implement this behaviour this RFC defines the mechanism to [add a
field to the schema](#field-addition) and the mechanism to [add an item to the
log](#item-addition).

Note: The explanation tries to avoid being prescriptive in how the Register is
implemented and because of that there is no explicit reference to entries and
how they participate in either mechanism.


### <span id="field-addition">Field addition</span>

The addition of a field, [`Schema.add`](#schema-add-function), adds a
[field](#field) to the [schema](#schema).


##### Example: Add a field to the schema

```elm
s_0 = Schema [("x", Int)]
s_1 = Schema.add s_0 ("y", Int) -- s_1 == Schema [("x", Int), ("y", Int)]
```


### <span id="item-addition">Item addition</span>

The item addition operation, [`Items.add`](#items-add-function), adds the item
if [`Item.validate`](#item-validate-function) succeeds. This RFC doesn't
change its behaviour, the only thing to keep in mind is that the validation
function checks against the current schema.

The addition of an item steps are:

1. Validate the incoming item `a` against the current schema `s`: `Item.validate s a`.
    * If the item is invalid, reject the operation.
2. Canonicalise the item `a`: `a_c = Item.canonicalise a`.
3. Add item `a_c` to the Register.


### REST API Implications

The REST API has no changes given that it is a read-only API. From the **API
consumer** perspective, nothing changes but let me make it explicit here.

One cornerstone assumption in the Registers' API is that a consumer is
expected to handle graciously any Record/Item response that doesn't match
exactly their knowledge of the schema.

Scenario:

Give a consumer that knows about a schema such as `Schema [("x", Int),
("y", Int), ("z", String)]` where `x` is the primary field, such consumer
is expected to handle reponses from the API such as:

* **All known fields are present**

  Given `{"x": 1, "y": 2, "z": "foo"}`, keep `{"x": 1, "y": 2, "z": "foo"}`.

* **Some known fields are present**

  Given `{"x": 1, "y": 2}`, keep `{"x": 1, "y": 2}` and ignore the fact that
  `z` is missing.

* **Some unknown fields present**

  Given `{"x": 1, "y": 2, "z": "foo", "n": 42}`, keep `{"x": 1, "y": 2, "z": "foo"}`
  and ignore the fact that `n` is unknown. If appropriate, update the schema
  and handle it as the first case.


## <span id="glossary">Glossary</span>

### <span id="datatype">Datatype</span>

Let `DataType` be any of the datatypes defined in the
[specification](http://openregister.github.io/specification/#datatypes). For
brevity, the RFC will express it as a union of strings and numbers.

```elm
type DataType = String | Int
```

### <span id="field-id">Field ID</span>

Let `Id` be any valid field id, [field name in the
specification](http://openregister.github.io/specification/#fieldname-datatype).

```elm
type Id
```

### <span id="field">Field</span>

Let `Field` be a pair of `Id` and `DataType`. Although a Field has more data
than just name and type, for the purpose of this document, we can simplify its
definition.

```elm
type Field = (Id, DataType)
```

### <span id="schema">Schema</span>

Let `Schema` be the set of fields allowed in items. A schema has a required
primary field and optionally a set of fields (unique by `Id`). We will
consider the first field, the primary field.

```elm
type Schema = Schema (Set Field)
```

### <span id="item">Item</span>

Let `Item` be a data set conforming to the schema.

```elm
type Item = Item (Set (Id, value))
```

Where `value` is any valid value for the corresponding datatype defined in the
schema.

### <span id="schema-add-function">Schema `add` function</span>

The function `Schema.add` takes a schema and a field and returns the new
schema.

```elm
Schema.add : Schema -> Field -> Schema
```

If the field already exists the operation behaves like the identity function
so:

```elm
schema' = Schema.add schema field -- schema' == schema
```

### <span id="items-add-function">Items `add` function</span>

The function `Items.add` takes a list of items, a schema and an item and
returns a new list of items if the given item is valid. Otherwise it returns a
validation error.

```elm
Items.add : (Items, Schema) -> Item -> Result ItemError Items
```

### <span id="item-validate-function">Item `validate` function</span>

The function `Item.validate` takes a schema and an item and returns the same
item if it is valid. Otherwise it returns a validation error.

```elm
Item.validate : Schema -> Item -> Result ItemError Item
```

## <span id="rest-api">REST API</span>

[API documentation](https://registers-docs.cloudapps.digital/#api-documentation-for-registers).
