---
name: tribunal
description: >-
  Adversarial code review of the current branch's diff against a base branch.
  Three stages: a hater accuses the code as if a clueless amateur wrote it, a
  judge looks for justification ("maybe it was deliberate"), and a verdict keeps
  only the spots where the hater is right AND the judge couldn't defend it.
  Writes a report to docs/reviews/ plus a chat summary. Portable, sub-agent
  based (no Workflow runtime) — works in any repository. MANDATORY TRIGGERS:
  '/tribunal', 'tribunal', 'run the tribunal', 'tear this code apart',
  'adversarial review', 'prosecutor defense verdict'. STRONG TRIGGERS (when the
  ask is a hard critical pass over branch changes): 'hard-review the diff',
  'find the weak spots in this branch', "what's actually bad here". Optional
  argument — a path filter, e.g. /tribunal content.
---

# /tribunal — put the code on trial

Prosecutor → defense → verdict. Adversarial review: a **hater** agent per file
tears the changes apart (focused on the diff), a separate **integration** hater
hunts for cross-module breakage, a **judge** agent looks for justification on
each accusation, and the **verdict** keeps only spots the judge couldn't defend
OR conceded are weak. The balance comes from the clash *between* agents, not from
any single one.

This version has no dependency on the Workflow tool — the skill orchestrates
everything itself via the **Agent** tool (`general-purpose` sub-agents). Invoking
the skill is the opt-in.

## CONFIG (edit to taste)

```
BASE_BRANCH      = main           # diff against this; auto-fallback to master/origin/HEAD
EXTENSIONS       = .py .js .ts .tsx .jsx .go .rs .java .rb .php .cs .cpp .c .h .kt .swift .scala .m .mm   # languages to review; empty = all text files
EXCLUDE_GLOBS    = node_modules/  .venv/  venv/  dist/  build/  vendor/  target/  alembic/versions/  migrations/
INCLUDE_TESTS    = yes            # no — drop paths under tests/
MAX_FILES        = 40            # circuit breaker: if more files — ask the user
PARALLEL         = 6             # how many hater agents to launch per batch
```

## Steps

### 1. Collect the list of files from the branch diff
Run (PowerShell or Bash):
- `git diff --name-only --diff-filter=ACMR <BASE_BRANCH>...HEAD` — changed/added tracked files relative to the merge base with the base branch.
- `git ls-files --others --exclude-standard` — new untracked files.

Merge both lists (dedupe), then filter:
- keep only files with an extension from `EXTENSIONS` (if empty — all);
- drop everything matching `EXCLUDE_GLOBS`;
- if `INCLUDE_TESTS = no` — drop paths under `tests/`;
- if the user passed an argument (e.g. `/tribunal content`) — keep only files whose path contains that fragment (a sub-filter on top of the diff).

If `BASE_BRANCH` doesn't exist — try `master`, then `git merge-base` with `origin/HEAD`. If files > `MAX_FILES` — show the count and ask whether to review all or narrow with a filter.

Also collect the **per-file diff** — the hater needs it as its focus:
- tracked: `git diff <BASE_BRANCH>...HEAD -- <file>`;
- untracked (new file): the whole file is the diff — mark it as "file is entirely new".

### 2. If there are no files
Say so and stop — nothing to review.

### 3. "Hater" stage — one agent per file (in parallel)
For each file, launch a sub-agent via the **Agent** tool (`subagent_type: "general-purpose"`). Launch in batches of `PARALLEL` — **multiple Agent calls in a single message** run in parallel. Substitute both the path and **this file's diff** from step 1 into the prompt. Agent prompt:

> You are the meanest, most nitpicky reviewer alive. A rival wrote this code, and you must find everything you can pick at — but strictly ON THE MERITS, no fluff, no style-for-style's-sake nitpicks.
>
> Read the WHOLE file (Read): `<path>` — you need this for context: to tell whether this is a hack or a deliberate decision driven by the code's specifics. But **only accuse what changed** in this branch. Here is the diff (your focus):
>
> ```diff
> <file diff; for a new file — "file is entirely new, review all of it">
> ```
>
> Targeting rule: complain about (a) changed code from the diff; (b) old code ONLY if your change leans on it or exposes it (a crutch this diff stands on). Old code unrelated to the change is NOT your target — leave it.
>
> IMPORTANT: ignore comments, docstrings and docs as "excuses" — they are not a free pass. Judge the code itself: correctness, error handling, races and async bugs, performance, duplication, resource leaks, edge cases, security, API design, cohesion/coupling, hidden assumptions.
>
> Don't invent problems out of thin air. **If the changes are genuinely clean — return an empty list; that's normal and honest.** Better 0-6 real complaints than a single fabricated one.
>
> Return ONLY JSON in a fenced ```json block, schema: `{"complaints":[{"id":"c1","location":"function/class + lines","severity":"critical|major|minor","accusation":"\"An amateur wrote this because …\" — point by point, citing the code","better":"\"it should have been done like …\" — how to do it right and WHY it's better"}]}`. If no complaints — `{"complaints":[]}`.

Collect `complaints` per file from the JSON responses.

### 3.5. "Integration" stage — one agent over the change set
Per-file haters see files in isolation and are blind to cross-module bugs. This stage hunts exactly those. Launch ONE sub-agent (`general-purpose`); if there are many changed modules — split into connected groups (by imports) and launch one agent per group. Feed it the list of changed files and their diffs (from step 1). Prompt:

> You are the integration hater. Per-file reviewers already went over each file separately — your remit is DIFFERENT: whether anything breaks BETWEEN files because of these changes. Don't repeat intra-file nitpicks.
>
> Changed files and their diffs:
>
> ```diff
> <list of files + their diffs>
> ```
>
> Read the relevant files and their callers/callees (Read, Grep on symbol names). Look specifically for cross-module issues: a function signature/contract changed but a caller still calls the old way; a return value/dict shape changed but a consumer expects the old one; invariants out of sync across modules; races/init-order across files; new code relying on an assumption not held in another file; places that should have been updated together with this change but weren't.
>
> Don't fabricate. No real cross-file problems — return an empty list.
>
> Return ONLY JSON in a ```json block with the same `{"complaints":[...]}` schema, but in `location` name the participants: `fileA:loc → fileB:loc`. In each complaint's `file`, put the primary affected file.

