# Import Wizard Frontend/Backend Sync Plan

## 1. Goal

Translate the proposed 6-step wizard UI into a production-ready, target-independent import system where frontend states and backend contracts stay tightly aligned.

The wizard must support exports to Mermaid, BPMN, ProcessMap-V1, JSON, CSV, and PDF while keeping a single canonical process model.

## 2. Architecture Snapshot

- **Frontend (Wizard UI):** step orchestration, user controls, visual QA, and export selection.
- **Backend (Import Intelligence API):** ingestion, extraction, normalization, QA comparison, and target compilation.
- **Canonical Process Model (CPM):** single source of truth between all steps.
- **Adapter Layer:** output-specific compilers (`mermaid`, `bpmn`, `processmap_v1`, `json`, `csv`, `pdf`).
- **Job Runtime:** async job orchestration with status events for long-running steps.

## 3. Step-by-Step Contract Mapping

### Step 1 - Upload
UI responsibilities:
- file selection, validation (type/size), and source metadata.

Backend contracts:
- `POST /api/import/sessions`
  - Creates import session.
- `POST /api/import/sessions/{sessionId}/files`
  - Multipart upload (PDF/DOCX/TXT/HTML/CSV/JSON).
  - Returns `fileId`, hash, detected mime, page estimate.

Frontend state produced:
- `sessionId`
- `selectedFileIds[]`

---

### Step 2 - Page map
UI responsibilities:
- page-by-page structure preview (title/sections/body/bullets/keywords).
- confidence bars and keyword panel.

Backend contracts:
- `POST /api/import/sessions/{sessionId}/parse`
  - Starts parse/extract job.
  - Payload: `{ fileIds, parseProfile, ocrPolicy }`
  - Returns `jobId`.
- `GET /api/jobs/{jobId}`
  - Poll status or stream events.
- `GET /api/import/sessions/{sessionId}/pages?index=N`
  - Returns page map blocks, interpretation tiles, keywords.

Frontend state produced:
- `pageMap[]`
- `parseArtifactsId`

---

### Step 3 - Settings
UI responsibilities:
- choose model strategy (`raw`, `wizard`, `gemini`, `claude`, future providers).
- output intent (`journey`, `swimlane`, `decision_tree`) and toggles.

Backend contracts:
- `POST /api/import/sessions/{sessionId}/settings`
  - Stores provider routing and extraction options.
  - Example:
    - `providerPolicy`: `single|fallback|consensus`
    - `primaryModel`: `gemini`
    - `secondaryModel`: `claude`
    - `features`: actor lanes, decisions, system refs, confidence labels.

Frontend state produced:
- `settingsVersion`
- immutable settings snapshot for downstream reproducibility.

---

### Step 4 - Review
UI responsibilities:
- cluster reordering, process/step type edits, row-level corrections, unassigned handling.

Backend contracts:
- `POST /api/import/sessions/{sessionId}/extract`
  - Runs LLM extraction + deterministic normalization to CPM.
  - Returns `jobId`.
- `GET /api/import/sessions/{sessionId}/review`
  - Returns grouped process list, clusters, themes, counts, and unresolved items.
- `PATCH /api/import/sessions/{sessionId}/cpm`
  - Applies user edits (rename/retype/reorder/reassign/remove).

Frontend state produced:
- `cpmDraft` (server-backed)
- `reviewEdits[]` (operation log)

---

### Step 5 - Preview
UI responsibilities:
- read-only map preview with pan/zoom and cluster filters.
- visual sanity checks before final import.

Backend contracts:
- `POST /api/import/sessions/{sessionId}/preview`
  - Materializes preview graph from latest CPM.
- `GET /api/import/sessions/{sessionId}/preview?clusterId=...`
  - Returns diagram payload (nodes/edges/layout hints) for UI renderer.

Optional advanced QA:
- `POST /api/import/sessions/{sessionId}/visual-qa/compare`
  - Compare source document map overview vs import map.
  - Returns alignment and mismatch queues.

