---
rfc: 0026
start_date: 2018-08-09
pr: openregisters/registers-rfc#9981
status: draft
---

# Use OpenAPI to describe the REST API

## Summary

This RFC proposes that we publish an OpenAPI document for each version of the specification.

The conformance tests for the specification should reuse this rather than defining additional schemas in the code.

Note that at the time of writing OpenAPI only supports JSON and YAML. Therefore the benefits of OpenAPI described in this CSV only apply to implementations that use the JSON representation.

## Motivation

OpenAPI is a standard for describing - in a machine readable format - REST APIs that return JSON or YAML.

In this RFC, when I say OpenAPI I'm refering to [version 3.0.2 of the specification](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.2.md).

Having a machine readable specification for the registers API allows implementors to take advantage of an existing ecosystem of tooling for building and hosting APIs.

OpenAPI is not the only standard for specifying APIs, but I'm restricting the proposal to OpenAPI because it is a [GDS recommended standard for API documentation](https://www.gov.uk/guidance/gds-api-technical-and-data-standards#document-your-api).

## Explanation
OpenAPI tools can provide API documentation, generate client libraries, and validate that implementations conform to the specification.

### API documentation
Currently the reference implementation comes with an API explorer, but no API documentation. The API explorer links out to the GOV.UK Registers API documentation, which is not appropriate for other implementors who might wish to host the software themselves.

Although [the source for this documentation is open](https://github.com/alphagov/registers-tech-docs/), it's not easy to reuse, because
- the content contains information that is specific to the GOV.UK Registers service
- the content must be manually updated whenever a change is made to the API (this is also more difficult now that we have multiple API versions)

Providing an OpenAPI document along with the specification would mean that any implementation can reuse it, using their choice of documentation tool.

This RFC does not propose removing the API explorer from the reference implementation, but using a 3rd party documentation tool such as [Swagger UI](https://swagger.io/tools/swagger-ui/) may reduce the need for us to continue to maintain our own solution.

### Conformance testing
Currently we provide a test suite to verify that an implementation obeys the specification.

Most of these tests relies on validating a response against a JSON schema. OpenAPI schemas are described as an extension to JSON schema (although they have [diverged somewhat](https://philsturgeon.uk/api/2018/03/30/openapi-and-json-schema-divergence/)) so it would be straightforward to base the conformance tests on OpenAPI test instead.

This proposal would essentially take the schemas we are already defining and put them in a single document that can be reused.

### Code generation
Various API client libraries exist that make it easier to integrate with the registers API. However, the GOV.UK Registers team does not officially maintain any of them. It is a lot of work to create good client libraries and to maintain them over a range of languages.

If we use OpenAPI, a code generation tool such as [OpenAPI generator](https://github.com/openapitools/openapi-generator) would be a low cost way to provide client libraries for a range of languages.

### What shouldn't be specified?

An OpenAPI specification can contain information about:
- where to find the API (optional)
- the methods and paths supported offered by the API
- schemas for requests and responses
- where to find external documentation

To keep the OpenAPI document implementation-independent, we should focus on the last 3 points, and not specify any server endpoints.

### Discoverability
The document should be linked from the corresponding specification page.

Each version of the API should have its own OpenAPI document.

### Limitations of OpenAPI

The Registers team has briefly investigated OpenAPI before (see [this comment from Paul Downey](https://github.com/alphagov/open-standards/issues/31#issuecomment-319334209)). A reason against adopting it is that parts of the API (such as HATEOAS, and especially links between registers) would be difficult or impossible to capture. This means that a purely machine generated documentation site would be missing information. This is something we should consider carefully when authoring the document, but I don't consider a strong enough reason to not adopt it.

OpenAPI does not currently allow you to define CSV responses. However, the text specification also only uses JSON as an example, as does the existing API documention for the GOV.UK Registers service. So while it would be great to have a machine readable specification that handled CSV as well, I think we can live without it.