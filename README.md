# Claude Dev Pipeline

A set of six Claude Code agents that implement a structured, review-gated software development pipeline. Drop them into any project to get a consistent workflow: plan → implement → review → test → document → commit.

---

## Installation

The agent files live in the `agents/` directory of this repo. Copy them into your project's `.claude/agents/` directory (create it if it doesn't exist):

```bash
# From anywhere
git clone https://github.com/your-org/claude-dev-pipeline.git
cp claude-dev-pipeline/agents/*.md /path/to/your-project/.claude/agents/
```

Or copy individual agents if you only want some of them:

```bash
cp claude-dev-pipeline/agents/project-manager.md /path/to/your-project/.claude/agents/
cp claude-dev-pipeline/agents/developer.md       /path/to/your-project/.claude/agents/
# ... and so on
```

Claude Code picks up agent files automatically when you run `claude` from your project directory. Verify they are loaded by typing `/agents` in the chat — you should see all six listed.

> **Note:** Installing these agents will not overwrite any existing agents in your project's `.claude/agents/` directory unless you already have files with the same names (`project-manager.md`, `developer.md`, etc.). If you do, review them before copying.

---

## What this is

The pipeline is driven by a **project-manager** agent that orchestrates five specialist agents:

| Agent | Role | Commits? |
|---|---|---|
| `project-manager` | Coordinates the whole pipeline; delegates and routes | No |
| `developer` | Implements the change | No |
| `code-reviewer` | Reviews for correctness, clarity, and simplicity | No |
| `security-reviewer` | Reviews for OWASP Top 10 vulnerabilities | No |
| `tester` | Runs the test suite; commits on clean pass | **Yes — only after all gates pass** |
| `documenter` | Updates README, CLAUDE.md, and design docs | No |

Every change passes through two review gates before the tester is allowed to commit. If either gate returns FAIL, the project-manager routes the findings back to the developer for fixes and the review runs again from the start.

---

## Workflow

The intended workflow for every feature or fix is:

### 1. Enter plan mode

Open Claude Code in your project directory and enter plan mode:

```
/plan
```

### 2. Describe what you want

Tell Claude what you want to build or fix. Work collaboratively to produce a written implementation plan. The plan should cover:

- What problem is being solved
- Which files will change and how
- Any constraints or acceptance criteria
- The spec, schema, or design doc that governs the change (if one exists)

Take the time to get the plan right — the agents will follow it faithfully, so ambiguity now becomes wasted work later.

### 3. Review and refine the plan

Read the plan file. Ask Claude to revise any section that is unclear, incomplete, or wrong. The plan is your contract with the pipeline — it is cheaper to fix here than after implementation.

### 4. Approve the plan

When you are satisfied, approve the plan in Claude Code. This exits plan mode and makes the approved plan available to the implementation session.

### 5. Implement with the project-manager

Ask the project-manager to execute the plan:

```
Use the project-manager to implement the plan.
```

The pipeline runs automatically from here:

```
Approved plan
     │
     ▼
developer  ──── implements change ────────────────────────────────────┐
     │                                                                │
     ▼                                                                │
code-reviewer  ── FAIL ──► relay findings to developer, repeat ──────┤
     │ PASS                                                           │
     ▼                                                                │
security-reviewer  ── FAIL ──► relay findings to developer, repeat ──┘
     │ PASS
     ▼
tester  ── FAIL ──► relay findings to developer, fix cycle restarts
     │ PASS (all tests pass, all gates passed)
     ▼
documenter  ── updates README / CLAUDE.md / relevant docs
     │
     ▼
Done — project-manager reports summary to you
```

The tester commits only when all tests pass **and** both review gates have confirmed PASS. If the tester finds failures, the whole fix cycle restarts from the developer.

---

## Grounding agents in a specification

The single most effective customisation you can make is giving the pipeline an authoritative specification to work from: an OpenAPI/Swagger file, a database schema, a formal design document, or any other machine-readable contract.

When both the developer and the reviewers can check their work against a spec:

- **Architecture drift is caught at review time**, not in production. The code-reviewer verifies that endpoint paths, request/response shapes, and status codes match the spec exactly.
- **Tests have a clear contract**. The tester knows what behaviour to assert against, not just whether the existing tests pass.
- **Scope is bounded**. The developer cannot accidentally extend an API in a way that contradicts the agreed design.

To use a spec, add its path to your project's `CLAUDE.md` and mark it as the source of truth:

```markdown
## API specification

`docs/api.yaml` is the source of truth for all API behaviour.
- Update the spec *before* writing any implementation code.
- Bump the version and add a changelog entry with every change.
- Code reviewers must reject any implementation that contradicts the spec.
```

---

## Customising for your project

### CLAUDE.md

Add a `CLAUDE.md` at your project root describing your tech stack, repo layout, key files, and any project-specific conventions. Every agent reads this file automatically. The more precise your CLAUDE.md, the less you need to repeat in every plan.

A useful CLAUDE.md typically covers:

