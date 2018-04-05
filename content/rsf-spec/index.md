---
rfc:
start_date: 2018-04-04
pr:
status: draft
---

# The Register Serialisation Format

## Summary

This RFC aims to collect in one place the current implementation of RSF so it
can be added to the specification and it can be evolved with new RFCs when
required.

The Register Serialisation Format, from now on RSF, is an event log describing
the evolution of the Register data and metadata.

## Motivation

TODO

## Explanation

### RSF Grammar

RSF is a positional line-based textual format separated by tabs. Each
line defines a command to apply to a Register state to obtain the next state.

This specification uses the Augmented Backus-Naur Form (ABNF) as defined by
[RFC5234](https://tools.ietf.org/html/rfc5234) and refined by
[RFC7405](https://tools.ietf.org/html/rfc7405). It assumes the following
definitions:

* RFC5234: `ALPHA` (letters), `CRLF` (carriage return, line feed), `DIGIT`
  (decimal digits), `HEXDIG` (hexadecimal digits) and `HTAB` (horizontal tab).
* Registers specification: [`CANONREP`][canon-rep] (canonical representation).
  Note that, in turn, it depends on [RFC8259](https://tools.ietf.org/html/rfc8259).

```abnf
log              = command *(CRLF command) [CRLF]
command          = add-item / append-entry / assert-root-hash

assert-root-hash = %s"assert-root-hash" HTAB hash

add-item         = %s"add-item" HTAB CANONREP

append-entry     = %s"append-entry" HTAB type HTAB key HTAB timestamp HTAB hash-list
type             = "user" / "system"
key              = alphanum / %x2D / %x5F
hash-list        = hash *(list-separator hash)
hash             = "sha-256:" 64(HEXDIG) ; sha-256
list-separator   = ";" ; hash list separator

alphanum         = ALPHA / DIGIT

;                timestamp
timestamp        = date "T" time
date             = century year DSEP month DSEP day ; date YYYY-MM-DD
time             = hour TSEP minute TSEP second TZ ; time HH:MM:SSZ

;                date
century          = 2DIGIT ; 00-99
year             = 2DIGIT ; 00-99
month            = 2DIGIT ; 01-12
day              = 2DIGIT ; 01-28, 01-29, 01-30, 01-31 based on month/year
DSEP             = "-"    ; date separator

;                time
hour             = 2DIGIT ; 00-24
minute           = 2DIGIT ; 00-59
second           = 2DIGIT ; 00-58, 00-59, 00-60 based on leap-second rules
TSEP             = ":"    ; time separator
TZ               = "Z"    ; timezone
```

### Media type

The current media type is `application/uk-gov-rsf`. It should change to
`application/vnd.rsf` to align with [RFC6838](https://tools.ietf.org/html/rfc6838).

### REST API

TODO

The current implementation uses `GET /download-rsf`. The main issue with that
is that diverges from the rest of the API where serialisation is expressed
either via suffix or via media type. The problem with using the same approach,
say `GET /register.rsf` is that we are not providing the same information when
querying `GET /register.json`.


### Commands

#### <a id="assert-root-hash-command">`assert-root-hash` command</a>

Asserts that the provided root hash is the same as the one computed from the
current entry log as defined in the [Digital Proofs][digital-proofs]
specification.

##### Arguments

1. The `hash` of the root of the tree.

For example, the empty root hash:

```
assert-root-hash	sha-256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

#### <a id="add-item-command">`add-item` command</a>

Adds a new [Item resource][item-res] to the register. It will require an
[`append-entry` command](#append-entry-command) to make it visible to users.

##### Arguments

1. The [canonical representation][canon-rep] of the item.

For exeample:

```
add-item	{"country":"GB","name":"United Kingdom","official-name":"The United Kingdom of Great Britain and Northern Ireland"}
```

#### <a id="append-entry-command">`append-entry` command</a>

Appends a new [Entry resource][entry-res] to the register.

##### Arguments

1. The `type` of the entry determines if the entry belongs to the data log
   (`user`) or to the metadata log (`system`).
2. The `key` of the entry. The primary key field is the field with the same
   name as the register.
3. The `timestamp` of the entry. This is the time at which the entry was
   appended to the register.
4. The `hash` of the item for which the entry was appended. This is the
   [sha-256 hash of the item][canon-rep].

For example:

```
append-entry	user	GB	2010-11-12T13:14:15Z	sha-256:08bef0039a4f0fb52f3a5ce4b97d7927bf159bc254b8881c45d95945617237f6
```


### Validation rules

A RSF list of commands is expected to conform to the following rules:

* [Commands](#commands) are executed in order of appearance, top to bottom.
* Entries are numbered in sequence in order of appearance starting with 1 if
  the register is empty, otherwise incrementing on the latest entry number
  found in the register.
* An [`append-entry` command](#append-entry-command) must always appear after
  the [`add-item` command](#add-item-command) that introduces the item is
  referencing.
* It is illegal to have orphan items. An `add-item` must have at least one
  `append-entry` referencing to the item.
* It is illegal to have broken references. An `append-entry` must reference an
  item previously introduced by an `add-item` command.
* The item in the `add-item` command must always be in the canonical form.


#### Type checking

Although not part of the RSF specification, it is worth mentioning that a
Registers' implementation is expected to type check the data according to the
computed schema.

##### Metadata

A metadata item must conform to the metadata schema

[TODO: This needs definition].

##### Data

A data item must conform to the current schema derived from the previous
system entries. A type checker is expected to verify:

* It has the primary key defined.
* Fieldnames exist in the schema.
* Cardinality is consistent.
* Datatype is consistent.

Given the example “[All commands in use](#all-commands-example)”, a new data
item is valid if:

* It has the primary key, `country` defined.
* It has at most one `name` field and one `official-name` field.
* The `country` field has cardinality 1.
* The `country` field is a String.
* The `name` field has cardinality 1.
* The `name` field is a String.
* The `official-name` field has cardinality 1.
* The `official-name` field is a String.

Each datatype must be parsed according to the [datatype specification][datatype-spec].

A RSF patch (set of commands) must be treated as a single transaction. If
there is a validation error, the whole patch must be rejected and any changes
to the state rolled back.


### Examples

#### Simple RSF

```
add-item	{"country":"GB","name":"United Kingdom","official-name":"The United Kingdom of Great Britain and Northern Ireland"}
append-entry	user	GB	2010-11-12T13:14:15Z	sha-256:08bef0039a4f0fb52f3a5ce4b97d7927bf159bc254b8881c45d95945617237f6
```

#### Multiple items

```
add-item	{"local-authority-eng":"LND","local-authority-type":"NMD","name":"London"}
add-item	{"local-authority-eng":"LEI","local-authority-type":"NMD","name":"Leicester"}
add-item	{"local-authority-eng":"CHE","local-authority-type":"NMD","name":"Cheshire"}
append-entry	user	NMD	2016-04-05T13:23:05Z	sha-256:490636974f8087e4518d222eba08851dd3e2b85095f2b1427ff6ecd3fa482435;sha-256:8b748c574bf975990e47e69df040b47126d2a0a3895b31dce73988fba2ba27d8;sha-256:eb3ee00e6149cd734a7fa7e1f01a5fbf5fb50e1b38a065fd97d6ad3017750351
```

#### <a id="all-commands-example">All commands in use</a>

```
assert-root-hash	sha-256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
add-item	{"cardinality":"1","datatype":"string","field":"country","phase":"beta","register":"country","text":"The country's 2-letter ISO 3166-2 alpha2 code."}
add-item	{"cardinality":"1","datatype":"string","field":"name","phase":"beta","text":"The commonly-used name of a record."}
add-item	{"cardinality":"1","datatype":"string","field":"official-name","phase":"beta","text":"The official or technical name of a record."}
append-entry	system	field:country	2017-01-10T17:16:07Z	sha-256:a303d05bdbeb029440344e0f1148f5524b4a2f9076d1b0f36a95ff7d5eeedb0e
append-entry	system	field:name	2017-01-10T17:16:07Z	sha-256:a7a9f2237dadcb3980f6ff8220279a3450778e9c78b6f0f12febc974d49a4a9f
append-entry	system	field:official-name	2017-01-10T17:16:07Z	sha-256:5c4728f439f6cbc6c7eea42992b858afc78c182962ba35d169f49db2c88e1e41
add-item	{"country":"GB","name":"United Kingdom","official-name":"The United Kingdom of Great Britain and Northern Ireland"}
append-entry	user	GB	2010-11-12T13:14:15Z	sha-256:08bef0039a4f0fb52f3a5ce4b97d7927bf159bc254b8881c45d95945617237f6
```


[item-res]: https://openregister.github.io/specification/#item-resource
[entry-res]: https://openregister.github.io/specification/#entry-resource
[canon-rep]: https://openregister.github.io/specification/#sha-256-item-hash
[digital-proofs]: http://openregister.github.io/specification/#digital-proofs
[datatype-spec]: http://openregister.github.io/specification/#datatypes
