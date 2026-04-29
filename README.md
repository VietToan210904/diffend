# Diffend

Do not trust AI-generated code blindly. Test every AI-produced diff with `diffend` before shipping.

Diffend means **Diff Defend**: a diff-aware, multi-agent security review system for code changes produced by developers, Codex, or other AI coding tools.

## Idea

AI-assisted coding makes it easy to produce large changes quickly, but security review often becomes the bottleneck. A single diff can include hidden risks that are easy to miss during normal code review, especially when reviewers are also checking functionality, style, and correctness.

Existing automated security tools are useful for common issues like leaked secrets, vulnerable dependencies, injection patterns, and unsafe cryptography. They are weaker at deeper security questions involving authentication, authorisation, privilege boundaries, business logic, session handling, and sensitive data flows.

Diffend is designed to review the code diff first, trace related context only when needed, and produce a persistent scan bundle that can be read by developers, security reviewers, Codex, or another AI coding assistant.

The goal is not just to produce a report. The goal is to create structured security context that an AI agent or human reviewer can use to continue safely.

## Product Shape

Diffend has four connected layers:

- **CLI:** `diffend scan` gives immediate terminal feedback after a code change.
- **SDK:** the reusable scan engine captures diffs, runs security checks, coordinates agents, creates findings, and writes scan bundles.
- **Multi-agent review system:** specialist security agents inspect the diff from different risk perspectives and return structured findings.
- **Context bundle:** generated `.md`, `.patch`, and `.json` files preserve what was scanned, what was found, what needs manual review, and what Codex or another agent should inspect next.

This means Diffend supports two workflows at the same time:

- quick local security feedback for the developer
- deeper AI-assisted security review through a structured handoff bundle

## Core Principle

Diffend should be:

- **Diff-first:** review changed lines before looking at the whole repository.
- **Evidence-driven:** every finding should point to a file, line, check, and reason.
- **Agent-friendly:** every scan should produce enough context for Codex or another AI agent to continue the task.
- **Human-safe:** uncertain security risks should be escalated to manual review instead of being guessed away.
- **Auditable:** every scan should save the exact patch and structured report used to make the decision.

## Multi-Agent Architecture

Diffend uses a supervisor-worker architecture.

The agents do not freely chat with each other. Each specialist agent receives the diff, inspects a specific security area, and returns structured findings to the orchestrator. The orchestrator merges results, deduplicates findings, decides the final status, and writes the scan bundle.

```mermaid
flowchart TD
  Developer["Developer / Codex / AI coding tool"] --> CLI["CLI: diffend scan"]
  CLI --> SDK["Diffend SDK"]

  SDK --> DiffCapture["Git Diff Capture<br/>git diff + git diff --cached"]
  DiffCapture --> RunContext["Run Context<br/>run_id, repo path, patch, changed files"]
  RunContext --> PatchFile["diff.patch"]
  RunContext --> Orchestrator["Scan Orchestrator Agent"]

  Config["Policy Config<br/>future: team rules, severity thresholds"] -.-> Orchestrator

  subgraph Runtime["Agent Runtime Inside The SDK"]
    Orchestrator --> Scope["Diff Scope Agent"]
    Orchestrator --> Secrets["Secrets Gate Agent"]
    Orchestrator --> Dependencies["Dependency Gate Agent"]
    Orchestrator --> Injection["Injection Risk Agent"]
    Orchestrator --> Authz["Auth/Authz Risk Agent"]
    Orchestrator --> SensitiveData["Sensitive Data Agent"]
    Orchestrator --> CryptoSession["Crypto/Session Agent"]

    Scope --> RelatedContext["Related Context Collector<br/>imports, routes, auth helpers, models, tests"]
    RelatedContext --> Injection
    RelatedContext --> Authz
    RelatedContext --> SensitiveData
    RelatedContext --> CryptoSession

    AIAdapter["AI Review Adapter<br/>phase 3: optional LLM-assisted reasoning"] -.-> Injection
    AIAdapter -.-> Authz
    AIAdapter -.-> SensitiveData
    AIAdapter -.-> CryptoSession
  end

  Secrets --> EvidenceStore["Evidence Store<br/>normalized findings + manual review items"]
  Dependencies --> EvidenceStore
  Injection --> EvidenceStore
  Authz --> EvidenceStore
  SensitiveData --> EvidenceStore
  CryptoSession --> EvidenceStore
  Scope --> EvidenceStore

  EvidenceStore --> Adjudicator["Risk Adjudicator Agent<br/>dedupe, severity, final status"]
  Adjudicator --> Reporter["Report + Codex Handoff Agent"]

  Reporter --> Summary["summary.md"]
  Reporter --> Findings["findings.md"]
  Reporter --> Manual["manual-review.md"]
  Reporter --> CodexInstructions["codex-instructions.md"]
  Reporter --> JsonReport["report.json"]
  Reporter --> Terminal["Terminal Result<br/>pass / fail / manual_review_required"]

  CodexInstructions --> FollowUp["Codex Follow-Up<br/>explain, inspect, fix with approval, rescan"]
  FollowUp --> CLI
```

