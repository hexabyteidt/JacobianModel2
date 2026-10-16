# ProtEva Integration Roadmap

**Status: proposed implementation plan.** The existing React frontend and mock demo remain in place. This document does not assert that Gemma, a genetic algorithm, BioEmu, a voice agent, or a backend exists. Primary judging choice supplied by the project owner: **Best Use of Gemma 4, G1 Autonomous Multi-Agent Collaboration**. Do not claim Track 2 as primary as well; participation rules require exactly one primary track and statement.

## Delivery Order

1. Re-establish the project Git boundary, install the existing frontend dependencies with Bun, and record build/lint/test baselines. This workspace currently resolves Git at `C:/`, outside `proteva_ui`; stop before any Git mutation and have the owner open/use the correct clone or correct workspace/repository boundary.
2. Add contract tests and migrate frontend view types from literal `simulated: true` to explicit execution mode and evidence provenance while preserving demo output.
3. Implement minimal persistent FastAPI experiment CRUD, validation, status, events, cancellation, artifact metadata and contract tests. Wire HTTP mode without mock fallback.
4. Build the scientific evaluation and deterministic genetic algorithm as independent testable Python modules.
5. Implement the asynchronous BioEm Linux/GPU worker and validate one genuine sampling job end-to-end.
6. Implement Gemma orchestration, structured tasks/messages, allowlisted tools and verifiable event recording. This is the primary judging feature; demonstrate actual agent exchanges, not a scripted transcript.
7. Add Sarvam as an optional isolated voice feature after controlled backend tools and user confirmation are available.
8. Deploy, monitor, license-check and document a reproducible public repository release.

## A. Gemma 4 Agent Orchestration (Primary: Track 1 / G1)

### Responsibilities and shared state

Use FastAPI as the API/control plane and a durable job/task store as the source of truth. A workflow coordinator schedules the six named roles. Agents communicate through validated, versioned messages referencing `experiment_id`, `task_id`, `parent_task_id`, sender, recipient, deadline, idempotency key and artifact/evidence references. Messages are appended to an immutable event log; mutable experiment state is updated through explicit, validated commands. Agents never communicate only through separate hidden prompts or unrecorded in-process variables.

| Agent | Inputs | Allowed outputs/responsibility |
| --- | --- | --- |
| Landscape Interpreter | User objective, sequence metadata, target constraints | Structured target specification, unresolved questions, constraint conflicts; no metric values |
| Search Strategist | Validated target, constraints, compute budget | Bounded search config, expected task plan, budget estimate clearly labeled as estimate |
| Evolution Coordinator | Plan, GA candidates, worker/evaluator statuses | Create generation/evaluation tasks, delegate, retry under policy, select next actions, request cancellation |
| Ensemble Analysis Worker | Candidate and BioEmu artifact refs | Invoke deterministic evaluator tools on verified artifacts; return typed metrics/results, not freeform measurements |
| Scientific Auditor | Metrics with provenance, constraints, uncertainty | Check completeness/constraints/uncertainty, flag indistinguishable rankings, record checks and rationale refs |
| Report Agent | Persisted inputs, events, metrics, audit and artifacts | Assemble a report from verified records, clearly separate facts, computed outputs, interpretations and limitations |

### Observable collaboration

Persist and expose (with private reasoning removed): task-created/delegated/accepted/completed events; agent-to-agent messages with typed payloads; allowlisted tool invocation and validated arguments; tool result or structured error with invocation ID; validation warnings; coordinator decision and referenced evidence; retry/cancel events; artifact references. UI activity is derived from these persisted records, not from a fabricated timeline.

### Model/tool boundary

Use Gemma 4 through a supported Google/Gemini API integration, with model ID, region/endpoint, API capabilities, structured-output and function-calling support confirmed against current official provider documentation before implementation. Keep credentials exclusively in server-side secret storage. Record model ID/version, prompt/template version, schema version, request correlation ID, token/latency metadata when permitted, and redacted input/output hashes.

Prefer constrained structured outputs with a schema per agent action. Treat all model output as untrusted: validate schema, enum, size, numeric ranges, experiment ownership and state; reject unknown tool names; authorize every tool server-side; enforce timeouts, idempotency and quotas. Tool arguments must be validated again at execution. The model may suggest a scientific action or explain a verified result; it must never directly generate occupancy, RMSD, uncertainty, structures, or other numerical scientific results. Such values come from deterministic code reading worker artifacts.

### Demonstration criteria

A demo submission must show one real run in which the Interpreter's checked target output is passed to the Strategist, the Coordinator assigns candidate work, the analysis worker calls an allowlisted evaluator with actual artifacts, the Auditor identifies/clears a check using evidence, and the Report Agent cites those records. Show correlation IDs, task parents, tool results and event ordering. A mock mode can remain, but label it as simulated and do not count it as Gemma execution.

