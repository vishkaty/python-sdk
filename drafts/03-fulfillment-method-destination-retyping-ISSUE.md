TITLE: bug: fulfillment_method.json destination retyping per method type is dropped

BODY:

## Observed versus expected

fulfillment_method.json (source, release/2026-08-25,
source/schemas/shopping/types/fulfillment_method.json) declares a base
destinations array typed via items pointing at
fulfillment_destination.json (line 39), a bare type and id. Two allOf
branches then retype destinations per the method own type: when type is
shipping, destinations items point at shipping_destination.json instead
(lines 81 to 90, postal address fields, type const shipping_address);
when type is pickup, destinations items point at
location_destination.json instead (lines 105 to 114, type const
business_location).

The generated FulfillmentMethod model on current main (b6f9b91c,
src/ucp_sdk/models/schemas/shopping/types/fulfillment_method.py) keeps
destinations typed to the base FulfillmentDestination regardless of
type, so this currently passes:

```python
from ucp_sdk.models.schemas.shopping.types.fulfillment_method import (
    FulfillmentMethod,
)
from ucp_sdk.models.schemas.shopping.types.fulfillment_destination import (
    FulfillmentDestination,
)

FulfillmentMethod(
    id="m1",
    type="shipping",
    line_item_ids=["li1"],
    destinations=[FulfillmentDestination(type="business_location", id="d1")],
)
# validates, though a shipping method destination should carry
# type shipping_address, not business_location
```

## Why this was not caught by CI

Same root cause as the sibling issues filed alongside this one:
tests/test_codegen_pipeline.py gates most of its suite behind a HAVE_SDK
flag, set by importing Description and Totals from
ucp_sdk.models.schemas.shopping.types (lines 34 to 41 on current main).
Those paths moved to ucp_sdk.models.schemas.common.types when #87
restructured the schema tree on 2026-08-25, so the import raises
ModuleNotFoundError and every test gated on HAVE_SDK skips rather than
fails. unittest reports a skip as passing, so the suite reads green
while a large share of it, including every semantic test that would
have caught this gap, never runs.

## Root cause

No scanner in postprocess_models.py has ever looked for this shape. The
module handles a discriminator that adds required fields
(find_conditional_required) and one that narrows a numeric range
(find_conditional_bounds), but a discriminator that retypes an array
property items to a schema file different from the property own base
ref is a third, distinct shape, and nothing in the module scans for it.
The two retyping branches were confirmed at
generate_models.sh run time to fall through to
find_conditional_bounds, which correctly declines them as an
unsupported bounds shape (a stderr warning, "unsupported conditional
bounds rule, skipped") rather than silently accepting them, but nothing
else picks them up in its place.

## Offer

A fix is ready (branch
fix/fulfillment-method-destination-retyping against current main). It
does not attempt to retype the field statically, pydantic has no clean
way to do that from a source text splice the way the other families
rewrite an annotation or add a validator against a field own declared
type, so it enforces the constraint with a runtime check instead: each
destination item is checked against the retyped schema own root level
required keys and const pinned properties (shipping_destination.json
and location_destination.json each declare a type const), an
approximation rather than a full re-derivation of the retyped type.
Failing tests added first, the generator level fix in
postprocess_models.py only, and a regeneration against
release/2026-08-25 as a separate commit. Happy to open the PR alongside
this issue if that is welcome.