Add these complaints to the shared pool (with `file` from the response; if it spans several — use the one in `file`). They go through the judge stage alongside the per-file ones.

### 4. "Judge" stage — one agent per file (only where there are accusations)
For each file with a non-empty `complaints`, launch a sub-agent (`general-purpose`), also in batches. Prompt:

> You are an impartial judge. The hater tore the file apart. For EACH accusation, look at the code and honestly decide: could this have been done deliberately, with a justification?
>
> File: `<path>`. Read it (Read), and if needed peek at neighboring files/imports (Grep).
>
> Consider context: domain requirements, project conventions, conscious trade-offs, simplicity, compatibility. Docs and comments MAY be used as evidence of intent. Don't side with either the hater or the author.
>
> Accusations (JSON): `<insert this file's complaints>`
>
> Separate two questions honestly (the defender role and the assessor role are different): first `defensible` — is there an objective justification / is the nitpick off-base; then, SEPARATELY, `agree_weak` — hand on heart, is the spot actually weak? You can defend the choice on context and still concede `agree_weak:true` if it really should be done differently.
>
> For cross-file accusations (location like `fileA → fileB`) look at both ends via Grep and verify the contract for real.
>
> Return ONLY JSON in a ```json block, schema: `{"verdicts":[{"id":"c1","defensible":true|false,"defense":"reply to the hater","agree_weak":true|false}]}`. `defensible:true` — there's a sound justification and it's fine, OR the nitpick is off-base; `false` — no defense, the code really is weak.

Merge: to each accusation add `file`, `defensible`, `defense`, `agree_weak` by its `id`. If the judge skipped an `id` — treat it as `defensible:false`, `agree_weak:true`, `defense:"(judge gave no answer)"`.

### 5. Selecting the confirmed ones
`confirmed` = all accusations where **`defensible === false` OR `agree_weak === true`** — i.e. the judge either couldn't defend it or conceded the spot is weak (even if it formally defended the choice). This puts the judge's honest signal to work so a real bug doesn't slip through under cover of "defensible". The rest go to the full transcript only.

### 6. Verdict (synthesis — you do this yourself, no separate agent)
From the `confirmed` list, group and rank by severity (critical→major→minor) and real risk. For each: `file`, `location`, `severity`, `problem` (the gist in 1-2 sentences), `fix` (how to fix, concretely). `summary` — a short takeaway: overall state of the code + the main systemic problem patterns. Don't add anything not in the data.

### 7. Write the report and print the summary
- Date: PowerShell `Get-Date -Format yyyy-MM-dd`; branch: `git rev-parse --abbrev-ref HEAD`.
- Report → `docs/reviews/<date>-<branch>-tribunal.md` using the template below (create the folder if needed).
- To chat — a short summary: path to the report, the numbers (files / total accusations / confirmed weak), and the top confirmed ones (critical/major) as a `file:loc — gist` list. No fluff.

## Report template

```markdown
# Tribunal — <branch> — <date>

> Hater → judge → verdict. Files: <N>. Accusations: <total>. Confirmed weak: <confirmed>.

## ⚖️ Verdict — genuinely weak spots

<summary>

<for each confirmed, by severity critical→major→minor:>
- **[<severity>] <file>:<location>** — <problem>
  _Fix:_ <fix>

## 📜 Full transcript (hate + defense per file)

<group all accusations by file; within a file, for each:>
### <file>
- **[<severity>] <location>**
  - 🔥 Hater: <accusation>
  - 💡 Should have been: <better>
  - ⚖️ Judge (<defensible ? "✅ defended" : "❌ not defended">): <defense>
```

## Notes

- **Diff-aware:** the hater reads the whole file (for context — hack vs. deliberate decision) but only accuses changed code + old code the diff leans on. It does not tear apart untouched old code.
- **Integration stage (3.5)** closes the blind spot of per-file haters — cross-module bugs (broken contracts, out-of-sync calls). Without it the most expensive class of bugs is structurally missed.
- "Genuinely weak" = `defensible === false` OR `agree_weak === true`. The judge's honest signal is used, not just a binary defense — a real bug won't slip through under "defensible".
- The hater deliberately ignores docs/comments as excuses; the judge does the opposite, weighing intent. The balance is between stages, not inside one agent. In the judge, the roles are split: defense first (`defensible`), then separately an honest assessment (`agree_weak`).
- The hater may return an empty list — on a clean diff it is not obligated to invent complaints.
- Parallelism — `PARALLEL` agents per batch, the rest in the next batch.
- Sub-agents return text — so we require strict JSON in a fenced block and parse it. If an agent returns garbage instead of JSON — re-run that one file.
- It does NOT grade the code "in general"; it judges only the current branch's diff.
