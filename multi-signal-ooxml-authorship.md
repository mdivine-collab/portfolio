# Multi-signal authorship for OOXML tracked changes

**One-liner:** A deterministic attribution layer for Word `.docx` redlines and comments on negotiated agreements: it classifies *who* introduced each change when native author metadata is missing, generic, or stripped—using editing-session and document-structure signals *before* optional language-model assistance, so automated workflows do not depend on brittle LLM guesses for identity.

---

## Summary

- **Problem:** Contract negotiation is shifting toward **LLM-assisted first passes**—drafting, summarizing, and responding to redlines faster. That only works if the system knows **which party** produced each tracked change and comment. **Large language models perform poorly on authorship** when Word’s **metadata is stripped or genericized** (Document Inspector, “remove personal information,” inconsistent saves across rounds) and when a single file **layers multiple negotiation turns**. Models tend to hallucinate or overfit surface text; they also lack guaranteed access to **non-textual** evidence embedded in the package (revision timestamps, revision session ids, thread structure). Relying on an LLM alone for “who wrote this?” is high-risk for any workflow that must not treat **your own** markup as the counterparty’s.

- **Approach:** Treat authorship as **structured inference** over explicit **signal tiers**, not as an open-ended model prompt. A **deterministic resolver** walks each candidate change or comment through an **ordered evidence stack**: trustworthy OOXML author fields first; then **session and locality** signals that often survive cleaning (timestamp clustering, RSID-style revision session clustering); then **comment threading**, **addressee cues** in comment bodies, and **paragraph-neighborhood** propagation from nearby anchored edits; then **deliberately weak** text-level hints; then a **safe default** (unknown → counterparty at low confidence) with an optional **constrained** model pass only where deterministic evidence is exhausted. Every decision carries **confidence**, **human-readable explanation**, and **signal codes** for audit. **Raw provenance** (author, date, RSID hooks, thread ids, placement) is preserved alongside derived party labels so later steps (for example multi-draft turn analysis) never need to re-open the XML.

- **Stack:** Python, OOXML / WordprocessingML parsing (**`lxml`**), typed analysis artifacts (**Pydantic**-style domain models).

---

## Why this problem is getting sharper

**More LLM tooling in the loop:** Teams are using models to accelerate review and response. A first-pass model that mislabels authorship can generate **wrong-sided** replies at scale.

**Metadata does not keep pace:** The same tools that make redlines portable also **break the simplest cue**—the `w:author` string—selectively within a file or across turns. **Multiple negotiation rounds** in one document compound the issue: some regions retain names, others do not, and narrative context alone is ambiguous.

**LLM limits:** Without a structured substrate, models see **flattened text** and guess. They do not reliably exploit **correlation** between anonymized revisions and sibling revisions that still carry a name, or **session boundaries** implied by revision metadata. A thin deterministic layer turns those correlations into **labeled, testable** inputs for any downstream model or rule engine.

---

## Attribution as a ranked cue inventory

Authorship probability is modeled as **competing evidence**, not a single field. Cues below are grouped by **intended reliability** in the design: stronger structural and session signals first; linguistic and stylistic signals only after structure is exhausted, because they are easier to spoof and harder to defend in a legal workflow.

### Tier A — Explicit identity (highest when available)

- **Trustworthy author metadata:** Per-revision and per-comment author attributes from OOXML when present and **not** treated as generic placeholders (for example blank, `unknown`, `author`, `editor`, `user`, `microsoft office user`, or similar tokens that appear after cleaning).

- **Configured party roster:** A known list of **internal** author names (normalized) maps to “our party”; any other non-generic name maps to counterparty. This encodes organizational reality without asking a model to memorize HR records.

### Tier B — Session and correlation (often survives metadata scrubbing)

- **Temporal clustering:** Revisions and comments that share the same **revision timestamp** (or tight temporal bucket) can be treated as one **editing burst**. If any member of that burst still has Tier A identity and all anchored members agree on party, **anonymized siblings in the same burst** inherit that party. Mixed bursts produce **no** propagation—ambiguity is not forced into a label.

