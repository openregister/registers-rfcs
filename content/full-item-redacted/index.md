---
rfc: 0017
start_date: 2018-08-23
decision_date: 2018-09-25
pr: openregister/registers-rfcs#30
status: approved
---

# Full item redaction

## Summary

This RFC proposes removing the old partially defined item redaction solution
in favour of [RFC0010: Item
hash](https://github.com/openregister/registers-rfcs/pull/24).


## Motivation

The specification defines “redaction” as:

> An item may be removed from a register, but the entry MUST NOT be removed
> from the register.

In some buried decision log and as a known thing for some components of the
Registers core team, a while ago it was decided that to fully redact an item,
the HTTP response would be a `410 Gone`. The specification doesn't reflect
this decision nor the details.

After some effort trying to define how this solution works for items and
records I have came to the conclusion that it is a broken solution. The
fundamental problem lays on the fact that a record is an entry with the item
embedded. In JSON it is possible to show that the item is redacted (or some of
them if the entry has many items). In CSV it's not possible without changing
the column structure which would cause problems to any user consuming records
expecting a particular column structure.

Worth mentioning that `410 Gone` can't be applied to a record because it is an
entry and these never get removed or redacted.


## Explanation

A fully redacted blob MUST be done by redacting every value according to
RFC0010.

Both item and record resource MUST return a `200 OK` response serialised with
the requested content type if available.

***
**EXAMPLE:**

For example, a redacted blob in JSON:

```http
GET /items/sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "country": "**REDACTED**fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21",
  "official-name": "**REDACTED**bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5",
  "name": "**REDACTED**94099b1e0b9a1e673bafee513080197fa1980895ca27e091fdd4c54fab2bed24",
  "citizen-names": "**REDACTED**e6ec7637c8df22f85b49d91ac98a37c96372cf3328e1b5aa96e00af1658f0dc0"
}
```

And in CSV:

```http
GET /items/sha-256:6b18693874513ba13da54d61aafa7cad0c8f5573f3431d6f1c04b07ddb27d6bb HTTP/1.1
Host: country.register.gov.uk
Accept: text/csv;charset=UTF-8
```

```http
HTTP/1.1 200 OK
Content-Type: text/csv;charset=UTF-8

country, official-name, name, citizen-names
**REDACTED**fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21, **REDACTED**bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5, **REDACTED**94099b1e0b9a1e673bafee513080197fa1980895ca27e091fdd4c54fab2bed24, **REDACTED**e6ec7637c8df22f85b49d91ac98a37c96372cf3328e1b5aa96e00af1658f0dc0
```

Similarly, a record would be redacted like:

```http
GET /records/GB HTTP/1.1
Host: country.register.gov.uk
Accept: application/json
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "GB": {
    "index-entry-number": 6,
    "entry-number": 6,
    "entry-timestamp": "2016-04-05T13:23:05Z",
    "key":"GB",
    "item":[
      {
        "country": "**REDACTED**fff7021c7df4426be0f9a3c83f236eb6f85d159e624b010d65e6dde267889c21",
        "official-name": "**REDACTED**bf1860175c77869938cf9f4b37edb00f2f387be7b361f9c2c4a2ac202c1ba2e5",
        "name": "**REDACTED**94099b1e0b9a1e673bafee513080197fa1980895ca27e091fdd4c54fab2bed24",
        "citizen-names": "**REDACTED**e6ec7637c8df22f85b49d91ac98a37c96372cf3328e1b5aa96e00af1658f0dc0"
      }
    ]
  }
}
```
