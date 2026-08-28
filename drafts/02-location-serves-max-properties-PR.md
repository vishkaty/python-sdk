TITLE: fix: add the missing maxProperties constraint family

BODY:

Fixes #ISSUE_NUMBER

## What this changes

Adds find_root_max_properties, inject_max_properties, and
_patch_max_properties to postprocess_models.py, mirroring their
minProperties counterparts one for one (same marker guarded
idempotency, same model_fields_set union model_extra key counting
semantics, same free form object exclusion for a bound without declared
properties, already handled natively by the generator through
Field(max_length=...)). Wired into main as an independent patch pass so
both bounds can be injected into the same class without either
clobbering the other, which location_serves.json needs since it
declares minProperties 1 and maxProperties 1 together.

One deliberate difference from the function it mirrors:
find_root_min_properties treats a falsy minProperties (0) as absent,
using not minimum, which is harmless since minProperties 0 permits
everything minProperties absent already does. maxProperties 0 is a real
and different constraint, no properties allowed at all, so
find_root_max_properties checks isinstance(maximum, int) instead of
truthiness. This is new code, not a change to the existing minProperties
function, which is out of scope here.

## Test plan

* Added MaxPropertiesInjectorTest, mirroring the existing InjectorTest
  for minProperties, plus LocationServesMaxPropertiesSemanticTest
  against the real committed model, including a negative control
  proving the existing minProperties check is untouched by this change.
* Full suite: 100 tests, 0 failures, 4 documented skips.
* Regenerated against release/2026-08-25 as a separate commit from the
  generator fix. Three files change: LocationServes,
  LocationServesCreateRequest, LocationServesUpdateRequest.
* Regenerated a second time and diffed the two outputs, excluding
  __pycache__. No difference.
* Kill test: reverted postprocess_models.py to its pre fix state,
  regenerated, reinstalled. The same failures reappeared exactly.
  Restored the fix and regenerated again to confirm green.
* pre-commit run on the changed files: clean.

## Not included

README.md, which ruff format also reformats when generate_models.sh
runs, unrelated to this fix and present before this PR. Left untouched
to keep the diff scoped.
