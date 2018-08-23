---
rfc: 0019
start_date: 2018-08-23
decision_date:
pr: openregister/registers-rfcs#32
status: draft
---

# Boolean datatype

## Summary

This RFC proposes a new datatype to represent boolean values.

## Motivation

The datatypes in registers cover most of the common values you find in other
systems. Although some systems opt to represent booleans as 0 and 1, it is
ambiguous without human interaction.

## Explanation

The **boolean** datatype can have two states: `true` or `false`.

```elm
type Boolean
  = True
  | False
```

The ABNF definition for the string representation is:

```abnf
boolean = "true" / "false"
```

***
**EXAMPLE:**

For example, in JSON:

```json
{"foo": "true", "bar": "false"}
```

And in CSV:

```csv
foo, bar
true, false
```
***
