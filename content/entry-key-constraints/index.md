---
rfc: 0008
start_date: 2018-08-02
decision_date: 2018-08-24
pr: openregister/registers-rfcs#22
status: approved
---

# Entry key constraints

## Summary

This RFC proposes to restrict the possible values allowed as an entry key
value.


## Motivation

Currently, identifiers (i.e. keys) are defined as of any datatype available.
Usage shows only strings have been used thus far.

Allowing any datatype imposes a hard task on the user trying to validate
what to expect in there. Other questions arise when you analyise how each
datatype is expected to behave in all contexts identifiers are used.

Questions like “What are the rules for an identifier when used as part of a
CURIE?”, “What does it mean to have a String (full UTF-8 range)?”.

Notice that [issue 18](https://github.com/openregister/registers-rfcs/issues/18)
has a discussion related to this RFC.

This RFC aims to improve the situation by proposing a set of constraints to
make validation viable, that works well with CURIEs and overall that guides
users without preventing valid use cases.


## Explanation

A valid identifier is a subset of the String datatype. It helps keeping
backwards compatibility and builds on top of how Registers are used in
reality.

Evidence says that identifiers are composed primarily by alphanumeric ASCII
characters with a few other printable characters:

* `-` (U+002D)
* `_` (U+005F)
* `.` (U+002E)
* `/` (U+002F)

Note that `/` has meaning in URLs and hence it has meaning in CURIEs. This
means that a CURIE must always percent-encode this character as `%2F`.

The proposal then is to not allow more characters than the ones that are
currently used:

```abnf
id = (ALPHA / DIGIT) *(ALPHA / DIGIT / restricted)
restricted = delimiters / legacy
delimiters = "-" / "_" / "."
legacy = "/"
```

Where `restricted` can't appear consecutively.

The restriction on `restricted` is to prevent legibility issues e.g `a____b`.

For example, the following identifiers are valid:

```
1
GB
01
10.5
ADR
CA-ZX
an_id
```

The following is discouraged but accepted due legacy reasons:

```
10.2/3
```

The exaple above is encoded as a CURIE as:

```
example:10.2%2F3
```

And, assuming `example: https://www.example.org/` expanded as a URL as:

```
https://www.example.org/10.2%2F3
```

And the following are invalid:

```
_1
.34
A..B
ALPHA--
C__34
C_/34
```

### Relationship with data in items

Historically Registers have stored the identifier of the element in the data
blob (item) as well as in the entry key. In this case, the attribute with the
same value as the entry key must have the exact same value.
