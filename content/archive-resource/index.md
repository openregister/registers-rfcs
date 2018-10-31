---
rfc: 0021
start_date: 2018-09-27
decision_date: 2018-10-31
pr: openregister/registers-rfcs/pull/35
status: approved
---

# Archive resource

## Summary

This RFC proposes removing the Archive resource to simplify the API.

This is a breaking change.

## Motivation

Currently, the specification defines the Archive resource as a zip file within
the whole register inside. That is entries, blobs and some contextual
information. Usage metrics show that no one is actually using it.

## Explanation

Remove the endpoint `GET /download-register`.

## Consequences

The archive resource will not be available anymore.
