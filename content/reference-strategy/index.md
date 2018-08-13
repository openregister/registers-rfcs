---
rfc: 0005
start_date: 2018-03-29
decision_date: 2018-08-13
pr: openregister/registers-rfcs#11
status: approved
---

# Reference strategy

## Summary

This RFC proposes CURIEs as the only way to reference between records and from a record
to a register (set of records). It can be seen as the natural consequence of
[RFC 0002: URI Consolidation](https://github.com/openregister/registers-rfcs/blob/master/content/uri-consolidation/index.md).

Note that other types of references (e.g. subset of records) are out of scope.
See the discussion [Group
References](https://github.com/openregister/registers-rfcs/issues/4) for more
on this.

## Motivation

Currently, there are two mechanisms for referencing: a foreign key and a CURIE.

### Foreign Key

A foreign key is a regular string that _happens_ to be an identifier in another
register (including itself). You have to deduce by the name of the field that
the values are in fact identifiers (primary keys) for records in a Register of
the same name. To get the URI of the referenced resource you have to:

1. Given a field with name `x`, where the record for `x` in the Field register
   has the `register` field populated with value `country`
1. And a value `GB`
1. Compose a register subdomain: `country.register.gov.uk`
1. Compose a record set URI: `https://country.register.gov.uk/records/`
1. Concatenate the URI with the value: `https://country.register.gov.uk/records/GB`

There is a case that can be seen as a special case where a foreign key refers
to a register register record. Then the meaning of the relation can be seen as
“everything in the register described by the linked record”.

1. Given a fieldname `register`
1. And a value `country`
1. Compose a register subdomain: `register.register.gov.uk`
1. Compose a record set URI: `https://register.register.gov.uk/records/`
1. Concatenate the URI with the value: `https://register.register.gov.uk/records/country`
1. Derive the record set URI: `https://country.register.gov.uk/records/`

The last step is not specified anywhere so it is an internal convention more
than anything a regular user might know. It is defined here in order to mirror
the capabilities for CURIEs.

#### Benefits

* Potentially familiar to people familiar with the Relational Data Model.
* Hints a single type of thing for any value in the field.

#### Limitations

* To know a field contains a foreign key, you need to look up the `register`
  field in the field register. If it is informed, the original field is a “key”
  of type defined in the `datatype` field.
* Only able to reference one register per field.
* You need a field for each register linked to even if they have the same type
  of relationship with the record (e.g. owned by, managed by).
* Linking a full set of records is not possible. Although there is a
  convention that could be standarised previous section.


### CURIE

A [CURIE](https://www.w3.org/TR/curie/) is a Compact URI so according to its
definition for a given CURIE should be able to be transformed into its
expanded URI form. The current use of CURIEs lacks any mechanism to resolve the
prefix into its equivalent URL which means the expansion mechanism relies on
the assumption that the prefix will be a known Register identifier, so to
expand it you have to:

1. Given a CURIE `country:GB`
1. Take its prefix: `country`
1. Compose a register subdomain: `country.register.gov.uk`
1. Compose a record URI: `https://country.register.gov.uk/records/`
1. Concatenate the URI with the CURIE value: `https://country.register.gov.uk/records/GB`

In the case of a CURIE to the full set of records you don't need the last
step:

1. Given a CURIE `country:`
1. Take its prefix: `country`
1. Compose a register subdomain: `country.register.gov.uk`
1. Compose a record URI: `https://country.register.gov.uk/records/`


#### Benefits

* Allows a field to reference a Record in any known register.
* A data consumer can know the field contains references by the datatype of the
  field.
* It's built on a existing open standard. Technically it is a note and not a
  recommendation but it is used in many standards, including RDF.

#### Limitations

* A consumer has to know the assumptions that lead from a prefix to a URI. And
  know it's not the standard way to work with CURIEs.
* It's not possible to infer a type constraint for any new value.


## Explanation

Although Registers could be built on any of the exposed solutions, a CURIE has
the edge in flexibility and builds on top of existing open standards for
linking resources.

### Expanding a CURIE

The current solution for expanding CURIEs relies on a few assumptions. In
order to remove this RFC proposes providing a **Context**, in a similar way
[JSON-LD](https://json-ld.org/) does it.

The definition of the context will happen at the Register Environment level,
this way it can be guaranteed that a single context will work for any Register
part of the Environment.

A Context is a dictionary of prefixes and URIs, in JSON it would look like:

```json
{
  "country": "https://country.register.gov.uk/records/",
  "lae": "https://local-authority-eng.register.gov.uk/records/",
  "example": "https://example.org/"
}
```

With this context, the following expansions are expected to be true:

* `country:GB` -> `https://country.register.gov.uk/records/GB`
* `country:ZZ` -> `https://country.register.gov.uk/records/ZZ`
* `lae:BIR` -> `https://local-authority-eng.register.gov.uk/records/BIR`
* `example:32` -> `https://example.org/32`

Note that currently the context is managed partially by the Register register.
Further work is expected to evolve it to match this definition of context.

## Naming conventions

Foreign keys have been using the same field definition as the primary key in
the targeted register. This means that a sigle entry in the field register
defines a field used in one register as the primary key and in another
register as a foreign key. This convention derives from another one where the
primary key field is always the same as the register identifier. E.g. the
register with id `country`, has a field `country` that acts as the primary key
for that register. Another register with a field `country` will use the field
as a foreign key.

With CURIEs this assumption weakens. A future RFC will define the new
convention for naming fields of type CURIE although this will be _informative_
not _normative_. The current thinking is to move away from using names and
start using relationships. For example, given a register with a link to the
organisation that manages the resource, it would use something like
`managed-by` instead of `manager` or `organisation`.

A future RFC will formalise the way to explicitly identify the primary key for
a register so the convention of using the register identifier will be
addressed there.
