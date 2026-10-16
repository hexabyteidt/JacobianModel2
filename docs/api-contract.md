# Proposed ProtEva API Contract

**Status: proposal only.** No endpoint below is implemented. The shape is designed to preserve the existing `ProtEvaApi` boundary while supporting real asynchronous runs. Use `/api/v1`; agree schemas before connecting a live backend.

## Principles

- The frontend selects exactly one client at startup: mock when `VITE_API_BASE_URL` is absent, HTTP when present. HTTP errors must remain errors; never silently return mock success.
- Backend owns identity/access control, persistence, uploads, artifact storage, scientific execution and audit records.
- Create and cancellation are asynchronous operations. Responses identify the durable experiment and current state.
- JSON responses are validated at runtime in both backend and frontend. Do not rely on TypeScript `as T` casts as validation.
- Use authenticated HTTPS. Prefer same-origin frontend/backend routing with secure HttpOnly session cookies and CSRF protection, or a short-lived authenticated bearer token minted by a trusted server. Do not place service/model credentials in `VITE_*` values.

## Core Schemas

The current request can remain nested as `{ input, target, config }`. Extend it with optional references to validated uploads and an execution request; the server chooses live execution only when explicitly requested and authorized.

```json
{
  "input": {
    "experimentName": "Protease lid-opening sweep",
    "sequence": "MKTAYIAKQRQISFVKSHFS...",
    "sequenceUploadId": null,
    "referenceStructureUploadId": "upl_01J..."
  },
  "target": {
    "objectiveText": "Increase open-pocket occupancy while preserving the catalytic triad.",
    "targetStateDescription": "Lid helix displaced more than 6 angstrom from reference.",
    "undesiredStateDescription": "Closed pocket state",
    "structuralMetric": "rmsd",
    "thresholdAngstrom": 6,
    "residueRegion": "180-215",
    "preservationConstraints": "Do not mutate S195, H57, D102"
  },
  "config": {
    "populationSize": 24,
    "generations": 12,
    "maxSubstitutions": 4,
    "screeningSamples": 64,
    "finalistSamples": 256,
    "mutationStrategy": "balanced",
    "diversityPreference": 55,
    "randomSeed": 42
  },
  "executionMode": "live"
}
```

### Execution mode and provenance migration

Do not retain the current `Experiment.simulated: true` literal as the universal model. Proposed distinction:

```ts
type ExecutionMode = "demo" | "live";
type EvidenceOrigin = "demo_simulated" | "computational" | "experimental" | "human_review";
interface Provenance {
  origin: EvidenceOrigin;
  producer: string;
  producerVersion?: string;
  modelName?: string;
  modelVersion?: string;
  configurationHash?: string;
  sourceArtifactIds: string[];
  generatedAt: string;
}
interface MetricEvidence extends Provenance {
  metricDefinition: string;
  sampleCount?: number;
  uncertainty?: { value: number; method: string; confidenceLevel?: number };
}
```

Each experiment has `executionMode`; every metric, candidate result, agent event and report carries provenance at the narrowest useful level. A computational result is not an experimental measurement. A demo execution can only emit `demo_simulated` evidence. Use `null` for unknown values, not plausible-looking defaults.

Affected files for a narrow frontend migration: `src/lib/protein/types.ts`, `src/lib/protein/mock-data.ts`, `src/lib/protein/api.ts`, `src/lib/protein/format.ts`, `src/lib/protein/hooks.ts`, `src/components/app/AppShell.tsx`, `src/components/app/LandingPage.tsx`, `src/components/agents/AgentViews.tsx`, `src/components/candidates/CandidateDrawer.tsx`, `src/routes/overview.tsx`, `src/routes/experiments.new.tsx`, `src/routes/experiments.$experimentId.tsx`, `src/routes/candidates.tsx`, `src/routes/agents.tsx`, `src/routes/documentation.tsx`, and `src/routes/settings.tsx`. This list is impact analysis, not authorization to change them in this documentation phase.

### Experiment and status

