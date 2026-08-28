TITLE: fix: restore JWK conditional rules and unit.json scale pin

BODY:

Fixes #ISSUE_NUMBER

## What this changes

Two scanner bugs in postprocess_models.py were dropping all five of
profile.json jwk_public_key if/then rules and the unit.json C62 scale
pin, described in the linked issue. This restores all six.

1. find_conditional_required now threads the enclosing object properties
   into an allOf branch that carries no properties of its own, the same
   way find_conditional_bounds already does. Every JWK required field
   rule is this shape.

2. find_conditional_bounds now recognizes a bare const constraint in a
   then.properties.<field> clause, not only the four numeric bound
   keywords. A const entry is added to _BOUND_KEYWORDS mapped to not
   equal, reusing the existing value versus limit comparison template
   unchanged.

3. Both scanners now decline to adopt a bare if/then branch own title as
   the enclosing class name. A new helper,
   is_bare_conditional_branch, recognizes a node that carries if and
   then but no properties of its own as rule documentation rather than a
   type, so its title is never mistaken for a generated class name. A
   node that carries both a conditional rule and its own properties (a
   titled type with an inline conditional) keeps adopting its title as
   before.

No hardcoded schema knowledge was added. Every rule is still derived
mechanically from the schema tree, the same as the existing conditional
required and conditional bounds families.

## Also included

The tests/test_codegen_pipeline.py HAVE_SDK import gate cited
ucp_sdk.models.schemas.shopping.types paths for Description and Totals
that moved to common.types in #87. That silently skipped about a third
of the suite instead of running it, hiding the six failures this PR
fixes. Corrected every stale shopping.types reference in the file
(description, totals and its request variants, signals and its request
variants, error_response), which moved under the same class names and
now pass unmodified. Two targets did not survive the schema restructure
at all, card_payment_instrument Constraints (a uniqueItems brands field)
and merchant_fulfillment_config nested additionalProperties false object
(now business_fulfillment_config, reshaped), so their four tests became
a documented unittest.skip with the reason stated in the schema, rather
than a silent deletion.

## Test plan

* Added JwkConditionalRulesSemanticTest and UnitScaleSemanticTest against
  the real committed models, plus injector level unit tests for both
  scanner fixes against synthetic fixtures mirroring the schema shape.
* Full suite: 109 tests, 0 failures, 4 documented skips.
* Regenerated against release/2026-08-25 (./generate_models.sh
  2026-08-25) as a separate commit from the generator fix.
* Regenerated a second time and diffed the two outputs, excluding
  __pycache__. No difference.
* Kill test: reverted postprocess_models.py to its pre fix state,
  regenerated, reinstalled. The same failures reappeared exactly.
  Restored the fix and regenerated again to confirm green.
* pre-commit run on the changed files: clean.

## Not included

README.md, which ruff format also reformats when generate_models.sh
runs (a spacing difference in an embedded code block, unrelated to this
fix and present before this PR). Left untouched to keep the diff scoped.
