---
rfc: 0014
start_date: 2018-08-13
decision_date: 2018-08-24
pr: openregister/registers-rfcs#27
status: rejected
---

# Summary resource

This RFC has been rejected (see comments
https://github.com/openregister/registers-rfcs/pull/27).

In summary, this resource doesn't fit well with the broader metadata stream of
work. RFC0018 Context resource obsoltes this one.

## Summary

This RFC proposes a summary resource to offer an overview of the current state
of the register.

Dependencies:

* [RFC0013: Multihash](https://github.com/openregister/registers-rfcs/pull/26)


## Motivation

The current summary is exposed as the register resource (`GET /register`).
The name of the resource is misleading as you don't get the register, only
some metadata about it. Also, the current specification and implementation
don't agree on what is the register resource.

## Explanation

The summary resource is a new resource that will replace the old register
resource. The summary resource will not contain any information about fields
as this will be addressed by the schema resource.

The most noticeable change is the addition of the `hashing-algorithm` object
describing the hashing algorithm used in the register to create hashes for
items, entries and proofs.

***
### Endpoint

```
GET /summary
```

### Response attributes

|Name|Type|Description|
|-|-|-|
|`copyright`| String |The copyright of the data.|
|`custodian`| Optional String |The custodian of the register.|
|`description`| Optional String |The description of the register.|
|`hashing-algorithm`| [HashingAlgorithm](#hahing-algorithm-attributes) |The hahing algorithm used throught the register.|
|`root-hash`| Hash | The Merkle tree root hash. |
|`last-updated`| Timestamp |The date the register was last updated.|
|`licence`| String |The licence of the data.|
|`total-entries`| Integer |The number of entries in the register.|
|`total-items`| Integer |The number of items in the register.|
|`total-records`| Integer |The number of records in the register.|

This RFC doesn't include anything related to signing root hashes. A future RFC
will expand this resource with it.

#### Hashing algorithm attributes

|Name|Type|Description|
|-|-|-|
|`digest-length`| Hexadecimal Integer |The length of the digest in bytes.|
|`function-type`| Hexadecimal Integer |The identifier of the hash function.|
|`name`| String |The common name of the algorithm.|

See also: [Multihash table function type
identifiers](https://github.com/multiformats/multihash/blob/master/hashtable.csv).

***

***
**EXAMPLE:**


```http
GET /summary HTTP/1.1
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "description": "List of multihash codes.",
  "hashing-algorithm": {
    "name": "sha2-256",
    "function-type": "12",
    "digest-length": "20"
  },
  "total-entries": 118,
  "total-items": 118,
  "total-records": 118,
  "custodian": "IPFS team",
  "last-updated": "2017-09-04T00:00:00Z",
  "copyright": "Copyright (c) 2014 Crown Copyright HM Government",
  "licence": "Open Government Licence v3.0 (OGL)",
}
```

***
