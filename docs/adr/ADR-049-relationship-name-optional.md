# ADR-049: Relationship.name defaults to empty string

**Status:** ACCEPTED
**Date:** 2026-04-27
**Scope:** Pydantic surface of `etcion.metamodel.concepts.Relationship` and all concrete relationship subclasses.

## Context

Until v0.11.0, `Relationship` (and every concrete subclass: `Serving`, `Composition`, `Aggregation`, `Access`, etc.) inherited `name: str` from `AttributeMixin` with no default. The field was therefore *required* by Pydantic, even though the validator accepted the empty string. This produced two related defects:

- **Ergonomic friction (#104).** Constructing a relationship that semantically has no name -- the common case for most relationship types -- forced callers to write `Serving(source=a, target=b, name="")`. The boilerplate added no validation value because `""` was always accepted.
- **CSV round-trip break (#102).** `to_csv` writes an empty cell for an unnamed relationship; `from_csv` interpreted blank cells as "key absent" and never forwarded `name=""` to the constructor, so the round-trip raised `ValidationError: name Field required`.

The ArchiMate Exchange Format XSD makes `<name>` optional on `<relationship>` (the existing XML serializer already guards with `if rel.name:` at `serialization/xml.py:118`), so the required-field model surface was inconsistent with the format we serialize to.

Link: #102, #104

## Decision

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | Override `name: str = ""` on `Relationship` (concepts.py) | Removes the boilerplate, fixes the round-trip, aligns the metamodel with the Exchange Format spec. |
| 2 | Do **not** propagate the default to `Element.name` | Elements without names are a smell in ArchiMate; the spec treats `<name>` on elements as effectively required. |
| 3 | Keep `AttributeMixin.name: str` (no default) unchanged | Avoids accidentally weakening the Element contract; only `Relationship` overrides. |
| 4 | Fix the CSV adapter symmetrically (issue #102) | The model-level default alone leaves an asymmetric `to_csv`/`from_csv` contract that future fields would re-trigger. |

## Alternatives Considered

| Alternative | Rejected Because |
|-------------|-----------------|
| Add a Pydantic validator that *rejects* `""` and keep the required-field schema | Breaks every existing model and contradicts the Exchange Format which permits unnamed relationships. |
| Default `name=""` on `AttributeMixin` (Element + Relationship together) | Weakens the Element contract for no benefit; ArchiMate elements should carry names. |
| Fix only the CSV adapter (#102) and leave the model surface alone | Closes the symptom but not the cause; #104 remains and every other adapter (DataFrame, Parquet, DuckDB, future JSON) hits the same trap. |

## Consequences

- **Behavior change, not a bug fix.** Existing user code that relied on `ValidationError` to catch missing names on relationships will now silently accept them. Documented in `CHANGELOG.md` under a "Behavior change" heading.
- **Public API surface change.** `Relationship.name` becomes optional. mypy / IDE autocomplete will see the change in stubs. Existing callers passing `name=...` continue to work unchanged.
- **XML and other serializers unaffected.** `serialize_relationship` already guards with `if rel.name:` and emits no `<name>` element when the name is empty -- the on-disk format is unchanged.
- **Element contract preserved.** `Element.name` remains required.
