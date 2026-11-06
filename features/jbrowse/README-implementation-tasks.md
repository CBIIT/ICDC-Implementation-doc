# Reference Mapping Implementation Tasks
**Parent story:** [ICDC-4299](https://tracker.nci.nih.gov/browse/ICDC-4299). All tasks below support this story.

# Reference Mapping Implementation Tasks

These tasks begin after solution approval. This file is a planning checklist; it does not indicate that implementation or Production inventory is complete.

## Ordered task list

| Order | Task title | Summary |
| --- | --- | --- |
| 1 | Identify reference files for S3 hosting | Inspect study-file headers, identify assemblies and custom additions, and inventory authoritative reference files and matching indexes. |
| 2 | Prepare S3 reference storage | Upload approved reference files and indexes, verify consistency, and configure browser byte-range access. |
| 3 | Create the reference catalog | Map assembly IDs to sequence dictionaries, asset URLs, and JBrowse configuration. |
| 4 | Confirm loader integration | Define resolver execution, file/index inputs, result storage, and publication ownership. |
| 5 | Review dependencies and security | Check versions and advisories; establish patching, permissions, resource limits, and credential protection. |
| 6 | Implement header collection | Extract reference metadata from remote BAM/VCF headers using bounded reads. |
| 7 | Implement reference matching | Validate evidence against the catalog and record confirmed, ambiguous, conflicting, or unresolved mappings. |
| 8 | Implement initial-region discovery | Use indexes and bounded record reads to find a region containing data, with explicit fallback outcomes. |
| 9 | Persist inspection results | Save mappings, regions, file/index identities, and evidence; reuse results while inputs remain valid. |
| 10 | Integrate OpenSearch and the API | Publish summary fields through the existing loader and metadata API, with model review where required. |
| 11 | Update JBrowse startup | Select the matching reference and region, restrict annotations, and separate incompatible reference groups. |
| 12 | Validate and roll out | Test correctness, security, failure handling, and reindex reuse; backfill eligible files and monitor Production behavior. |

## Task details

### 1. Identify reference files for S3 hosting

**Scope:** Enumerate Production studies and their viewable BAM/VCF files and matching indexes. Initially inspect 2–5 randomly selected files per study, or all available files when fewer exist. Record the sample method, file UUIDs, and actual coverage. Use manual/header tooling for this discovery task; it does not depend on the future automated resolver.

Create a hosting inventory with study/file evidence, assembly name and accession/version, sequence naming, authoritative source URL, format, estimated size, required reference indexes, and missing assets. Check custom additions such as ERCC or decoys and distribution differences such as chromosome naming. Include the currently configured references in the assessment; do not assume six browser labels cover all files.

**Dependencies:** Approved design and authorized access to Production file records and headers.

**Acceptance:** Each study has sampled evidence and each discovered reference has a proposed compatible asset set or a documented gap. Sampling does not assign a validated mapping to uninspected files. Unexplained differences are recorded for follow-up.

### 2. Prepare S3 reference storage

**Scope:** Acquire approved exact reference distributions from authoritative sources. Publish complete reference files and indexes generated from those exact files, using stable versioned locations. Use a JBrowse-supported format, such as uncompressed FASTA with FAI. Configure read access, browser CORS, and byte-range responses.

**Dependencies:** Task 1 inventory and approved S3 locations/access policies.

**Acceptance:** Each published asset set passes sequence-name/length, index consistency, and browser range-access checks. Record provenance and checksums. Reference-only viewing works for known loci; missing custom sequences remain explicit gaps.

### 3. Create the reference catalog

**Scope:** Define versioned catalog entries containing assembly ID, reference distribution ID where needed, sequence dictionary, verified aliases, reference/index URLs, and JBrowse configuration. Record which annotations are compatible.

**Dependencies:** Task 1 inventory and Task 2 validated asset locations.

**Acceptance:** Each supported distribution resolves to one complete catalog entry. Dictionary and alias checks pass. Unsupported assemblies are represented clearly, and canFam3 annotation data is not assigned to other assemblies.

### 4. Confirm loader integration

**Scope:** Identify the existing indexing/reindexing entry point, file-service access, durable result store, and loader ownership of published fields. Specify file UUID and immutable content identity, matching index identity, and the module/task or container interface.

**Dependencies:** Approved design; coordinate with the existing service owners. Discovery can run alongside Tasks 1–3.

**Acceptance:** Document the selected runtime, input/output contract, retry behavior, and result storage. Both incremental indexing and full reindexing can consume saved results. No additional Production processing trigger is required.

### 5. Review dependencies and security

**Scope:** Inventory actual Python, SAMtools, BCFtools, bundled/linked HTSlib, compression/TLS, frontend, and container OS versions. Scan for CVEs and other security advisories, assess affected processing paths, and select patched pinned versions.

Restrict file endpoints and redirects to approved destinations. Run parsing without elevated privileges, enforce resource/byte limits, use argument lists rather than shell commands, and redact credentials and signed URLs. Prefer a restricted resolver container if the loader holds broad Production credentials.

**Dependencies:** Task 4 runtime decision and actual dependency manifests/image details. Complete the initial review before deploying the resolver; repeat scans on the final image.

**Acceptance:** Record findings, affected versions, remediation or approved exceptions, and scan date. Security tests cover endpoint restrictions, malformed files, oversized responses, and credential redaction. Do not claim a clean CVE audit without the deployed dependency inventory.

### 6. Implement header collection

**Scope:** Use SAMtools/HTSlib for BAM and BCFtools/HTSlib for VCF. Read remote headers within configured time, memory, compressed/decompressed-byte, and response budgets. Obtain current access when processing starts.

Extract BAM sequence names/lengths and available AS/M5/UR/AN fields, plus reference evidence in PG/CO text. Extract VCF reference/contig declarations and applicable command metadata. Treat header text as data.

**Dependencies:** Tasks 4–5.

**Acceptance:** Header extraction works without a BAM index and stops without scanning the full alignment file. Transfer metrics demonstrate bounded behavior. Truncated/malformed headers, expired access, and unsupported server range behavior yield explicit failures.

### 7. Implement reference matching

**Scope:** Generate assembly candidates from declarations and command evidence, then compare the full available sequence dictionary with catalog entries. Preserve accession versions and use verified aliases. Distinguish exact matches, compatible subsets, custom additions, ambiguous results, conflicts, and unresolved references.

**Dependencies:** Tasks 3 and 6.

**Acceptance:** Known ROS, GSD, and canFam3 examples and synthetic conflict/custom-reference cases produce expected outcomes. A filename hint or one shared contig is not enough for automatic acceptance. Never silently select canFam3 for an unmatched file. Name/length matching establishes dictionary compatibility, not proof of identical reference bases.

### 8. Implement initial-region discovery

**Scope:** Use the matching BAI/CSI or TBI/CSI and a bounded number of actual records to choose a compatible contig and window containing visible reads or variants. Check sequence bounds and define the coordinate convention.

**Dependencies:** Tasks 6–7 and usable matching data indexes.

**Acceptance:** Successful discovery opens visible data. Missing or invalid indexes, retrieval errors, and exhausted search budgets have explicit statuses. A valid reference default is labeled as not confirmed to contain data. Failure to find records within the budget is not reported as an empty file.

### 9. Persist inspection results

**Scope:** Save file UUID, immutable data identity, index identity, reference mapping, initial region, evidence location, and resolver/catalog/schema versions. Keep detailed headers and dictionaries separately from the search summary.

Reuse accepted results for unchanged inputs. Invalidate assembly mapping when data identity or relevant resolver/catalog versions change. Recompute the preview when index identity changes; index arrival alone need not invalidate header-only mapping.

**Dependencies:** Task 4 storage contract and Tasks 7–8 result definitions. Storage implementation can progress alongside resolver development.

**Acceptance:** Results survive service restart and full reindex. Stale completions cannot overwrite current results. Cache reuse and invalidation are tested, including missing-index arrival and file replacement.

### 10. Integrate OpenSearch and the API

**Scope:** Have the existing loader join accepted results to the correct file and publish explicit summary fields. Expose the assembly/catalog reference, mapping/browser status, and initial-region summary through the existing metadata API. Confirm field names and complete the established data-model update process if official scientific properties are added.

**Dependencies:** Tasks 4 and 9, plus approval for any governed model changes.

**Acceptance:** Both incremental loads and full rebuilds preserve mapping summaries. Explicit field mappings prevent raw headers or contig names from becoming dynamic fields. Bulk publication checks each item and retries only failed writes; the loader remains the single publication owner.

### 11. Update JBrowse startup

**Scope:** Build the assembly/session from accepted metadata and versioned catalog configuration. Use the discovered initial region or a labeled valid fallback. Group multiple selected files by compatible reference distribution and offer a per-file jump to data.

**Dependencies:** Tasks 2–3 and 10.

**Acceptance:** Supported files open on the correct reference. Unknown, conflicting, or unavailable references produce actionable status messages. Different assemblies open in separate views/groups. NCBI, Ensembl, and human-to-canine tracks containing canFam3 data are restricted to canFam3.

### 12. Validate and roll out

**Scope:** Pilot representative BAM/VCF files, custom references, malformed inputs, missing indexes, expired URLs, and mixed-reference selections. Verify security controls and resource budgets. Process every file requiring automatic reference selection, reconcile study coverage, and roll out using the existing indexing path.

**Dependencies:** Tasks 1–11 and any required operational/model approvals.

**Acceptance:** Record inspection coverage, unresolved exceptions, preview outcomes, bytes transferred, latency, retries, and dependency scan results. Full reindex reuses valid results. Automatic selection is enabled only for accepted mappings with compatible assets. Document operational ownership and rollback before Production rollout.

