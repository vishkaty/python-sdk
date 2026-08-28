TITLE: bug: location_serves.json maxProperties is never enforced

BODY:

## Observed versus expected

location_serves.json (source, release/2026-08-25,
source/schemas/common/types/location_serves.json lines 6 and 7) declares
both minProperties 1 and maxProperties 1 on the same object. The
description states the rule plainly: "The Platform MUST supply exactly
one target form."

Only the minimum is enforced on current main (b6f9b91c,
src/ucp_sdk/models/schemas/common/types/location_serves.py). The
generated LocationServes model has an enforce min properties validator
but nothing checking the maximum, so this currently passes:

```python
from ucp_sdk.models.schemas.common.types.location_serves import (
    LocationServes,
)
from ucp_sdk.models.schemas.common.types.geo import Geo
from ucp_sdk.models.schemas.common.types.location_serves import Address

LocationServes(
    point=Geo(latitude=1.0, longitude=2.0),
    address=Address(address_country="US"),
)
# validates, though the schema allows exactly one of point or address
```

## Why this was not caught by CI

Same root cause as the sibling issue filed alongside this one:
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

find_root_min_properties (postprocess_models.py) scans the preprocessed
schemas for a root level minProperties constraint on an object with
declared properties, and injects a matching validator. That function was
added in #55 for #49. It never grew a maxProperties counterpart. There is
no find_root_max_properties in the module at all, so location_serves.json
maxProperties 1 was unenforced from the day minProperties support landed,
alongside its own minProperties 1 on the same schema, which was caught.

## Offer

A fix is ready (branch fix/location-serves-max-properties against
current main): find_root_max_properties and inject_max_properties,
mirroring the existing minProperties functions one for one, with a
failing test added first, the generator level fix in
postprocess_models.py only, and a regeneration against
release/2026-08-25 as a separate commit. Both bounds coexist on the same
class without either clobbering the other. Happy to open the PR
alongside this issue if that is welcome.