### Runtime Sequence

```mermaid
sequenceDiagram
  participant Dev as Developer or Codex
  participant CLI as diffend scan CLI
  participant SDK as Diffend SDK
  participant Orch as Scan Orchestrator Agent
  participant Agents as Specialist Agents
  participant Adj as Risk Adjudicator Agent
  participant Report as Report + Handoff Agent

  Dev->>CLI: Run diffend scan
  CLI->>SDK: Start scan for repository
  SDK->>SDK: Capture staged and unstaged Git diff
  SDK->>SDK: Create .diffend/runs/{run_id}/
  SDK->>Orch: Send run context and patch
  Orch->>Agents: Run diff scope and security agents
  Agents->>Agents: Inspect changed lines and related context
  Agents->>Orch: Return structured findings
  Orch->>Adj: Send all findings and evidence
  Adj->>Adj: Deduplicate, assign severity, choose final status
  Adj->>Report: Send final report model
  Report->>SDK: Write Markdown, JSON, and diff.patch
  SDK->>CLI: Return terminal summary and scan folder
  CLI->>Dev: Show pass, fail, or manual_review_required
```

### Agent Responsibilities

- **Scan Orchestrator Agent:** owns the run, captures the Git diff, creates `.diffend/runs/<run-id>/`, starts specialist agents, waits for results, and coordinates final output.
- **Diff Scope Agent:** identifies changed files, added lines, deleted lines, file types, package files, route files, tests, and areas that may need related context.
- **Secrets Gate Agent:** detects API keys, tokens, passwords, private keys, credentials, and accidental secret exposure.
- **Dependency Gate Agent:** detects dependency additions, lockfile changes, risky packages, version downgrades, and dependency files that need extra review.
- **Injection Risk Agent:** looks for SQL injection, shell injection, path traversal, unsafe template rendering, unsafe deserialisation, `eval`-style execution, and unsafe user input handling.
- **Auth/Authz Risk Agent:** looks for changes to authentication, authorisation, role checks, permission boundaries, middleware, guards, sessions, and access control logic.
- **Sensitive Data Agent:** looks for personal data exposure, unsafe logging, data returned to clients, analytics leaks, and sensitive fields crossing trust boundaries.
- **Crypto/Session Agent:** looks for weak hashing, insecure randomness, token handling problems, cookie/session risks, and unsafe cryptographic changes.
- **Risk Adjudicator Agent:** merges findings, removes duplicates, assigns severity, decides whether the scan passes, fails, or requires manual review.
- **Report + Codex Handoff Agent:** writes human-readable Markdown, machine-readable JSON, the scanned patch, and focused instructions for Codex or another AI coding agent.

## Workflow

Diffend focuses only on the code diff. It does not try to review the whole repository unless changed lines require related context.

### Execution Flow

1. A developer, Codex, or another AI coding tool makes changes to a repository.
2. The developer runs:

```bash
diffend scan
```

3. The CLI triggers the Diffend SDK.
4. The SDK captures the current Git diff.
5. Diffend creates a scan output folder for the current run.
6. The Scan Orchestrator Agent sends the diff to specialist agents.
7. Automated gate agents detect concrete common vulnerabilities.
8. Risk review agents flag suspicious changes that need deeper judgment.
9. The Risk Adjudicator Agent combines all findings into one final decision.
10. The Report + Codex Handoff Agent writes the final scan bundle.
11. The developer can ask Codex or another agent to read the generated Markdown files for deeper review, explanation, or remediation.