```json
{
  "id": "exp_01JABC",
  "name": "Protease lid-opening sweep",
  "executionMode": "live",
  "status": "sampling",
  "stage": { "name": "sampling", "index": 3, "total": 7, "progress": 0.25, "message": "Sampling generation 3 finalists" },
  "timestamps": {
    "createdAt": "2026-10-16T10:00:00Z",
    "startedAt": "2026-10-16T10:00:03Z",
    "updatedAt": "2026-10-16T10:05:00Z",
    "completedAt": null,
    "cancellationRequestedAt": null
  },
  "input": { "experimentName": "Protease lid-opening sweep", "sequence": "MKTAYIAK...", "referenceStructureUploadId": "upl_01J..." },
  "target": { "objectiveText": "Increase open-pocket occupancy", "targetStateDescription": "Open lid", "structuralMetric": "rmsd", "thresholdAngstrom": 6, "residueRegion": "180-215", "preservationConstraints": "S195,H57,D102" },
  "config": { "populationSize": 24, "generations": 12, "maxSubstitutions": 4, "screeningSamples": 64, "finalistSamples": 256, "mutationStrategy": "balanced", "diversityPreference": 55, "randomSeed": 42 },
  "currentGeneration": 3,
  "candidateCount": 72,
  "requiresReview": 2,
  "failure": null,
  "provenance": { "origin": "computational", "producer": "proteva-orchestrator", "producerVersion": "0.1.0", "sourceArtifactIds": [], "generatedAt": "2026-10-16T10:05:00Z" }
}
```

States: `queued`, `validating`, `planning`, `sampling`, `evaluating`, `auditing`, `completed`, `failed`, `cancellation_requested`, `cancelled`. Expected forward path is queued → validating → planning → sampling → evaluating → auditing → completed. `failed` may occur from any nonterminal stage and includes an actionable failure object. Cancellation moves a cancellable nonterminal run to `cancellation_requested`; workers stop at a safe boundary, then set `cancelled`. A completed/failed/cancelled run cannot be restarted through the cancel endpoint. Record `stageStartedAt`, `stageCompletedAt`, overall timestamps, and append-only transitions/events. Do not jump states silently; retries should be explicit events/attempt metadata.

```json
{
  "failure": {
    "code": "WORKER_UNAVAILABLE",
    "message": "Sampling worker did not accept the job.",
    "retryable": true,
    "stage": "sampling",
    "action": "Retry after checking worker availability.",
    "occurredAt": "2026-10-16T10:05:00Z",
    "requestId": "req_01J..."
  }
}
```

### Candidate and scientific evidence

Keep candidate sequence and mutation lineage explicit. `objectiveScore` is nullable and must state which evaluator produced it. Metrics have per-value provenance and uncertainty; never label model-generated text as a metric.

```json
{
  "id": "cand_01JXYZ",
  "experimentId": "exp_01JABC",
  "parentId": "cand_parent",
  "generation": 3,
  "mutations": ["A42V", "T88S"],
  "sequence": "MKTAY...",
  "evaluationStatus": "evaluated",
  "objectiveScore": { "value": 0.71, "definition": "target occupancy objective v1", "provenance": { "origin": "computational", "producer": "proteva-evaluator", "producerVersion": "0.1.0", "modelName": "BioEmu", "modelVersion": "configured-release", "configurationHash": "sha256:...", "sourceArtifactIds": ["art_ensemble"], "generatedAt": "2026-10-16T10:04:00Z" } },
  "metrics": {
    "targetOccupancy": { "value": 0.71, "sampleCount": 256, "metricDefinition": "fraction of sampled frames meeting target predicate v1", "uncertainty": { "value": 0.05, "method": "bootstrap percentile interval", "confidenceLevel": 0.95 }, "origin": "computational", "producer": "proteva-evaluator", "sourceArtifactIds": ["art_ensemble"], "generatedAt": "2026-10-16T10:04:00Z" }
  },
  "diversityScore": 0.42,
  "audit": { "status": "flagged", "checks": [{ "code": "OVERLAPPING_INTERVALS", "passed": false, "message": "Candidate ranking is not separated at the configured uncertainty." }], "reviewedAt": "2026-10-16T10:04:30Z", "reviewer": "scientific-auditor" },
  "warnings": ["Ranking is not separated at the configured uncertainty."],
  "provenance": { "origin": "computational", "producer": "proteva-evaluator", "sourceArtifactIds": ["art_ensemble"], "generatedAt": "2026-10-16T10:04:00Z" }
}
```

