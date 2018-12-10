---
rfc: 0027
start_date: 2018-12-10
decision_date:
pr: openregister/registers-rfcs#44
status: draft
---

# Blob collection resource

## Summary

This RFC proposes changing the blob resource to an array of records instead of
an object.

This is a breaking change.

## Motivation

The current blob collection is an object in JSON and a list in CSV. Also, the
rest of the collections (entries, records) are arrays.

## Explanation

To address the above this RFC proposes changing the JSON representation from
an object to an array. To keep the blob hash available, the blob resource must
incorporate a new attribute `_id` (aligning with
[RFC0025](https://github.com/openregister/registers-rfcs/blob/master/content/snapshot-resource/index.md)).


### REST API

#### Get a blob

***
#### Endpoint

```
GET /blobs/{id}
```

#### Parameters

|Name|Type|Description|
|-|-|-|
|`id`|Hash|The blob hash.|
***

***
**EXAMPLE:**

For example, a blob in JSON would look like

```json
{
  "_id":
  "12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb",
  "country":"GB",
  "official-name":"The United Kingdom of Great Britain and Northern Ireland",
  "name":"United Kingdom",
  "citizen-names":["Briton","British citizen"]
}
```

And in CSV

```csv
_id, country, official-name, name, citizen-names
12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb, GB, "The United Kingdom of Great Britain and Northern Ireland", "United Kingdom", "Briton;British citizen
```
***


#### Get the list of blobs

Returns an array of blob resources.

This resource MAY be paginated like the rest of the collections.

***
#### Endpoint

```
GET /blobs
```
***

***
**EXAMPLE:**

For example, a list of two blobs in JSON would look like

```json
[{
  "_id": "12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb",
  "country":"GB",
  "official-name":"The United Kingdom of Great Britain and Northern Ireland",
  "name":"United Kingdom",
  "citizen-names":["Briton","British citizen"]
  }, {
  "_id": "1220a99cacb3728427303b349279f2671b5a4a11aa6a596b5628c517e7e963b1f2ce",
  "country":"SU",
  "official-name":"Union of Soviet Socialist Republics",
  "name":"USSR",
  "citizen-names":"Soviet citizen"
}]
```

And in CSV

```csv
_id, country, official-name, name, citizen-names
12206b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb, GB, "The United Kingdom of Great Britain and Northern Ireland", "United Kingdom", "Briton;British citizen
1220a99cacb3728427303b349279f2671b5a4a11aa6a596b5628c517e7e963b1f2ce, SU, "Union of Soviet Socialist Republics", "USSR", "Soviet citizen"
```
***

## Consequences

This is a breaking change.

The core collections provided by the API (entries, items, records) can be
processed the same way by API clients.
