# Wednesday September 10, 2025

## Plan

- Add Backend CI Implementation Summary

    ### Backend CI Implementation Summary

    #### Test Harness (`test.sh`)

    *   **Purpose**: Canonical entrypoint for running tests in CI.  
    *   **Mode**: Container-centric — spins up Docker services (Postgres, Redis, app containers).  
    *   **Flow**:

        1.  Builds `sdfk-backend-web` and `sdfk-backend-trainer` images.  
        2.  Configures runtime sysctls for Redis (e.g., `vm.overcommit_memory`, `somaxconn`).  
        3.  Starts services via `docker-compose` (or equivalent).  
        4.  Runs the test suite (`pytest`, integration tests, etc.) inside the containers.  

    This approach makes `test.sh` self-contained: local dev and CI both use it, ensuring reproducibility.

    ---

    #### Preflight Guardrails (`scripts/preflight_guardrails.sh`)

    *   **Purpose**: Run checks before or alongside `test.sh`.

    *   **Checks Included**:

        *   **Version SSOT**: Frontmatter in README/docs matches `libs/synesthetic-schemas/version.json`.  
        *   **Dependency Pin Consistency**: Aligns root Poetry vs. schemas/python/pyproject.toml.  
        *   **Lint & Type Surfacing**: Runs `ruff` and `pyright` (non-blocking locally, strict in CI).  
        *   **No blanket `# noqa` / `# type: ignore`**.  
        *   **Deprecation Guard**: Flags `jsonschema.RefResolver`.  
        *   **Codegen Drift Check**: Runs `libs/synesthetic-schemas/scripts/ensure_codegen_clean.sh` **only to confirm vendored code is fresh**.  
        *   **Legacy Example Audit**: Flags old “auto-converted” bundle notes.

    *   **Execution Modes**:

        *   **Local dev**: Non-blocking, surfaces warnings.  
        *   **CI**: Enforced strict (`STRICT_GUARDRAILS=1 STRICT_CODEGEN=1`).  

    ---

    #### Current CI Workflows

    *   **`.github/workflows/preflight-guardrails.yml`**:

        *   Runs on PRs and pushes to `main`.  
        *   Steps:

            1.  Check out repo (with submodules).  
            2.  Install Python runtime deps (`requirements.txt`).  
            3.  Install dev tooling (pinned `ruff`, `pyright`).  
            4.  Run `scripts/preflight_guardrails.sh` in strict mode.  

    *   **Main test CI** (not fully expanded here, but implied by `test.sh`):  
        *   Builds containers, runs integration/unit tests inside them.  
        *   Uses Docker services (Postgres/Redis) instead of local Poetry/Nix installs.  
        *   Enforces green test suite before merge.  

    ---

    #### The Disconnect

    *   **Schemas Repo Expectation**:  
        It owns codegen. `ensure_codegen_clean.sh` ensures checked-in artifacts are consistent.  

    *   **Backend CI Reality**:  
        Backend preflight tries to *re-run codegen* in schemas, which requires bootstrapping schemas’ dev env. That drags in fragile deps.  

    *   **Why It’s Fragile**:  
        Backend shouldn’t generate at all — just validate the committed outputs. The single source of truth must remain in the schemas repo.  

    ---

    #### Key Takeaway

    Backend CI has two layers:  

    1.  **Containerized `test.sh`** → isolated, reproducible app/runtime testing.  
    2.  **Host-level `preflight_guardrails.sh`** → validation only (no schema generation).  

    Fix = change backend guardrails so they **only check for drift** in vendored schemas artifacts.  
    If drift exists → fail and instruct developer to regenerate in the schemas repo, commit, and bump submodule.  

---

## Backend CI ↔ Schemas CI Integration

### Current State (Broken)

```mermaid
flowchart TD

    subgraph Backend_CI["Backend CI (sdfk-backend)"]
        A1["Checkout repo + submodules"]
        A2["Run preflight_guardrails.sh"]
        A3["Run test.sh (containerized)"]
    end

    subgraph Guardrails["Preflight Guardrails"]
        G1["Version SSOT check"]
        G2["Dependency pin check"]
        G3["Lint/Type surfacing (ruff, pyright)"]
        G4["❌ Codegen cleanliness tries to regenerate schemas"]
    end

    subgraph Schemas_CI["Schemas CI (synesthetic-schemas)"]
        S1["Poetry/Nix dev env (datamodel-code-generator installed)"]
        S2["Schema normalization & validation"]
        S3["✅ Codegen generation + drift check"]
    end

    %% Backend Flow
    A1 --> A2 --> Guardrails
    A2 --> A3

    %% Guardrails Flow
    G1 --> G2 --> G3 --> G4
    G4 -.->|fragile call| Schemas_CI

    %% Disconnect
    N1["Backend CI mistakenly attempts to run schemas' generation tools\ninstead of just verifying committed outputs"]
    G4 -.-> N1
```

### Target State (Fixed)

```mermaid
flowchart TD

    subgraph Backend_CI["Backend CI (sdfk-backend)"]
        A1["Checkout repo + submodules"]
        A2["Run preflight_guardrails.sh (validate only)"]
        A3["Run test.sh (containerized)"]
    end

    subgraph Guardrails["Preflight Guardrails"]
        G1["Version SSOT check"]
        G2["Dependency pin check"]
        G3["Lint/Type surfacing (ruff, pyright)"]
        G4["✅ Codegen drift check: ensure_codegen_clean.sh only verifies"]
    end

    subgraph Schemas_CI["Schemas CI (synesthetic-schemas)"]
        S1["Poetry/Nix dev env (datamodel-code-generator installed)"]
        S2["Schema normalization & validation"]
        S3["Codegen generation + drift check"]
    end

    %% Backend Flow
    A1 --> A2 --> Guardrails
    A2 --> A3

    %% Guardrails Flow
    G1 --> G2 --> G3 --> G4
    G4 -->|verify artifacts match| Schemas_CI

    %% Fix
    N2["Backend CI no longer re-generates.\nIt validates schemas artifacts are clean.\nIf not, fail and instruct to update schemas repo + bump submodule."]
    G4 -.-> N2
```


## Meetings

## Log

## Thoughts
- Teaching a cat to fetch
    - Trying to wrangle ai into some sort of tool to project vision feels a bit like that time I tried to teach a cat to fetch
        - Throw the ball. Watch the cat look at it. Cat looks at me. I get the ball.
        - Repeat

# Review
    - Highlights:e2e running and serving assets with schema ssot.
    - Roadblocks: - backend ci fragile

    - Next steps: Triage backend CI

**PROMPTS**
