# Session & Prompt Guide — AI Agent Ecosystem Project

**Purpose:** This document tells you exactly what to type at the start, middle, and end of every working session in the Claude Project space, so continuity is never lost and progress is always tracked.

This guide assumes:
- A Claude Project named (e.g.) **"AI Agent Ecosystem"** has been created
- The 7 reference cards, `00-MASTER-EXECUTION-PLAN.md`, and this guide are uploaded to the Project's knowledge base
- `AGENTS.md` is uploaded once it exists (Phase 0)

---

## Starting a New Session

Open the Project, then use one of these prompts depending on the situation:

### A. Resuming work mid-phase
```
Resume the AI agent ecosystem build. Check 00-MASTER-EXECUTION-PLAN.md
for current status. The last file in progress was [FILE NAME]. Let's
continue scrutinizing/refining it.
```

### B. Starting the next file in sequence
```
Per 00-MASTER-EXECUTION-PLAN.md, [PREVIOUS FILE] is now APPROVED.
Move to the next file in the sequence: [NEXT FILE]. Draft it for review.
```

### C. Unsure where you left off (most common case)
```
I'm picking this back up — review 00-MASTER-EXECUTION-PLAN.md, tell me
the current phase, the last APPROVED file, and what the logical next
step is.
```
Claude will read the status table and tell you exactly where things stand. This is your fallback any time you're unsure — always safe to use.

### D. Working on a specific component out of sequence
*(Use sparingly — sequence exists for a reason, but sometimes a question arises that requires looking ahead.)*
```
Before we continue the sequence, I have a question about [TOPIC] that
affects [FILE]. Let's discuss before proceeding further.
```

---

## During the Session — Review Discipline

For every file we draft, follow this loop:

1. **Claude drafts** the file (spec or implementation) based on AGENTS.md and the relevant reference card(s)
2. **You scrutinize** — ask questions, request changes, push back on anything unclear or that doesn't fit your vision
3. **Claude revises** until you explicitly say it's approved
4. **Status gets updated** — say:
```
Approved. Update 00-MASTER-EXECUTION-PLAN.md status for [FILE] to
APPROVED and add a progress log entry.
```

**Do not let a file slide to "approved" by default or by moving on without saying so explicitly.** This is the discipline that prevents backtracking later.

---

## Ending a Session

Before closing out, always run:

```
Update 00-MASTER-EXECUTION-PLAN.md: set today's progress log entry
summarizing what we covered, and confirm the status flags for every
file we touched today are accurate.
```

Claude will give you the updated file. Replace the version in your local repo and re-upload to the Project knowledge base if the Project doesn't auto-sync (Claude Projects currently require manual re-upload of edited files — check this hasn't changed before assuming otherwise).

---

## Useful Standing Prompts (Reference)

| Situation | Prompt |
|---|---|
| Want a refresher on a concept | `Pull up what card 04 (Skills Framework) says about progressive disclosure before we draft this SKILL.md.` |
| Want to sanity-check token cost of a design | `Before finalizing this spec, check it against docs/architecture/model-routing-table.md — which tier should handle this?` |
| File just reached APPROVED or BUILT | `This is APPROVED/BUILT — give me the exact git commit message and command per the GitHub Strategy convention.` |
| Want to see the big picture again | `Show me the full Phase 0-7 status at a glance.` |
| Suspect scope creep | `Are we straying from AGENTS.md's original scope here? Sanity check.` |
| Ready to write actual code for an approved spec | `[FILE]'s spec is APPROVED. Let's implement it now in ds-core.` |

---

## What Lives Where (Quick Reference)

| Artifact | Location | Re-read every session? |
|---|---|---|
| 7 reference cards | Project knowledge base | No — Claude pulls as needed |
| `AGENTS.md` | Project knowledge base + repo root | Yes, implicitly — it's the spec for everything |
| `00-MASTER-EXECUTION-PLAN.md` | Project knowledge base + repo root | Yes — at start and end of every session |
| This guide | Project knowledge base | Only when you forget the workflow |
| Per-file specs (`docs/specs/*.md`) | Repo, not necessarily uploaded | Only when working on that specific file |

**Important:** Keep the master plan and this guide in the Project's knowledge base at all times, but you don't need to upload every spec file — only upload what's actively relevant if the Project knowledge base starts getting cluttered. Specs can simply be pasted into chat when you're working on that specific component.

---

## Sample Full Session Walkthrough (For Reference)

```
You: Resume the AI agent ecosystem build. Check 00-MASTER-EXECUTION-PLAN.md
for current status.

Claude: [Reports: Phase 0, file 0.1 (architecture-taxonomy card) is
IN REVIEW. Last session we drafted it but you hadn't approved yet.]

You: [Reviews the draft, requests two changes]

Claude: [Revises]

You: This looks good. Approved.

Claude: Updating status to APPROVED for 0.1. Next file in sequence is
0.2 (tools-mcp card). Want me to draft it now?

You: Yes, go ahead.

[... work continues ...]

You: Let's stop here for today. Update the master plan with today's
progress.

Claude: [Updates status flags + adds progress log entry, outputs the
full updated file]

You: [Saves the file, replaces it in the repo and Project knowledge base]
```

---

## Notes on Discipline

This guide exists because architecture work degrades quietly when continuity breaks — a half-finished spec gets reinterpreted differently three sessions later, or a security decision gets silently relitigated. The fix is not memory, it's a written, mutually-referenced state. Treat `00-MASTER-EXECUTION-PLAN.md` the way a real engineering team treats a sprint board: boring, unglamorous, and the only thing that actually keeps a multi-week build coherent.
