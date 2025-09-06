# Tuesday September 5, 2025


## Caryover resume, - blocks removed


## Plan

-establish baseline on sdfk-backend refactor

-draft implemenmtation plan

-execute implementation  plan



## Meetings
-

## Log
- cleaned up key issues
- dev and test for nix and poetry and node setup in schemas
- ci implemeted green
- pushed nix setup to main


# Review
    - Highlights:
    - Roadblocks:
    - Next steps:









**OLD NOTES**




What I fixed

Asset import source: Examples now come from the SSOT by default (libs/synesthetic-schemas/examples). Local app/examples stays only for comparison or explicit override via EXAMPLES_DIR.
Asset schema for validation: Switched SynestheticAsset validation to the external synesthetic_schemas.synesthetic_asset.SynestheticAsset. This aligns the loader with the exact request model your /synesthetic-assets/nested route expects, avoiding drift between a local nested model and the SSOT examples.
No derived posts: Removed the control/modulation extraction and standalone posts from derived data to avoid the ModulationItem serialization mess and reduce duplication.
Payload posting: We still post the raw dicts and only strip $schemaRef before sending. We don’t pass Pydantic instances to JSON insertion, eliminating enum/object serialization problems.
What remains (and how to finish cleanly)

Duplicates: Standalone component examples (Shader/Tone/Control/Haptic) in the SSOT are still posted because tests rely on them being imported. If you want a pure asset-only import during local runs, I can add a small env gate you can flip in codex.sh, for example:
ONLY_ASSETS=1 → loader uses only the asset pattern and skips standalone components.
Zero assets symptom you saw: That happened in the unit test scenario because EXAMPLES_DIR is set to a temp dir that does not contain SynestheticAsset_Example*.json. In full runs against SSOT, assets should now import via the external schema path. If you still see 0 assets with SSOT examples available, tell me which exact command/environment is used and I’ll instrument the loader to print:
examples_dir being used
matched files for SynestheticAsset pattern
first post status for nested endpoint
If you want, I’ll add the env toggle (ONLY_ASSETS) in the loader and wire codex.sh to set it so the local runner imports assets-only from SSOT, and leaves standalone examples out — while tests keep using their own EXAMPLES_DIR fixture.



- Refactor synesthetic schemas submodule into backend

** COMPLETE **


# Backend Interrogation — synesthetic_schemas Integration

- Objective: Audit SDFK Backend for clean SSOT (synesthetic-schemas) integration with CRUD-only scope, stable response envelopes, version header policy, schema pruning, and CI parity.
- Constraints: KISS, deterministic, Python 3.11, no DB migrations.

## B1. Patch/Preview Removal (CRUD-only)

- Findings:
  - No `/patches` router; file `app/routers/patches.py` removed.
  - No preview/apply endpoints in `app/routers/synesthetic_assets.py`; only CRUD + nested read remain.
  - Patch storage helpers removed from `app/main.py`; no `get_ring_buffer_patch_storage`.
  - Tests referencing preview/apply updated/disabled accordingly; ring buffer fixture removed.
- Verification:
  - Search: `rg -n "patch|preview|ring_buffer|get_ring_buffer" app tests`
  - Result: Only docstrings/comments remain; no functional references.

## B2. SSOT Requests & Envelope Responses

- Findings:
  - Requests validate via SSOT models:
    - SynestheticAsset create: `app/routers/synesthetic_assets.py` imports `synesthetic_schemas.synesthetic_asset.SynestheticAsset`.
    - Controls, Tones, Shaders, Haptics, Modulations: factories use SSOT create schemas via `app/routers/factory.py` and router-specific modules (`app/routers/shaders.py`).
  - Responses preserve IDs and use `.model_dump(mode="json")` in formatters:
    - `app/services/asset_utils.py` uses SSOT models for nested serialization and dumps with `mode="json"` (fixes enum serialization).
    - Ensures list-typed fields are `[]`, not `null`, to satisfy response validation.
    - Nested asset formatting omits nested component IDs (e.g., drops `shader_id` inside `shader`).
- Verification:
  - Search: `rg -n "from synesthetic_schemas" app/routers app/services`
  - Search: `rg -n "model_dump\(mode=\"json\"\)" app`

## B3. Version Header Consistency

- Findings:
  - `X-Schema-Version` optional dependency added to all routers (controls, tones, shaders, shader_libs, embeddings, search, protobuf-assets, synesthetic-assets). Absent header allowed; mismatched header → 409 Conflict.
  - `/schema/version` endpoint returns the version from `libs/synesthetic-schemas/version.json`.
- Verification:
  - Search: `rg -n "X-Schema-Version" app`

## B4. Local Schema Pruning

- Findings:
  - Local request models replaced by SSOT across routers.
  - Local response wrappers retained to preserve API IDs and shapes (e.g., `ShaderAPIResponse`, `SynestheticAssetResponse`, `NestedSynestheticAssetResponse`).
  - Remaining local schemas are wrappers, error/embedding models, and MCP/compute-related definitions.
- Verification:
  - Search: `rg -n "from app\\.schemas" app/routers app/services | rg -v "mcp|error|embedding|patch_index|compute_shader|ShaderLib"`

## B5. Contracts & Test Parity

- Findings:
  - Response payloads stabilized with IDs; nested responses validated by `NestedSynestheticAssetResponse`.
  - Normalization added to CRUD factory to ensure list fields are arrays (not null) for Tone/Haptic/Modulation.
  - Protobuf asset export fixed to pass `modulations` list (not the Modulation row) into the exporter.
- Status:
  - Local SQLite suite: expected to pass after changes; full Docker `./test.sh` should pass after environment is available.
- Notes:
  - If any failures persist, re-check nested response list fields and ensure control bundle mappings use arrays for `keys`, `mouseButtons`, `wheel` coercions.

## B6. CI & Docs Hygiene

- Findings:
  - Docs should reflect CRUD-only endpoints; preview/apply removed.
  - README/docs mention `X-Schema-Version` policy and `/schema/version`.
  - CI should ensure submodules are present (recursive checkout) before tests.
- Verification:
  - Searches to run in CI: `rg -n "/patch" docs/api/endpoints.md`, `rg -n "schemaVersion" README.md docs`, `rg -n "submodules:.*recursive" .github/workflows`.

## Summary

- CRUD-only enforcement complete; patch infrastructure removed.
- Requests validate against `synesthetic-schemas`.
- Responses keep IDs; nested payloads conform and serialize with `.model_dump(mode="json")`.
- Version header dependency consistent; `/schema/version` available.
- Local schemas pruned to wrappers.
- Test parity addressed through payload normalization and exporter fix.
- Action Items:
  - Run `./codex.sh` and `./test.sh` in CI to confirm green.
  - Update docs to remove patch references and document version policy if not already.






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

Controls/Tones aligned
Requests: external synesthetic_schemas models.
Responses: API models with IDs; no extra strict validators.
Fixed enum serialization to JSON before DB writes/updates to avoid 500s.
Added X-Schema-Version check for controls and tones.








- Refactor synesthetic schemas submodule into frontend


