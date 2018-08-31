---
rfc: 0003
start_date: 2018-04-26
decision_date: 2018-05-01
pr: openregisters/registers-rfc#16
status: approved
---

# Entry timestamp meaning

## Summary

This RFC proposes a definition for the timestamp found in an [entry
resource][entry-res] to address the current ambiguity raised in
[Issue 48][issue-48].


## Motivation

Every entry in a register has a timestamp (`entry-timestamp`). The current
observed behaviour is that the timestamp is the moment when the entry was
minted but there is no assurance that this is the expected behaviour for a tool
complying with the specification. This makes the timestamp an unreliable piece
of data.

## Explanation

The proposal is to formalise the current behaviour as the expected behaviour:
The timestamp entry is the time when the entry was minted, qualifying for
being part of a given register.

This means that the timestamp is **no guarantee** of entry order. The entry
number is. Initial intuition might let you think the timestamp reflects the
natural order of entries, let's describe a situation where this is not
necessarily true.

Imagine a minting system with two machines (e.g. load balanced system,
blue-green deploys, team components with a personal computer each). Every time
the system stamps an entry, it uses the machine clock, which is as correct as
the setup for that machine. So, an entry will always be stamped with the
**local time** from machine stamping it.

Note that **local time** here has nothing to do with time zones. It's about
what's the internal clock and how well synchronised (e.g. NTP).

So, you should expect:

* The timestamp to inform when the entry was minted according to the minting
  system's clock.
* An ordered sequence of entries by entry number can have unordered timestamps.

So, the entry timestamp has similar expectations and meaning as you would
expect from distributed systems like Git.


[entry-res]: http://openregister.github.io/specification/#entry-resource
[issue-48]: https://github.com/openregister/specification/issues/48
