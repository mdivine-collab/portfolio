# Constraining probabilistic classification inside an idempotent event pipeline

**One-liner:** A reaction-triggered routing engine that uses deterministic gates, idempotent business-event claims, and compiled configuration bundles to constrain an LLM classifier's output to a narrow, validated action space, falling back to interactive human disambiguation when model confidence is insufficient.

---

## Summary

- **Problem:** A team uses Slack emoji reactions to manage work items in a project tracker. Some reactions have a fixed meaning ("move this ticket to In Review"), but others are open-ended ("do the right thing with this thread"). The open-ended case requires reading a Slack thread, extracting context (issue keys, participants, URLs), and deciding whether to create a new ticket, update an existing one, add a comment, or transition a status. That classification is a natural fit for a language model. But model output is probabilistic: it may return a plausible-sounding action with no grounding, hallucinate issue keys, or fail silently. In a workflow that writes to a project tracker, an unconstrained model response can create duplicate tickets, update the wrong issue, or transition a ticket to an invalid state. The system must also handle the real-world problem that Slack delivers reaction events at least once, meaning the same emoji can arrive multiple times for the same message.

- **Approach:** Build a **typed event pipeline** where the model is one participant, not the orchestrator. Reactions enter through a **webhook intake layer** that normalizes payloads, enforces identity and reaction allowlists, and claims each business event against a **deduplication store** using a composite key (workspace, channel, message, user, reaction). Duplicate deliveries are rejected before any classification runs. Accepted events are routed through one of two **lanes**: a **deterministic lane** for reactions with fixed semantics (direct emoji-to-status mapping, confidence 1.0, no model involved), or a **generic AI lane** where a model classifier receives the normalized thread context and returns a structured prediction validated against a **Zod schema** (action, confidence score, reasoning, candidate issue keys). The schema enforces that the model's output is one of exactly four actions; anything else is treated as malformed. A **configurable confidence threshold** gates whether the prediction is accepted or deferred to the user. Below-threshold predictions, malformed outputs, and classifier failures all follow the same path: the system persists a **pending ambiguity request**, delivers an interactive prompt to Slack with action buttons, and waits for the user to resolve it through a **signed webhook callback**. The entire configuration (identity allowlists, reaction rules, routing parameters, board metadata) is managed as **source inventory files** that are compiled into an **immutable release bundle** with a content-addressable SHA-256 hash, cross-checked for referential integrity at compile time, and loaded at runtime from the compiled artifact rather than raw config.

- **Stack:** TypeScript, Zod for runtime schema validation, Slack Web API and interactivity webhooks with HMAC signature verification, SQLite (local) and Upstash Redis (serverless) as swappable persistence backends behind a store interface, Vercel for deployment.

---

## Why the model needs scaffolding

**Classification is easy to attempt, hard to trust.** Given a Slack thread about a contract negotiation, a model can plausibly output "update the existing Jira ticket" with high apparent confidence. But "update" requires a target issue key. If the thread mentions two issue keys, which one? If it mentions none, the model may fabricate one. The scaffolding's job is to guarantee that every model output either resolves to a valid, executable action or is explicitly deferred to a human, with no silent middle ground.

**At-least-once delivery breaks naive handlers.** Slack's Events API guarantees at least one delivery, not exactly one. A reaction event for the same emoji on the same message can arrive two or three times within seconds. Without deduplication, each delivery triggers a separate classification and potentially a separate write to the project tracker. The deduplication layer must fire before the model, not after, because model calls are expensive and non-deterministic (the same thread context can produce different confidence scores across calls).

**Configuration drift causes invisible failures.** Reaction-to-status mappings, identity allowlists, board metadata, and AI routing parameters are interdependent. A status reactji that references a board status ID which no longer exists will fail at runtime. Catching this at deploy time requires the configuration to be compiled and cross-checked as a unit, not loaded piecemeal from scattered config files.

---

## Intake: normalize, gate, deduplicate

Every reaction event enters through a single webhook endpoint. The intake layer applies three filters in sequence before any routing logic runs:

### Payload normalization

Raw webhook payloads vary by trigger source. An adapter normalizes each payload into a typed event structure (channel, message timestamp, reacting user, reaction name, optional workspace and delivery identifiers). Malformed payloads are rejected with a typed failure reason. The adapter is an interface, so alternate trigger sources (direct API calls, test harnesses) can provide their own normalizer without changing the intake pipeline.

### Identity and reaction gates

The normalized event is checked against the compiled bundle's allowlists: is this Slack user permitted to trigger actions? Is this reaction name configured? For the AI routing lane specifically, is this user authorized to invoke model classification? Each gate produces a typed result with a reason string, so rejected events are auditable. No downstream logic runs for events that fail a gate.

### Business event deduplication

A composite business key is constructed from the event's workspace, channel, message timestamp, reacting user, and reaction name. This key is claimed against a **deduplication store** using a first-writer-wins pattern: the store returns either `first-seen` or `duplicate`. Duplicates are rejected with no side effects. The store interface has two implementations behind the same contract: SQLite for local development (file-backed, no external dependencies) and Upstash Redis for serverless deployment (survives cold starts and redeploys). A write-enable flag in the release manifest can suppress all downstream writes globally, allowing the full intake pipeline to run in dry-run mode against production traffic.

---

## Routing: two lanes, one contract

Events that survive intake are dispatched to one of two routing lanes. Both lanes produce the same typed result structure: a schema version, the business event key, a selected action (create, update, comment, or transition), a confidence score, a reason string, candidate issue keys extracted from the thread, and an optional target status for transitions. This shared contract means everything downstream of routing (persistence, Jira execution, audit logging) is indifferent to whether the decision came from a lookup table or a language model.

### Deterministic lane

Reactions with fixed semantics (e.g., a specific emoji always means "move to In Review") are resolved by a direct lookup in the compiled bundle's status-reactji mapping table. The mapping references a board status by ID; the status must exist in the bundle's board metadata (enforced at compile time by the cross-checker). Confidence is always 1.0. No model is invoked. No ambiguity is possible.

### Generic AI lane

The open-ended reaction triggers a model classifier. Before calling the model, the system loads the full Slack thread context: all messages in chronological order, with each message annotated as parent, reacted message, or reply; all Jira issue keys extracted by regex from message text; all URLs; and all participant user IDs in encounter order. This normalized context is the model's entire input, alongside the bundle's model selection parameters (provider and model name).

The model returns a raw prediction. The system validates it against a **Zod schema** that enforces: `selectedAction` is one of exactly four enum values; `confidence` is a number between 0 and 1; `reason` is a non-empty string; `candidateIssueKeys` is an array of non-empty strings. Any response that fails schema validation is treated as a classification failure: confidence drops to 0, reason is set to "generic AI output was malformed and could not be trusted," and the event routes to ambiguity resolution.

For predictions that pass validation, two additional checks apply:

- **Confidence threshold:** The bundle defines a minimum confidence for autonomous action. Below-threshold predictions are deferred to the user, regardless of what the model claimed.
- **Issue targeting check:** Non-create actions (update, comment, transition) require a `selectedIssueKey`. If the model selected an action that implies an existing ticket but did not identify one, the system overrides the state to `needs-user-choice` with an appended reason, even if confidence was above threshold.

The result: every model call either produces a validated, complete action that the system can execute, or an explicit deferral with a machine-readable reason. There is no path where a malformed or incomplete model output silently becomes a write.

---

## Ambiguity resolution: interactive fallback

When routing produces a `needs-user-choice` state (whether from low confidence, malformed output, classifier failure, or missing issue targeting), the system:

1. Generates a unique ambiguity request ID.
2. Persists the full pending request (business event key, lane, predicted action, confidence, reason, candidate issue keys, thread context) to the routing store.
3. Delivers an interactive Slack message to the reacting user with action buttons (Create, Update, Comment, Transition) and the model's predicted action and confidence for context.
4. Waits for resolution via a Slack interactivity webhook.

The interactivity webhook verifies Slack's HMAC signature (timing-safe comparison, 5-minute freshness window), confirms the responding user is the original reactor (not an arbitrary team member), resolves the selected action, and persists the choice. The pending request and resolved choice are separate records, preserving the full decision history: what the model predicted, why it was deferred, and what the human chose instead.

---

## Compiled configuration

All runtime configuration is managed as **source inventory files** (identity allowlists, reaction rules, Jira targets, board statuses, AI routing parameters) that are compiled into a single **immutable release bundle**. The compiler:

1. Loads and validates each source file against its Zod schema.
2. Runs a **cross-checker** that enforces referential integrity: every status reactji must reference a status ID that exists in the board metadata; every array must be non-empty and deduplicated; every required string must be non-blank.
3. Produces a **canonical JSON** representation (keys sorted deterministically at every nesting level) and computes a SHA-256 content hash.
4. Writes the bundle and a manifest (version, creation timestamp, content hash, source input list, write-enable flag) to a versioned release directory.

Existing releases are immutable: the compiler refuses to overwrite a complete release directory. This means any deployed version can be reproduced exactly from its manifest, and configuration changes are always a new version, never an in-place mutation.

---

## Tradeoffs

- **Schema validation over prompt engineering.** Rather than engineering the model prompt to reliably produce valid output, the system treats model output as untrusted foreign input and validates it structurally. This means the model can be swapped (provider, model name, prompt strategy) without changing the routing logic, because the contract is enforced at the boundary, not inside the model call.

- **Deduplication before classification.** Claiming business events before invoking the model wastes no compute on duplicates but means a failed claim is permanent: if the first-seen event's processing fails after claiming, the duplicate that could have succeeded is already rejected. The tradeoff favors idempotency over retry, which is correct for a system that writes to an external tracker where duplicate writes are harder to undo than missed writes are to retry manually.

- **Compiled bundles over runtime config.** Loading configuration from a pre-validated, content-hashed bundle is more rigid than reading config files at startup. It prevents hot-reloading and requires a compile step for every change. The benefit is that referential integrity errors (a reactji pointing to a deleted board status) are caught at deploy time, not at 2 AM when someone reacts to a message and the system crashes on an invalid status ID.

- **Ambiguity as a first-class state.** The system does not retry low-confidence classifications or fall back to a default action. It parks the event and asks the human. This is slower but produces a decision audit trail (model predicted X at confidence Y; human chose Z) that is unavailable when the system guesses silently. Over time, the accumulated ambiguity records become training signal for improving the classifier or tightening the confidence threshold.

---

## Outcome

The pipeline handles both the easy case (deterministic reaction mapping) and the hard case (open-ended thread classification) through a single typed contract, with the model constrained to a narrow, schema-validated output space and a human fallback for every failure mode. The architecture's value is not in the classifier itself but in the guarantees around it: every event is deduplicated before processing, every model output is validated before acting, every low-confidence prediction is surfaced rather than guessed, and every configuration change is compiled and integrity-checked before deployment.
