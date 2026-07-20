---
name: cicd-repair-agent
description: Design and build a complete closed-loop, self-healing CI/CD system for any project — one that is continuously monitored, diagnosed, and repaired by AI agents running inside GitHub Actions, not just static predefined checks. The skill first ingests and analyzes ALL project documents (READMEs, design docs, ADRs, manifests, existing workflows) to build a system model, then designs the full pipeline with an explicit risk and edge-case register, then generates the deterministic CI layer (lint, typecheck, tests, builds), the agentic layer (auto-diagnosis, auto-repair, PR review, flaky-test triage, scheduled health sentinels using claude-code-action), the security hardening (prompt-injection and poisoned-pipeline defenses), and tests for the repair system itself. Use this skill whenever the user asks for: a self-healing or self-repairing pipeline, AI agents in CI/CD, automatic fixing of failing builds or tests, a CI system that monitors and diagnoses itself, closed-loop DevOps automation, "make CI fix itself", agentic GitHub Actions, autonomous PR remediation, or a complete CI/CD setup that goes beyond predefined checks. Also use when the user wants to retrofit an existing repo's CI with AI repair agents.
---

# CI/CD Repair Agent — Closed-Loop System Builder

Build CI/CD that doesn't just detect failure — it **diagnoses, repairs, verifies, and learns**. The output of this skill is a complete, runnable system with two layers:

1. **Deterministic layer** — classic gates (lint, typecheck, tests, build, migrations). Cheap, reproducible, boring. This is the sensor network.
2. **Agentic layer** — AI agents (via `anthropics/claude-code-action@v1` or headless `claude -p`) that react to what the sensors report: triage failures, produce root-cause diagnoses, push candidate fixes as PRs, quarantine flaky tests, review incoming code, and run scheduled repo-health patrols.

The deterministic layer decides *whether* something is wrong. The agentic layer decides *what* is wrong and *what to do about it*. Never blur the two: agents must never be the gate that decides merge-worthiness on their own, and static checks must never attempt judgment calls.

If the `cicd-architect` skill is available, use it for the deterministic layer (Phases 3 of this skill delegates to it) and use this skill for everything agentic on top. If it is not available, build the deterministic layer inline following the same principles: fail fast/fail cheap, affected-only, locally reproducible, secrets never in repo.

## Core ideology

1. **The loop is Detect → Triage → Diagnose → Repair → Verify → Learn.** Every design decision maps to one of these six stages. If a stage is missing, the system is not closed-loop — it's just alerts with extra steps.
2. **Agents propose, CI disposes.** Every agent-produced fix lands as a branch + PR and must pass the same deterministic gates as human code. An agent never force-pushes, never merges its own work into main without the repo's normal protections, and never edits workflow files or security config as part of an auto-repair.
3. **Read the whole system before designing any of it.** Docs, ADRs, manifests, existing CI, test layout, deploy scripts. The pipeline must match the system that exists, not a generic template.
4. **Every failure gets classified before any agent touches code.** A failure taxonomy with per-class repair policy (auto-fix / propose-PR / comment-only / escalate) is the control law of the loop. Without it, the agent either does too little or does something dangerous.
5. **Loop prevention is a first-class requirement.** Agent commits that trigger CI that triggers the agent is a runaway. Every agent workflow needs at minimum: attempt counters, actor guards (`github.actor != 'claude[bot]'`-style checks), concurrency groups, and a hard iteration ceiling.
6. **The agent runs with the least privilege that lets it do its one job.** `permissions: {}` at workflow level, explicit per-job grants, no secrets exposed to jobs that read untrusted content. Untrusted content (fork PRs, issue bodies, dependency code) is hostile input to an LLM — treat every agent workflow as a prompt-injection target.
7. **Cost is a design axis.** Model tiering (small model to classify, capable model to repair), `--max-turns` ceilings, and per-repo monthly budget stated as a number. An unbounded agent loop is a billing incident.
8. **The repair system itself is code and gets tested.** Seeded-failure drills, dry-run modes, and a verification checklist. If you can't demonstrate the loop closing on a deliberately broken build, it isn't done.

