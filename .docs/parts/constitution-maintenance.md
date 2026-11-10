# Constitution Maintenance — Parts Reference

## Amendment Workflow

This document describes how to amend the constitution (`.docs/constitution.md`) and track changes.

### When to Amend

Amend the constitution when:
- A new engineering practice is adopted that affects all contributors
- A working rule needs clarification or correction
- A quality gate tool changes (e.g., ruff → eslint, shellcheck added)
- Branch/push policy needs adjustment
- New "parts" reference file is added

### Amendment Process

1. **Propose**: Create a branch `amend/<short-description>`
2. **Edit**: Modify `.docs/constitution.md` and any affected `parts/*.md`
3. **Document**: Add entry to **Changelog** below with date, author, summary
4. **Review**: PR review required (at least one other contributor)
5. **Merge**: Squash merge to `main` with conventional commit:
   ```
   chore(constitution): <summary of change>
   ```

### Versioning

Constitution uses **semantic versioning** (MAJOR.MINOR.PATCH):
- **MAJOR**: Breaking change to workflow (e.g., new quality gate, policy reversal)
- **MINOR**: New rule, new parts file, clarified guidance
- **PATCH**: Typo fixes, formatting, minor clarifications

Update `Version` header in constitution.md on every amendment.

## Changelog

### Version 1.1.0 — 2026-11-10

**Author**: Mike Stitt (rules drafted by the OpenCode agent on 2026-06-18 at the
owner's request; changelog entry recorded with Claude Code)

**Summary**: Add Working Rules 12–13 so conclusions are treated as assumptions
until validated, and so the reasoning behind decisions is recorded.

**Changes**:
- Working Rules: add rule 12, "Validate assumptions by research or test".
  Every architectural assumption must be traced to a primary source or
  validated by experiment, with the source and why it is valid recorded in an
  ADR or `bibliography.md`.
- Working Rules: add rule 13, "Document the 'why', not just the 'what'".
  Commits, ADRs, comments and PR descriptions must explain the reasoning behind
  a decision.

**Why**: During the 2026-06-17 session the agent asserted, without evidence,
that Romi runs robot code on the Pi. The WPILib docs say it runs on the
desktop. That unverified claim drove a design recommendation before it was
caught.

**Process note**: The rules were written on 2026-06-18 without this workflow,
then committed straight to `main` on 2026-11-10 (`df385e2`, subject
`docs(constitution): ...` instead of `chore(constitution): ...`) with no
changelog entry. This entry records the amendment after the fact. `df385e2` is
left as it is to avoid rewriting published history again.

**Parts affected**:
- `parts/constitution-maintenance.md`: this entry, the Quick Reference rule
  range, and the footer version.

---

### Version 1.0.0 — 2026-06-18 (Ratified)

**Author**: Initial project setup

**Summary**: Created constitution for romi-wpilib-photonvision project, adapted from danno constitution. Establishes:
- Working Rules (1-11)
- Engineering Discipline (Quality Gates, Config as Code, Branch Policy, Conventional Commits, Documentation Hygiene, Plan Etiquette, Upstream Provenance, Scratch Escape Hatch)
- Parts references for: image-builder, romi-dashboard, photonvision, wpilib, hardware-testing, testing, constitution-maintenance
- Governance compliance requirements

**Parts created**:
- `parts/image-builder.md` — Shell script pipeline for image building
- `parts/romi-dashboard.md` — Javalin + Vue dashboard
- `parts/photonvision.md` — PhotonVision install/configure
- `parts/wpilib.md` — WPILib/NT4 integration
- `parts/hardware-testing.md` — QEMU + hardware validation
- `parts/testing.md` — Test strategy and organization
- `parts/constitution-maintenance.md` — This file

---

### Template for Future Amendments

```
### Version X.Y.Z — YYYY-MM-DD

**Author**: <name>

**Summary**: <one-sentence summary>

**Changes**:
- <category>: <specific change>
- <category>: <specific change>

**Parts affected**:
- `parts/<file>.md`: <what changed>
```

---

## Adding a New Parts File

1. Create `parts/<name>.md` following the template:
   ```markdown
   # <Title> — Parts Reference
   
   ## Overview
   <what this covers, when to read it>
   
   ## <Key Sections>
   ...
   
   ## References
   <links to upstream docs, specs>
   ```

2. Add entry to **Parts table** in `.docs/constitution.md`

3. Update this changelog

## Removing/Archiving a Parts File

1. Move to `parts/archive/<name>.md` (preserve history)
2. Remove from Parts table in constitution
3. Update changelog

## Compliance Checking

In PR reviews, verify:
- [ ] Constitution version bumped if amended
- [ ] Changelog entry added
- [ ] Parts table updated if parts added/removed
- [ ] Conventional commit message used
- [ ] No unrelated changes in same PR
- [ ] Quality gates pass (shellcheck, shfmt, ruff, gradle check)

## Quick Reference: Constitution Structure

| Section | Purpose | Read When |
|---------|---------|-----------|
| Working Rules | Behavioral contract (1-13) | Every task |
| Quality Gates | Lint/test requirements | Before commit |
| Config is Code | Verify by exercising | Changing build/config |
| Branch Policy | Git workflow | Starting new work |
| Conventional Commits | Commit format | Every commit |
| Doc Hygiene | Update docs with code | Behavior changes |
| Plan Etiquette | Plan file handling | Planning mode |
| Upstream Provenance | PhotonVision/WPILib handling | Vendor updates |
| Scratch Escape Hatch | Throwaway scripts | Prototyping |

## Enforcement

- **Pre-commit hooks** run shellcheck, shfmt, ruff, spotlessCheck
- **CI** reproduces all gates on PR
- **PR reviews** must verify constitution compliance
- **Violations** block merge until resolved

---

*Last amended: 2026-11-10 | Version: 1.1.0*