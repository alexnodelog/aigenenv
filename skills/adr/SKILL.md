---
name: adr
description: Append an entry to DECISIONS.md. Use when a choice departs from the archetype rule's Default (a listed Alternative, or something the rule does not list), when a rule is deliberately broken or the human overrides a required check or review, when an earlier decision is reversed, when the human approves collecting, storing or sending sensitive personal data, or when a rule says to record something in DECISIONS.md or to write an ADR.
---

# adr

An ADR (architecture decision record) is one entry in `DECISIONS.md` at the repository
root. Its reader is the project owner years later, who remembers none of the discussion.

## 1. Before writing

1. Open `DECISIONS.md`. If it does not exist, create it containing only `# Decisions`.
2. Search it for an entry on the same question. If one exists and this choice changes it,
   follow §3 as well.
3. Number: the highest existing number plus one, four digits (`ADR-0001`). Numbers are
   never reused or reassigned.

## 2. The entry

Append at the end of the file:

```markdown
## ADR-0001: <decision in one line>
**Date:** YYYY-MM-DD
**Status:** Accepted

**Context.** What forced a choice.
**Decision.** What was chosen.
**Consequences.** What this costs and rules out.
**Alternatives.** What was rejected, and why.
```

- **Context** — the constraint or event, not the history of the discussion.
- **Decision** — concrete enough to check against the code: names, versions, paths.
- **Consequences** — include what is now harder or impossible, not only the benefit.
- **Alternatives** — when the decision departs from a rule's Default, the Default is one
  of them, with the reason it did not fit.
- A deliberately broken rule or an overridden check: name the rule or check, the reason,
  and what exposure remains.
- Sensitive personal data: why it is needed, what is kept and where, how it is protected,
  whether it is encrypted at rest, and what happens if that key is lost.

## 3. Reversal

1. Write a new entry as in §2. Its Context names the entry it reverses (`Reverses
   ADR-0003.`).
2. In the old entry, change only the Status line to `**Status:** Superseded by ADR-0007`.
   Nothing else in it is edited or deleted.

## 4. Commit

- The entry goes in the same commit as the change it explains.
- A decision made before any code exists — an approval, an override — is committed on its
  own: `docs(decisions): add ADR-0007 <decision>`.
