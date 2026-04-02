## Projects
| Project | One-liner |
|---------|-----------|
| [Multi-signal authorship for OOXML tracked changes](projects/multi-signal-ooxml-authorship.md) | Attributing redline/comment when Word's author metadata is stripped or generic, using session and structure signals before optional LLM assistance. |
| [Preserving native Word tracked changes in automated counter-redlines](projects/native-tracked-changes-counter-redlines.md) | Emitting real OOXML revisions and threaded comments by mutating document XML directly, so automated counter-redlines stay auditable in Word's native track-changes UI. |
| [Deterministic playbook matching and auto-decision for contract negotiation](projects/deterministic-playbook-matching.md) | Matching each tracked change to a negotiation knowledge base using TF-IDF keyword scoring, section alignment, and factual anchoring, then auto-decide from structured entry fields. |
| [Constraining probabilistic classification inside an idempotent event pipeline](projects/constraining-probabilistic-classification.md) | Wrapping an LLM classifier in deterministic gates, idempotent event claims, schema validation, and compiled configuration so it can only produce narrow, validated actions or defer to a human. |

## Themes
- **Attribution:** Recover party identity when personal metadata is unreliable or cleaned, using OOXML revision signals and conservative propagation.
- **Output fidelity:** Preserve Word's native revision model so accept/reject/counter actions remain auditable tracked changes, not a substitute document format.
- **Retrieval and decision:** Match unstructured contract fragments to a structured knowledge base using weighted multi-signal scoring, then derive actions deterministically from entry fields rather than open-ended model prompts.
- **Containment:** Use typed schemas, idempotent event claims, and compiled configuration to constrain a model classifier to a narrow, validated action space with interactive human fallback.

## Adding an entry
1. Add `projects/<short-slug>.md` with a clear title, one-liner, and body (problem → approach → outcome).
2. Append a row to the table above with the same title text as the markdown `#` heading and a one-line summary.