Frontend state produced:
- `previewGraph`
- optional `qaCompareReport`

---

### Step 6 - Complete
UI responsibilities:
- show final stats/validation and select export target.

Backend contracts:
- `POST /api/import/sessions/{sessionId}/compile`
  - Payload: `{ targets: ["mermaid","processmap_v1","bpmn","json","csv","pdf"] }`
  - Returns artifact manifest + warnings/loss reports per target.
- `GET /api/import/sessions/{sessionId}/artifacts`
  - Download links and metadata.
- `POST /api/import/sessions/{sessionId}/open-in-editor`
  - Optional launch handoff for selected target/editor integration.

Frontend state produced:
- `artifactManifest`
- `selectedTarget`

## 4. Canonical Data Contracts (Minimum)

### 4.1 Session envelope
- `sessionId`
- `createdAt`
- `status`
- `settingsVersion`
- `sourceFiles[]`
- `currentStage`

### 4.2 Canonical Process Model (CPM)
- `process`: id, title, type, language
- `clusters[]`: id, name, themes[]
- `actors[]`
- `nodes[]`: id, type, label, clusterId, actorId, confidence, evidenceRefs[]
- `edges[]`: id, from, to, edgeType, condition, confidence, evidenceRefs[]
- `facts[]` and policies
- `unassigned[]`
- `validation`: disconnected nodes, decision exit counts, warnings[]

### 4.3 Evidence reference
- `sourceFileId`
- `page`
- `chunkId`
- `charStart`, `charEnd`
- `excerpt`

## 5. Async Runtime and Event Sync

Use job events for parse/extract/compile:
- `job.accepted`
- `job.progress` (stage + percentage + counters)
- `job.partial` (streaming partial artifacts)
- `job.completed`
- `job.failed`

Transport options:
- preferred: SSE (`/api/jobs/{jobId}/events`)
- fallback: polling (`GET /api/jobs/{jobId}`)

Frontend behavior:
- persist local step state but treat backend session as source of truth.
- allow resume at any step using `sessionId`.

## 6. Secrets and Model Provider Integration

- All provider keys (Gemini/Claude/others) must be backend-managed.
- Never store model API keys in frontend local storage.
- Provider adapter interface:
  - `extractStructure(input, settings)`
  - `extractFlow(input, settings)`
  - `classifyThemes(input, settings)`
  - `reconcileDiffs(primary, secondary)`

Recommended policy:
- `single`: one model for speed.
- `fallback`: primary with automatic fallback.
- `consensus`: dual pass + disagreement queue for high-stakes imports.

## 7. Testing Plan to Keep Front/Back in Sync

1. **Contract tests** for all endpoints and schemas.
2. **Fixture-based E2E** per wizard step with stable test documents.
3. **Golden output tests** for target compilers (Mermaid/BPMN/ProcessMap-V1).
4. **Regression metrics**:
   - node/edge precision/recall
   - cluster/theme/title match rates
   - unassigned ratio
5. **UI-state replay tests**:
   - save session mid-step and resume.
   - verify no drift between frontend cache and backend state.

## 8. Immediate Next Steps

1. Freeze v1 JSON schemas:
   - session
   - page map
   - CPM
   - preview graph
   - artifact manifest
2. Generate OpenAPI spec from the endpoint list in this document.
3. Implement mock backend returning deterministic fixtures for all 6 steps.
4. Wire frontend to mock endpoints before real extraction logic.
5. Add one end-to-end happy-path test from upload to export (`processmap_v1` + `mermaid`).

## 9. Notes for the Provided HTML Concept

- The 6-step flow is production-viable and maps cleanly to backend stages.
- The model selector in Step 3 should map to backend `providerPolicy`, not direct client API calls.
- Step 4 edits should be sent as patch operations (not full document overwrite) for traceability.
- Step 5 preview should consume backend-rendered layout hints for deterministic cross-client visuals.
- Step 6 export list is aligned with target-independent architecture and can be expanded via adapters.
