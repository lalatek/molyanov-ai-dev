# Code Research: ui-preview-approval

Feature: add a "generate + approve UI mockup" step to the methodology, inserted after user-spec
approval and before implementation, producing a static HTML/CSS mockup (multi-screen + responsive
preview) that the user explicitly approves/rejects in chat before any code/layout is written.

## 1. `skills/user-spec-planning/SKILL.md` — current workflow

File: `/Users/macbook/IdeaProjects/sz/molyanov-ai-dev/skills/user-spec-planning/SKILL.md`

Exact current `## Workflow` steps (confirmed, lines 34–213):

1. **`### 1. Start or Resume`** (lines 36–96) — resolves `work/{feature}` folder, derives slug
   (plain kebab-case locally, or via placeholder GitHub issue when online-pipeline is enabled),
   runs `scripts/init-feature-folder.sh {slug}`, initializes interview metadata, begins interview.
   Also contains the splitting-user-specs branch (optional, user-gated).
2. **`### 2. Interview and Research`** (98–117) — scores topics, runs interview loop, launches
   `code-researcher` once the outcome is clear enough (writes `code-research.md`), covers
   flows/integration/edge cases, re-runs research if a later answer exposes a factual gap.
3. **`### 3. Check Interview Completeness`** (119–131) — launches
   `interview-completeness-checker`; `user_decision_required` findings go back through the
   interview loop; drafting happens only after a `clean` result.
4. **`### 4. Draft the User Spec`** (133–146) — fills `work/{feature}/user-spec.md` in place from
   the template, preserves the executor instruction, commits
   `draft(userspec): create user-spec for {feature}`.
5. **`### 5. Validate the User Spec`** (148–180) — 3 parallel reviewers
   (`userspec-quality-validator`, `userspec-adequacy-validator`, `skeptic`), up to 3 rounds, commits
   `chore(userspec): validation round {N} — {summary}` between rounds. Has an
   `ONLINE_PIPELINE_AUTOMATED`-gated branch that persists `automated_validate_state` in
   `interview.yml` across runs.
6. **`### 6. Obtain Approval`** (182–193) — shows spec path + validation summary to the user in
   chat; on explicit approval sets `user-spec.md` frontmatter `status: approved`,
   `interview_metadata.status: completed`, commits
   `chore(userspec): approve user-spec for {feature}`, returns the spec path and tells the user it
   can be implemented "in a new chat."

**Where a new mockup step would fit structurally:** Step 6 is the last step in this skill and is
explicitly the terminal action — it already says "tell the user it can be implemented in a new
chat" immediately after commit. There is no Step 7 today. A new mockup-generate-and-approve step
is a candidate new **Step 7**, inserted after Step 6's approval commit and before the "return the
spec path" hand-off — i.e. structurally it would run *between* `user-spec-planning` finishing and
whatever skill starts implementation (`code-writing`/`layout-writing`), not as a sub-step inside
validation (Step 5 validates the *document*, not a visual artifact, and its 3-round cap is a
document-review mechanism, not a build-and-preview mechanism).

`online-pipeline/SKILL.md`'s stage `userspec-turn` explicitly maps to "`user-spec-planning` Steps
1–5" and explicitly skips calling user-spec-planning's own Step 6 ("do not run Step 6") because it
replaces it with its own PR + `/approve` comment flow (see §3 below). Any new Step 7 inserted into
`user-spec-planning/SKILL.md` would need an equivalent online-pipeline mapping decision: either
`userspec-turn` also skips it and a parallel online stage implements the mockup gate, or it becomes
part of the existing PR-based approval instead of a separate step — this is unresolved and is
exactly the kind of decision the interview must capture.

### Templates / artifact conventions (`skills/user-spec-planning/assets/`)

- `user-spec.md.template` — YAML frontmatter (`created`, `status: draft|approved`,
  `type: feature|bug|refactoring`), then an "Executor instruction" blockquote, then fixed H2
  sections (`What We Are Building`, `Why`, `Expected Behavior`, `Acceptance Criteria`,
  `Constraints`, `Risks`, `Accepted Decisions`, `Testing`, `Verification` with `Agent
  Verification`/`User Verification` subsections). Bracketed placeholders throughout.
- `decisions.md.template` — minimal: title `# Решения: {Feature Name}` + an HTML comment
  explaining the only-material-decisions rule and the per-entry shape (`Что решили` /
  `Почему` / `Отклонение от user-spec`). Note: this template's own body is in Russian even though
  the skill's scaffold-English convention applies elsewhere — it is filled in the user's language
  per the Language rule, same as user-spec.md content.
