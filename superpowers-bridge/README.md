# superpowers-bridge Schema

[English](./README.md) · [繁體中文](./README.zh-TW.md)

Bridges OpenSpec's artifact governance with [obra/superpowers](https://github.com/obra/superpowers) execution skills, plus an evidence-first `retrospective` artifact filling a gap Superpowers does not natively cover.

## Install

### Method 1: Claude Code one-shot prompt (recommended)

Copy and paste this into Claude Code in your project root:

```
Install the superpowers-bridge schema for OpenSpec into this project:

1. Verify the project has an `openspec/` directory (run `openspec init` if missing).
2. Clone https://github.com/JiangWay/openspec-schemas to a temp dir.
3. Copy the `superpowers-bridge/` subdirectory to `openspec/schemas/superpowers-bridge/`.
4. Run `openspec schema validate superpowers-bridge` to verify.
5. Run `openspec schemas` and confirm `superpowers-bridge` is listed.
6. Clean up the temp directory.
7. Verify Superpowers plugin is installed by running `claude plugin list`.
   If not listed, run `claude plugin install superpowers@claude-plugins-official`.
8. Show me the final state.
```

### Method 2: Manual bash (CI / non-Claude environments)

```bash
git clone https://github.com/JiangWay/openspec-schemas /tmp/oss
cp -R /tmp/oss/superpowers-bridge ~/your-project/openspec/schemas/superpowers-bridge
rm -rf /tmp/oss
cd ~/your-project
openspec schema validate superpowers-bridge
claude plugin install superpowers@claude-plugins-official  # if not already
```

## What problem does this schema solve?

OpenSpec governs **what to do** (artifact lifecycle: proposal / specs / tasks / verify, etc.). Superpowers governs **how to do it** (execution discipline: brainstorming, writing-plans, TDD, code-review). Each is solid on its own, but interleaving them in real development surfaces three structural problems:

1. **Output duplication** — brainstorming writes design output to the Superpowers directory (`docs/superpowers/specs/`); OpenSpec then re-authors `proposal.md` / `design.md` in the change directory, with overlapping content.
2. **Task fragmentation** — OpenSpec's `tasks.md` (coarse checkboxes) and Superpowers' `plan.md` (TDD micro-steps) describe the same work in different formats, locations, and progress trackers.
3. **Manual orchestration** — the user has to decide on every step which skill to invoke; the two systems do not connect on their own.

### Why a custom schema rather than modifying existing skills?

Two alternatives were considered and rejected:

- **Adding custom fields to `config.yaml`** (e.g., `skill_bindings`): the OpenSpec CLI does not recognize these — no validation, no discoverability, and reading them would require editing multiple SKILL.md files.
- **Editing the opsx skill files directly**: invasive (affects every change) and fragile (overwritten on SKILL.md upgrade).

A custom schema uses OpenSpec's **native project-level schema mechanism**:

- The CLI validates schema structure
- `openspec schemas` lists it automatically
- Each change can pick its schema independently (`--schema spec-driven` or `--schema superpowers-bridge`)
- No existing SKILL.md or command file is modified

---

## Workflow overview

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan ──→ [apply] ──→ verify ──→ retrospective
                  │                     ↑
                  └──→ design ──────────┘
                       (optional)
