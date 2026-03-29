# AI Agent Instructions

This file provides guidance for AI agents working in this repository. Read it fully before making any changes.

## Repository purpose

This is a homelab repository. It contains documentation, configuration runbooks, and infrastructure-as-code for a self-hosted homelab built on OPNsense, Proxmox, Pi-hole, and Kubernetes.

## Repository structure

```
docs/                   # Documentation and runbooks
  NETWORK.md            # Source of truth: VLANs, subnets, IPAM
  FIREWALL.md           # OPNsense configuration runbook
  diagrams/             # Network topology diagrams
.github/
  instructions/         # Scoped AI skill files (one per domain)
    network-docs.instructions.md   # Rules for editing NETWORK.md and FIREWALL.md
README.md
```

## Skills

Each file under `.github/instructions/` is a scoped skill. When working on a task, identify which skill files apply and read them before proceeding. Skill files are the authoritative source of domain-specific rules — they take precedence over general assumptions.

| Skill file | Applies to |
|-----------|------------|
| `network-docs.instructions.md` | `docs/FIREWALL.md`, `docs/NETWORK.md` |

New skill files must be created whenever a new domain is introduced to the repo (e.g., Proxmox configs, Kubernetes manifests, Pi-hole settings). See [Maintaining skills](#maintaining-skills).

## Documentation

| Document | Purpose |
|----------|---------|
| `docs/NETWORK.md` | VLANs, IP address management (IPAM), device inventory |
| `docs/FIREWALL.md` | OPNsense setup: aliases, NAT, firewall rules, DHCP |
| `README.md` | Repo overview and links to all docs |

## Rules for all changes

### Always update documentation
- Any change to infrastructure, configuration, or runbooks must be reflected in the relevant doc under `docs/`.
- If no doc exists for the area being changed, create one and link it from `README.md`.

### Always update skills
- If you establish a new convention, constraint, or pattern while working on a task, add it to the relevant skill file under `.github/instructions/`.
- If no skill file exists for the domain, create one with an appropriate `applyTo` frontmatter scope.
- Skills must stay in sync with the docs they describe. If a rule in a skill file no longer matches the docs, update the skill.

### Commit hygiene
- Do not reference AI tools, assistants, or code generation in commit messages or code comments.
- Commits that change both a doc and a skill file for the same reason must be in the same commit.
- Commits that change both `docs/FIREWALL.md` and `docs/NETWORK.md` for the same reason must be in the same commit.

## Maintaining skills

When creating a new skill file:

1. Place it at `.github/instructions/<domain>.instructions.md`.
2. Add a `applyTo` frontmatter that scopes it to the relevant files or glob pattern.
3. Add a row to the Skills table in this file.
4. Commit the skill file together with the change that motivated it.

Example frontmatter:
```
---
applyTo: "docs/PROXMOX.md,proxmox/**"
---
```
