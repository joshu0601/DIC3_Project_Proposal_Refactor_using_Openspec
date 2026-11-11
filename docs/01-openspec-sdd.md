# 1. OpenSpec & Spec-Driven Development (SDD)

## Installed

```bash
npm install -g @fission-ai/openspec@latest   # v1.17.0
openspec init --tools claude .                # creates openspec/ + .claude/commands/opsx, .claude/skills
```

## What SDD means

Write down **what the system must do** (a spec) and agree on it before writing code. The AI coding agent then implements against the spec, and the spec stays as the living contract.

| Idea | In OpenSpec |
|---|---|
| Source of truth = current behavior | `openspec/specs/<capability>/spec.md` |
| A change is proposed, not just coded | `openspec/changes/<change>/` |
| Why → What → How → Steps | `proposal.md` → `specs/` (delta) → `design.md` → `tasks.md` |
| Testable requirements | `### Requirement:` with SHALL/MUST + `#### Scenario:` WHEN/THEN |
| Deltas | `## ADDED / MODIFIED / REMOVED / RENAMED Requirements` |
| Merge back after done | `openspec archive <change>` moves deltas into `openspec/specs/` |

## Workflow (spec-driven schema)

```
/opsx:explore  → think about the idea, no files
/opsx:propose  → proposal.md + specs/ + design.md + tasks.md
openspec validate <change> --strict
/opsx:apply    → implement tasks.md one by one, tick checkboxes
/opsx:archive  → specs/ updated, change moved to changes/archive/
```

Useful CLI: `openspec list`, `openspec list --specs`, `openspec show <id>`, `openspec status --change <id>`, `openspec view`.

## DESIGN.md vs. OpenSpec

| DESIGN.md section | OpenSpec artifact |
|---|---|
| §1 Problem | `proposal.md` → Why |
| §2 Requirements F1–F8, N1–N6 | `specs/*/spec.md` (6 capabilities, each with scenarios) |
| §3–4 Architecture, Components | `design.md` → Architecture |
| §6 AI/Analytics, Domain mapping | `design.md` → Decisions D1–D8 |
| §7 Algorithm mapping | `design.md` → Algorithm mapping |
| §8 Implementation, planned additions | `tasks.md` groups 1–7 |
| §10 Evaluation | `tasks.md` group 8 |
| §11 Limitations | `design.md` → Risks / Trade-offs |
| Project constraints | `openspec/config.yaml` → `context` + `rules` |

Requirement → capability mapping:

| Capability | Covers |
|---|---|
| `diagnosis-intake` | F1, F3 |
| `sensor-telemetry` | F2, N4 |
| `differential-diagnosis` | F4, F5, N1, N2 |
| `ipm-recommendation` | F6, N3 |
| `disease-risk-alerts` | F7 |
| `interaction-logging` | F8 |
| (cross-cutting, in `config.yaml` + design D8) | N5, N6 |