## Workflow

For a narrow question ("how do I make Claude fix failing tests on PRs?"), jump straight to `references/agent-workflows.md` and generate the specific workflow — but still apply the security defaults and loop-prevention guards. For a full system, run the phases in order.

### Phase 1 — System document analysis

Before proposing anything, build a **system model** from the repository and every document the user provides or the repo contains:

- **Docs**: README, `docs/`, ADRs, design documents, runbooks, CONTRIBUTING, existing `CLAUDE.md`/agent config. Extract: what the system does, its components, its deploy targets, its stated invariants.
- **Manifests**: package manifests, lockfiles, workspace configs, Dockerfiles, IaC. Extract: languages, frameworks, package managers, build commands, test commands.
- **Existing CI**: every file in `.github/workflows/`. Extract: current gates, triggers, secrets referenced, gaps.
- **Test landscape**: where tests live, how they run, current coverage signals, known flaky suites (search issues/comments for "flaky", "retry", "skip").
- **Operational surface**: what "broken" means for this project — failed build? failed deploy? bad data? Each needs its own sensor.

Output of this phase: a short **System Model** section — component inventory, command inventory (install/lint/test/build per component), current CI state, and the project's definition of healthy. If the repo isn't accessible and documents weren't provided, ask for them in ONE focused round; otherwise state assumptions explicitly and proceed.

### Phase 2 — Risk register & edge-case analysis

Before designing the pipeline, enumerate what can go wrong — both in the project and in the repair system you're about to add. Produce a **risk register table**: risk → likelihood → blast radius → mitigation → residual. Cover at minimum:

**Project-side failure modes**: flaky tests, environment drift, dependency breakage (upstream releases), migration failures, secret expiry, cache poisoning/staleness, platform-specific breaks, slow-creep pipeline duration.

**Repair-system failure modes** (these are the ones most designs miss):
- Runaway loops (agent fix triggers CI triggers agent).
- Fix-the-test-instead-of-the-bug: agent makes a failing test pass by weakening the assertion. Mitigation: repair policy forbids editing test expectations for `test-failure` class without explicit human approval; diff-scope guards.
- Prompt injection via PR bodies, issue text, commit messages, or code comments steering the agent (see `references/security-hardening.md` — this is actively exploited in the wild, not theoretical).
- Privilege escalation via `pull_request_target` / `workflow_run` misuse and artifact poisoning.
- Cost blowout from retries on a persistently broken main.
- Trust erosion: agent PRs that are wrong often enough that humans stop reading them. Mitigation: confidence gating — the agent posts diagnosis-only when confidence is low.
- Masking real problems: auto-retry and auto-fix hiding a genuine regression. Mitigation: everything the agent does is logged to the failure ledger; repeated same-class failures escalate instead of silently re-fixing.

Edge cases to explicitly design for: first PR from a fork, force-push during an agent run, two failures racing, agent running while a human is pushing to the same branch, GitHub Actions outage, model API outage (the deterministic layer must function with the agent layer completely down — the agent layer is an enhancement, never a dependency).

### Phase 3 — Deterministic pipeline foundation

Delegate to `cicd-architect` if available (pass it the System Model from Phase 1); otherwise build inline. Non-negotiables regardless of who builds it:

- Jobs ordered cheap→expensive; affected-only in monorepos; lockfile-keyed caching; concurrency groups cancelling superseded runs.
- Every gate reproducible locally with one documented command per component.
- **Structured failure output**: this is where this skill goes beyond a normal pipeline. Every job must emit machine-readable failure context for the agent layer — upload logs as artifacts with predictable names, use `$GITHUB_STEP_SUMMARY`, produce JUnit/JSON test reports. The repair agent's diagnosis quality is bounded by the quality of the failure signal, so design the sensors for the consumer.
- A **test baseline**: if the project has thin or no tests, generate a starter test suite in this phase (smoke tests per component, one integration path) — the loop cannot verify repairs without tests to verify against.

Produce the pipeline as a Mermaid `flowchart LR` (deterministic DAG and agent triggers on one diagram, agents as distinct shapes) plus a stated time/cost budget.

### Phase 4 — Closed-loop architecture

Read `references/closed-loop-design.md`. Design the six stages concretely for THIS project:

- **Detect**: which events feed the loop — `workflow_run` (completed, conclusion=failure) for CI failures; `schedule` for the health sentinel (dependency drift, security advisories, cache health, duration trends); `pull_request` for review; `issue_comment` with `@claude` for on-demand.
- **Triage**: a cheap classification pass (small model or pure heuristics on log patterns) that assigns every failure to a class in the failure taxonomy and selects the repair policy. Never send a capable model to look at a failure that a regex can classify.
- **Diagnose**: the agent pulls logs (`gh run view --log-failed`), the diff, and relevant source; produces a root-cause statement with confidence. Diagnosis is always produced and posted even when repair is not attempted — a good diagnosis alone saves the 20 minutes of log-spelunking.
- **Repair**: per-class policy from the taxonomy. Repairs land on a `ci-repair/<run-id>` branch as a PR with the diagnosis in the body, labeled (e.g. `ci-repair`, `needs-human-review`), scoped by diff guards (max files, max lines, forbidden paths: workflows, lockfile-only unless class=dependency, test assertions unless approved).
- **Verify**: the repair PR runs the full deterministic pipeline. The agent monitors the outcome; on failure it gets N more attempts (default 2), then escalates with everything it learned.
- **Learn**: append every loop execution to a failure ledger (`.github/ci-repair/ledger.jsonl` or issues with a label): class, root cause, fix, outcome. The sentinel reads the ledger to spot repeat offenders (same test flaking 5× → quarantine proposal + issue; same dependency breaking → pin proposal). Distilled lessons go into `CLAUDE.md` so future agent runs start smarter.

Define the **escalation policy as a table**: failure class → auto-action → who is notified → hard stop condition. This table is the single most important deliverable for the human operators; put it in the runbook.

### Phase 5 — Agent workflow implementation

