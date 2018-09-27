---
rfc: 0021
start_date: 2018-09-27
decision_date:
pr: openregister/registers-rfcs/pull/35
status: draft
---

# Archive resource

## Summary

This RFC proposes to change the endpoint for the Archive resource to match the
name of the resource instead of using the action “download”. Also, it
consolidates the non-standardised endpoint for RSF to use the same enpoint.

## Motivation

Currently, the specification defines the Archive resource endpoint as
`GET /download-register` but the rest of resources use the RESTful convention
of using the name of the resource. For example, to get the list of records you
would use `GET /records`.

Also, there is another non-standardised endpoint, `GET /download-rsf` that
allows getting a resource that in fact is an archive in another serialisation
format, RSF.

## Explanation

The proposal is to consolidate both `GET /download-register` and `GET
/download-rsf` into a single endpoint `GET /archive`.

The content negotiation must happen based on the same rules that apply to
other resources but, instead of JSON and CSV, this enpoint expects either
ZIP or RSF. ZIP is the default response when there is no extension or `Accept`
header.

The decision logic is as follows:

1. Let _req_ be the incoming request.
2. If _req_ has an extension:
    1. If it is `.zip`, return (200, `application/octet-stream`)
    2. If it is `.rsf`, return (200, `application/vnd.rsf`)
    3. Otherwise, return 406, Not Acceptable.
3. If _req_ has an `Accept` header:
    1. If it is `application/octet-stream`, return (200, `application/octet-stream`)
    2. If it is `application/vnd.rsf`, return (200, `application/vnd.rsf`)
    3. If it is `application/uk-gov-rsf`, return (200, `application/vnd.rsf`)
    4. Otherwise, return 406, Not Acceptable.
4. Otherwise, return (200, `application/octet-stream`)


Note that `application/uk-gov-rsf` is handled for legacy reasons.


### Zip

* Extension: `.zip`
* Content type: `application/octet-stream`

### RSF

* Extension: `.rsf`
* Content type: `application/vnd.rsf`

### Endpoint definition

The normative endpoint is:

```
GET /archive
```

The RSF extension needs the ability to request for an slice of the log, the
current implementation uses `GET /download-rsf/{from}/{to}` and  `GET /download-rsf/{from}`.
The proposed extension is:

```
GET /archive?from={n}&to={n}
```

Parameters:

|Name|Type|Description|
|-|-|
|from|Integer|The log size to use as starting point for the slice|
|to|Integer|The log size to use as ending point for the slice|


## Consequences

The transition from the old endpoints to the new ones will follow the version
strategy.
