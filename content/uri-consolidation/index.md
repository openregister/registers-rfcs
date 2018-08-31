---
rfc: 0002
start_date: 2018-03-08
decision_date: 2018-03-14
pr: openregisters/registers-rfc#10
status: approved
---

# URI Consolidation

## Summary

This RFC proposes a change in the REST API to consolidate endpoints of the
same resource type to use the same naming convention so CURIEs can be used to
refer to the full collection of records within a Register.


## Motivation

Currently the REST API uses the plural form of a resource type for collection
endpoints and the singular form for collection elements. Based on how Registers
use CURIEs to reference records this design choice makes it impossible to
refer to the whole list of records using the same CURIE prefix.


## Explanation

To address this limitation, we have two options: use the same form for the
endpoints, or define two prefixes per register. Using the latter option would
make defining and evolving registers more complex than using the former
option, so we propose to only use the plural form for endpoints.

We selected the plural form because it's more common in well-known APIs (e.g.
[GitHub API v3](https://developer.github.com/v3)).

Endpoint mappings:

* From `/record/{field-value}` to `/records/{field-value}`.
* From `/record/{field-value}/entries` to `/records/{field-value}/entries`.
* From `/entry/{entry-number}` to `/entries/{entry-number}`.
* From `/entry/{entry-number}/proofs` to `/entries/{entry-number}/proofs`.
* From `/item/{item-hash}` to `/items/{item-hash}`.
* From `/proof/entry/{entry-number}/{total-entries}/{proof-identifier}` to
`/proof/entries/{entry-number}/{total-entries}/{proof-identifier}`.

Note that we have not yet implemented `/items`, but it is in the pipeline.

The `/records/{field-value}` change is the principal change that enables defining a
CURIE mapping like:

```turtle
@prefix country <https://country.register/records/> .
```

So the current CURIEs' `{prefix}:{id}` such as `country:GB` will still work and
a CURIE like `country:` will expand to `https://country.register/records/`
which means “all records within the Country register”.

As a consequence of this change any record containing a CURIE of the form
`register:{register}` with the intention to express “all records within
register `register`” must be amended to have the new CURIE form `{register}:`.
E.g. From `register:local-authority-eng` to `local-authority-eng:`.


### Backwards compatibility

In order to guarantee backwards compatibility and to have a smooth transition,
the endpoint mappings listed above should be HTTP permanent redirects (301).
