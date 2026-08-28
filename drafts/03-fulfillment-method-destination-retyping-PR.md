TITLE: fix: approximate discriminator array item retyping

BODY:

Fixes #ISSUE_NUMBER

## What this changes

Adds an eighth constraint family to postprocess_models.py:
find_conditional_array_retyping and inject_conditional_array_retyping,
for a discriminator that retypes an array property items to a schema
file different from the property own base ref, described in the linked
issue.

find_conditional_array_retyping scans for the shape mechanically: a
single key const or enum discriminator naming a property on the
enclosing object, and a then that narrows exactly one array property,
also on the enclosing object, to a different items ref. For each match
it resolves the retyped file own root level required keys and const
pinned properties through a new helper, resolve_referenced_shape.
inject_conditional_array_retyping then injects a model_validator that,
for each item in the array field when the discriminator matches, checks
the item carries every required key (through model_fields_set union
model_extra, the same key counting idiom used throughout this module)
and that every const pinned field matches its expected value.

This is deliberately an approximation rather than a full re-derivation
of the retyped type. Only the referenced schema own required and const
fields are checked, not fields it in turn allOf references (for example
shipping_destination.json own allOf ref to postal_address.json is not
inspected). For fulfillment_method.json the meaningful check is
primarily the type const pin, id and type were already required by the
base FulfillmentDestination, so checking them again there is redundant,
but the mechanism itself is general and both required keys and const
pins are read mechanically from whatever the referenced schema
declares, nothing is hardcoded to this one case.

A request variant that omits the retyped field entirely
(fulfillment_method_create_request.json never carries destinations at
all) makes the rule inapplicable rather than malformed, silently,
mirroring find_conditional_bounds existing handling of a stripped
field.

## Test plan

* Added ConditionalArrayRetypingInjectorTest, injector level unit tests
  against synthetic fixtures mirroring fulfillment_method.json exact
  shape, plus FulfillmentMethodDestinationRetypingSemanticTest against
  the real committed models, including negative controls for an open
  vocabulary method type and for a method with no destinations at all.
* Full suite: 101 tests, 0 failures, 4 documented skips.
* Regenerated against release/2026-08-25 as a separate commit from the
  generator fix. One file changes, fulfillment_method.py.
* Regenerated a second time and diffed the two outputs, excluding
  __pycache__. No difference.
* Kill test: reverted postprocess_models.py to its pre fix state,
  regenerated, reinstalled. The same failures reappeared exactly.
  Restored the fix and regenerated again to confirm green.
* pre-commit run on the changed file: clean.

## Not included

README.md, which ruff format also reformats when generate_models.sh
runs, unrelated to this fix and present before this PR. Left untouched
to keep the diff scoped.