- `interview.yml.template` — `interview_metadata` block, `automated_validate_state` (null unless
  online-pipeline automation), `conversation_history: []` (verbatim Q&A log), then three topic
  groups (`general_understanding`, `user_experience`, `integration`), each topic having
  `score`/`value`/`gaps`/`required`. Notably `user_experience.ui_ux_requirements` already exists as
  an **optional** (`required: false`) topic — "Describe UI or UX requirements when applicable." A
  mockup-approval step would likely hang off this existing topic rather than requiring a new one,
  though scoring/gating a *generated artifact* (not just a text answer) doesn't fit the current
  score/value/gaps shape cleanly.
- `scripts/init-feature-folder.sh` — creates `work/{feature}/{user-spec.md, decisions.md,
  logs/userspec/interview.yml, logs/working/}` from the three templates above, idempotently (skips
  existing files). A `mockup.html` artifact would need either a new template added here or to be
  generated ad hoc by the step itself (no init-time placeholder currently exists for it).

No other artifact file type is referenced by this skill. `code-research.md` (this file) is written
by the `code-researcher` agent, not by a template in this skill.

## 2. `skills/layout-writing/SKILL.md` — Figma/Claude Design/screenshot consumption

File: `/Users/macbook/IdeaProjects/sz/molyanov-ai-dev/skills/layout-writing/SKILL.md` (+
`references/reproduce.md`, `references/design-decisions.md`, `scripts/capture.mjs`,
`scripts/overlay.mjs`).

- `layout-writing` is explicitly scoped to **implementation**, not planning: "Deliver a layout that
  matches its exact source... in the existing codebase." It takes an already-existing visual
  source and reproduces it as real markup/CSS in the target project's own stack.
- Source handling (`reproduce.md`) is organized by source *type*: Figma (via Figma MCP
  `get_design_context`/`get_metadata`/`get_screenshot`/`download_assets`, or the Figma REST API as
  fallback), **Claude Design** (prefers a "Project HTML ZIP or standalone HTML" export — this is
  the closest existing precedent to consuming a static HTML artifact), and "Screenshot or Existing
  Page" (treated as a looser, partially-specified source requiring viewport/scale assumptions to be
  flagged).
- Verification mechanism: `scripts/capture.mjs` renders the implemented site at explicit viewports
  (baseline widths `360/430/768/1440` plus source-specific and breakpoint-adjacent widths) and
  `scripts/overlay.mjs` produces pixel-diff/overlay images between the source screenshot and the
  live capture, inspected block-by-block (`design-decisions.md`'s evidence order, plus the Long
  Pages section of `reproduce.md` for multi-section sources).
- Review: one or two `layout-reviewer` waves per the methodology's shared review model, consuming
  "source and difference images" explicitly.

**Reusability for the new mockup.html source:** `layout-writing`'s "Claude Design" branch
(standalone HTML export, treated as "evidence, not production code," then adapted to the project's
real stack/components, with authoritative visual values preserved) is architecturally the closest
match to "consume a static mockup.html as the visual source." A generated `mockup.html` is not
Figma (no node IDs/design-variables/components payload) and not a live page (no DOM/viewport to
capture against) — it is a static, self-authored HTML file, so of the three existing branches in
`reproduce.md`, the Claude-Design-export branch is the one with the right shape (render to native
image, diff against the implemented result) rather than requiring a fourth branch. `layout-writing`
has no existing concept of a "pre-implementation-approved mockup," only "exact source" vs.
"partially-specified source + project-first decisions" — whether the new mockup becomes a declared
third source type inside `reproduce.md` or is just handed through the existing Claude-Design-export
path is an open design question, not yet answered by the skill itself.

## 3. `skills/online-pipeline/SKILL.md` — existing human-in-the-loop approval mechanics

File: `/Users/macbook/IdeaProjects/sz/molyanov-ai-dev/skills/online-pipeline/SKILL.md`. This is the
most load-bearing section for designing a consistent mockup-approval gate, since online-pipeline
has **no interactive chat** and **no way to render HTML for a human to look at** inside a `claude -p`
CI job — every existing approval gate is GitHub-native.

### Exact mechanism for user-spec approval today (replaces Step 6 of user-spec-planning)

