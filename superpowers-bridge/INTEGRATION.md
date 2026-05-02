# OpenSpec × Superpowers integration runbook

[English](./INTEGRATION.md) · [繁體中文](./INTEGRATION.zh-TW.md)

> This document explains how the `superpowers-bridge` schema fuses
> OpenSpec's artifact governance with Superpowers' execution skills
> into a single workflow. Use it as an onboarding reference, a
> change-review companion, and required reading before modifying
> the schema.
>
> Schema version: `superpowers-bridge` v1

---

## 1. The integration in one sentence: who handles what

OpenSpec handles the **what (WHAT)** — governance, validation, and archival of `proposal` / `specs` / `design` / `tasks` markdown artifacts.

Superpowers handles the **how (HOW)** — execution skills like brainstorming dialogue, TDD discipline, subagent dispatch, code review.

The two are connected through a custom schema ([schema.yaml](./schema.yaml)). The integration is not a code-level fork; it lives entirely in OpenSpec artifact `instruction:` fields, where each step says "use the Skill tool to invoke `superpowers:xxx`". **No Superpowers skill files are modified, and no OpenSpec CLI changes are made** — the wiring sits purely at the prompt layer.

---

## 2. The seven Superpowers touchpoints

| # | Superpowers skill | Where it's invoked | Trigger |
|---|---|---|---|
| 1 | `superpowers:brainstorming` | `brainstorm` artifact instruction | Direct |
| 2 | `superpowers:writing-plans` | `plan` artifact instruction | Direct |
| 3 | `superpowers:using-git-worktrees` | apply step 1 | Direct |
| 4 | `superpowers:subagent-driven-development` | apply step 2a | Direct |
| 5 | `superpowers:test-driven-development` | (activated inside #4) | **Transitive** (SKILL.md L205 / L274) |
| 6 | `superpowers:requesting-code-review` | (activated inside #4) | **Transitive** (SKILL.md L270) |
| 7 | `superpowers:finishing-a-development-branch` | apply step 4 | Direct |

Plus one **fallback**:

- `superpowers:executing-plans` (apply step 2b) — only used when the host platform lacks subagent support. On Claude Code, 2a is always preferred. Per `superpowers:executing-plans` SKILL.md L14: "If subagents are available, use `superpowers:subagent-driven-development` instead of this skill."

---

## 3. Artifact DAG (with Superpowers injection points)

```text
┌──────────────┐
│  brainstorm  │ ◄── superpowers:brainstorming
│  (root)      │     (2-3 approaches + Alternatives Considered)
└──────┬───────┘
       │
       ├──► ┌──────────┐
       │    │ proposal │    Why (50-1000 chars) / What Changes / Capabilities
       │    └────┬─────┘
       │         │
       │         ▼
       │    ┌──────────────────┐
       │    │ specs/**/*.md    │    ADDED / MODIFIED / REMOVED / RENAMED
       │    │ (delta specs)    │    Each requirement: SHALL/MUST + scenario
       │    └────┬─────────────┘
       │         │
       │         ▼
       │    ┌──────────┐
       │    │  tasks   │    Coarse checkboxes (apply's tracking signal)
       │    └────┬─────┘
       │         │
       │         ▼
       │    ┌──────────┐
       │    │  plan    │ ◄── superpowers:writing-plans
       │    └────┬─────┘     (2-5 minute micro-steps)
       │         │
       │         │ ─────────┐
       │         │          │
       │         │     ┌────▼──────┐
       │         │     │  apply    │ ◄── superpowers:using-git-worktrees
       │         │     │  (phase)  │ ◄── superpowers:subagent-driven-development
       │         │     │           │         ├── superpowers:test-driven-development (transitive)
       │         │     │           │         └── superpowers:requesting-code-review (transitive)
       │         │     │           │ ◄── superpowers:finishing-a-development-branch
       │         │     └────┬──────┘
       │         │          │
       ▼         ▼          ▼
    ┌──────────┐    ┌──────────┐    ┌────────────────┐
    │  design  │    │  verify  │ ◄──│ openspec-      │
    │(optional)│    │          │    │ verify-change  │
    └──────────┘    └────┬─────┘    └────────────────┘
                         │
                         ▼
                   ┌──────────────┐
                   │retrospective │   Evidence-first, 6 sections
                   └──────────────┘
```

**A few things to note:**

- `design` is an **optional leaf**. brainstorming may pre-fill `design.md`, but `tasks.requires: [specs]` only — matching OpenSpec conventions where `design.md` exists only for non-trivial technical decisions.
- `verify.requires: [plan]` and `retrospective.requires: [verify]` are file-existence dependencies in the graph, but their instructions explicitly state they MUST run AFTER apply / after verify passes. See section 6.6 — this is a known limitation; engine-level enforcement awaits a `post_apply` phase concept upstream in OpenSpec.
- `apply` does not produce an artifact; it is a **phase** that mutates source code and ticks `tasks.md` checkboxes.

---

## 4. Full development workflow (one change's lifecycle)

### Step 0: Decide whether you need a change at all

Ask first: is this a behavioral change?

| Type | Needs a change? | Which schema |
|---|---|---|
| New feature / new capability | ✅ Yes | `superpowers-bridge` |
| Breaking change | ✅ Yes | `superpowers-bridge` |
| Architectural change | ✅ Yes | `superpowers-bridge` |
| Bug fix (restoring original behavior) | ❌ No | Direct PR |
| Test backfill / coverage | ❌ No | Direct PR |
| Build tweaks (linter rules, coverage thresholds, etc.) | ❌ No | Direct PR |
| Dependency upgrades (non-breaking) | ❌ No | Direct PR |
| Documentation updates | ❌ No | Direct PR |

The full triage rubric lives in [openspec/specs/README.md](../../specs/README.md), section "When NOT to create a Spec".

---

### Step 1: Create change + enter brainstorming

```bash
/opsx:new my-feature --schema superpowers-bridge
# → creates openspec/changes/my-feature/ + .openspec.yaml
# → displays brainstorm artifact instructions
```

Then:

```bash
/opsx:continue
# → triggers brainstorm artifact
# → instruction says "use the Skill tool to invoke superpowers:brainstorming"
# → enters multi-turn dialogue: context exploration → clarifying Qs → 2-3 options + tradeoffs → design approval
# → on completion, writes brainstorm.md (with Alternatives Considered)
# → if a design doc is produced, also writes to design.md (pre-fill)
```

**Why this matters**: this step is the alignment ritual for the entire flow. Subsequent `proposal` / `specs` are extracted from `brainstorm.md`, not invented anew.

---

### Step 2: Sequentially produce proposal → specs → tasks → plan

Either step through with `/opsx:continue` (review opportunity at each step) or run `/opsx:ff` to fast-forward through all remaining artifacts.

| Step | Output | Key rule |
|---|---|---|
| 2a | `proposal.md` | Why section 50-1000 chars; Capabilities section lists new / modified capabilities |
| 2b | `specs/<capability>/spec.md` | Four delta types (ADDED / MODIFIED / REMOVED / RENAMED); each requirement has SHALL/MUST + `#### Scenario:` |
| 2c (opt) | `design.md` | Only when explaining technical decisions; brainstorm may have pre-filled |
| 2d | `tasks.md` | Coarse checkboxes (`- [ ] X.Y description`), tracked by apply for progress |
| 2e | `plan.md` | `/opsx:continue` triggers `superpowers:writing-plans`, decomposing tasks into 2-5 minute micro-steps |

When done:

```bash
openspec validate --all --json
# → if you've installed the local pre-commit hook, this also runs automatically on commit
```

---

### Step 3: Apply (implementation phase)

```bash
/opsx:apply
```

This triggers the steps inside [schema.yaml](./schema.yaml)'s `apply.instruction`:

#### 3-0. Pre-flight — verify required Superpowers skills

Before creating the worktree, the schema checks that this apply phase's required skills are installed:

- `superpowers:using-git-worktrees`
- `superpowers:subagent-driven-development`
  - Transitive: `superpowers:test-driven-development`, `superpowers:requesting-code-review`
- `superpowers:finishing-a-development-branch`

If any required skill is missing, **STOP and inform the user** — do not silently fall back to manual implementation. The user can install Superpowers, or explicitly opt in to the manual fallback path documented at the end of `apply.instruction`.

**Why this design**: this schema's apply is tightly coupled to Superpowers skill names (hard-coded in `schema.yaml`). If a skill is missing and we keep going, the LLM ends up improvising prompt interpretations and producing irreproducible results. We prefer to fail loudly and early. Note: this PRECHECK is prompt-level enforcement; OpenSpec's engine does not yet recognize the concept of "skill". If OpenSpec adds schema-level capability detection (modeled on spec-kit's `extension.yml` `requires.tools[]`), this step will migrate to a manifest declaration.

> The v0 version of this schema once placed an "auto-commit change artifacts to current branch" step here. It was removed after the [PR #970 review](https://github.com/Fission-AI/OpenSpec/pull/970): handling untracked change directories is the worktree skill's responsibility, not the schema's. A schema should not silently rewrite user git history.

#### 3-1. Workspace — invoke `superpowers:using-git-worktrees`

- Create an isolated workspace at `.worktrees/<change-name>/`
- Switch to a new branch
- Run project setup; confirm a clean test baseline

#### 3-2. Executor — invoke `superpowers:subagent-driven-development` (2a, the default)

- Main agent reads `plan.md`, **dispatches a fresh subagent per micro-task**
- Each subagent automatically inherits:
  - **TDD enforcement** (`superpowers:test-driven-development` activated transitively)
    - Write failing test first
    - Watch it fail
    - Write the minimal code to make it pass
    - Production code without a test? Delete and redo
  - **Per-task code review** (`superpowers:requesting-code-review` activated transitively)
    - Spec compliance review (does it match the plan?)
    - Code quality review (any smells?)
    - Critical issues block forward motion
- Coarse `tasks.md` checkboxes are ticked as tasks complete
- After all tasks, a final code review covers the whole implementation

> **2b fallback**: only used on platforms without subagent support. Claude Code has subagents, so 2a is always the right choice. If you are forced into 2b (constrained runtime), you must enforce TDD manually and invoke `superpowers:requesting-code-review` yourself — neither activates transitively under 2b.

#### 3-3. Verification — invoke `openspec-verify-change` (produces `verify.md`)

Five checks:

1. **Structural validation**: `openspec validate --all --json` returns valid for every item
2. **Task completion**: every `- [ ]` in `tasks.md` becomes `- [x]`
3. **Delta spec sync state**: each directory under `changes/<name>/specs/` has been synced into `openspec/specs/`
4. **Design / specs coherence**: spot-check that design decisions align with spec requirements (non-blocking warnings)
5. **Implementation signal**: no unstaged files in the worktree

If any check fails, return to the corresponding artifact, fix, and re-run verify.

#### 3-4. Completion — invoke `superpowers:finishing-a-development-branch`

- Confirm tests are green
- Present options: merge / PR / keep branch / discard
- Clean up the worktree

#### 3-5. Retrospective — `retrospective` artifact (recommended; trivial fixes may skip)

Before archiving, produce `retrospective.md` in the change directory. The retrospective is a self-review of the entire change — capturing what the diff alone cannot: why decisions were made, what surprised you, which lessons deserve promotion to long-term memory.

**Why it matters**: each retrospective raises the quality of the next change. Without one, blind spots compound; with one, both team and AI learn from each pass.

The 6 sections (evidence-first; every claim cites a commit / file / measurable fact):

1. **Wins** — what worked (with commit / test evidence)
2. **Misses** — what didn't (🔴 blocking / 🟡 painful / 📌 nit)
3. **Plan deviations** — tasks whose scope changed, and why
4. **Skill / workflow compliance** — skills actually invoked vs. deliberately skipped (with reasons)
5. **Surprises** — assumptions that turned out wrong
6. **Promote candidates** — learnings worth moving to long-term memory / `CLAUDE.md` / schema or skill updates (classify each, so insights don't die silently in the archive)

For trivial single-commit fixes, the overhead exceeds the value — skip with a one-line note in `retrospective.md` explaining why.

---

### Step 4: Archive

```bash
/opsx:archive my-feature
```

- Validate + check task completion (incomplete tasks warn but don't block)
- Sync delta specs back to `openspec/specs/<capability>/spec.md`
  - Order: RENAMED → REMOVED → MODIFIED → ADDED
  - If you've already synced manually, pass `--skip-specs`
- Move `changes/my-feature/` to `changes/archive/YYYY-MM-DD-my-feature/`
- History is locked; the unix timeline is the source of truth

---

## 5. Practical CLI cheat sheet

| Scenario | Command |
|---|---|
| **First clone of a project** | `bash scripts/install-git-hooks.sh` |
| New change (interactive, step-by-step) | `/opsx:new <name> --schema superpowers-bridge` then several `/opsx:continue` |
| New change (one-shot) | `/opsx:ff <name>` |
| Resume an interrupted change | `/opsx:continue <name>` |
| Enter implementation | `/opsx:apply <name>` |
| Manual verify | `/opsx:verify <name>` |
| Archive | `/opsx:archive <name>` |
| Use built-in OpenSpec schema (skip brainstorm) | `/opsx:new <name> --schema spec-driven` |
| List all schemas in the project | `openspec schemas` |
| Inspect a change's progress | `openspec status --change <name> --json` |
| List active changes | `openspec list` |
| Validate the entire project | `openspec validate --all --json` |

---

## 6. Six design touches worth remembering

### 1. Skill-name PRECHECK (Layer 1 capability detection)

Each artifact / apply step that invokes a Superpowers skill runs a PRECHECK at the start of its instruction, confirming the skill exists in the LLM's available skills list:

- `brainstorm` artifact → checks `superpowers:brainstorming`
- `plan` artifact → checks `superpowers:writing-plans`
- `apply.instruction` Step 0 → checks 4 skills (`using-git-worktrees`, `subagent-driven-development`, `finishing-a-development-branch`, plus the two transitive ones)

**Missing skill = STOP, no silent fallback.** This is the concrete answer to layer 1 of [PR #970 review](https://github.com/Fission-AI/OpenSpec/pull/970)'s concern #1 — alfred-openspec worried that "if Superpowers renames a skill, OpenSpec would silently ship a stale built-in schema". Our response is **fail loud, fail early.**

### 2. Output redirection

Superpowers' brainstorming defaults to writing into `docs/superpowers/specs/`; writing-plans goes to `docs/superpowers/plans/`. Our artifact instructions **override** this by injecting prompt context that redirects output into the change directory. No Superpowers source is modified; no OpenSpec CLI is touched.

### 3. Schema-level vs prompt-level integration

Integration lives entirely in `instruction:` fields (pure prompts). If Superpowers upgrades a skill's behavior, **our schema doesn't change at all**. We only need to touch `schema.yaml` if a skill is renamed or removed.

### 4. Transitive dependencies made explicit

TDD and code-review are normally hidden inside `subagent-driven-development` (only visible if you read its SKILL.md). Our schema's apply step 2a instruction **lists these two transitive activations explicitly**, so a reader can see "what actually happens during apply" at a glance.

### 5. Fallback path honestly labeled

2b (`executing-plans`) exists but is labeled as the "platforms without subagent support" fallback, citing Superpowers' own SKILL.md L14. We don't invent a self-serving rule like "use 2b for small changes."

### 6. verify and retrospective are time-mismatched artifacts (known limitation)

`verify.requires: [plan]` and `retrospective.requires: [verify]` are file-existence dependencies in the schema graph, but each instruction explicitly states "MUST run AFTER apply phase / verify pass". This is an intentional misalignment — OpenSpec's engine only checks predecessor file existence; it cannot verify whether apply actually completed or whether verify actually passed.

**v1 mitigation**: each artifact has an evidence-based PRECHECK using `git log` / `grep` against observable runtime state (commit count, checkbox completion, verify.md content). The LLM does not need to understand timing — it just runs shell commands and reads 0 / non-zero results.

**Full fix**: when OpenSpec adopts a `post_apply` phase concept (spec-kit already has the analogous `after_implement` hook), `verify` and `retrospective` will migrate from artifacts to `post_apply` steps, and the engine will enforce timing natively.

---

## 7. Recommended snapshot section for adopting projects

Each project that adopts `superpowers-bridge` should keep a snapshot like this in its own docs, so a new contributor can see the current state of the repo at a glance:

```markdown
## Current project state (snapshot: YYYY-MM-DD)

- **OpenSpec CLI**: v<version>
- **Schema**: `superpowers-bridge` v<n>
- **Specs (bounded-context granularity)**: <n> domains exist, <n> domains reserved for lazy backfill
  - Existing: `<capability-a>` / `<capability-b>` / ...
  - Reserved: `<capability-c>` / ...
- **Automation**: <which openspec commands run in pre-commit / CI>
- **Superpowers plugin**: `superpowers@<version>` installed at `<path>`, this integration uses N skills
```

> The snapshot will go stale; for authoritative live state, run `openspec list` + `openspec schemas`.

---

## 8. The single most important takeaway

The integration's core value is not "stitching many skills together," but:

> **Connecting requirement alignment (OpenSpec) with execution rigor (Superpowers), so a change's path from "what we want to build" to "code has passed TDD + code review" is fully traceable, replayable, and auditable.**

The traditional break points are:

- Requirements live in Slack / chat → during apply, the LLM works from memory → output drifts from spec
- Or: spec lives in Confluence → code lives in repo → the two diverge silently

`superpowers-bridge`'s two layers solve this:

1. **OpenSpec's delta-spec governance** → ensures "what we are building" doesn't drift
2. **Superpowers' subagent-driven + TDD + review** → ensures "what we built" has discipline

In short: OpenSpec **rescues requirements from chat**; Superpowers **rescues discipline from human willpower**. Combined, they constitute complete spec-driven development.

---

## Related files

- [schema.yaml](./schema.yaml) — the machine-readable schema definition
- [README.md](./README.md) — design rationale and high-level overview
- [templates/](./templates/) — markdown templates per artifact
- [../../specs/README.md](../../specs/README.md) — capability-domain triage guide
- [openspec-conventions spec](https://github.com/Fission-AI/OpenSpec/blob/main/openspec/specs/openspec-conventions/spec.md) — official OpenSpec conventions
- [obra/superpowers](https://github.com/obra/superpowers) — Superpowers skill source
