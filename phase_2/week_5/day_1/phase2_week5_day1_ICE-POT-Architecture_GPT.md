# ICE POT — Test-case generator architecture

Version 1.0 • 26 September 2026 • Design based on the completed 20-question questionnaire

## 1. Architecture decision

Build a project-isolated retrieval-augmented generation application with a React frontend, a modular Node.js/TypeScript backend, MongoDB Atlas on AWS, and separately scalable API and background worker services on Amazon ECS with Fargate. Use a fixed, persisted workflow with bounded AI checks rather than autonomous agents. Sources are approved, sanitized, versioned, and embedded before they become available to hybrid retrieval. Generated cases retain source evidence and require user approval before publishing.

MongoDB Atlas is an architectural recommendation implementing the requested MongoDB stack; its deployment region, tier, and connectivity remain to be selected. S3 stores original documents and export files; MongoDB stores structured records, sanitized chunks, and vectors. “All data in vector DB” therefore means indexing useful approved content, not replacing original files or structured test data with vectors.

![Architecture flow](ICE-POT-Architecture.png)

The diagram is a logical flow; the AWS placement and operational details are specified below. This document is a design, not a deployed or performance-tested application.

## 2. Confirmed requirements

| Question | Confirmed decision |
|---|---|
| 1 | Generate for one requirement, a selected group, or a release/sprint. |
| 2 | First-release sources: requirements, existing tests, API/UI specifications, and test data. |
| 3 | Uploads, pasted content, and on-demand connected imports; no continuous content synchronization. |
| 4 | Input connectors: Jira, Azure DevOps, and Figma. |
| 5 | Downloads and direct publishing. |
| 6 | Jira output; Xray, Zephyr, or ordinary Jira issue configuration is not yet known. |
| 7 | Detailed cases: requirements, category, priority, data, preconditions, steps and per-step expected results. |
| 8 | User chooses categories after application recommendations. |
| 9 | Generate supported cases; flag gaps and hold back affected cases for clarification. |
| 10 | Draft downloads allowed; approval required for publishing. |
| 11 | Multiple internal teams with project-based access. |
| 12 | Retain source versions, retrieve latest by default, flag affected linked tests. |
| 13 | Configurable pilot; determine capacity using representative load tests. |
| 14 | Project administrator approves external processing of sources; sensitive values are masked. |
| 15 | Fixed workflow with bounded coverage, grounding, and duplicate checks. |
| 16 | Hybrid search for every generation request, followed by reranking and deduplication. |
| 17 | OpenAI or Groq generation; OpenAI or Mistral embeddings; one active configuration per project. |
| 18 | TXT, Markdown, CSV, JSON, YAML, text-based PDF, DOCX, XLSX uploads. |
| 19 | XLSX, CSV, JSON exports. |
| 20 | ECS/Fargate with separate API and background workers. |

### Scope boundaries

Meeting recordings, OCR, release-note ingestion, Confluence/wiki connectors, dedicated defect ingestion, and developer repositories are future extensions, not first-release work. Selecting a release/sprint is a generation scope, not release-note ingestion. Existing tests and test data initially enter through uploads/paste; do not assume Jira or Azure DevOps requirements connectors also import test suites. Swagger/OpenAPI enters through JSON/YAML uploads or paste. Figma contributes accessible design text and structure, not proof of hidden application behavior.

If code repositories are later added, preserve the original no-vector requirement: use permission-checked, revision-pinned code reads/search through a separate adapter. Do not embed repository code.

## 3. Logical components and ownership

| Component | Responsibility |
|---|---|
| React UI | Project selection, imports, source approval, generation scope/categories, job progress, evidence review, clarification, approval, downloads and publishing. |
| TypeScript API | Authentication, project authorization, validation, job submission, review/version operations, export and publishing gates. |
| Ingestion worker | Connector reads, parsing, masking, approval enforcement, chunking, embedding, index readiness and change impact. |
| Generation worker | Scope resolution, query preparation, hybrid retrieval, reranking, evidence packing, generation, bounded validation and gap records. |
| Delivery worker | Format exports, validate destination mapping, publish approved revisions, reconcile uncertain remote outcomes. |
| Provider adapters | Separate generation and embedding interfaces with capability checks, timeouts, quotas and model/version tracking. |
| MongoDB Atlas | Application records, source versions, chunks, vectors, text/vector indexes, job state, evidence links and publication ledger. |
| Amazon S3 | Restricted original uploads, normalized artifacts when needed, and downloadable exports. |
| SQS + dead-letter queues | Separate ingestion, generation, and delivery work queues, retry isolation and backlog visibility. |

