---
rfc: 0013
start_date: 2018-08-13
pr: openregister/registers-rfcs/pull/26
status: draft
---

# Multihash

## Summary

This RFC proposes to change the method to qualify hashes from using prefixes
`sha-256:` to using [Multihash](https://multiformats.io/multihash/).

**This RFC is not backwards compatible in the same sense as
[RFC0010](https://github.com/openregister/registers-rfcs/pull/24)**

## Motivation

Currently, the specification defines a custom `sha-256:` prefix for
stringified hexadecimal hashes to qualify them as SHA256. With this approach
parsers must be done specifically for registers and registers must standarise
any hashing algorithm needed by users. Relying on a standard helps benefiting
from that standards ecosystem.

## Explanation

The proposal is to adopt [Multihash](https://multiformats.io/multihash/).
Their specification is quite thorough but the gist is to replace any of our
`sha-256:` to `1220`.

* `0x12` stringified as `12`: Hashing function sha2-256.
* `0x20` stringified as `20`: Digest length.

For example, instead of the current JSON:

```json
[{
  "index-entry-number":"6",
  "entry-number":"6",
  "entry-timestamp":"2016-04-05T13:23:05Z",
  "key":"GB",
  "item-hash":["sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
}]
```

We will use

```json
[{
  "index-entry-number":"6",
  "entry-number":"6",
  "entry-timestamp":"2016-04-05T13:23:05Z",
  "key":"GB",
  "item-hash":["12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb"]
}]
```

And similarly, the `GET /items/{hash}` endpoint will expect something like
this:

```http
GET /items/12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Accept: application/json
}]
```
