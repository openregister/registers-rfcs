# Notes on verifiable data structures

How schema evolution affects Verifiable data structures? And how it affects
the ergonomics of the API? (porcelain vs plumbing).

## Plumbing API

The _plumbing_ API is intended to be used as a low-level access to the data
model. That is Items and Entries as they are the building blocks for the rest.

* Is a Record a high-level abstraction or a building block?
* Is a Field a building block?
* Is a Schema a building block?
* Is a Lens a building block?

## Porcelain API

The _porcelain_ API is intended to be used as a shortcut to get data you can
derive from the building blocks. An example of such resource is the
Projection. It's only intention is to be consumed by a user with few
dependencies or knowledge around the building blocks.



## The challenge

Be ready to find agreement on what should the REST API be providing:

a) Plumbing access to data (i.e. low-level access).
b) Plumbing and Porcelain (i.e. high-level access) access to data.

The tension between the two approaches is, on one side the need to provide
robust ways to proof authenticity and on the other side the need to provide
access to data as the Custodian intended it to be consumed (according to the
RFC, this means providing direct access to the projections).

The RFC: https://github.com/openregister/registers-rfcs/pulls

Some ideas that came up since last time (not necessarily exclusive with each
other):

* Record as the projection (breaks the Record proof and it's the root of this
discussion).
* Record + Lens so the consumer can derive the projection.
* Record as is. New endpoint for a Thing that returns the projection.
* Record with the data redacted like objecthash.


### Record as the projection

porcelain oriented.

This option breaks the Record proof and it's the root of this discussion. 

### Consumer requirements

* Regular access to the API.

### Guarantees

* No new concepts/resources added to the API. Although it is not
  backwards-compatible because it no longer allows the Record Proof.


### Record + Lens so the consumer can derive the projection.

plumbing oriented.

This option keeps the Record as is but allows the consumer to obtain the Lens
either as part of the Record or in a separate call (e.g. `/lenses/{size}`).

#### Consumer requirements

* The Lens [TODO: Clarify how is the lens obtained].
* Function to extract the Item from a Record.
* Function to compute the Projection for any given (item, lens) pair.

#### Guarantees

* Backwards compatible including Record proofs.


### Record as is. New endpoint for a Thing that returns the projection.

porcelain oriented.

This option keeps the Record as is and adds another top-level resource to the
API.

#### Consumer requirements

* Regular access to the API.

#### Guarantees

* Backwards compatibility including Record proofs.
* Few consumer requirements for consuming the data (i.e. porcelain).


### Record with the data redacted like objecthash.

plumbing oriented.

This is not really any different from the (Record + Lens) option as it doesn't
solve the actual problem of expressing Lens Projections.

#### Consumer requirements

* Function to clean the redacted information. Probably evident on its own.

#### Guaranteed

* No new concepts/resources added to the API. Although it is not
  backwards-compatible because it changes the way item hashes are generated.