## B. Deterministic Genetic Algorithm

Keep sequence evolution in a separate Python package/module, independent of FastAPI route handlers and Gemma. Example future layout: `backend/proteva/ga/` with sequence validation, mutation operators, selection, diversity, generation runner and typed records.

- Input: validated parent sequence, mutable/frozen residue constraints, population/generation limits, max substitutions, strategy, diversity policy and explicit integer seed.
- Outputs: candidate sequence, candidate ID, parent ID, generation, exact mutation list and RNG/config metadata. Do not contain predicted fitness.
- Fitness interface: accept only evaluator-produced `FitnessRecord` keyed by candidate ID and provenance; GA may rank these records and perform selection, but cannot calculate structural fitness itself.
- Determinism: use an injected local PRNG (`random.Random(seed)` or an explicitly versioned equivalent), stable input ordering and documented tie breaks. Persist implementation version and seed. Never rely on global RNG state.
- Constraints: validate amino-acid alphabet and sequence length, freeze preserved positions, bound distinct substitutions relative to the declared parent, avoid no-op/duplicate mutation events, and emit clear infeasible-constraint errors.
- Diversity: define sequence distance (for example normalized Hamming distance for equal-length aligned sequences), population-level calculation, and strategy-specific diversity pressure.
- Selection: specify elitism, parent selection, mutation/crossover policy and deterministic tie resolution; maintain lineage.

Tests: fixed-seed exact-output golden test; same seed/config gives same population; different seed typically changes outputs; every output preserves length/alphabet/frozen residues; mutation count bound holds; parent links point to earlier generations; no duplicate IDs; empty/invalid input fails; selection/tie rules deterministic. Compare against a random-search baseline with the same evaluator call/sample budget. Report best-so-far, diversity and compute/evaluation counts; do not claim superiority without repeated, controlled benchmarks. Connect candidate output to an evaluator queue, then return fitness records for the next generation.

## C. BioEmu Linux/GPU Worker

BioEmu runs in a separately deployed Linux/GPU worker image, never in Vercel frontend code or ordinary API request handlers. An API operation enqueues an asynchronous job; a worker consumes it and stores artifacts/results in persistent storage. Keep provider/model setup, weight access and accepted sequence limits aligned with the official BioEmu release/model card. Verify model and dataset licenses, redistribution limits, GPU memory, CUDA/runtime versions and commercial/demo use before committing to hardware or artifact publication. Do not guess resource requirements; benchmark the selected model/config on the intended GPU first.

Proposed worker request:

```json
{
  "jobId": "job_01J",
  "experimentId": "exp_01JABC",
  "candidateId": "cand_01JXYZ",
  "sequence": "MKTAY...",
  "sampling": { "sampleCount": 256, "seed": 42, "configuration": "screening-v1" },
  "inputArtifactRefs": ["art_reference_structure"],
  "callback": { "protocol": "queue_result_v1" }
}
```

Result is typed success/failure metadata: job ID, worker/image version, BioEmu model/version/config hash, seed, start/end timestamps, sample count actually completed, artifact IDs and checksums, logs/reference, validation state and error/retryability. Structural artifacts (for example trajectory and topology/metadata as supported) are stored in private object storage and referred to by opaque IDs. Evaluator consumes artifacts from storage under narrowly scoped service identity; the model worker does not return arbitrary public URLs.

Operational requirements: worker capability preflight; bounded concurrency/queue; GPU health and utilization metrics; per-job timeout and heartbeat; cancellation at safe points; idempotent job IDs; retry only for transient errors; quota/resource estimates; cleanup policy; temporary disk limits; encrypted transport and at-rest storage; no credentials in logs; model cache keyed by model version/config and safely isolated. Record container digest, driver/CUDA, model checksum, sampling config and random seed. Never mark sampling complete until artifact checksums and expected sample count are verified.

Verification gate: run a single validated real sequence through the selected BioEmu release on target Linux/GPU, verify output structure can be parsed, sample count and checksums match, evaluator can consume it, artifact authorization works, cancellation/failure paths are recorded, and report records model/config/worker provenance. Keep this as an integration test requiring licensed model access and GPU; it is not replaced with mock output.

## D. Scientific Evaluation and Trust

Make an independent deterministic evaluator interface: validated target specification + immutable candidate + verified sampled artifacts + evaluator version/config → typed metric records, warnings and audit inputs. Unit-test the evaluator with small fixture structures that are openly licensed or generated solely as test fixtures and clearly not model evidence.

