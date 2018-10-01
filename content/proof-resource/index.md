---
rfc: 0023
start_date: 2018-10-01
decision_date:
pr: openregister/registers-rfcs/pull/40
status: draft
---

# Proof resource

## Summary

This RFC proposes to fix the type of proofs to Certificate Transparency's
Merkle tree approach.

## Motivation

Currently, the specification define the Proof resource endpoints parametrized
by type of proof through the `proof-identifier` parameter. The implications of
implementing a proof system are substantial as demonstrated by the long time
is taking the reference implementation to have a stable implementation for a
SHA2-256 Merkle tree. Also, a Register uses a single hashing function which
means the proof type has to align with that fact.

## Explanation

The proposal is to restrict the possible proofs to Merkle-based (i.e.
Certificate Transparency based) ones. And rely on the register hashing
function to determine what to use to compute the root hash.

### Endpoints

The implications for each endpoint are mainly the removal of the paramter
`proof-identifier` and the pluralisation of the resource scope to align with
the rest of the API.

#### Register proof

```
GET /proofs/register
```

##### Response attributes

|Name|Type|Description|
|-|-|-|
|`root-hash`|Hash|The root hash for the log when the proof was issued.|
|`total-entries`|Integer|The size of the log when the proof was issued.|

Note that `tree-head-signature` will be addressed in another RFC when we have
done the work to scope implications.

***
**EXAMPLE:**

For example, a register using a SHA2-256 hashing function:

```http
GET /proofs/register HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "root-hash": "12208d92e1e0af1d43c41e498e6baed0d0b3ea2770d1bf9d2afc04e9c4dad7795729",
  "total-entries": 208
}
```
***

#### Consistency proof

```
GET /proofs/consistency/{small-log-size}/{large-log-size}
```

##### Parameters

|Name|Type|Description|
|-|-|-|
|`small-log-size`|Integer|The size of the smaller log.|
|`large-log-size`|Integer|The size of the larger log.|

##### Response attributes

|Name|Type|Description|
|-|-|-|
|`audit-path`|List of Hash|The list of node hashes.|

***
**EXAMPLE:**

For example, a register using a SHA2-256 hashing function:

```http
GET /proofs/consistency/197/200 HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "audit-path": [
      "122073f13521226acdfa2a610c7bfdc955fa52aea1d554dd247011312ee48686a538",
      "12208a16bb948f55ef959a5a7ddad5e2d1d398b50f3d7095aba1e97ad50c1fa374a9",
      "1220be8a541a0a763f88c8e4ff5f013e701e5f89c3f9cb744aadfaf19668189de514",
      "1220733c1adf88daff4ba4275b4ff86d373266c17eeb547ef54093ed14649d168865",
      "12206242c4d6fde2c79c26144deab292fc6702d321a7e79c535e146d25f356191f7c",
      "122020b0c02232b50a587671ed9f465fb1a99923a08ff53951b8b9f4bb29648aa112"
    ]
}
```
***

#### Entry proof

```
GET /proofs/entry/{entry-number}/{log-size}
```

##### Parameters

|Name|Type|Description|
|-|-|-|
|`entry-number`|Integer|The entry number to proof.|
|`log-size`|Integer|The size of the log.|

##### Response attributes

|Name|Type|Description|
|-|-|-|
|`audit-path`|List of Hash|The list of node hashes.|

***
**EXAMPLE:**

For example, a register using a SHA2-256 hashing function:

```http
GET /proofs/entry/10/200 HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "audit-path": [
    "1220f0ebeef6be205cfc5fb6b4a314294bdff471f5409594f742b0f30c8551278b4a",
    "12208dc980062c4e6ffd2300b72cd5a6a67e23070aabec31911691c657c2e1dd37a6",
    "1220c48916df15f3f6e030d84bf0f8bb59460c472250d38db27b4cd2e7394fe0741d",
    "122008e9d6bd5717717c1c40ba518ccf02cad9c412eae6052739552ecd4a668b4ec3",
    "122043834a10ac7dcecc7bb274d67f79dc5da4c03efb6dadc20657595ca4b261df4d",
    "122010d897e8df0096412f45e9c16c61eed7b335267d803872f85ce0d25218fc82eb",
    "1220e483ea76d5ca3fdcef64ae8a2c910d1e47b90507a364da8dc4878cacd48cd414",
    "1220ca77ecfa5a4e847c65fda8f41f73758456814acf473bab2811516aeaac17f7cc"
  ]
}
```
***

### Consequences

This change is a breaking change to anyone using the experimental proofs API.
Given it is experimental, users have to expect this kind of change.
