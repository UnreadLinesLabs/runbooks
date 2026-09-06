# AGENTS.md — UnreadLines Labs runbooks

A README for AI agents working directly in this repository (opened on its own, without the wider
project folder above it). `CONTRIBUTING.md` (same folder) is authoritative on structure, writing
style, and the pre-publish checklist — read it before touching a runbook, rather than deducing the
format from an existing one. `reference/naming-conventions.md` is authoritative on every name
(servers, accounts, groups, OUs); check it before inventing one.

## Non-negotiable

- Never commit or push without an explicit request from the user.
- English only in this repository — commit messages included.
- Never a real secret, private key, tenant ID, or person's name (`CONTRIBUTING.md`, "What never goes
  in this repository").

## Scope

One runbook = one flat, kebab-case folder at this root, with a `README.md`, linked from the index
`README.md`. `reference/` holds design documents, not procedures. A runbook covers its own subject
only — a topic spanning several runbooks gets its own document in `reference/` instead.