- Target score: define structural alignment, atom/residue selection, target-state predicate, denominator/frame handling, missing-frame policy and weighting before scoring. Score only valid sampled frames and expose numerator/denominator.
- Optional off-target score: require an explicit off-target predicate/reference and report absent as `null`, never zero.
- Structural checks: parser validity, chain/residue mapping, clashes/geometry checks if supported, missing atoms, sequence-to-structure correspondence and artifact integrity. Distinguish invalid samples from low-scoring samples.
- Uncertainty: select a sample-count-aware interval/resampling method before production; report sample count, effective sample assumptions, interval/method and dependence caveats. Correlated trajectory frames may invalidate simple binomial assumptions, so evaluate autocorrelation/block-bootstrap or justified independent sampling. An auditor flags overlapping/insufficient intervals; do not imply separation.
- Diversity: compute deterministic sequence diversity from sequences, distinct from structural-state diversity. State formula and population denominator.
- Provenance: every metric links to sequence hash, sample artifact IDs, BioEmu/model version, evaluator and target-predicate version, config hash, sample count and uncertainty procedure.

Deterministic code can validate inputs, mutations, sequence distance, file checksums, structure parsing, geometric metrics over provided coordinates, target predicates and audit rules. BioEmu supplies sampled coordinates; it does not establish thermodynamic equilibrium by itself. Occupancy in generated samples does not prove equilibrium populations, protein stability, binding affinity, biological function or safety. Experimental claims require appropriate external experiments and independently recorded provenance.

## E. Sarvam Voice Agents (Optional)

