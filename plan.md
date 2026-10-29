# Implementation plan

## Ordered steps

1. Confirm the paths with `ls`: `src/portal/result-route.mjs` and `src/service/queue.mjs` exist; the archived helper `sendAllResults.mjs` does not exist and is rejected, not used.
2. Add the release gate (R1) in a proposed new file `src/service/notify.mjs` (new file): only reviewed and released results continue; sending on upload is rejected.
3. Add idempotency by released result version in the new file `src/service/notify.mjs` (R5), so one version is never delivered twice.
4. Add batching of at most 20 messages and a limit of 50 requests per second (R6), handing jobs to the existing `enqueueNotification` in `src/service/queue.mjs`.
5. Add bounded retries on provider failure that reuse the same idempotency key (R7), with logging of event IDs and delivery states only (R3).
6. Build the notification text with no clinical content and a link from the existing route in `src/portal/result-route.mjs` (R2, R4); a portal redesign is rejected.
7. Write the named tests listed below (they do not exist yet; none are created in this lab).

## Paths and dependencies

- Existing, reused: `src/portal/result-route.mjs` (the portal route for the link) and `src/service/queue.mjs` (the queue adapter, `enqueueNotification`).
- Proposed, new file: `src/service/notify.mjs` (new file) holds the gate, idempotency, batching and retry logic.
- Not used: the archived helper `sendAllResults.mjs` never existed and is rejected; nothing depends on it.
- Order: the gate (step 2) comes first, then idempotency (step 3), then batching and rate limit (step 4), then retries and logging (step 5), then the message and link (step 6). Tests follow each step.
- Open decisions block only the channel and referring-doctor work: OPEN-1 (channel) is owned by product/privacy and OPEN-2 (referring doctors) by clinical/privacy; both are out of scope until approved.

## Risks and verification

Clinical content stays out of notifications and logs; only event IDs and delivery states are logged, and the provider limits are 50 requests per second and 20 messages per batch.

- Risk: a missing release gate sends early notifications; verified by gate_blocks_unreleased_result.
- Risk: a retry delivers twice; verified by duplicate_retry_same_version_single_delivery and bounded_retry_on_provider_failure.
- Risk: provider limits exceeded; verified by throughput_boundary_50_rps_20_per_batch at exactly 50 and 20, and just above.
- Risk: privacy leak in text or logs; verified by payload_has_no_clinical_content and logs_only_event_id_and_state.
- Risk: the link drifts from the existing route; verified by link_uses_existing_portal_route.
- Check: `grep -on 'src/[A-Za-z0-9_./-]*\.mjs' spec.md plan.md`, then `ls` each path; `git status` shows only `spec.md` and `plan.md` changed.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1 | 2 | - | gate_blocks_unreleased_result |
| R2 | 6 | - | payload_has_no_clinical_content |
| R3 | 5 | - | logs_only_event_id_and_state |
| R4 | 6 | src/portal/result-route.mjs | link_uses_existing_portal_route |
| R5 | 3 | - | duplicate_retry_same_version_single_delivery |
| R6 | 4 | src/service/queue.mjs | throughput_boundary_50_rps_20_per_batch |
| R7 | 5 | src/service/queue.mjs | bounded_retry_on_provider_failure |
