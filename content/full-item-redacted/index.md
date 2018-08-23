---
rfc: 0017
start_date: 2018-08-23
pr: openregister/registers-rfcs#30
status: draft
---

# Full item redaction

## Summary

This RFC proposes a way to signal an item has been fully redacted.

Dependencies:

* [RFC0010: Item hash](https://github.com/openregister/registers-rfcs/pull/24).
* [RFC0013: Multihash](https://github.com/openregister/registers-rfcs/pull/26).


## Motivation

The specification defines “redaction” as:

> An item may be removed from a register, but the entry MUST NOT be removed
> from the register.

In some buried decision log and as a known thing for some components of the
Registers core team, a while ago it was decided that to fully redact an item,
the HTTP response would be a `410 Gone`. The specification doesn't reflect
this decision nor the details.


## Explanation

A fully redacted item MUST respond with a `410 Gone` HTTP status keeping the
same payload structure as a `200 OK` response but redacted following RFC0010
`**REDACTED**` mark and strategy.

Note: Partially redacted items are addressed by RFC0010.

***
**EXAMPLE:**

For example,

```http
GET /items/sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 410 Gone
Content-Type: application/json

**REDACTED**sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb
```

With RFC0013 in mind, this is the expected behaviour:


```http
GET /items/12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 410 Gone
Content-Type: application/json

**REDACTED**12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb
```

For CSV, the payload MUST be the same as the expected list of attributes
doesn't apply:

```http
GET /items/12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Host: country.register.gov.uk
Accept: text/csv;charset=UTF-8
```

```http
HTTP/1.1 410 Gone
Content-Type: text/csv;charset=UTF-8

**REDACTED**12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb
```
***