Current official docs reviewed: [Voice Agents overview](https://docs.sarvam.ai/voice-agents/overview), [Web TypeScript SDK](https://docs.sarvam.ai/voice-agents/deploy/sdks/web), and [API/HTTPS tool](https://docs.sarvam.ai/voice-agents/build/tools/https-tool). The platform supports voice/text conversational agents and the browser SDK uses `sarvam-conv-ai-sdk/browser`, `ConversationAgent`, and `BrowserAudioInterface`. The docs require a user gesture to start audio and HTTPS (localhost for development) for microphone access. SDK callbacks expose transcript, session state/events and audio level; `stop()` must run on cleanup. Reconnect requires a new agent/session.

For a tightly integrated ProtEva workbench, prefer the browser SDK **only through a same-origin controlled proxy** (`apiKey: ""`, `baseUrl: "/api/sarvam/"`) so the custom interface can bind to authenticated user/session and display confirmation. The docs explicitly warn against embedding the API key client-side and recommend this proxy mode. Agent configuration requires platform `org_id`, `workspace_id`, committed `app_id`, user identifier/type, `InteractionType.CALL`, and supported input/output sample rates. Pin a committed agent version for reproducibility. Store credentials in a secure server/platform secret store, never in `VITE_*`, JavaScript, browser storage or Git.

Sarvam's API tool configuration supports an HTTPS endpoint, method, params/headers/body, authentication via secure workspace secrets, response mapping, timeout and failure response; tools can be configured for mid-conversation, on-start or on-end. Call context may include user identifier, transcript so far, interaction ID and call length. A webhook body is configured by the tool, not a single fixed schema: define a strict ProtEva request schema and verify actual mapped fields in Sarvam's test agent before production. Restrict tool endpoints to controlled backend operations, verify platform-authenticated requests/signatures or a high-entropy secret where the platform supports it, authorize the user's experiment access and reject arbitrary URLs/operations. Keep timeout tight and responses small.

Possible tool set:

- `create_experiment_draft`: turn spoken intent into a draft only; API validates, returns missing fields and a `draftId`/version.
- `get_experiment_status`: reads only experiments visible to the authenticated user.
- `summarize_candidate`: returns backend-selected candidate fields plus evidence/provenance and limitations.
- `prepare_report_export`: creates an export job and returns status/artifact metadata, not arbitrary storage access.
- `confirm_launch`: separate endpoint requiring draft version, a short-lived one-use confirmation token, user identity, explicit acknowledgement, and validated cost/resource bounds. Creating a draft never launches. A positive confirmation must be a distinct user turn/action and be recorded; when identity/consent is uncertain, require an in-app confirmation control.

Never let Sarvam's language model call Gemma tools directly. Sarvam voice invokes these controlled API endpoints; the backend validates input and the Gemma coordinator owns the scientific workflow. Gemma agent events are the source of truth for scientific collaboration.

Fallback: voice UI is behind a feature/config switch; when credentials, proxy or Sarvam is unavailable, hide/disable voice and retain all existing routes and manual wizard functionality. Store no transcript by default; if retained, disclose retention, minimize/redact sequence or personal data, and obtain consent.

Testing: use Sarvam mock API tools for draft/status/report conversation prototyping; then configure a test agent and verify start gesture, mic denial/no-device handling, transcript callback, end cleanup, proxy credential isolation, request schema, timeout/error speech, authenticated user binding, confirmation token replay rejection, cancellation and tool authorization. Test live platform sessions only with user-provided account IDs/credentials and consent. Never require these credentials in normal frontend CI. Docs mention paid platform usage; confirm current account pricing and rate limits before enabling it.

## F. Repository, Open Source and Deployment

Preserve current frontend at the existing project boundary. Proposed additions after owner fixes the repository boundary:

```text
proteva_ui/                   # existing TanStack Start frontend; no redesign
  docs/
  src/lib/protein/            # keep frontend domain/API boundary
backend/
  pyproject.toml
  src/proteva_api/            # FastAPI routes, auth, schemas, services
  src/proteva_ga/              # independent deterministic GA
  tests/
worker/
  bioemu/                      # GPU job entrypoint/container and worker tests
  evaluator/                   # deterministic structure evaluation
  Dockerfile
contracts/
  openapi.json                 # generated/checked API schema
  events.schema.json
infra/
  vercel/                       # frontend config only where needed
  deployment/                   # environment-neutral service manifests/examples
```

Keep responsibilities separated: Vercel serves the frontend; persistent FastAPI owns auth, experiment state, queue API and controlled tool execution; a durable queue feeds a separate Linux/GPU worker; relational storage holds metadata/events; private object storage holds uploads, structures and reports. Use a service identity and least privilege between API, queue and worker. Do not put GPU execution into Vercel functions. Choose deployment by actual GPU model fit, region/data-residency, queue/storage/network needs, idle/scale-to-zero behavior and budget. GPU instances, model storage, managed database/queue and Sarvam per-minute usage can dominate cost; estimate from a measured workload before choosing a paid vendor.

Recommend Apache-2.0 for a public open-source project if the owner wants permissive reuse with an explicit patent grant; confirm contributor/IP preferences before adding it. Add LICENSE and attribution only after owner approval. Audit direct/transitive dependency licenses, model/weights licenses, BioEmu terms, sample/test structure data, sequence/database sources and Sarvam SDK/platform terms. Do not infer weights or scientific datasets are redistributable merely because code is open source.

Reproducible setup: specify supported Bun and Python versions after CI verification; pin dependencies and lockfiles; document `bun install --frozen-lockfile`, frontend scripts, Python venv/lock install, tests, local mock mode and optional backend mode. Provide `.env.example` containing names and safe placeholders only; `.env` and `.env.*` remain ignored, with `.env.example` explicitly allowed. Add secret scanning in CI and document rotation procedure. Keep real API/model/platform credentials in deployment secret stores.

## Verification and Acceptance Plan

### No GPU/model credentials required

1. Frontend: `bun run build`, `bun run lint`, `bun run test`; route tests include `/`, all workbench routes and unknown route handling.
2. Wizard: required fields, invalid amino acids, minimum sequence length, numeric bounds, FASTA parse, PDB upload validation errors, draft payload and no launch before confirmation.
3. Client contract: mock and HTTP client satisfy the same interface; HTTP 4xx/5xx, timeout, invalid JSON/schema and offline errors surface; assert no mock fallback when base URL is configured.
4. State: legal/illegal status transitions, timestamps, retry attempts, cancellation idempotency/races and terminal-state behavior.
5. Events/tools: task-parent references, sender/recipient, schema validation, allowlist denial, typed tool result/error and artifact provenance; no credential or chain-of-thought leakage.
6. GA: deterministic seeded population, mutation/frozen-residue bounds, lineage, diversity, selection, invalid inputs and random-search baseline accounting.
7. Upload/artifacts/reports: malformed/oversized FASTA/PDB rejection; checksums, ownership, no public artifact links, authorization, signed-link expiry, export includes evidence and limitations.
8. Voice: mock webhook payloads, tool authorization, user/experiment binding, one-use confirmation and replay rejection, no credentials in browser bundle, graceful disabled mode.
9. Security/config: `.env`/`.env.production` ignored, `.env.example` not ignored, required config validation, CORS/auth/CSRF checks, secret scanner and dependency license report.

### Requires real services or GPU/model credentials

- Gemma: exact supported model ID/provider API, structured/function-call behavior, schema failure/adversarial argument handling, rate/timeout behavior and an actual end-to-end orchestrated task exchange. Use a gated integration test; never fake it in the release evidence.
- BioEmu: Linux/GPU preflight, actual sample job, verified structures/checksums/sample counts, evaluator consumption, worker failure/cancellation and output provenance.
- Sarvam: platform account/app/version, secure proxy secret, microphone over HTTPS, real transcript/audio lifecycle, voice tool webhook and consent/confirmation behavior. Never require these credentials in normal frontend CI.
- Deployment: Vercel frontend plus persistent API/database/queue/private storage and worker; verify authorization, restart persistence, queue recovery, backup/restore and monitoring in staging.

## Unresolved Owner Inputs

No Sarvam account identifiers, committed agent/app ID, Gemma API credential/model choice, BioEmu model entitlement/weights, GPU budget, backend hosting choice, database/object-store provider, public license preference, user/auth model or production retention policy is available in the inspected workspace. Keep all optional integrations disabled until these are supplied and tested. Avoid asking for credentials in source files or chat; enter secrets directly into approved secret management.
# ProtEva Integration Roadmap

**Status: proposed implementation plan.** The existing React frontend and mock demo remain in place. This document does not assert that Gemma, a genetic algorithm, BioEmu, a voice agent, or a backend exists. Primary judging choice supplied by the project owner: **Best Use of Gemma 4, G1 Autonomous Multi-Agent Collaboration**. Do not claim Track 2 as primary as well; participation rules require exactly one primary track and statement.

## Delivery Order

1. Re-establish the project Git boundary, install the existing frontend dependencies with Bun, and record build/lint/test baselines. This workspace currently resolves Git at `C:/`, outside `proteva_ui`; stop before any Git mutation and have the owner open/use the correct clone or correct workspace/repository boundary.
2. Add contract tests and migrate frontend view types from literal `simulated: true` to explicit execution mode and evidence provenance while preserving demo output.
3. Implement minimal persistent FastAPI experiment CRUD, validation, status, events, cancellation, artifact metadata and contract tests. Wire HTTP mode without mock fallback.
4. Build the scientific evaluation and deterministic genetic algorithm as independent testable Python modules.
5. Implement the asynchronous BioEm Linux/GPU worker and validate one genuine sampling job end-to-end.
6. Implement Gemma orchestration, structured tasks/messages, allowlisted tools and verifiable event recording. This is the primary judging feature; demonstrate actual agent exchanges, not a scripted transcript.
7. Add Sarvam as an optional isolated voice feature after controlled backend tools and user confirmation are available.
8. Deploy, monitor, license-check and document a reproducible public repository release.

## A. Gemma 4 Agent Orchestration (Primary: Track 1 / G1)

### Responsibilities and shared state

Use FastAPI as the API/control plane and a durable job/task store as the source of truth. A workflow coordinator schedules the six named roles. Agents communicate through validated, versioned messages referencing `experiment_id`, `task_id`, `parent_task_id`, sender, recipient, deadline, idempotency key and artifact/evidence references. Messages are appended to an immutable event log; mutable experiment state is updated through explicit, validated commands. Agents never communicate only through separate hidden prompts or unrecorded in-process variables.

| Agent | Inputs | Allowed outputs/responsibility |
| --- | --- | --- |
| Landscape Interpreter | User objective, sequence metadata, target constraints | Structured target specification, unresolved questions, constraint conflicts; no metric values |
| Search Strategist | Validated target, constraints, compute budget | Bounded search config, expected task plan, budget estimate clearly labeled as estimate |
| Evolution Coordinator | Plan, GA candidates, worker/evaluator statuses | Create generation/evaluation tasks, delegate, retry under policy, select next actions, request cancellation |
| Ensemble Analysis Worker | Candidate and BioEmu artifact refs | Invoke deterministic evaluator tools on verified artifacts; return typed metrics/results, not freeform measurements |
| Scientific Auditor | Metrics with provenance, constraints, uncertainty | Check completeness/constraints/uncertainty, flag indistinguishable rankings, record checks and rationale refs |
| Report Agent | Persisted inputs, events, metrics, audit and artifacts | Assemble a report from verified records, clearly separate facts, computed outputs, interpretations and limitations |

### Observable collaboration

Persist and expose (with private reasoning removed): task-created/delegated/accepted/completed events; agent-to-agent messages with typed payloads; allowlisted tool invocation and validated arguments; tool result or structured error with invocation ID; validation warnings; coordinator decision and referenced evidence; retry/cancel events; artifact references. UI activity is derived from these persisted records, not from a fabricated timeline.

### Model/tool boundary

Use Gemma 4 through a supported Google/Gemini API integration, with model ID, region/endpoint, API capabilities, structured-output and function-calling support confirmed against current official provider documentation before implementation. Keep credentials exclusively in server-side secret storage. Record model ID/version, prompt/template version, schema version, request correlation ID, token/latency metadata when permitted, and redacted input/output hashes.

Prefer constrained structured outputs with a schema per agent action. Treat all model output as untrusted: validate schema, enum, size, numeric ranges, experiment ownership and state; reject unknown tool names; authorize every tool server-side; enforce timeouts, idempotency and quotas. Tool arguments must be validated again at execution. The model may suggest a scientific action or explain a verified result; it must never directly generate occupancy, RMSD, uncertainty, structures, or other numerical scientific results. Such values come from deterministic code reading worker artifacts.

### Demonstration criteria

A demo submission must show one real run in which the Interpreter's checked target output is passed to the Strategist, the Coordinator assigns candidate work, the analysis worker calls an allowlisted evaluator with actual artifacts, the Auditor identifies/clears a check using evidence, and the Report Agent cites those records. Show correlation IDs, task parents, tool results and event ordering. A mock mode can remain, but label it as simulated and do not count it as Gemma execution.

## B. Deterministic Genetic Algorithm

Keep sequence evolution in a separate Python package/module, independent of FastAPI route handlers and Gemma. Example future layout: `backend/proteva/ga/` with sequence validation, mutation operators, selection, diversity, generation runner and typed records.

- Input: validated parent sequence, mutable/frozen residue constraints, population/generation limits, max substitutions, strategy, diversity policy and explicit integer seed.
- Outputs: candidate sequence, candidate ID, parent ID, generation, exact mutation list and RNG/config metadata. Do not contain predicted fitness.
- Fitness interface: accept only evaluator-produced `FitnessRecord` keyed by candidate ID and provenance; GA may rank these records and perform selection, but cannot calculate structural fitness itself.
- Determinism: use an injected local PRNG (`random.Random(seed)` or an explicitly versioned equivalent), stable input ordering and documented tie breaks. Persist implementation version and seed. Never rely on global RNG state.
- Constraints: validate amino-acid alphabet and sequence length, freeze preserved positions, bound distinct substitutions relative to the declared parent, avoid no-op/duplicate mutation events, and emit clear infeasible-constraint errors.
- Diversity: define sequence distance (for example normalized Hamming distance for equal-length aligned sequences), population-level calculation, and strategy-specific diversity pressure.
- Selection: specify elitism, parent selection, mutation/crossover policy and deterministic tie resolution; maintain lineage.

Tests: fixed-seed exact-output golden test; same seed/config gives same population; different seed typically changes outputs; every output preserves length/alphabet/frozen residues; mutation count bound holds; parent links point to earlier generations; no duplicate IDs; empty/invalid input fails; selection/tie rules deterministic. Compare against a random-search baseline with the same evaluator call/sample budget. Report best-so-far, diversity and compute/evaluation counts; do not claim superiority without repeated, controlled benchmarks. Connect candidate output to an evaluator queue, then return fitness records for the next generation.

## C. BioEmu Linux/GPU Worker

BioEmu runs in a separately deployed Linux/GPU worker image, never in Vercel frontend code or ordinary API request handlers. An API operation enqueues an asynchronous job; a worker consumes it and stores artifacts/results in persistent storage. Keep provider/model setup, weight access and accepted sequence limits aligned with the official BioEmu release/model card. Verify model and dataset licenses, redistribution limits, GPU memory, CUDA/runtime versions and commercial/demo use before committing to hardware or artifact publication. Do not guess resource requirements; benchmark the selected model/config on the intended GPU first.

Proposed worker request:

```json
{
  "jobId": "job_01J",
  "experimentId": "exp_01JABC",
  "candidateId": "cand_01JXYZ",
  "sequence": "MKTAY...",
  "sampling": { "sampleCount": 256, "seed": 42, "configuration": "screening-v1" },
  "inputArtifactRefs": ["art_reference_structure"],
  "callback": { "protocol": "queue_result_v1" }
}
```

Result is typed success/failure metadata: job ID, worker/image version, BioEmu model/version/config hash, seed, start/end timestamps, sample count actually completed, artifact IDs and checksums, logs/reference, validation state and error/retryability. Structural artifacts (for example trajectory and topology/metadata as supported) are stored in private object storage and referred to by opaque IDs. Evaluator consumes artifacts from storage under narrowly scoped service identity; the model worker does not return arbitrary public URLs.

Operational requirements: worker capability preflight; bounded concurrency/queue; GPU health and utilization metrics; per-job timeout and heartbeat; cancellation at safe points; idempotent job IDs; retry only for transient errors; quota/resource estimates; cleanup policy; temporary disk limits; encrypted transport and at-rest storage; no credentials in logs; model cache keyed by model version/config and safely isolated. Record container digest, driver/CUDA, model checksum, sampling config and random seed. Never mark sampling complete until artifact checksums and expected sample count are verified.

Verification gate: run a single validated real sequence through the selected BioEmu release on target Linux/GPU, verify output structure can be parsed, sample count and checksums match, evaluator can consume it, artifact authorization works, cancellation/failure paths are recorded, and report records model/config/worker provenance. Keep this as an integration test requiring licensed model access and GPU; it is not replaced with mock output.

## D. Scientific Evaluation and Trust

Make an independent deterministic evaluator interface: validated target specification + immutable candidate + verified sampled artifacts + evaluator version/config → typed metric records, warnings and audit inputs. Unit-test the evaluator with small fixture structures that are openly licensed or generated solely as test fixtures and clearly not model evidence.

- Target score: define structural alignment, atom/residue selection, target-state predicate, denominator/frame handling, missing-frame policy and weighting before scoring. Score only valid sampled frames and expose numerator/denominator.
- Optional off-target score: require an explicit off-target predicate/reference and report absent as `null`, never zero.
- Structural checks: parser validity, chain/residue mapping, clashes/geometry checks if supported, missing atoms, sequence-to-structure correspondence and artifact integrity. Distinguish invalid samples from low-scoring samples.
- Uncertainty: select a sample-count-aware interval/resampling method before production; report sample count, effective sample assumptions, interval/method and dependence caveats. Correlated trajectory frames may invalidate simple binomial assumptions, so evaluate autocorrelation/block-bootstrap or justified independent sampling. An auditor flags overlapping/insufficient intervals; do not imply separation.
- Diversity: compute deterministic sequence diversity from sequences, distinct from structural-state diversity. State formula and population denominator.
- Provenance: every metric links to sequence hash, sample artifact IDs, BioEmu/model version, evaluator and target-predicate version, config hash, sample count and uncertainty procedure.

Deterministic code can validate inputs, mutations, sequence distance, file checksums, structure parsing, geometric metrics over provided coordinates, target predicates and audit rules. BioEmu supplies sampled coordinates; it does not establish thermodynamic equilibrium by itself. Occupancy in generated samples does not prove equilibrium populations, protein stability, binding affinity, biological function or safety. Experimental claims require appropriate external experiments and independently recorded provenance.

## E. Sarvam Voice Agents (Optional)

Current official docs reviewed: [Voice Agents overview](https://docs.sarvam.ai/voice-agents/overview), [Web TypeScript SDK](https://docs.sarvam.ai/voice-agents/deploy/sdks/web), and [API/HTTPS tool](https://docs.sarvam.ai/voice-agents/build/tools/https-tool). The platform supports voice/text conversational agents and the browser SDK uses `sarvam-conv-ai-sdk/browser`, `ConversationAgent`, and `BrowserAudioInterface`. The docs require a user gesture to start audio and HTTPS (localhost for development) for microphone access. SDK callbacks expose transcript, session state/events and audio level; `stop()` must run on cleanup. Reconnect requires a new agent/session.

For a tightly integrated ProtEva workbench, prefer the browser SDK **only through a same-origin controlled proxy** (`apiKey: ""`, `baseUrl: "/api/sarvam/"`) so the custom interface can bind to authenticated user/session and display confirmation. The docs explicitly warn against embedding the API key client-side and recommend this proxy mode. Agent configuration requires platform `org_id`, `workspace_id`, committed `app_id`, user identifier/type, `InteractionType.CALL`, and supported input/output sample rates. Pin a committed agent version for reproducibility. Store credentials in a secure server/platform secret store, never in `VITE_*`, JavaScript, browser storage or Git.

Sarvam's API tool configuration supports an HTTPS endpoint, method, params/headers/body, authentication via secure workspace secrets, response mapping, timeout and failure response; tools can be configured for mid-conversation, on-start or on-end. Call context may include user identifier, transcript so far, interaction ID and call length. A webhook body is configured by the tool, not a single fixed schema: define a strict ProtEva request schema and verify actual mapped fields in Sarvam's test agent before production. Restrict tool endpoints to controlled backend operations, verify platform-authenticated requests/signatures or a high-entropy secret where the platform supports it, authorize the user's experiment access and reject arbitrary URLs/operations. Keep timeout tight and responses small.

Possible tool set:

- `create_experiment_draft`: turn spoken intent into a draft only; API validates, returns missing fields and a `draftId`/version.
- `get_experiment_status`: reads only experiments visible to the authenticated user.
- `summarize_candidate`: returns backend-selected candidate fields plus evidence/provenance and limitations.
- `prepare_report_export`: creates an export job and returns status/artifact metadata, not arbitrary storage access.
- `confirm_launch`: separate endpoint requiring draft version, a short-lived one-use confirmation token, user identity, explicit acknowledgement, and validated cost/resource bounds. Creating a draft never launches. A positive confirmation must be a distinct user turn/action and be recorded; when identity/consent is uncertain, require an in-app confirmation control.

Never let Sarvam's language model call Gemma tools directly. Sarvam voice invokes these controlled API endpoints; the backend validates input and the Gemma coordinator owns the scientific workflow. Gemma agent events are the source of truth for scientific collaboration.

Fallback: voice UI is behind a feature/config switch; when credentials, proxy or Sarvam is unavailable, hide/disable voice and retain all existing routes and manual wizard functionality. Store no transcript by default; if retained, disclose retention, minimize/redact sequence or personal data, and obtain consent.

Testing: use Sarvam mock API tools for draft/status/report conversation prototyping; then configure a test agent and verify start gesture, mic denial/no-device handling, transcript callback, end cleanup, proxy credential isolation, request schema, timeout/error speech, authenticated user binding, confirmation token replay rejection, cancellation and tool authorization. Test live platform sessions only with user-provided account IDs/credentials and consent. Docs mention paid platform usage; confirm current account pricing and rate limits before enabling it.

## F. Repository, Open Source and Deployment

Preserve current frontend at the existing project boundary. Proposed additions after owner fixes the repository boundary:

```text
proteva_ui/                   # existing TanStack Start frontend; no redesign
  docs/
  src/lib/protein/            # keep frontend domain/API boundary
backend/
  pyproject.toml
  src/proteva_api/            # FastAPI routes, auth, schemas, services
  src/proteva_ga/              # independent deterministic GA
  tests/
worker/
  bioemu/                      # GPU job entrypoint/container and worker tests
  evaluator/                   # deterministic structure evaluation
  Dockerfile
contracts/
  openapi.json                 # generated/checked API schema
  events.schema.json
infra/
  vercel/                       # frontend config only where needed
  deployment/                   # environment-neutral service manifests/examples
```

Keep responsibilities separated: Vercel serves the frontend; persistent FastAPI owns auth, experiment state, queue API and controlled tool execution; a durable queue feeds a separate Linux/GPU worker; relational storage holds metadata/events; private object storage holds uploads, structures and reports. Use a service identity and least privilege between API, queue and worker. Do not put GPU execution into Vercel functions. Choose deployment by actual GPU model fit, region/data-residency, queue/storage/network needs, idle/scale-to-zero behavior and budget. GPU instances, model storage, managed database/queue and Sarvam per-minute usage can dominate cost; estimate from a measured workload before choosing a paid vendor.

Recommend Apache-2.0 for a public open-source project if the owner wants permissive reuse with an explicit patent grant; confirm contributor/IP preferences before adding it. Add LICENSE and attribution only after owner approval. Audit direct/transitive dependency licenses, model/weights licenses, BioEmu terms, sample/test structure data, sequence/database sources and Sarvam SDK/platform terms. Do not infer weights or scientific datasets are redistributable merely because code is open source.

Reproducible setup: specify supported Bun and Python versions after CI verification; pin dependencies and lockfiles; document `bun install --frozen-lockfile`, frontend scripts, Python venv/lock install, tests, local mock mode and optional backend mode. Provide `.env.example` containing names and safe placeholders only; `.env` and `.env.*` remain ignored, with `.env.example` explicitly allowed. Add secret scanning in CI and document rotation procedure. Keep real API/model/platform credentials in deployment secret stores.

## Verification and Acceptance Plan

### No GPU/model credentials required

1. Frontend: `bun run build`, `bun run lint`, `bun run test`; route tests include `/`, all workbench routes and unknown route handling.
2. Wizard: required fields, invalid amino acids, minimum sequence length, numeric bounds, FASTA parse, PDB upload validation errors, draft payload and no launch before confirmation.
3. Client contract: mock and HTTP client satisfy the same interface; HTTP 4xx/5xx, timeout, invalid JSON/schema and offline errors surface; assert no mock fallback when base URL is configured.
4. State: legal/illegal status transitions, timestamps, retry attempts, cancellation idempotency/races and terminal-state behavior.
5. Events/tools: task-parent references, sender/recipient, schema validation, allowlist denial, typed tool result/error and artifact provenance; no credential or chain-of-thought leakage.
6. GA: deterministic seeded population, mutation/frozen-residue bounds, lineage, diversity, selection, invalid inputs and random-search baseline accounting.
7. Upload/artifacts/reports: malformed/oversized FASTA/PDB rejection; checksums, ownership, no public artifact links, authorization, signed-link expiry, export includes evidence and limitations.
8. Voice: mock webhook payloads, tool authorization, user/experiment binding, one-use confirmation and replay rejection, no credentials in browser bundle, graceful disabled mode.
9. Security/config: `.env`/`.env.production` ignored, `.env.example` not ignored, required config validation, CORS/auth/CSRF checks, secret scanner and dependency license report.

### Requires real services or GPU/model credentials

- Gemma: exact supported model ID/provider API, structured/function-call behavior, schema failure/adversarial argument handling, rate/timeout behavior and an actual end-to-end orchestrated task exchange. Use a gated integration test; never fake it in the release evidence.
- BioEmu: Linux/GPU preflight, actual sample job, verified structures/checksums/sample counts, evaluator consumption, worker failure/cancellation and output provenance.
- Sarvam: platform account/app/version, secure proxy secret, microphone over HTTPS, real transcript/audio lifecycle, voice tool webhook and consent/confirmation behavior. Never require these credentials in normal frontend CI.
- Deployment: Vercel frontend plus persistent API/database/queue/private storage and worker; verify authorization, restart persistence, queue recovery, backup/restore and monitoring in staging.

## Unresolved Owner Inputs

No Sarvam account identifiers, committed agent/app ID, Gemma API credential/model choice, BioEmu model entitlement/weights, GPU budget, backend hosting choice, database/object-store provider, public license preference, user/auth model or production retention policy is available in the inspected workspace. Keep all optional integrations disabled until these are supplied and tested. Avoid asking for credentials in source files or chat; enter secrets directly into approved secret management.