Use one TypeScript repository with shared domain models and deployable API/worker entry points. Start with a modular backend, not a service per workflow step. No agent framework is required: durable stage records and explicit transitions are sufficient.

## 4. Ingestion pipeline — critical

**Upload/paste/on-demand import → authorize → validate → parse → mask → source approval gate → normalize and chunk → embed → MongoDB → verify index readiness → activate version.**

1. Authenticate and resolve the application project. Verify import permission and connector scope. Select specific source items; do not crawl entire accounts by default. Record connector instance, external ID, URL, revision, timestamp and importing user.
2. Store originals in a restricted S3 area with a content hash. Validate actual file type and configured size limits; scan uploads and reject unsupported or unsafe files. Never execute document macros or embedded content.
3. Parse within the application infrastructure before external model calls. Extract headings, paragraphs, tables, acceptance criteria, API operations, test steps and data schemas with locators. Low-text PDFs are reported as unsupported scans; OCR is not silently introduced.
4. Detect credentials and sensitive values. Replace them with stable placeholders within the project as appropriate. Preserve useful constraints such as data type and allowed length without retaining raw secrets in retrieval content. Quarantine items whose sanitization is uncertain. Masking is best effort and must be reviewable.
5. Require administrator approval covering the source, external processing policy and selected providers. A changed version is rechecked; new sensitive classifications require renewed approval. Block embeddings as well as generation when approval is missing. Do not send unapproved content to an LLM to perform the masking itself.
6. Normalize into source-specific records. Keep an entire acceptance criterion together, each test with its steps, each API operation with parameter/response constraints, and each Figma node with its screen/component context. For large documents, split at logical boundaries with bounded overlap. Start by evaluating approximately 400–800-token chunks; this is a tuning proposal, not a confirmed requirement.
7. Preserve structured sanitized test-data rows or fixtures separately. Embed descriptions, field constraints and useful sanitized examples, not every row indiscriminately. Resolve exact fixture values through authorized structured lookup after retrieval.
8. Embed sanitized chunks through the project's active embedding adapter. Record provider, model, dimensions, preprocessing version and embedding generation. Enforce compatible query/document embeddings.
9. Write versioned chunks and update the text/vector indexes. Use unique keys for project, source, revision, chunk and embedding generation to make retries safe. Verify completeness and search visibility before exposing the version to generation.
10. Switch the source's current-version pointer after successful readiness checks. Retain older versions for traceability. Diff semantic content/structured fields; flag linked test revisions as potentially stale, without overwriting them.

### Connector behavior

