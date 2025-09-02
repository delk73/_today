# Tuesday September 2, 2025

## Plan
- Refactor synesthetic schemas submodule into backend




What We Lost (Shader)

Uniform stage enum: not enforced as {'vertex','fragment'}; any string now passes.
Uniform type enum: not enforced (e.g., 'float','vec2','vec3','vec4','int','bool'); any string now passes.
Non-empty shader sources: vertex_shader and fragment_shader no longer validated as non-empty.
InputParameter range checks: no min < max, and no “default within [min,max]” guard.
OpenAPI precision: enums/validators above no longer surface in the docs, so client generation is less constrained.
Test signals: cases that used to fail-fast with 422 (invalid uniform stage/type, empty shader source) now succeed.
What Stayed Intact

Top-level strictness: external Shader keeps extra = forbid (e.g., we removed shader_lib_id).
Field typing: basic types are still enforced (strings, lists, numbers).
Proposed Upstream Additions (to restore SSOT constraints)

UniformDef.stage: enum {'vertex','fragment'}.
UniformDef.type: enum {'float','int','bool','vec2','vec3','vec4'} (extend if needed).
Shader.vertex_shader/fragment_shader: non-empty string (min_length=1).
InputParameter:
type enum {'float','int','bool','vec2','vec3','vec4'}.
validator: min < max when both provided.
validator: default within [min,max] for numeric types.
JSON Schema examples for the above (to improve FE dev-experience and OpenAPI).
Interim Backend Posture

Requests: validated by synesthetic_schemas (current behavior).
Responses: API model includes DB ids (e.g., shader_id) but avoids reintroducing stricter local validators.
Optional runtime guardrails (kept minimal): version header check; we can add compile/sanity checks in services without diverging from SSOT.


Progress update: migrated Controls and Tones to use synesthetic_schemas for request validation, kept API responses stable with IDs, and updated tests accordingly. All tests remain green on your side.

What changed

Controls

Router: app/routers/controls.py
Requests: validates with synesthetic_schemas.control_bundle.ControlBundle
Responses: unchanged, still use local ControlResponse (includes control_id)
Tests: tests/unit/test_controls.py
Import now from synesthetic_schemas.control_bundle
Relaxed local-only constraints not present in SSOT (external schema):
No min < max enforcement at schema level
No default-in-range enforcement
Combo can lack explicit input method (keys/mouseButtons/wheel)
String options are optional; default not required to be in options
Still rejects invalid axis via enum (e.g., “mouse.x” vs invalid)
Tones

Router: app/routers/tones.py
Requests: validates with synesthetic_schemas.tone.Tone
Responses: unchanged (local schemas.Tone including tone_id)
No test changes required; existing payloads align with external Tone.
Why this is safe

We keep frontend-facing response shapes stable by preserving ID fields in responses.
Requests now align to the shared submodule (SSOT), reducing divergence between FE/BE.
We’ve documented lost local validations (for shader earlier, and controls here) so we can propose reintegrating them upstream in synesthetic_schemas later.
What we’re explicitly losing (Controls)

Required input method in combo (local validator); external allows stricter or looser combos.
Numeric range constraints (min < max, default within range) enforced at schema level.
Options presence/containment checks for string type at schema level.
OpenAPI no longer shows the above as schema constraints, so FE generators won’t enforce them without client-side checks.
Mitigations (not done yet, optional)

Add service-level validators if we need to keep these behaviors temporarily without violating SSOT.
Prefer upstreaming constraints to synesthetic_schemas after the migration settles.

Changes made

Tones

app/routers/tones.py
Requests: use synesthetic_schemas.tone.Tone for creation (kept).
List: removed redundant schemas.Tone(...) validation inside the loop to prevent 500s.
Responses: still return local schemas.Tone (with tone_id) to keep IDs and stable shape.
Controls

app/routers/controls.py
Requests: use synesthetic_schemas.control_bundle.ControlBundle (kept).
Responses: switched from ControlResponse (strict) to a new permissive model.
app/schemas/control_api.py
New ControlAPIResponse: includes control_id, and represents control_parameters as a list of dicts to avoid strict validation clashes.
Tests

tests/unit/test_controls.py (updated earlier)
Now imports from synesthetic_schemas.control_bundle and no longer expects local-only validations.
Expected outcomes