### Git Diff Scope

The first version should scan the current working tree diff:

- unstaged tracked changes from `git diff`
- staged tracked changes from `git diff --cached`

Untracked files should be documented as a limitation in the first version, then added later through explicit file capture.

## Terminal Experience

The terminal output should be short, readable, and useful during normal development.

Example:

```text
Diffend scan started

Checking git diff... done
Checking secrets... done
Checking dependency changes... done
Checking injection risks... warning
Checking auth and permission changes... manual review required

Status: manual review required
Report written to: .diffend/runs/2026-04-29-001/
Next: ask Codex to read .diffend/runs/2026-04-29-001/codex-instructions.md
```

## Final Status Rules

Each scan should end with one final machine-readable status:

- `pass`: no findings and no suspicious areas requiring deeper review
- `fail`: at least one concrete high-confidence security problem was found
- `manual_review_required`: no confirmed failure, but one or more suspicious security-sensitive changes need human or AI-assisted review

The CLI may display `manual_review_required` as `manual review required` for readability. Individual checks may produce lower-level states like `done`, `warning`, or `skipped`, but the final scan status must always be one of the three statuses above.

## Scan Bundle

Each scan should always generate an output folder, even when no problems are found.

Example:

```text
.diffend/
  runs/
    2026-04-29-001/
      summary.md
      findings.md
      manual-review.md
      codex-instructions.md
      diff.patch
      report.json
```

### File Responsibilities

- `summary.md`: human-readable scan summary, final status, checks performed, and next steps.
- `findings.md`: concrete automated findings with severity, evidence, location, agent name, and recommendation.
- `manual-review.md`: suspicious areas that require human or AI-assisted security judgment.
- `codex-instructions.md`: focused prompt-style handoff file telling Codex what was scanned, what needs deeper review, which files to inspect, and how to continue safely.
- `diff.patch`: the exact Git diff that Diffend scanned.
- `report.json`: structured machine-readable report for future integrations, CI, IDE plugins, dashboards, and AI coding tools.

## Finding Format

All agents should return findings using a shared structure.

Example:

```json
{
  "agent": "auth-authz-risk",
  "type": "authorization_change",
  "severity": "high",
  "status": "manual_review_required",
  "file": "src/routes/admin.ts",
  "line": 42,
  "evidence": "Role check was removed from an admin route.",
  "recommendation": "Verify whether this route still requires admin-only access.",
  "related_files": [
    "src/middleware/auth.ts",
    "src/models/user.ts"
  ]
}
```

## Report JSON Shape

The first version of `report.json` should follow a simple schema that can evolve later.

```json
{
  "tool": "diffend",
  "run_id": "2026-04-29-001",
  "status": "manual_review_required",
  "repository": {
    "path": "/path/to/repo",
    "base_ref": null,
    "head_ref": null
  },
  "diff": {
    "files_changed": 3,
    "added_lines": 42,
    "deleted_lines": 8,
    "patch_file": "diff.patch"
  },
  "checks": [
    {
      "agent": "secrets-gate",
      "status": "pass",
      "findings_count": 0
    },
    {
      "agent": "auth-authz-risk",
      "status": "manual_review_required",
      "findings_count": 1
    }
  ],
  "findings": [],
  "manual_review": [],
  "codex_next_steps": [
    "Inspect src/routes/admin.ts and verify the removed role check.",
    "Check whether src/middleware/auth.ts still protects the route."
  ]
}
```

## Context Handoff Contract

Every scan should leave enough context for a future reviewer or AI coding assistant to answer these questions without starting from scratch:

- What diff was scanned?
- Which files and added lines were involved?
- Which automated checks ran?
- Which findings are concrete security problems?
- Which areas are suspicious but need deeper judgment?
- Which related files should Codex inspect next?
- What is the safest next action for the developer?

## AI Agent Follow-Up Workflow

Diffend should support an AI-assisted review loop after the scan bundle is created.

Example:

