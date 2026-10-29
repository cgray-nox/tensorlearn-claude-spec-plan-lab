# Specification

## Requirements

- R1 (I-1): Release gate: a patient is notified only after a doctor has reviewed and released the result. An unreleased or unreviewed result is never enqueued.
- R2 (I-2): A notification contains no clinical content. It only tells the patient a result is ready and links to the authenticated portal.
- R3 (I-2): Operational logs record event IDs and delivery states only, never patient identifiers or clinical payloads. Clinical content stays out of notifications and logs.
- R4 (I-3): The notification link reuses the existing portal route in `src/portal/result-route.mjs` (`/portal/results/:resultId`); the portal is not changed.
- R5 (I-4): The idempotency key is the released result version, so one released version is never delivered twice, including on retry.
- R6 (I-4): Provider limits are respected: at most 50 requests per second and at most 20 messages per batch.
- R7 (I-4): On provider failure, retries are bounded (a fixed maximum attempt count) and reuse the same idempotency key, so a retry cannot create a duplicate delivery.
- R8 (OPEN-1, OPEN-2): The channel choice and referring-doctor notifications are out of scope until their owners approve them (see the open decisions below).

## Design

- Gate first: the release check from R1 runs before anything reaches the queue, so notifying on upload is rejected and not used.
- Sending logic goes in a proposed new file `src/service/notify.mjs` (new file, not yet written). It applies the release gate, the idempotency key from R5, 20-message batching and the 50 requests per second limit, and it hands jobs to the existing queue adapter `src/service/queue.mjs` (`enqueueNotification`).
- The existing portal route `src/portal/result-route.mjs` is reused as the only place clinical content is shown, behind authentication. A portal redesign is rejected and excluded from this work.
- The archived helper `sendAllResults.mjs` was never built and is rejected; this design does not use it or depend on it.
- Archived advice to include result details in the message is rejected; messages carry no clinical content (R2).
- Retries: a small fixed number of attempts with backoff, each reusing the same idempotency key; after the last attempt the delivery state is recorded as failed and logged by event ID only (R3, R7).

## Open decisions and owners

- OPEN-1: Which notification channel is used. Owner: product/privacy. Not decided; no channel is assumed in this spec.
- OPEN-2: Whether referring doctors are notified. Owner: clinical/privacy. Not currently in scope and not approved.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 | R1 | gate_blocks_unreleased_result: an unreleased or unreviewed result enqueues nothing; a released one enqueues once |
| I-2 | R2, R3 | payload_has_no_clinical_content: the message holds no result values or identifiers; logs_only_event_id_and_state: log lines hold only event ID and delivery state |
| I-3 | R4 | link_uses_existing_portal_route: the link is built from the existing route and no new portal route is added |
| I-4 | R5, R6, R7 | duplicate_retry_same_version_single_delivery: a retry of one released version delivers once; throughput_boundary_50_rps_20_per_batch: 20 per batch and 50 requests per second are not exceeded; bounded_retry_on_provider_failure: retries stop at the maximum |
