---
rfc: 0022
start_date: 2018-09-28
decision_date:
pr: openregisters/registers-rfc#39
status: draft
---

# Facets endpoint

## Summary

This RFC proposes to change the endpoint for the records' “facets” to simplify
path handling.

The facets requirement is not essential for a Register to operate so this RFC
proposes relaxing the specification and making its implementation optional.


## Motivation

Currently the REST API for facets uses a path structure that overlaps with the
standard record structure. This makes path handling (e.g. analytics, logging)
harder than it needs to be. Observation suggests that users can get confused
of what is the resource about.


## Explanation

For context, the current endpoint for facets is:

```
GET /records/{attribute-name}/{attribute-value}
```

The record endpoint is:

```
GET /records/{key}
```

And the records endpoint is:

```
GET /records
```


This RFC proposes to change the facets endpoint to extend the records endpoint
with a query string to make more obvious that a “facet” is filter on the
records collection.

```
GET /records{?name, value}
```

Where `{?name}` is required if `{?value}` is provided.


### Filter by strict value match

```
GET /records?name={attribute-name}&value={attribute-value}
```

Exactly the current behaviour.


### Backwards compatibility

In order to guarantee backwards compatibility and to have a smooth transition,
the old endpoint should issue HTTP permanent redirects (301) to the new
equivalents.