```

Differences from `spec-driven`:

| | spec-driven | superpowers-bridge |
|---|---|---|
| Entry | proposal (manual) | **brainstorm** (invokes brainstorming skill) |
| Plan layer | tasks (coarse) | tasks + **plan** (TDD micro-steps) |
| apply requires | tasks | **plan** |
| apply method | standard task-by-task | **worktree + subagent-driven-development** (with TDD + code-review transitive) |
| Post-apply | (none) | **verify** + **retrospective** artifacts |
| New artifacts | — | brainstorm, plan, verify, retrospective |

---

## Integrated Superpowers skills

| Schema phase | Superpowers skill invoked | Trigger |
|--------------|--------------------------|---------|
| brainstorm artifact | `superpowers:brainstorming` | artifact instruction (with PRECHECK) |
| plan artifact | `superpowers:writing-plans` | artifact instruction (with PRECHECK) |
| apply phase Step 1 | `superpowers:using-git-worktrees` | apply instruction (with PRECHECK) |
| apply phase Step 2a | `superpowers:subagent-driven-development` | apply instruction |
| (transitive) | `superpowers:test-driven-development` | activated inside #4 |
| (transitive) | `superpowers:requesting-code-review` | activated inside #4 |
| apply phase Step 3 | `openspec-verify-change` (OpenSpec built-in, not Superpowers) | produces verify.md |
| apply phase Step 4 | `superpowers:finishing-a-development-branch` | apply instruction |
| retrospective artifact | (embedded procedure, no external skill) | 6-step procedure inlined in instruction |

All integration happens via the `instruction:` field in `schema.yaml` — instructing the AI to invoke skills via the Skill tool at appropriate moments. Each Superpowers skill is preceded by a PRECHECK that STOPs without silent fallback if the skill is unavailable. **No Superpowers skill files are modified.**

The `retrospective` artifact fills a v1 capability gap (Superpowers has no retro skill); its procedure is embedded directly in the schema instruction. v1.x may upgrade to a Claude Code plugin if real demand surfaces (see [docs/roadmap.md](../docs/roadmap.md)).

### Output redirection

Superpowers skills have default output paths (e.g., brainstorming writes to `docs/superpowers/specs/`). This schema's artifact instructions **override** that behavior by injecting context that redirects output into the change directory:

- brainstorming → `openspec/changes/<name>/brainstorm.md` (+ optional `design.md`)
- writing-plans → `openspec/changes/<name>/plan.md`

This is implemented purely via context injection at invocation time, not by modifying the skills' source.

---

## Usage

### Quick flow (recommended)
```bash
/opsx:ff my-feature    # one-shot: scaffold + brainstorm + proposal + design + specs + tasks + plan
/opsx:apply            # worktree + subagent-driven-development (with TDD + code-review)
/opsx:verify           # produces verify.md (5 checks)
/opsx:continue         # → retrospective (produces retrospective.md, 6 sections)
/opsx:archive          # archive
```

### Step-by-step flow
```bash
/opsx:new my-feature --schema superpowers-bridge
/opsx:continue         # → brainstorm (interactive dialogue)
/opsx:continue         # → proposal
/opsx:continue         # → design (optional, only when explaining technical decisions)
/opsx:continue         # → specs
/opsx:continue         # → tasks
/opsx:continue         # → plan
/opsx:apply            # → implementation + worktree + subagent-driven-development
/opsx:verify           # → verify.md (post-apply, runs the 5 checks)
/opsx:continue         # → retrospective.md (post-verify, evidence-first 6 sections)
/opsx:archive
```

### Switching back to spec-driven
```bash
# Use a different schema for one change
/opsx:new my-simple-fix --schema spec-driven

# Or change project default
# openspec/config.yaml: schema: spec-driven
```

---

## Design decisions

### Why `brainstorm` is an artifact, not a hook

Brainstorming is multi-turn interactive dialogue requiring user participation. Modeling it as the first artifact (rather than a schema-level hook) gives two advantages:

1. **Skippable** — if the user already knows what they want to build, they can author `brainstorm.md` directly without invoking the skill.
2. **Trackable** — `openspec status` reports brainstorm completion, and downstream artifacts have explicit dependencies on it.

### Why `plan` is separate from `tasks`

`tasks.md` is a coarse checkbox ("Add PdfServiceTest"); `plan.md` is micro-steps ("scaffold test → write downloadPdf test → run → commit"). They serve different purposes:

- `tasks.md` → tracks overall progress (apply phase's `tracks` field parses these checkboxes)
- `plan.md` → guides subagents step by step (the executor's input)

Apply phase requires `plan` (not `tasks`) because the executor needs micro-steps to work effectively; `tracks: tasks.md` ensures progress is still surfaced via the coarse checkboxes.

### Fallback strategy

If a Superpowers skill is unavailable (not installed, version mismatch, etc.), every relevant instruction includes an explicit fallback path:

- brainstorm → write `brainstorm.md` manually
- plan → write `plan.md` manually
- apply → standard task-by-task implementation

The PRECHECK at the start of each instruction ensures missing skills surface as a clear error rather than silent degradation.
