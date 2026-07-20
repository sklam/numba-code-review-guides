# <Topic> Guide for Numba Code Reviews

Template for review guides. Keep guides terse and concise; technical accuracy
comes first. Delete this template's guidance comments before shipping a guide.

## Principles

- **Accuracy over completeness.** Every claim must be verifiable. Read the
  linked PR/issue/blog before asserting anything. Mark uncertain claims as
  *arguable* rather than stating them as fact.
- **Terse.** No filler, no history ("10+ years ago"), no motivational lines, no
  restating the same rule in three sections. If a table says it, don't also
  write it as prose.
- **No stale pins.** Don't cite commit hashes, line numbers, or "current state"
  of in-flight code — it rots. Cite PRs/issues as the *source of the lesson*,
  not as a snapshot of code.
- **No versions/dates in the doc.** Git tracks that.
- **Show, don't lecture.** Prefer a ❌ Wrong / ✅ Correct code pair over
  paragraphs. Annotate the specific wrong line and say *why* in a comment.
- **Actionable only.** Cut anything a reviewer can't act on (compiler-internal
  pass names, heuristics) unless it changes what to look for.

## Required structure

Header, then sections in this order. Drop any section with nothing terse to say.

```
# <Topic> Guide for Numba Code Reviews

**Document Purpose**: Educational reference for code reviewers and contributors
working with <area> in Numba, based on lessons from <PRs/issues>.

**Key References**:
- GitHub PR #NNNNN: "<title>"
- GitHub Issue #NNNNN: "<title>"

---

## Table of Contents
1. ...

---
```

- **Concept** — what the thing is and the one key principle. A few sentences.
- **Common Mistakes / Anti-Patterns** — ❌ Wrong code, one-line *why*, ✅ Correct.
- **Correct Patterns** — the idioms to use, as code.
- **Review Checklist** — `- [ ]` items a reviewer ticks off. This is the payload.
- **Worked Example** — one realistic ❌/✅ pair from the source PR. Optional.
- **Quick Reference** — a Wrong→Right table. Optional.
- **Further Reading** — source PRs/issues and authoritative external links.
- **Conclusion** — a short "Golden Rules" list. No new information.

## Style rules

- Use ❌/✅ consistently for wrong/right.
- Code blocks over prose. Comments on the offending line, not below the block.
- One idea per checklist item; phrase as something checkable.
- Link the primary source for any non-obvious technical claim inline.
