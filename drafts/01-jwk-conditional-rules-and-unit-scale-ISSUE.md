TITLE: bug: profile.json JWK conditional rules and unit.json C62 scale pin are dropped by the generator

BODY:

## Observed versus expected

profile.json describes a jwk_public_key definition with five if/then
rules under allOf (source, release/2026-08-25,
source/schemas/profile.json lines 104 to 192): an EC key must carry crv,
x, and y; an OKP key must carry crv and x; and a curve pins its
algorithm (P-256 with ES256, P-384 with ES384, Ed25519 with EdDSA). The
description on crv in the same file calls the algorithm pairing out
directly: "When present for a well known curve it MUST match: ES256 with
P-256, ES384 with P-384, EdDSA with Ed25519."

None of the five rules reach the generated JwkPublicKey model on current
main (b6f9b91c, src/ucp_sdk/models/schemas/profile.py). The class has no
validator at all, so this currently passes:

```python
from ucp_sdk.models.schemas.profile import JwkPublicKey

JwkPublicKey(kid="k1", kty="EC")
# validates, though the schema requires crv, x, and y on an EC key

JwkPublicKey(kid="k2", kty="EC", crv="P-256", x="AA", y="BB", alg="EdDSA")
# validates, though the schema pins alg to ES256 for a P-256 curve
```

A profile that publishes a mismatched or incomplete key like this passes
SDK validation and would only surface a problem later, at signature
verification time against the actual key material.

Separately, unit.json (same release,
source/schemas/common/types/unit.json lines 20 and 27 to 44) pins scale
to exactly 0 when unit is C62. That is dropped too:

```python
from ucp_sdk.models.schemas.common.types.unit import Unit

Unit(unit="C62", scale=5, display_text="pieces")
# validates, though the schema requires scale to be 0 for C62
```

## Why this was not caught by CI

tests/test_codegen_pipeline.py gates most of its suite behind a HAVE_SDK
flag, set by importing Description and Totals from
ucp_sdk.models.schemas.shopping.types (lines 34 to 41). Those paths moved
to ucp_sdk.models.schemas.common.types when #87 restructured the schema
tree on 2026-08-25 (merged 2026-08-27, one approving review from a
contributor who had also just pushed a commit to the same PR, no
comments left on the diff). The stale import raises ModuleNotFoundError,
which the surrounding except catches and sets HAVE_SDK to False, so
every test gated on it skips rather than fails. unittest reports a skip
as passing, so the suite reads green while a large share of it never
runs. This affected 33 of 89 collected tests before the paths were
corrected, including every semantic test that would have caught the two
gaps described above.

## Root cause

Two independent bugs in the postprocessing scanners that would otherwise
restore these rules.

First, find_conditional_required (postprocess_models.py, function
starting at line 696) only looks at the properties key belonging to the
branch itself. Every JWK required field rule is carried as an allOf
branch with just if and then, no properties of its own, since the
object it constrains is the enclosing jwk_public_key. The branch is
invisible to the scan as a result, silently, with no warning.
find_conditional_bounds already threads the enclosing object properties
into such branches; the sibling function never picked up that fix.

Second, find_conditional_bounds (function starting at line 827) only
recognizes four numeric bound keywords in _BOUND_KEYWORDS (line 169):
minimum, maximum, exclusiveMinimum, exclusiveMaximum. A bare
{"const": ...} constraint in a then.properties.<field> clause falls
outside that set, so the whole rule is rejected as an unsupported shape
and dropped. Both the unit.json scale pin and all three JWK curve and
algorithm pairings are const shaped, not numeric bounds.

A third, shared issue affects both scanners once the first two are
fixed. Several of the JWK branches carry their own human readable title
(for example, "EC keys carry crv, x, y"), and both scanners currently
adopt any node title as the enclosing class name. Applied to a bare
rule branch, that overwrites the correct class name (JwkPublicKey) with
an alias derived from the rule documentation string, misattributing the
rule to a class that does not exist.

## Offer

A fix for all three is ready (branch
fix/jwk-conditional-rules-and-unit-scale against current main), with
failing tests added first, a generator level fix in
postprocess_models.py only, and a regeneration against
release/2026-08-25 as a separate commit. The full suite passes with the
same four documented skips the corrected import gate leaves. Happy to
open the PR alongside this issue if that is welcome.
