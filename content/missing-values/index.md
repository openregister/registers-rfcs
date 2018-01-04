---
rfc:
start_date: 2018-01-04
status: draft
---

# Missing values

TODO:

* A-mark: Missing and applicable => Unknown, Null
* I-mark: Missing and inapplicable => NA (or an explicit value so no longer
  _missing_).

Can we answer the following?

* What kind of information is missing?
* What is the main reason for being missing?

The first, assumed we are talking about atomic/datum type, could be inferred
from the schema. Contextual “kind” or composite “kind” sounds unlikely as it
would require domain knowledge.
The second requires domain knowledge, so unlikely to have a systematic
solution.

Assuming the above, this document only deals with A-marks in the addition
operation.

> The semantics of the fact that a datum is missing are not the same as the
> semantics of the datum itself.

Application of equality:  (1) semantic equality and (2) symbolic equality.
* Semantic equality needs context (domain knowledge).
* Symbolic equality can be applied systematically with a 4-value logic: (true,
  false, A-mark, I-mark).

It probably doesn’t matter in terms of what Registers tries to offer.