- Tech stack (language, framework, database, test runner)
- How to run the service locally
- How to run the tests
- Which files are most important to understand first
- Any hard rules (e.g., "no direct SQL — always use the ORM", "no commits without passing tests")
- The location and role of any authoritative spec or schema

### Combining with ponytail and Matt Pocock's skills

Here is a _partial_ CLAUDE.md file that combines [Matt Pocock’s skills[(https://github.com/mattpocock/skills) with these agents to implement an improved software engineering workflow.

It seems to work pretty well. The idea is that you can just ask it for a new feature and it should run through the entire process without you needing to tell it to use the agents or skills directly.

```text
## Feature workflow (mandatory)

Every request for a new feature MUST run this gate sequence, in
order. Do not skip a gate unless the user explicitly waives it for
that request.

1. **Grill** — invoke `mattpocock-skills:grilling` to stress-test the
   idea and requirements before any design work. If grilling surfaces
   new domain terms or an architectural decision, run
   `mattpocock-skills:domain-modeling` (CONTEXT.md / ADR) before
   planning. If a design question can only be answered by trying it,
   run `mattpocock-skills:prototype`; if it needs external facts,
   `mattpocock-skills:research`.
2. **Plan** — enter plan mode; get the plan approved before touching
   code.
3. **Delegate** — hand the approved plan to the `project-manager`
   agent. The PM runs the pipeline: `developer` implements via
   `mattpocock-skills:tdd` (test-first; red-green-refactor) →
   `code-reviewer` gate → `security-reviewer` gate; failed gates
   route fixes back to the `developer` until both PASS.
4. **Review** — run `mattpocock-skills:code-review` against the
   merge-base (Standards axis + Spec axis vs the approved plan);
   route confirmed findings back through the `project-manager`.
5. **Commit** — `tester` runs the full test suite(s) (both repos if
   the change touches both) and commits per repo only when everything
   passes; then `documenter` updates docs (including
   `docs/integration-contract.md` if the handoff changed).
6. **Verify** — invoke `superpowers:verification-before-completion`:
   show fresh test output before claiming the feature is done.

## Bug workflow

Bugs skip Grill/Plan. Diagnose first via
`mattpocock-skills:diagnosing-bugs` (reproduce before you touch
code), then enter the pipeline at **Delegate**: the `developer`
turns the reproduction into a failing test, fixes it, and the same
gates (code-reviewer → security-reviewer → tester → documenter)
apply.
```

It is still advised to use other approaches, such as [ponytail](https://github.com/DietrichGebert/ponytail) to review for dead code, duplications, and similar abstractions.

### Specialised agents

The six generic agents work for any project, but you will get better results by specialising them for your own codebase. For example, if you have a Python backend and a TypeScript frontend in the same repo, you might create:

- `backend-developer.md` — knows the backend framework, ORM, and Python test conventions
- `frontend-developer.md` — knows the frontend framework, TypeScript, and UI test conventions
- `backend-security-reviewer.md` — adds SQL injection, auth bypass, and API token checks
- `frontend-security-reviewer.md` — adds XSS, CSP, and client-side data exposure checks

Create the specialised agents in your project's `.claude/agents/` directory alongside the generic ones. Then update your project's CLAUDE.md to tell the project-manager which specialist to use for each area:

```markdown
## Agent routing

- Backend changes: use `backend-developer` and `backend-security-reviewer`
- Frontend changes: use `frontend-developer` and `frontend-security-reviewer`
- Changes touching both: use both pairs; never assign two developers to the same file
```

### Models

Each agent specifies a model in its frontmatter. The defaults are:

| Agent | Default model | Reason |
|---|---|---|
| `project-manager` | `claude-opus-4-8` | Orchestration benefits from the most capable model |
| `developer` | `claude-sonnet-4-6` | Good balance of capability and cost for implementation |
| `code-reviewer` | `claude-sonnet-4-6` | Same |
| `security-reviewer` | `claude-sonnet-4-6` | Same |
| `tester` | `claude-sonnet-4-6` | Same |
| `documenter` | `claude-haiku-4-5-20251001` | Documentation tasks are lighter; Haiku is faster and cheaper |

Edit the `model:` field in any agent file to change the model for that role.

---

## Agent roles and tools reference

| Agent | Model | Tools | Writes code? | Commits? |
|---|---|---|---|---|
| `project-manager` | Opus | Read, Glob, Grep, Bash, Agent | No | No |
| `developer` | Sonnet | Read, Write, Edit, Bash, Glob, Grep | Yes | No |
| `code-reviewer` | Sonnet | Read, Bash, Glob, Grep | No | No |
| `security-reviewer` | Sonnet | Read, Bash, Glob, Grep | No | No |
| `tester` | Sonnet | Read, Bash, Glob, Grep | No | Yes (on PASS) |
| `documenter` | Haiku | Read, Write, Edit, Glob, Grep | Docs only | No |

Reviewers have no `Write` or `Edit` access by design — they identify problems but cannot modify the code themselves.