Read `references/agent-workflows.md` and generate the actual workflow files. The standard set (tailor to the project — a solo repo doesn't need all six):

1. `agent-repair.yml` — `workflow_run`-triggered auto-repair on CI failure (the core of the loop).
2. `agent-review.yml` — PR review with inline comments against the project's stated conventions.
3. `agent-interactive.yml` — `@claude` mention handler for on-demand work.
4. `agent-sentinel.yml` — scheduled health patrol: flaky-test detection from the ledger, dependency/security advisories, pipeline-duration trend, doc drift.
5. `agent-triage.yml` — issue triage/labeling (optional).
6. A `CLAUDE.md` (or update to it) giving agents the project's commands, conventions, repair policy, and forbidden actions — this file is the agent's standing orders and must encode the taxonomy's rules.

Every generated workflow must ship with: explicit `permissions`, concurrency group, actor guard, attempt ceiling, `--max-turns`, model choice justified by role, and timeout. No pseudo-YAML: correct action versions, real event payloads (`github.event.workflow_run.id`, not invented fields), and commands that match the System Model's command inventory.

### Phase 6 — Security hardening

Read `references/security-hardening.md` and apply it as a pass over everything generated. Minimum bar:

- `permissions: {}` at workflow level; per-job grants; `persist-credentials: false` on checkout in agent jobs.
- No agent job that reads untrusted content (fork PR code/body, issue text) also holds write perms or secrets beyond what its one task needs; privileged reactions to untrusted events go through the split `workflow_run` pattern without checking out untrusted code in the privileged half.
- Third-party actions pinned to full SHA in any workflow with secret access.
- Injection surfaces: never interpolate `github.event.*` text directly into `run:` shell; pass via `env:`.
- Repo settings stated explicitly in the runbook: require approval for first-time contributors' workflows, branch protection on main, no agent bypass of required reviews.
- The agent's own config files (`CLAUDE.md`, `.claude/`, workflow files) are on the forbidden-paths list for auto-repair and flagged for mandatory human review in `CODEOWNERS`.

### Phase 7 — Testing the repair system & verification

The loop must be demonstrated, not assumed:

- **Seeded-failure drills**: a documented script/checklist that intentionally breaks the build in one representative way per failure class (lint error, failing unit test, broken import, bad dependency bump) on a scratch branch, and the expected loop behavior for each. Run at least one drill before calling the system live.
- **Dry-run mode**: an env flag (`REPAIR_DRY_RUN=true`) under which the repair agent produces its diagnosis and diff as a PR comment/artifact but pushes nothing. Ship with dry-run ON by default; the runbook documents the promotion to live.
- **Kill switch**: one documented action that disables the whole agent layer (repo variable `AGENT_LAYER_ENABLED`, checked by every agent workflow's `if:`), so an operator can stop the loop in one click during an incident.
- Verification checklist: fresh clone passes local commands; dummy PR goes green; seeded failure produces a correct diagnosis; repair PR passes CI; ledger entry written; kill switch actually stops runs.

### Phase 8 — Runbook, trade-offs & evolution

Close with:

- **Operator runbook**: the escalation table, dry-run→live promotion steps, kill switch, how to read the ledger, monthly cost review, what to do when the agent is wrong.
- **Decision table**: each major choice → alternative → why → what was given up (e.g., "repair PRs over direct pushes: slower loop closure, accepted for auditability").
- **What breaks at 10x** (PRs/day, repo size, failure rate) and the mitigation for each.
- **Open questions** for the team before scaling trust in the loop (e.g., when to allow auto-merge of green low-risk repair classes).

## Output structure

For a full build, deliver in this shape (real files with exact repo paths when a filesystem is available; otherwise fenced blocks with paths):

```
# [Project] — Closed-Loop CI/CD Repair System
## 1. System Model                     (from document analysis)
## 2. Risk Register & Edge Cases       (table, incl. repair-system risks)
## 3. Deterministic Pipeline           (diagram, budget, workflow YAML, tests)
## 4. Closed-Loop Design               (six stages, taxonomy, escalation table)
## 5. Agent Workflows                  (YAML + CLAUDE.md standing orders)
## 6. Security Hardening               (applied pass + repo settings)
## 7. Drills, Dry-Run & Verification
## 8. Runbook, Trade-offs & 10x Plan
```

Every file internally consistent with every other (same commands, same branch names, same labels, same ledger path).

## Reference files

Read on demand; skip for questions answerable from this file:

- `references/closed-loop-design.md` — the six-stage loop in depth; the full failure taxonomy with per-class repair policies; escalation table template; loop-prevention mechanisms; the failure ledger format; confidence gating; learning/distillation into CLAUDE.md; flaky-test quarantine protocol.
- `references/agent-workflows.md` — complete runnable YAML for every agent workflow type (repair, review, interactive, sentinel, triage); claude-code-action usage in both interactive and automation modes; headless `claude -p` alternative; log retrieval patterns via `gh`; model tiering and cost controls; CLAUDE.md standing-orders template.
- `references/security-hardening.md` — prompt injection in agentic CI (real attack classes: config-file injection, credential exfiltration, poisoned pipeline execution / MITRE T1677); pull_request_target and workflow_run pitfalls incl. artifact poisoning; permission templates; injection-safe interpolation; supply-chain pinning; incident response for a compromised or misbehaving agent.

## Anti-patterns to actively avoid

- An agent that merges or force-pushes its own fixes, edits workflow/security files during auto-repair, or weakens test assertions to go green.
- Agent workflows with default (broad) token permissions, or secrets available to jobs that read fork-PR content.
- `pull_request_target` with checkout of untrusted head code — this is the canonical breach path.
- No actor guard / no attempt ceiling — the runaway loop.
- Sending every failure straight to the most capable model with no triage — cost blowout and slow feedback.
- A "self-healing" system with no dry-run mode, no kill switch, no drills, and no ledger — that's an unaccountable robot, not infrastructure.
- Designing the agent layer as a dependency of the deterministic layer: if the model API is down, CI must still work.
- Generic template output that ignores the project's actual documents, commands, and constraints — Phase 1 exists precisely so the system fits the repo.