- **Jira:** Import selected requirement issues and mapped acceptance criteria/custom fields. Jira edition and field mapping must be confirmed; Cloud REST v3 is a possible adapter, not an assumption about the user's installation. The official [Jira issues API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issues/) exposes issue operations and creation metadata.
- **Azure DevOps:** Import selected requirement work items and mapped fields. Resolve sprint scope from configured iteration paths. Release membership requires an explicit team mapping or selected work-item list; do not infer that every deployment release maps directly to requirements. The [Work Items API](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items?view=azure-devops-rest-7.1) supports reading work items.
- **Figma:** Read selected files/nodes with granted permissions and preserve node references. Its [file API](https://developers.figma.com/docs/rest-api/file-endpoints/) provides accessible file/node content. Extract labels and design structure; design-to-requirement links are user-confirmed when ambiguous. API limits and permissions must be validated for the team's account.

Connector credentials belong in Secrets Manager, not source content or prompts. Pagination, rate-limit backoff, resumable imports and per-item errors are required. Content changes appear only after reimport; UI must display the last successful import time.

## 5. Retrieval pipeline — critical

**Scope/query → authorization and masking → query preprocessing → vector + BM25 search → rank fusion → rerank → deduplicate → evidence-preserving compression → prompt/context.**

1. Freeze the selected requirement IDs and source versions for the job. For large scopes, create bounded requirement-group tasks and a parent aggregation job. Show source freshness and do not label old indexed material as current when a newer imported version is still processing.
2. Normalize whitespace/case where safe. Expand abbreviations using a project glossary and approved synonyms. Preserve exact API paths, identifiers, enum values and original terms. Search the original query as well as expansions; unknown abbreviations remain unchanged or become clarification requests.
3. Apply project, allowed-source, active-version and embedding-generation constraints inside both searches. Recheck access before constructing model context. A post-search check alone is not sufficient isolation.
4. Run MongoDB Vector Search for semantic matches and MongoDB Search for BM25 lexical matches. Use exact-match boosts for requirement IDs, API paths and field names. Do not substitute MongoDB's ordinary text index for the selected Search capability. [MongoDB scoring documentation](https://www.mongodb.com/docs/search/query/score/get-details/) describes BM25 scoring details.
5. Combine ranks using reciprocal rank fusion. Prefer application-side fusion of two filtered queries initially, making behavior explicit and avoiding a dependency on a particular server feature version. Native fusion can replace it after deployment compatibility testing; MongoDB documents [hybrid retrieval](https://www.mongodb.com/docs/vector-search/hybrid-search/hybrid-search-overview/).
6. Rerank the bounded candidate set using the configured generation provider in a scoring-only call, supplied only with approved sanitized snippets. This avoids adding a provider outside the chosen stack. Validate returned candidate IDs. If reranking fails after a bounded retry, retain fused ranking and mark the job degraded; measure its effect in evaluation.
7. Deduplicate repeated imports by content/source identity, then identify semantic overlaps. Preserve contradictory passages and provenance; similarity is not proof that two requirements are interchangeable. Keep existing tests distinct from authoritative requirements.
8. Pack original selected requirements and the best supporting excerpts into the context budget. Compress long supporting material with source IDs and locators, preserving conditions, numbers and negations. Keep decisive expected-result evidence verbatim where possible; a summary never becomes the source of truth.
9. Produce an evidence bundle linking every excerpt to a source version and locator. Record retrieval configuration, candidate ranks and final evidence IDs for reproducibility.

**Proposed pilot settings:** retrieve 40 candidates per search, fuse up to 80, rerank 30, and pack around 12–20 distinct excerpts within the selected model's budget. These are adjustable starting points, not capacity guarantees. Explicitly reserve context for the requested requirements so retrieved examples cannot displace them.

Historical tests guide reuse and style, but cannot override a conflicting current requirement. Contradictions produce gap records. Exact test-data lookup follows the same authorization and approval controls as vector retrieval.

## 6. Generation and bounded checks

Persist this workflow:

`QUEUED → RESOLVE_SCOPE → RETRIEVE → GENERATE → VALIDATE → DRAFT_READY | PARTIAL_WITH_GAPS | FAILED`

Create a coverage plan across selected requirements, acceptance criteria and user-selected categories. Offer positive, negative, boundary, validation, business-rule, state-transition, permission, security, performance, accessibility and compatibility categories where relevant. Categories with no supporting basis are marked not applicable or blocked, not filled with invented cases.

The generation prompt contains task instructions, output schema, selected categories, evidence IDs and an explicit rule: source text is data, not instructions. The LLM receives no publication credentials and cannot invoke external writes. Separate supported cases from gap records; never fabricate a performance threshold, security policy, browser matrix or expected behavior.

Run checks in this order:

1. **Deterministic validation:** schema, required fields, ordered steps, source IDs, category enums, locator existence and sanitized output.
2. **Coverage check:** map cases to acceptance criteria and identify uncovered combinations. Report gaps; do not equate case count with coverage.
3. **Grounding check:** assess whether expected results are supported by the cited evidence, including conflicting sources. An AI judgment is advisory, not a proof of truth.
4. **Duplicate check:** compare intent, preconditions, steps and expected outcomes against the batch and existing approved tests. Suggest reuse or flag overlaps; do not delete distinct boundary/negative variants solely because wording is similar.
5. **Bounded revision:** proposed maximum two repair attempts per requirement group. Exhaustion returns valid supported cases plus explicit gaps; it does not loop indefinitely.

Clarifications are saved as attributed, versioned supplemental requirements and pass the same processing policy. Regenerate only the affected scope when users resolve a gap. Generated drafts are not automatically indexed as trusted existing tests; only reviewed/approved cases are eligible for reuse and remain labeled as tests.

### Test-case contract

```typescript
type TestCase = {
  id: string;
  projectId: string;
  revision: number;
  title: string;
  requirementRefs: Array<{ sourceId: string; versionId: string; locator: string }>;
  category: string;
  priority: { value: 'high' | 'medium' | 'low'; rationale: string; proposed: boolean };
  preconditions: string[];
  testData: Array<{ fixtureRef?: string; values?: Record<string, unknown> }>;
  steps: Array<{
    number: number;
    action: string;
    expectedResult: string;
    evidenceIds: string[];
  }>;
  status: 'draft' | 'approved' | 'rejected';
  freshness: 'current' | 'needs_review';
  validation: { schemaValid: boolean; issues: string[] };
  provenance: { jobId: string; provider: string; model: string; promptVersion: string };
};
```

Configure category vocabulary centrally. Priority follows a project rubric; absent explicit evidence, mark it proposed for review. Approval records include actor, time and exact revision hash. Any edit or relevant source change invalidates approval for future publishing until re-reviewed.

## 7. MongoDB records and indexes

| Collection | Essential content |
|---|---|
| projects / memberships | Team access, roles, approved providers, active embedding generation, glossary and category settings. |
| connectors / sources | External identity, secret reference, source ownership, classification, approval and current version. |
| source_versions | Immutable revision, hash, S3 reference, extraction status, import time and supersession details. |
| chunks | Sanitized content, source/version/locator, project and access tags, embedding vector and model metadata. |
| test_data | Sanitized fixtures, types, constraints and source-version links. |
| jobs / job_items | Durable workflow stage, attempts, lease, cancellation, progress, errors and model/config snapshots. |
| test_cases / evidence_links | Immutable case revisions, evidence, requirement dependencies and freshness. |
| gaps / approvals | Clarification state and revision-bound review decisions. |
| outbox / publications / audit_events | Reliable dispatch, destination mapping, remote identity, outcomes and actor history. |

Create B-tree indexes for project-scoped lists, source lookup, job state and requirement-to-case impact lookup. Use unique indexes on source identity/revision and delivery idempotency keys. Configure Search fields for text plus exact identifiers; configure Vector Search vector dimensions and filter fields for project, source/version and embedding generation.

Use a compatible index/collection generation for each embedding model configuration. Never compare vectors from different models, even if dimensions happen to match. A model change builds and validates a new complete index generation before switching the project pointer; retain the old generation for rollback under retention policy. Jobs pin one generation throughout execution.

## 8. Review, exports and Jira publishing

Review UI shows requirement evidence beside expected results, missing-information records, duplicate warnings, source freshness and category coverage. Approval is per case revision or an explicitly selected batch; a separate reviewer is not required by the selected workflow.

Generate all formats from the same canonical records:

- **XLSX:** Cases, Steps, Traceability and Gaps sheets, joined by test-case ID. Include status, revision and freshness.
- **CSV:** A documented one-row-per-step layout, repeating case-level fields; ship a separate gaps CSV when needed. Escape delimiters and neutralize spreadsheet-formula injection.
- **JSON:** Preserve nested steps, evidence references, test data and validation status under a versioned export schema.

Draft files display DRAFT in filenames and status fields. Use short-lived authorized download links. Exports contain sanitized content, not original sensitive values.

Publishing is **designed but configuration-blocked** until the team identifies Xray, Zephyr or direct Jira issues, confirms Jira edition, and supplies target field mappings. Keep the publishing control disabled until its adapter is configured and contract-tested. Do not guess an app-specific endpoint or claim Jira's issue API provides native test-step fields.

Delivery sequence: recheck project permission → require current approved revision → validate mapping → record pending delivery → call destination adapter → record remote ID/result → display confirmation or per-case error. An ordinary Jira adapter, if selected, maps canonical cases into the configured issue type and fields; an app adapter uses that app's supported API.

Use an idempotency ledger keyed by project, case revision and destination. A timed-out remote write may have succeeded: reconcile using a stored correlation marker or remote lookup before retrying. If reconciliation is impossible, flag manual review instead of risking duplicate creation. Do not claim exactly-once delivery across an external API. Previously published cases are never silently rewritten by regeneration.

## 9. AWS deployment

Recommended deployment layout:

- React static assets in S3 behind CloudFront; TLS for frontend and API.
- Application Load Balancer in public subnets; Fargate API and workers in private subnets across availability zones for production.
- Separate ECS services for API, ingestion, generation and delivery; share images where practical but configure distinct entry points, IAM roles and scaling policies.
- SQS queues with dead-letter queues; S3 private buckets; Secrets Manager for credentials; KMS encryption; ECR for images; CloudWatch for logs, metrics and alarms.
- MongoDB Atlas deployed on AWS in the selected region, preferably with private connectivity supported by the chosen tier. Atlas is managed separately from the application VPC; do not depict it as an ECS container.
- Controlled outbound access to selected SaaS connectors and model APIs. External AI processing is outside the AWS application boundary and requires the approved-source policy.

For authentication, recommend Amazon Cognito federated with the organization's identity provider where available. The exact identity provider remains an implementation decision. API authorization derives project membership server-side; a client-supplied project ID alone never grants access.

Fargate networking uses the documented [task networking model](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html). Assign narrow [task IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html) independently to API and workers.

Scale API on request latency/utilization and workers on queue age/backlog, while enforcing project concurrency and provider quota limits. Keep instance counts, resource sizes and budgets configurable. No throughput, latency, availability or cost guarantee is inferred from the questionnaire.

## 10. Reliability and security controls

SQS standard queues have [at-least-once delivery](https://docs.aws.amazon.com/en_gb/AWSSimpleQueueService/latest/SQSDeveloperGuide/standard-queues-at-least-once-delivery.html); every worker therefore uses idempotent writes, stage checkpoints and job leases. Extend visibility while running; cap attempts and route exhausted work to dead-letter queues. Use a MongoDB transactional outbox for job creation plus dispatch intent, with an idempotent dispatcher. Cancellation is checked between stages and before external writes.

Handle provider 429/temporary failures with bounded backoff and jitter. Do not automatically fail over to an unapproved provider. If either search branch fails, retry then mark generation blocked/failed rather than silently representing single-mode retrieval as hybrid. Partial batches report completed, failed and gap counts separately.

Source deletion or access revocation removes retrieval eligibility promptly, independent of index update lag. Recheck current source permissions before any external model request and export/publication. Connector access must not broaden application access: restricted source items retain restrictions or are rejected if those restrictions cannot be represented. Approval does not grant source access.

Treat retrieved documents as untrusted content. They cannot change system instructions, authorize tool calls or trigger writes. Restrict parser network access, remote references in OpenAPI documents and connector destinations to avoid unintended outbound fetches. Log IDs, timing and sanitized errors rather than raw prompts/documents by default.

Retain encrypted originals, source versions and audits according to an agreed policy; retention period, backup objectives and deletion rules remain open. Deletion must cover S3 objects, chunks/vectors, cached content and derived artifacts as applicable; preserve only permitted audit metadata.

## 11. Provider abstraction

Expose `generateStructured`, `embedDocuments`, and `embedQuery` interfaces separately. Configure generation provider/model, embedding provider/model, dimensions, token limits, timeout and data-approval status per project. Pin the complete configuration per job. Reuse the generation adapter for reranking and semantic checks with separate prompt versions.

OpenAI documents [schema-constrained outputs](https://developers.openai.com/api/docs/guides/structured-outputs). Groq's [structured output support](https://console.groq.com/docs/structured-outputs) varies by model and mode; capability-test the exact selected model. Validate output in TypeScript even when provider schema enforcement is available. Mistral provides a documented [embedding API](https://docs.mistral.ai/api/endpoint/embeddings).

No exact model winner is asserted. Evaluate approved candidates against the same reference cases for grounding, coverage, latency and cost. Never switch embedding providers for an individual failed call; embedding changes require the complete migration described above. Check current provider retention terms and region options before real project data is enabled.

## 12. API and workflow boundaries

Suggested project-scoped endpoints:

| Endpoint | Behavior |
|---|---|
| `POST /projects/:id/imports` | Enqueue selected files/items; return job ID. |
| `POST /projects/:id/sources/:sourceId/approval` | Administrator records external-processing decision. |
| `POST /projects/:id/generations` | Accept scope, categories and idempotency key; return job ID. |
| `GET /projects/:id/jobs/:jobId` | Progress, gaps, failures and links to results. |
| `POST /projects/:id/gaps/:gapId/clarifications` | Save attributed supplemental requirement. |
| `PATCH /projects/:id/test-cases/:caseId` | Create new draft revision with optimistic concurrency. |
| `POST /projects/:id/test-cases/:caseId/approve` | Approve the specified validated, current revision. |
| `POST /projects/:id/exports` | Queue export of selected draft/approved revisions. |
| `POST /projects/:id/publications` | Queue only approved current revisions for configured Jira destination. |

Use polling initially for progress, keeping long-running jobs outside HTTP request lifetimes. Enforce authorization on every endpoint, including job and artifact reads.

## 13. Validation and acceptance plan

Build a human-reviewed evaluation set from representative approved requirements, API specifications, existing tests and sanitized test data. Include ambiguous rules, conflicting versions, duplicates, exact identifiers and missing performance criteria.

| Area | Required evidence before production |
|---|---|
| Ingestion | Accurate table/step extraction, repeat-import idempotency, no external calls for unapproved sources, recoverable partial imports. |
| Retrieval | Relevant-evidence recall and ranking reviewed by testers; exact identifier retrieval; no cross-project or revoked-source disclosure in adversarial tests. |
| Generation | Measured unsupported-expected-result rate, criterion coverage, duplicate rate and reviewer acceptance; empty evidence yields gaps, not invented cases. |
| Revision control | Source change flags affected tests, edits invalidate approval, jobs reproduce their pinned versions. |
| Delivery | Export round-trip checks, mapping contract tests, draft/stale publication blocked, timeout reconciliation avoids duplicate writes. |
| Operations | Crash/retry recovery, queue replay, provider throttling, cancellation and representative release-size load tests. |

Track p50/p95 duration by stage, queue age, failure rate, tokens/cost per job, ingestion freshness, retrieval quality and reviewer edits. Set numeric quality and service targets with pilot evidence; they are not established user requirements. Schema validity is a deterministic gate; semantic correctness remains subject to evidence checks and human review.

## 14. Implementation sequence and unresolved decisions

1. Implement access control, canonical records, uploads, source approval/masking, versioning and export schema.
2. Add Jira/ADO/Figma imports, model adapters and index readiness/migration handling.
3. Implement hybrid retrieval, evidence bundles, bounded generation/checks and clarifications.
4. Complete review, impact tracking and all three export formats.
5. Configure and test the confirmed Jira publishing adapter; it remains a required first-release capability, not removed from scope.
6. Deploy the ECS/Fargate services, run security/recovery/evaluation/load tests, and tune capacity.

Still to confirm during implementation: Jira edition and test-management app; destination custom fields; source field mappings and release membership; organizational identity provider; exact approved models/provider terms; AWS/Atlas region and tier; upload limits; retention/backup targets; operating budget and measured capacity targets. These are explicitly unresolved, not invented requirements. The architecture can be implemented incrementally, but Jira publishing cannot be completed correctly without its destination details.