test_invalid_synth_type: external Tone enum still rejects invalid synth types with 422.
get tones schema validation: no 500s due to internal re-validation; test continues to validate items externally.
controls router lifecycle + pagination: responses validate against the new API response model and include control_id.
Notes on strictness

This is a known trade-off in the migration: requests now align to synesthetic_schemas (SSOT), while responses remain stable and include IDs via API-specific wrappers. We’ve avoided reintroducing stricter validators in responses that could contradict the SSOT.
We documented stricter checks we removed so we can propose adding them upstream in synesthetic_schemas after the full migration.



We switched request models to synesthetic_schemas.control_bundle (SSOT). Those models use Enums (e.g., DataType, AxisType).
In the CRUD create/update path, we were using payload.model_dump() (python mode), which can leave Enum members in the resulting dict.
SQLAlchemy’s JSON column then tries to persist Enum objects in nested control_parameters and fails, bubbling up as a 500 Internal Server Error.
Fix

app/utils/crud_router.py
Use payload.model_dump(mode="json") for create and for update (with exclude_unset), so Enums serialize to their values before DB insert/update.
This aligns all nested structures with JSON-serializable types.
Related stabilizers already in place

Control-specific normalization: defaults combo fields (wheel=False, keys=[], mouseButtons=[]) to avoid nulls in DB JSON.
Control API response shaping: returns a clean dict via ControlAPIResponse, using format_control_response to normalize nested data and including control_id.
Unit test assertion for control type compares enum.value.
Outcome

The two failing control tests should now pass:
tests/routers/test_controls_router.py::test_full_control_lifecycle
tests/routers/test_controls_router.py::test_controls_list_pagination
Note on example import warnings

The duplicate control names in synesthetic asset examples trigger 409s (unique constraint). That’s expected and logged during the importer; not part of the router tests.






Immediate Next Steps

Controls/Tones Review: Confirm no remaining code relies on old local validators; keep API wrappers for IDs only.
Synesthetic Assets (requests): Switch nested components to synesthetic_schemas (Tone, Shader, Haptic, Modulation, RuleBundle, ControlBundle). Keep existing response wrappers with IDs.
Services: Ensure asset_utils uses external models for all nested serialization (we updated RuleBundle; extend for other nested components if needed).
Header Checks: Extend optional X-Schema-Version validation to controls, tones, and synesthetic-assets routers for consistency.
Examples/Loader: Keep shim that strips deprecated fields; consider light idempotency (skip/ignore duplicates by name for controls during importer to reduce noise).
Cleanup & Pruning

Prune migrated local schemas:
Remove local: control.py, tone.py, shader.py, modulation.py, synesthetic_asset.py, rule_bundle.py (once their usages are fully replaced).
Keep local-only: compute_shader.py, vectors.py, embedding.py, patch_index.py, patch.py, lab.py, mcp/, error.py, ShaderLib-related.
Update app/schemas/__init__.py exports to reflect removals.
Replace any remaining from app.schemas import … in app code and tests for migrated models.
Validation & CI

OpenAPI sanity: Check docs show external schema shapes for requests; ensure API wrappers return IDs only.
Tests: Run ./codex.sh locally; if feasible, also run full ./test.sh to surface Postgres/Redis edge cases.
CI readiness: When adding workflows, enable submodules checkout (actions/checkout@v4 with submodules: 'recursive').
Upstream Alignment (after migration is stable)

Tighten synesthetic_schemas:
Shader: enums for uniform stage/type; non-empty source; InputParameter type/range checks.
Controls: optional validators (combo input presence, numeric ranges, string options).
Bump submodule version.json; update backend submodule and rerun tests.
Remove any temporary service-level checks if upstreamed.
Frontend Coordination

Communicate changes:
Removed shader_lib_id in shader requests/responses; linkage handled separately.
Requests now follow synesthetic_schemas; responses remain stable with IDs.
Optional X-Schema-Version header available for client gating.
FE to add client-side validations where the external schema is permissive (e.g., uniform stage/type).








- Refactor synesthetic schemas submodule into frontend






## Meetings
-

## Log
-

# Review
    - Highlights:
    - Roadblocks:
    - Next steps: