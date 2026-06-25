# Deterministic playbook matching and auto-decision for contract negotiation

**One-liner:** A retrieval and decision engine that maps each tracked change in a redlined contract to the correct entry in a structured negotiation knowledge base, then pre-populates accept, reject, or counter decisions from the matched entry's fields, so the pipeline can run end-to-end with zero LLM calls when every change has a clean playbook match.

---

## Summary

- **Problem:** A redlined contract contains dozens of tracked changes. Each one must be matched to the organization's negotiation playbook (the authoritative source for what to accept, reject, or counter) before a decision can be made. The playbook is generally large, and each entry is a structured record with position statements, pre-approved counter-language, hard limits, escalation triggers, and search keywords. Matching is not a simple lookup: tracked changes are fragments of contract language with no labels, while playbook entries are organized by negotiation concept. The same change can plausibly match multiple entries, and the right match often depends on which agreement section the change falls in, what factual values it contains, and how discriminating the overlapping keywords are. Getting this wrong means the pipeline applies the wrong position, which is worse than doing nothing.

- **Approach:** Treat matching as a **weighted multi-signal scoring problem**, not as keyword search or semantic similarity. For each tracked change, build a **search corpus** from the change's text, its detected section heading, and its surrounding paragraph context. Score every playbook entry against that corpus using three independent signals combined by fixed weights: **keyword relevance** (TF-IDF weighted, 70%), **section alignment** (heading overlap, 25%), and **factual anchoring** (duration and monetary values, 5%). Apply confidence thresholds to sort matches into auto-decide-eligible (high confidence), LLM-review (medium), and playbook gaps (no viable match). Then run a **deterministic decision engine** over each high-confidence match: read the matched entry's structured fields in a fixed order (escalation triggers first, then hard limits, then position signals, then available counter-language) to emit an action without interpretation. Guardrails catch cases where the match score is high but the change is too substantive to auto-accept without review.

- **Stack:** Python, regex-based keyword matching with IDF weighting, structured text parsing, JSON artifacts for pipeline handoff.

---

## Why matching is harder than it looks

**Changes are fragments, not topics.** A tracked insertion might read "three times the fees paid during the twelve months preceding the claim" with no heading, no label, and no indication that this is a liability cap proposal. The matcher must connect that fragment to the right playbook entry about limitation of liability, not the entry about fee disputes or the entry about payment terms, all of which share overlapping vocabulary.

**Keyword overlap is high across entries.** Negotiation playbooks reuse the same language: "indemnification," "limitation," "liability," "breach," "termination." A keyword-only approach produces ambiguous rankings where the top five matches are all plausible. Without a second signal to break ties, the system either picks wrong or punts everything to human review, which defeats the purpose.

**Context matters, but loosely.** The paragraph surrounding a change provides useful matching surface (it names the section, references defined terms, establishes the contractual context), but it also introduces noise. A change inside the indemnification section that references "breach" should not match the breach-notification playbook entry just because the word appears nearby.

**Factual values are rare but decisive.** When a change proposes "30 days" and a playbook entry specifically addresses a 30-day notice period, that is strong evidence of a correct match. But most changes contain no factual values at all, so the signal must be weighted as a tiebreaker, not a primary discriminator.

---

## Scoring model

Matching uses a **composite score** built from three independent signals, each targeting a different failure mode:

### Keyword relevance (70% weight)

Each playbook entry carries a curated set of **search keywords**. These are matched against the change's text using **inverse document frequency weighting**: keywords that appear in fewer playbook entries receive higher weight because they are more discriminating. A keyword like "termination" that appears in eight entries contributes less to any single match than a keyword like "subprocessor" that appears in one.

The matching runs against two corpora per change:

- **Primary corpus:** The change's own text, summary, and section heading. This is the high-signal surface.
- **Full corpus:** Primary plus surrounding paragraph context. This is the fallback surface, discounted (multiplied by a factor below 1.0) to reflect that context matches are weaker evidence than direct text matches.

The matcher uses the full corpus only when the primary corpus score is below a threshold, preventing paragraph noise from inflating scores when the change text alone is sufficient.

**Score:** Ratio of matched keyword weight to total keyword weight for each entry.

### Section alignment (25% weight)

The change's detected section heading is compared against the playbook entry's topic area. Exact containment (the topic name appears inside the section heading, or vice versa) scores 1.0. Partial word overlap is scored proportionally after removing trivial words.

This signal is the primary tiebreaker: when multiple entries share keywords with a change, the one whose topic area matches the EA section usually wins.

### Factual anchoring (5% weight)

Duration values, monetary amounts, and multipliers extracted from the change text are checked against the playbook entry's full text. A literal match (the same "30 days" or "$1,000,000" string appears in both) scores 1.0. No factual overlap scores 0.0.

This signal fires rarely but resolves ambiguity when it does: two entries about notification periods might score identically on keywords and section, but only one mentions the specific duration the customer proposed.

### Combined score and thresholds

`combined_score = keyword_score * 0.70 + section_score * 0.25 + fact_score * 0.05`

Three confidence bands:

- **>= 0.6:** High confidence. The match is eligible for deterministic auto-decision.
- **>= 0.3 but < 0.6:** Low confidence. The match is surfaced as context for an LLM or human reviewer, but the system does not act on it autonomously.
- **< 0.3:** Discarded. If no match survives this threshold, the change is flagged as a **playbook gap** (a change the playbook does not address).

### Two-pass fallback

If no matches survive the combined threshold, a second pass runs **section-only matching** (section score >= 0.5, no keyword requirement). This catches changes where the playbook entry lacks good search keywords but lives in the correct topic area. These matches are scored conservatively and always routed to review, never auto-decided.

Up to five matches are retained per change, sorted by combined score, so downstream steps can consider alternatives when the top match is wrong.

---

## Auto-decide engine

For each change with a high-confidence match (combined score >= 0.6), a **deterministic decision engine** reads the matched playbook entry's structured fields in a fixed order and emits an action. The ordering encodes institutional priorities: safety constraints fire before convenience logic.

### Decision logic (evaluated in order, first match wins)

1. **Escalation triggers present:** If the entry's Approval Required field is populated, action = `needs_review`. Rationale: the playbook requires a specific person or group to approve any deviation. The system does not substitute its own judgment for an escalation gate.

2. **Hard limit present:** Action = `reject`. Comment = the entry's pre-approved customer-facing language (used verbatim). Hard limits are non-negotiable by definition; no further analysis is needed.

3. **Reject signals in the organization's stated position:** If the position field contains explicit rejection language ("reject," "decline," "cannot accept," "non-negotiable"), check whether the entry also provides pre-approved counter-language. If yes, action = `counter` (offer the alternative). If no, action = `reject`.

4. **Accept signals in internal guidance:** If the guidance field contains explicit acceptance language ("accept," "agree," "pre-approved," "reasonable"), action = `accept`. Comment = the entry's customer-facing language if available, otherwise a generic acknowledgment.

5. **Pre-approved counter-language available:** If the entry provides alternate contract language but none of the above signals fired, action = `counter`. The presence of alternate language implies the organization anticipated this change and prepared a response.

6. **None of the above:** Action = `needs_review`. The entry exists but its guidance requires interpretation that a deterministic engine should not attempt.

### Guardrails

Auto-decisions are subject to safety checks before they become final:

- **Substantive insertion guard:** Insertions longer than 15 words are not auto-accepted unless the match score is >= 0.7. Long insertions are more likely to contain language that deserves human review even when the playbook signals acceptance.
- **Group size guard:** The same 15-word / 0.7-score threshold applies to change groups (multiple related tracked changes evaluated as a unit).
- **Structural dependency guard:** Format-only changes (font, spacing, numbering) that share a paragraph with substantive changes are not auto-accepted, because the formatting may be structurally dependent on the substantive edit (accepting the format change while rejecting the substance can produce an inconsistent document).

### Output per decision

Each auto-decision carries: the action, a confidence level (high / medium / low / none), a color code for triage dashboards, the matched playbook entry reference with score and keywords, a rationale string explaining why this action was chosen, a pre-populated comment draft (sourced from the playbook's pre-approved language when available, with source attribution), pre-populated counter-text (from the playbook's alternate language when applicable), and an array of flags (hard_limit, escalation, playbook_gap, format_only, ambiguous_match, structural_dependency).

Items that the auto-decide engine cannot resolve are passed to an LLM with the matched entry as structured context, not as an open-ended prompt. The deterministic layer does the retrieval and narrows the problem; the model does the interpretation.

---

## Tradeoffs

- **Fixed weights over learned weights.** The 70/25/5 split was tuned by hand against real redlines, not trained. This sacrifices optimality for transparency: an attorney can understand why a match scored the way it did by inspecting three numbers, and the weights can be adjusted without retraining anything. In a domain where the knowledge base has 150 entries and the document volume is dozens per quarter (not millions), learned weights would overfit.

- **IDF over embeddings.** Semantic similarity via embeddings would handle paraphrase better, but it would also introduce opaque match explanations and require a vector store. IDF-weighted keyword matching is legible, deterministic, and sufficient when the playbook's search keywords are well-curated. The tradeoff is that the system depends on keyword curation quality; a playbook entry with poor keywords will underperform regardless of how relevant it is.

- **Conservative thresholds over aggressive automation.** The 0.6 auto-decide threshold means some clear matches get routed to review unnecessarily. The alternative (lowering the threshold) risks auto-deciding on weak matches, which in a legal workflow is strictly worse than asking a human. The system is tuned for **precision over recall** on autonomous actions.

- **First-match-wins decision logic over weighted interpretation.** The decision engine does not weigh competing signals within a playbook entry (e.g., an entry with both accept signals and an escalation trigger). It applies a fixed priority: escalation wins. This avoids the complexity of signal balancing but means the engine's behavior is only as good as the playbook's internal consistency. An entry that says "accept" in guidance but has an Approval Required field will always route to review, even if the approval requirement is stale. That is the correct failure mode for a governed workflow.

- **Guardrails as overrides, not filters.** The auto-accept guardrails can demote a decision from `accept` to `needs_review`, but they never promote. A change that fails a guardrail check is always reviewed by a human or model; a change that passes is still subject to the decision logic. This asymmetry is intentional: false negatives (unnecessary review) cost time; false positives (wrong autonomous action) cost trust.

---

## Outcome

For a typical redlined agreement, the matcher resolves 60-80% of tracked changes to a high-confidence playbook entry, and the auto-decide engine pre-populates decisions for those changes without any LLM call. The remaining changes are routed to review with their best-match context attached, so the reviewer starts with a hypothesis rather than a blank slate. The entire matching and decision layer is deterministic, unit-testable with fixture data, and produces a structured JSON artifact that downstream phases (validation, document building) consume without re-interpreting the playbook.