- Stage `userspec-turn` (triggered per issue comment / issue open by
  `online-pipeline-userspec.yml`) runs `user-spec-planning` Steps 1–5 **and explicitly skips Step
  6** ("do not run Step 6 ('Obtain Approval')... That step is replaced entirely by the PR +
  `/approve` comment flow below").
- When Step 5 validation ends clean (all three reviewers `clean`), the agent writes `status:
  ready_for_review` to the status marker file
  (`work/{slug}/logs/working/online-pipeline-status.yml`), commits/pushes it to branch
  `userspec/{slug}`, and exits. **The agent never opens the PR itself.**
- The surrounding **workflow YAML** (deterministic bash, not the agent) is what opens/updates the
  spec PR (label `userspec-spec`) after the agent exits — "The workflow opens or updates the spec
  PR after you exit — do not call `gh pr create` yourself."
- The actual human approval signal is a literal **`/approve` comment on the spec PR**. This is
  implemented **entirely outside the agent**: "`!approve` and new-feature/new-project creation are
  entirely Worker-side... posts the real `/approve` comment (spec PR — the existing route job in
  `online-pipeline-userspec.yml` handles it unchanged)." So: artifact posted = the spec PR itself
  (diff of `user-spec.md`/`decisions.md` on GitHub); where = a GitHub PR; how the human signals
  approval = commenting the literal string `/approve` on that PR, which the route job
  (deterministic YAML logic, not agent logic) recognizes to merge/continue.
- A requested change after the PR is open routes back into the normal `userspec-turn` dispatch (new
  issue comment → Resume path → back through Step 5 validation → PR updates in place since it's the
  same branch).

### Why this matters for a mockup-approval gate

- online-pipeline communicates with the human exclusively via `gh issue comment` / PR
  review+comment — "the agent communicates with the user exclusively through `gh issue comment
  {issue_number} --body ...`" and PR-level `/approve`. There is **no mechanism anywhere in
  online-pipeline for rendering or "showing" an HTML artifact to a human** beyond what GitHub itself
  can render inline (e.g., a PR diff, or an image attached to a comment). A static HTML file
  committed to a branch is visible to the human only by them opening the PR's "Files changed" tab
  (raw HTML/CSS source, not rendered) or by them checking out the branch locally — GitHub does not
  render arbitrary repo HTML as a live page in a PR view.
- Given the existing pattern (artifact = PR, signal = PR comment keyword), a mockup-approval gate
  modeled the same way would most naturally be: commit `mockup.html` (+ CSS/assets) to the same
  `userspec/{slug}` branch (or a new one), open or reuse a PR, and gate on a comment keyword (e.g.
  another `/approve` or a distinct keyword) on that PR or the issue — but *how the human actually
  previews a static HTML file's rendered look from inside a GitHub PR* is not solved by any existing
  online-pipeline mechanism and was not found anywhere in this skill. (GitHub Pages, a PR preview
  deployment, or screenshotting the mockup into a PR/issue comment are possible approaches but none
  of them exist in this codebase today — this is a gap to raise in the interview, not an existing
  pattern to reuse.)
- The Worker-side `/approve` detection (`control-plane` repo, not in this repo) only currently
  recognizes the spec-PR and code-PR cases ("looks up the two possible open PRs (`userspec/{slug}`,
  `feature/{slug}`)") — a third PR/branch for a mockup gate is not accounted for in the Worker logic
  described here (that logic lives entirely in the separate `control-plane` repo per this file).
- Naming Contract conventions that a new mockup stage would need to extend if modeled the same way:
  branch-naming scheme, a new PR label analogous to `userspec-spec`/`userspec-implement`, a new
  stage name in `online-pipeline-userspec.yml` (or a new workflow file), and a route-job dispatch
  rule analogous to the `ready_for_review`/`ready_for_pr` status-marker convention.

## 4. `skills/methodology/SKILL.md` — high-level pipeline description to update

File: `/Users/macbook/IdeaProjects/sz/molyanov-ai-dev/skills/methodology/SKILL.md`.

Relevant passages that name the pipeline stages and would need to mention a new mockup step:

- `## Planned Feature Lifecycle` (lines 43–48): the literal lifecycle diagram —
  ```
  user-spec-planning → explicit approval → new task: implement the approved spec
  → verified implementation commit → documentation-writing feature finalization
  ```
  A mockup-approval step sits exactly between "explicit approval" and "new task: implement."
- `### Plan the Feature` (50–74): step-by-step restatement of `user-spec-planning`'s workflow
  (numbered 1–7, ending "Obtain explicit user approval ... return the absolute user-spec path for a
  new task") — mirrors `user-spec-planning/SKILL.md` almost 1:1 and would need a new numbered item
  if Step 7 is added there.
- `### Implement the Feature` (76–98): names which skills own implementation
  (`code-writing`/`layout-writing`/`infrastructure-setup`/`prompt-master`/`skill-master`) — this is
  the section that currently assumes implementation starts directly from the approved spec with no
  intermediate visual-approval gate.
- `## Sources of Truth` → `### Feature Folder` (167–182): the literal `work/{feature}/` tree
  diagram (`user-spec.md`, `code-research.md`, `decisions.md`, `logs/`) — currently has no
  `mockup.html` (or similar) entry; this is the canonical list of expected feature-folder contents
  referenced elsewhere in the methodology.
- `## Skill Responsibilities` table (184–197) — lists owning skill per area; would need a row if the
  mockup step becomes its own named responsibility.
- No existing mention of Figma, mockups, or visual approval anywhere in this file — confirmed via
  grep, this concept is entirely new to the methodology today.

## 5. Existing Design System / style tokens / Figma references in the repo

Grepped the whole repo (`.md` files, excluding `work/`) for "design system", "style token",
"design-token", "figma" (case-insensitive):

- **No design-system or style-token file exists for this repo itself.** This repository
  (`molyanov-ai-dev`) is the methodology/skills source repo, not an end-user application — it has
  no own `.claude/skills/project-knowledge/` (confirmed: no `project-knowledge` directory exists at
  the repo root outside of the scaffold template).
- Figma is referenced only inside `layout-writing/SKILL.md` and `references/reproduce.md` (as a
  *consumption* source for implementing layouts) and inside the **scaffold template** for new
  projects:
  `skills/project-initialization/assets/new-project/.claude/skills/project-knowledge/references/ux-guidelines.md`
  — a fill-in-the-blank template with a `## Design System` section (`Design files: [e.g., "Figma:
  [link]" / "No design files..."]`, color palette placeholders, key-components placeholders). This
  is a per-project template an end user fills in during `documentation-writing`'s initial interview;
  it is not itself a design system and carries no real values.
- No CLAUDE.md in this repo (root `/Users/macbook/IdeaProjects/sz/molyanov-ai-dev/CLAUDE.md`)
  mentions design system, Figma, or UI conventions — it only carries the generic
  Language/Behavior/Task-Scope/Security rules shared across projects (identical content to the
  user's global `~/.claude/CLAUDE.md`).
- Conclusion: there is no existing style-token/design-system source a generated mockup must match
  *for this repo*. For an arbitrary end-user project using this methodology, the only existing
  convention is `ux-guidelines.md`'s optional `## Design System` section (filled per-project) plus
  `layout-writing/references/design-decisions.md`'s "Evidence Order" (project docs → same component
  elsewhere → neighboring components → project tokens → general conventions) — any new mockup step
  would need to decide whether it reads that same evidence order before generating a mockup for a
  project that already has established UI.

## 6. Git commit conventions for skill-authoring changes

`git log --oneline -- skills/` (most recent 30, confirmed verbatim):

```
65fea2b feat(online-pipeline): decouple finalize from code-PR-merge event
a893179 fix(online-pipeline): make the /approve commit step idempotent on retry
b67ac8d feat(online-pipeline): add implement-followup stage for post-PR change requests
e0a57af feat(online-pipeline): resume same session on rate-limit instead of reset-and-retry
6126c6b fix(online-pipeline): checkout implement job with GH_PAT so pushes touching .github/workflows/* succeed
a58cf92 fix(online-pipeline): remove invalid 'workflows' permission key from implement.yml
befc04c feat(online-pipeline): detect Claude rate-limit and auto-switch control-plane token pool
bdbee60 fix(online-pipeline): grant workflows permission to implement stage
c237ed2 fix(online-pipeline): prevent agent from self-relaying to Slack (duplicate posts)
3f95bea fix(online-pipeline): deterministic Slack relay for userspec-turn and implement
78397db fix(online-pipeline): correct bot-comment login check in knowledge-init relay
a586fe7 fix(online-pipeline): relay knowledge-init comments to Slack deterministically
0d4b209 feat(online-pipeline): add /init-knowledge trigger and knowledge-init stage
4b8f77d feat(online-pipeline): add Slack bridge and automated /new-project bootstrap
3931d3b feat(online-pipeline): add hybrid local/online switch and placeholder-issue start path
2ad08bc fix(online-pipeline): fix review findings and vendor the skill closure into target repos
a6a816f feat(online-pipeline): run feature lifecycle through GitHub Actions
```

Also observed from the broader (non-`skills/`-filtered, but touching skills) history at the start
of this session (per git status/log context): `chore(userspec): approve user-spec for {feature}`,
`chore(userspec): validation round {N} — {summary}`, `draft(userspec): create user-spec for
{feature}` — these three are defined verbatim inside `user-spec-planning/SKILL.md` itself (lines
146, 168, 191) as the required commit messages for its own steps, not just observed convention.

**Confirmed convention:** `{type}({skill-or-area}): {imperative summary}` —
`type` ∈ {`feat`, `fix`, `chore`, `draft`, `docs`, `update`, `refactor`, `i18n`, `sync`} observed;
scope is the skill name in parentheses (`online-pipeline`, `userspec`, `skill-master`, `codex`);
summary is lower-case, imperative, no trailing period. For a new mockup-approval step the
consistent pattern would be e.g. `feat(user-spec-planning): add mockup approval step` or
`feat(skill-name): ...` as the user's own research question anticipated — confirmed by the real
history above, not assumed.
</content>