- **RSID / revision-session clustering:** Word associates runs and revisions with **revision identifiers** (RSID family). These frequently persist when display names disappear. The same **unambiguous-cluster** rule applies: propagate only when anchors agree on a single party; never override Tier A when it is already trustworthy.

### Tier C — Thread and discourse context

- **Comment threading:** Comments in the same thread often share authorship. Party inferred from any **anchored** comment in the thread can propagate to **orphaned** comments in that thread.

- **Addressee / audience cues:** Comment bodies that **address** the internal party by name, handle, or company (for example `@` mentions or known internal names) are weak evidence the speaker is the **counterparty**—a common pattern in customer-facing negotiation notes.

- **Comment text alongside structure:** The literal text of a comment is a secondary signal combined with thread position, not a standalone authority.

### Tier D — Document locality

- **Paragraph neighborhood:** A change with no identity may **inherit** from the same paragraph or **immediately adjacent** paragraphs when those neighbors contain a **single** unambiguous anchored party. This captures “I cleaned metadata mid-paragraph but the next edit still has my name” situations without claiming long-range stylistic continuity.

### Tier E — Weaker linguistic and stylistic hints (low confidence by design)

These were explicitly scoped as **supporting only after** Tiers A–D, because they are negotiable in tone and easy to misread:

- **Phrase-level drafting hints:** Stock phrases or templates sometimes correlate with internal vs external drafting (for example acceptance boilerplate vs customer-request language). These are **low-confidence** signals and may flag items for human or **constrained** model review rather than standing alone.

- **Stylometric micro-signals:** Small textual patterns (formality, hedging, defined-term usage) can nudge probability when nothing structural fires; they are **not** treated as proof.

- **Negotiation-direction heuristics:** Whether a change **narrows** or **broadens** obligations relative to surrounding baseline text can hint at “who usually proposes this direction” in a given deal type—usable only as a **weak** prior, not a decision rule.

- **Formatting fingerprints and template deviation:** Run-level properties (fonts, spacing, numbering, style ids) and drift from an expected **house template** can suggest which party last touched a region, especially when one side’s template differs from the other’s. Noisy under paste-from-PDF workflows, so ranked **below** session and thread evidence.

### Tier F — Default and optional model pass

- **Safe default:** When no tier yields a defensible label, default to **counterparty** at **low** confidence so conservative automation does not assume internal authorship.

- **Constrained LLM (optional last mile):** Only for items that remain low-confidence after deterministic passes, a model may **reclassify** under strict guardrails (for example explicit metadata must still be absent; output should cite which signals it relied on). The deterministic layer remains the **source of structural truth**; the model fills gaps, not the other way around.

---

## Resolver behavior (how tiers combine)

**Fixed precedence:** The resolver applies tiers in order (A through F). The first tier that yields a decisive label sets the outcome; later tiers are skipped for that item. That ordering makes behavior **reproducible**, **unit-testable**, and easy to explain to counsel or in a security review.

**Confidence labels:** Outcomes are labeled **high**, **medium**, or **low** to match how far down the stack the decision came. **Low-confidence** items are **summarized in aggregate** for review rather than cluttering every row with warnings.

**Non-negotiable rule:** Session identifiers (RSID family) and clusters **never override** trustworthy explicit author metadata—structure supports identity; it does not erase it.

---

## Tradeoffs

- **Deterministic core first:** Testable, diffable behavior; LLMs assist **only** where structure ends, aligning with high-stakes document workflows.

- **Conservative propagation:** Mixed-party clusters and ambiguous neighborhoods do **not** propagate labels, reducing silent wrong-sided automation.

- **Explicit uncertainty:** The design prefers **labeled doubt** over false precision—especially important when LLM tooling amplifies the cost of a mistake.

---

## Outcome

The result is a **self-contained authorship substrate** for redline automation: any LLM or rule-based step that follows receives **party**, **confidence**, **explanation**, and **structured provenance** per change and comment, instead of being asked to infer identity from prose alone after metadata loss and multi-turn markup.