Candidate list response should include `items`, `nextCursor`, and `total` when known. Filters should accept `experimentId`, `generation`, `maxMutations`, `evaluationStatus`, `auditStatus`, and `search`. Candidate detail returns complete sequence, mutation/parent lineage, each metric, audit checks, warnings and provenance. Never manufacture metrics for pending/rejected candidates.

### Agent events and tasks

Events are durable records, not generated summaries. The frontend's current `from`, `to`, `kind`, `summary` shape can be adapted in one client mapper while retaining a richer backend record.

```json
{
  "id": "evt_01J",
  "experimentId": "exp_01JABC",
  "taskId": "task_01J",
  "timestamp": "2026-10-16T10:04:00Z",
  "sender": { "type": "agent", "id": "evolution-coordinator" },
  "recipient": { "type": "agent", "id": "ensemble-analysis-worker" },
  "eventType": "task_delegated",
  "summary": "Evaluate generation 3 finalists",
  "message": { "schemaVersion": "1", "content": { "candidateIds": ["cand_01JXYZ"], "sampleCount": 256 } },
  "tool": null,
  "artifactRefs": [],
  "provenance": { "origin": "computational", "producer": "proteva-orchestrator", "sourceArtifactIds": [], "generatedAt": "2026-10-16T10:04:00Z" }
}
```

For `tool_invoked`, include tool name, invocation ID, validated arguments, allowlist decision and start time. For `tool_result`, include invocation ID, typed result or typed error, completion time and artifact references. Redact credentials, sensitive input and internal chain-of-thought. Record concise decisions, evidence references and validation warnings, not hidden reasoning.

### Error envelope

Use RFC 9457 Problem Details-compatible JSON with stable application code and request ID. Do not return stacks, secrets, model credentials, raw prompts or filesystem paths.

```json
{
  "type": "https://proteva.example/problems/sequence-invalid",
  "title": "Sequence validation failed",
  "status": 422,
  "code": "INVALID_SEQUENCE",
  "detail": "Sequence contains unsupported residues.",
  "fieldErrors": [{ "field": "input.sequence", "code": "INVALID_RESIDUE", "message": "Unsupported residue at position 8." }],
  "requestId": "req_01J..."
}
```

Status usage: `400` malformed request, `401/403` unauthenticated/unauthorized, `404` unknown or inaccessible resource, `409` invalid state transition, `413` upload limit, `415` unsupported media type, `422` semantic validation, `429` rate limit, `5xx` service failure. Client must expose request ID and actionable safe detail. Network, HTTP, parse, and schema errors remain errors and never trigger a mock fallback.

## Proposed Endpoints

| Method and path | Behavior | Success |
| --- | --- | --- |
| `POST /api/v1/experiments` | Validate draft and enqueue explicit demo/live run | `202` experiment with `queued` status |
| `GET /api/v1/experiments?cursor=&limit=&status=` | Paginated experiment list | `200` page |
| `GET /api/v1/experiments/{id}` | Full experiment details | `200` experiment |
| `GET /api/v1/experiments/{id}/status` | Small polling response: state, stage, timestamps, progress, failure | `200` status |
| `GET /api/v1/experiments/{id}/events?after=` | Append-only agent/system event page after cursor/sequence | `200` events and cursor |
| `GET /api/v1/experiments/{id}/candidates?...` | Paginated filtered candidates | `200` page |
| `GET /api/v1/candidates/{id}` | Candidate sequence, lineage, metrics, audit and provenance | `200` candidate |
| `POST /api/v1/uploads` | Multipart FASTA/PDB upload and validation | `201` upload metadata and validation |
| `GET /api/v1/uploads/{id}` | Read validation result/metadata for owned upload | `200` upload |
| `GET /api/v1/experiments/{id}/artifacts` | Authorized metadata list | `200` artifact page |
| `GET /api/v1/artifacts/{id}/download` | Authorized backend stream or short-lived redirect | `200` stream or `302` signed URL |
| `POST /api/v1/experiments/{id}/cancel` | Idempotently request safe cancellation | `202` experiment/status |
| `POST /api/v1/experiments/{id}/exports` | Queue/compose reproducibility report | `202` export job |
| `GET /api/v1/exports/{id}` | Export status and artifact reference | `200` export status |
| `GET /api/v1/health` | Non-sensitive liveness/readiness | `200` minimal status |

