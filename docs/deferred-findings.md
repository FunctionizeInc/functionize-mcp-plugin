# Deferred findings

**Status: CURRENT. Append here; do not delete entries, mark them RESOLVED.**

What a review found, judged real, and deliberately did not fully fix here. Every entry
is a known defect or a known limit, not a suspicion.

**Why this file exists.** A finding nobody else can read is not recorded, it is
forgotten with extra steps. See `functionize-mcp-go/docs/deferred-findings.md` for the
sibling repo's version of this convention.

---

## `functionize`/`functionize-prompting` sync from FunctionizeSandbox/claude_skills (2026-08-26)

Lite-lane review (skeptic + native) on the `3edf9df` update (functionize 2.12.0 /
functionize-prompting 2.40.0, dead SAP + Salesforce cross-refs removed). Native: no
findings. Skeptic found real inconsistencies in the upstream edit itself — this repo
only *distributes* these two skills, the source of truth is
[`FunctionizeSandbox/claude_skills`](https://github.com/FunctionizeSandbox/claude_skills)
(`functionize-core/skills/`), so the durable fix belongs there. What's recorded here is
what we patched locally as a stopgap, plus what's still owed upstream.

- **Patched locally, root cause is still upstream (high).**
  `skills/functionize/references/operational-notes.md` — the premise-check sentence
  had been rewritten to hand the operating agent an unbounded, unperformable check
  ("confirm every field, record, or relationship... exists") with no fact source,
  contradicting this same skill's own "no independent back-channel read, never invent
  app facts" rules elsewhere. Reworded locally back to a routing note (premise check
  belongs to the domain/app-context skill, or is a question to the user, never inferred).
  Matt owns the real fix at the source so the next sync doesn't reintroduce it.
- **Patched locally, root cause is still upstream (medium).**
  `skills/functionize-prompting/SKILL.md` — the permission-grant paired-control routing
  note lost its `functionize-coverage` ownership clause when SAP/Salesforce refs were
  stripped, leaving "a two-test / coverage concern" with no coverage owner named, while
  a sibling line in the same file (§"Suite requests") still correctly references
  `functionize-coverage` with an install guard. Restored the ownership clause locally
  with the same guard pattern.
- **Corrected an over-eager first fix; now matches this file's own established pattern
  (medium).** `skills/functionize-prompting/references/fundamentals.md` anti-pattern #10
  (SAP-shaped guidance, line 858) survived the cross-ref cleanup with its
  `functionize-sap-s4hana` pointer removed, while the same file's line 70 had already
  been genericized the identical way ("Salesforce CPQ quote lines", app named, skill not
  named — that line used to carry both a SAP and a Salesforce pointer, per the same sync).
  First attempt re-added a SAP pointer at 858 alone, which created a NEW asymmetry
  *within this one file* (858 names a skill, 70 doesn't) — a re-review caught this before
  push. Corrected: line 858 now matches line 70's convention instead of reversing it
  (plain "SAP especially", no skill pointer). The two `functionize-salesforce`
  naming-pattern mentions elsewhere (`skills/functionize/SKILL.md:13`,
  `skills/functionize-prompting/SKILL.md:25`) are a different kind of reference (how
  skills are named, not domain-guidance) and aren't in tension with this.
- **Patched locally, corrected once (medium).** `skills/functionize-prompting/references/templates.md`
  "Domain-specific extensions" §3 referenced "what does **the skill** need to decide"
  with the antecedent (a specific domain skill example) removed by the same cleanup,
  reading ambiguously. First fix reworded to "a domain skill" — but that section's own
  opening two lines up explicitly cover the no-domain-skill-loaded case ("elicit those
  specifics from the user... build from the primitives above"), so attributing the
  checklist to a domain skill undercuts the exact fallback the section exists to serve.
  Corrected to actor-neutral phrasing ("what has to be decided") instead, matching items
  1 and 2 in the same list, which name no owner either. No upstream action needed, this
  was a wording gap introduced by the edit, not a missing skill/pointer.
- **Not fixed, flagged to Matt (low).** Commit `3edf9df`'s stated rationale for dropping
  `plugin.json`'s `version` field ("a pinned version freezes delivery") doesn't match
  this machine's own installed-plugin state (`~/.claude/plugins/installed_plugins.json`
  keys updates on `gitCommitSha` regardless of a pinned semver; other installed plugins
  with pinned versions update fine). The removal itself is harmless and matches other
  real plugins that omit it, so left as-is, but the reasoning behind it may not hold and
  is worth Matt knowing in case it matters for how his own repo versions things.

**Action owed**: relay this whole list to Matt so `FunctionizeSandbox/claude_skills`
gets the real fix, not just our local patch — otherwise the next sync silently
reintroduces all four content issues.
