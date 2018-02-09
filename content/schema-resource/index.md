---
rfc:
start_date: 2018-02-09
pr:
status: draft
---

# Schema resource

## Summary

This RFC proposes a schema resource to describe the register set of attributes
with their datatypes.


## Motivation

The current mechanisms to get information about the schema are the `GET
/register` endpoint or the Register register complemented with the Field
register. It is cumbersome to get the full picture of the schema and, even
more the day the schema gets the ability to evolve.


## Explanation

***
### Endpoint

```
GET /schema
```

### Parameters

|Name|Type|Description|
|-|-|-|
|`log-size` | Optional `Integer`| Set the log size to the given entry number instead of the latest value.|


### Response attributes

|Name|Type|Description|
|-|-|-|
|`attributes` | List of `Attribute`| The list of attributes.|


### Attribute attributes

|Name|Type|Description|
|-|-|-|
|`label` | `Attrname` | The attribute machine-readable name.|
|`title` | `String` | The attribute human-readable title.|
|`description` | `Text` | The attribute description.|
|`datatype` | `Datatype` | The attribute datatype.|
|`cardinality` | `Cardinality` | The attribute cardinality (either `1` or `n`).|
***

Note that meanwhile there is no schema evolution, `log-size` doesn't affect
the result.

***
**EXAMPLE:**

```http
GET /schema HTTP/1.1
Host: approved-open-standard-guidance.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "attributes": [
    {
      "label": "name",
      "title": "Name",
      "description": "The commonly-used name of a record.",
      "datatype": "string",
      "cardinality": "1"
    },
    {
      "label": "approved-open-standards",
      "title": "Approved open standards",
      "description": "Open standards that have been approved by government technology.",
      "datatype": "curie",
      "cardinality": "n"
    },
    {
      "label": "website",
      "title": "Website",
      "description": "The website for a record.",
      "datatype": "url",
      "cardinality": "1"
    },
    {
      "label": "start-date",
      "title": "Start date",
      "description": "The date a record first became relevant to a register.",
      "datatype": "datetime",
      "cardinality": "1"
    },
    {
      "label": "end-date",
      "title": "End date",
      "description": "The date a record stopped being applicable.",
      "datatype": "datetime",
      "cardinality": "1"
    },

  ]
}
```
***