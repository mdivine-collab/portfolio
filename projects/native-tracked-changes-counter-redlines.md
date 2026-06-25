# Counter-redline generation as OOXML surgery

**One-liner:** A builder that takes a customer-redlined Word `.docx` plus a structured instruction set (accept, reject, or counter each tracked change) and emits a new package where **our party’s** responses appear as **valid tracked changes and comments**—without round-tripping through Word’s UI, by mutating `word/document.xml` and comment parts directly.

---

## Summary

- **Problem:** Negotiation automation is only useful if attorneys receive a **normal Word document**: customer markup must remain visible, and **our** accept/reject/counter moves must appear as **first-class revisions** (insertions, deletions, nested revisions) with correct **authors**, **timestamps**, and **revision ids**. High-level Word libraries either **skip revision subtrees** or expose APIs too coarse for “counter this insertion with minimally invasive tracked edits.” Pasting HTML or plain text loses the audit trail counsel expect.

- **Approach:** Treat the `.docx` as a **ZIP of XML parts**. Parse `word/document.xml` with a namespace-aware tree API, **locate each target** `w:ins`, `w:del`, `w:rPrChange`, or `w:pPrChange` by stable **`w:id`**, and apply **deterministic mutations** per instruction: accept unwraps or removes markup; reject inverts visibility with **nested** revisions authored as our party; **counter** on insertions uses a **token-aligned diff** inside the existing insertion so unchanged spans keep original run structure, deleted spans become nested deletions, and new language becomes new insertions—all **inside** the customer’s insertion when the texts overlap enough, otherwise **reject-then-sibling-insert** for a clean narrative. **Comments** attach to mutated anchor ranges or **reply in existing threads** via `comments.xml` plus `commentsExtended.xml` **parent/child paragraph ids**. **Global id allocation** scans the document and comment parts so new `w:id` values never collide. After rewriting parts, the builder refreshes **relationships** and **`[Content_Types].xml`** when comment parts are added or were missing, re-zips the package, and runs a **library open smoke test** on the bytes so obviously broken OOXML is surfaced before handoff.

- **Stack:** Python, **`lxml`** for OOXML, **`python-docx`** only as a **load-time integrity check**, in-memory **ZIP** reassembly for output.

---

## Why this is hard (and easy to get wrong)

**Tracked changes are a tree, not a string.** An insertion is an element whose children are runs, breaks, and possibly nested markup. A naive “replace inner XML” approach breaks **run properties**, **proofing**, and **Word’s revision merge** behavior.

**Counters are not a single operation.** Countering a **customer insertion** often means: keep the parts of their text we are not fighting, **strike** only what we replace, and **insert** our alternate language—while staying inside revision semantics so history remains legible.

**Ids and relationships are global.** Each new revision needs a fresh **`w:id`**; comments need **`w15:paraId`** and, for replies, consistent links in **comments extended**. Collisions or missing overrides cause Word to **repair** or **drop** content.

**Optional parts.** Many files lack `comments.xml` or `commentsExtended.xml`; the builder must **create roots**, wire **relationships**, and register **content types** idempotently.

---

## Instruction model (what the builder consumes)

Upstream phases produce **typed output instructions**: an action (`accept`, `reject`, `counter`), **target OOXML ids** (possibly multiple elements for grouped edits), optional **verbatim counter text**, and **comment strategy** (anchor a new comment on affected spans, or **reply** to an existing thread with resolved parent ids). The builder does not interpret legal meaning; it **executes** geometry and revision mechanics.

---

## Mutation strategies (core behaviors)

**Accept**  
- **Insertion:** unwrap—children promote to the parent flow, insertion element removed.  
- **Deletion / property change:** remove the revision wrapper so the document reflects the post-negotiation state.

**Reject**  
- **Insertion:** rewrite the insertion’s content so the proposed text is **marked deleted** under our author (nested `w:del` with `w:delText` where required), preserving strike styling expectations.  
- **Deletion:** re-insert the deleted text as a new **visible** insertion after the deletion marker.

**Counter (insertion)**  
- Tokenize **visible text** from the customer insertion and **counter proposal** text.  
- Run a **sequence alignment** (`difflib.SequenceMatcher`) on tokens.  
- If overall similarity is **below a conservative threshold**, treat the proposal as structurally unrelated: **reject** the customer insertion and add a **sibling** insertion for our text (avoids a misleading “partial overlap” inside the same bubble).  
- Otherwise, **clear** the insertion’s children and rebuild from opcodes: **equal** spans rehydrate runs from **original run tokens** (preserving `w:rPr`); **delete** spans become **nested deletions**; **insert** spans become new **insertions** with run properties **inferred** from neighboring original material so formatting does not reset arbitrarily.

**Counter (deletion)**  
- For split or multi-id deletions, walk elements and **insert replacement text** after the appropriate deletion node, with **empty** follow-up inserts where the instruction spans multiple deletions until the final segment carries the counter language.

---

## Comments and package wiring

- **Anchor strategy:** After mutations, record **anchor nodes and paragraphs**; the comment subsystem inserts **comment range markers** into `document.xml` tied to new comment ids.  
- **Reply strategy:** Append a new `w:comment` and link it in **`w15:commentsEx`** with **`w15:paraId`** and **`w15:paraIdParent`** resolved from prior comment ids.  
- **Package:** Ensure `[Content_Types].xml` and `word/_rels/document.xml.rels` reference comment parts when needed; serialize with XML declaration and UTF-8.

---

## Verification

Opening the generated bytes with a **document loader** smoke test catches gross structural mistakes early. Warnings are **surfaced on the build result** so reviewers know the file may still need manual inspection even when bytes are produced.

---

## Tradeoffs

- **Deterministic output** beats “ask a model to emit OOXML”: reproducible, testable, and safe for governed **verbatim** comment language from a playbook.  
- **Token threshold** for counter vs sibling-insert is a **product choice**: favor readable Word history over forcing every counter into a single insertion shell.  
- **python-docx** is intentionally **not** the mutation engine: its object model is the wrong abstraction for arbitrary revision trees; it is used as a **consumer sanity check** only.

---

## Outcome

Attorneys get a **standard counter-redlined `.docx`** suitable for review: customer track changes remain interpretable, **our** revisions are attributed consistently, comments land on the right spans or threads, and the pipeline can be **regression-tested** with XPath assertions on fixture XML—something that is impractical when generation is delegated to opaque document exports or LLM-authored markup.