### Upload and validation contract

Request: `multipart/form-data` with `file`, `kind` (`fasta` or `pdb`), and optional `experimentId`. Enforce configurable byte limits, extension plus content sniffing, authorization, malware/content policy, and parse limits. Do not trust MIME type or filename. FASTA validation returns normalized sequence, header/name, residue count, invalid positions and warnings. PDB validation returns detected format, chain IDs, residue count, parse warnings/errors and content hash; it does not claim scientific validity.

Response example:

```json
{
  "id": "upl_01J",
  "kind": "pdb",
  "originalFilename": "reference.pdb",
  "contentType": "chemical/x-pdb",
  "sizeBytes": 84321,
  "sha256": "...",
  "validation": { "status": "valid_with_warnings", "format": "pdb", "chains": ["A"], "residueCount": 220, "warnings": ["Alternate locations were resolved by the parser policy."] },
  "createdAt": "2026-10-16T10:00:00Z"
}
```

Upload IDs must be bound to the authenticated owner and accepted only by experiments that owner can access. Never return a public object-store URL.

### Artifacts and exports

Artifact metadata includes opaque ID, kind, safe display name, media type, byte size, checksum, creation time, provenance and availability. Access control is checked on every list and download. Preferred default is an authenticated backend stream. If object storage requires signed URLs, mint a one-use or short-lived URL after authorization; keep TTL limited, do not log it, and do not store it in experiment records. The backend owns storage and cleanup.

Export request:

```json
{ "format": "markdown", "include": ["inputs", "configuration", "events", "candidate_metrics", "artifacts"], "idempotencyKey": "..." }
```

Report must include input hashes, normalized input/config, software and model versions, random seed, state history, event references, measurement/evidence provenance, uncertainty, warnings, failed/omitted outputs and limitations. Distinguish computational outputs from experimental measurements. Export is not proof of reproducibility unless all referenced inputs/artifacts remain available and executable versions are recorded.

### Cancellation and live updates

`POST /cancel` should return `202` with `cancellation_requested` or `cancelled`, be idempotent for repeated requests, and return `409` for terminal experiments. The worker acknowledges cancellation at a safe checkpoint. UI must show request/terminal state separately.

Start with React Query polling: poll status about every 2-4 seconds for active runs, use a slower interval or stop after terminal status, and fetch events with an `after` cursor. Honor `Retry-After`, back off on errors, and keep errors visible. Do not invalidate every query for any one server event. Add authenticated Server-Sent Events later if polling load/latency justifies it; provide reconnect cursor (`Last-Event-ID`) and polling fallback. The current mock subscription remains demo-only.

## Compatibility and Acceptance Criteria

- Keep `ProtEvaApi` methods or adapt them behind a small mapper; add upload APIs and pagination deliberately rather than leaking transport shapes into UI.
- Verify returned JSON at runtime and map backend statuses/provenance to UI view models.
- A live `GET`/`POST` failure, malformed body, unauthorized response, timeout or offline condition displays an error and never presents demo data as success.
- A created experiment is durable and immediately returns an ID; reloading detail/list routes retrieves the same experiment.
- Every status transition includes timestamps and is validated against the transition table. Cancellation has a race-safe terminal result.
- Event log proves task delegation, messages, allowlisted tool invocation, tool result/error, decision and artifact references; no private reasoning is returned.
- Every numeric metric names its definition, evidence origin, sample count where applicable, uncertainty method/value where available, source artifact, model/config version and producer.
- FASTA/PDB upload tests reject malformed/oversized/mismatched files and return actionable errors; no filename-only PDB path is considered uploaded.
- Artifact list/download is access-controlled; expired links fail safely. Export cites provenance and limitations.
- Contract tests run against mock HTTP server without Gemma, BioEmu, or GPU credentials; live integration tests are separately gated.