1. Developer runs `diffend scan`.
2. Diffend writes `.diffend/runs/<run-id>/codex-instructions.md`.
3. Developer asks Codex to read the scan bundle.
4. Codex inspects the exact diff and related files listed by Diffend.
5. Codex explains the security risk.
6. Codex suggests or applies a fix.
7. Developer reruns `diffend scan`.
8. Diffend compares the new scan result and produces an updated bundle.
9. The final result is `pass`, `fail`, or `manual_review_required`.

Later versions can support an automated remediation agent, but the first version should keep code modification under developer control.

## Implementation Phases

### Phase 1: Deterministic CLI and SDK

Build the first working version of `diffend scan`.

- Implement the CLI command.
- Capture staged and unstaged Git diffs.
- Create `.diffend/runs/<run-id>/` for each scan.
- Write `summary.md`, `findings.md`, `manual-review.md`, `codex-instructions.md`, `diff.patch`, and `report.json`.
- Define shared finding and report types.
- Print progress for each check in the terminal.
- Return `pass`, `fail`, or `manual_review_required`.

### Phase 2: Automated Security Gates

Add deterministic specialist agents for common security checks.

- Secrets scanning.
- Dependency change detection.
- Simple injection pattern checks.
- Risky authentication or authorisation change detection.
- Sensitive data exposure checks.
- Weak cryptography and unsafe session checks.

### Phase 3: AI-Assisted Risk Review

Add LLM-powered or AI-assisted specialist agents for deeper review.

- Trace related files from the diff when needed.
- Identify suspicious changes in security-sensitive areas.
- Generate manual review checklists.
- Produce higher-quality Codex handoff instructions.
- Keep every conclusion evidence-based.

### Phase 4: Remediation and Verification Loop

Add optional agents that help fix and verify security issues.

- Suggest safe fixes.
- Optionally apply fixes with developer approval.
- Rerun `diffend scan`.
- Compare before and after scan results.
- Produce final review notes.

### Phase 5: Integrations

Extend Diffend beyond local CLI use.

- CI integration.
- GitHub pull request comments.
- IDE plugin support.
- Team dashboards.
- Historical scan comparison.
- Policy configuration for teams.

## Roadmap

### 29/04/2026

- Implement the first version of the `diffend scan` command.
- Implement Git diff capture from the current working tree.
- Define the core Diffend SDK interface:
  - input: repository path and captured diff
  - output: structured security report, final status, and scan output folder
- Create `.diffend/runs/<run-id>/` for each scan.
- Write the first scan bundle files.
- Build the first automated gates runner.
- Add initial gates for secrets, dependency changes, simple injection patterns, and risky auth/authz changes.
- Define shared finding and report formats.
- Print progress for each check in the terminal.
- Make `codex-instructions.md` useful as a direct handoff prompt for Codex.
- Test the command against small sample diffs.

Goal for the day: `diffend scan` can capture a diff, run basic automated gates, print terminal progress, and write a structured scan bundle that developers can hand to Codex for follow-up.

### 30/04/2026

- Implement the first version of the multi-agent review architecture.
- Add the Scan Orchestrator Agent.
- Add specialist agents for secrets, dependencies, injection, auth/authz, sensitive data, and crypto/session review.
- Add the Risk Adjudicator Agent.
- Add the Report + Codex Handoff Agent.
- Generate a manual review checklist when suspicious security risk is found.
- Merge automated gate findings and security risk findings into one final report.
- Improve `codex-instructions.md` for AI-agent follow-up.
- Run end-to-end tests on sample vulnerable diffs.
- Document known limitations and next steps.

Goal for the day: Diffend can run a multi-agent diff security review and produce a final report with Markdown files that support Codex-assisted follow-up.

## Known Limitations For V1

- Untracked files may not be scanned at first.
- Findings will initially rely on simple deterministic patterns.
- Dependency vulnerability lookups may require later registry or advisory database integration.
- AI-assisted review should be treated as advisory, not as a replacement for human security judgment.
- Diffend should not claim that a scan proves code is secure. It should say what was scanned, what was found, and what still needs review.

## Resources

- [AI-Generated Code Security Risks - Why Vulnerabilities Increase 2.74x and How to Prevent Them](https://www.softwareseni.com/ai-generated-code-security-risks-why-vulnerabilities-increase-2-74x-and-how-to-prevent-them/)
