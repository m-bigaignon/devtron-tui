# CLAUDE.md

`devtron-tui`: a keyboard-driven terminal UI (k9s / lazygit style) for one or more Devtron
instances, written in Rust. Personal project, not a Xefi one. Xefi org conventions do not apply
unless this file or the constitution adopts them.

## Layout

```
.specify/memory/constitution.md   ← governing rules, read before planning or coding
specs/NNN-feature/                ← spec-kit artifacts per feature (spec, plan, tasks)
backlog.md                        ← agreed future features, not yet specified
Cargo.toml, src/, tests/          ← the crate (created by the first implementation)
```

Features go through spec-kit: `/speckit-specify` → `/speckit-clarify` → `/speckit-plan` →
`/speckit-tasks` → `/speckit-implement`. The constitution wins over anything else here.

## Build and test

```bash
cargo build
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

All four must pass before a commit lands on `main`. Tests never call a live Devtron instance:
they use scrubbed fixtures recorded from a real one (constitution, Principle V).

## Secrets

The Devtron API token lives in `~/.devtron-token` (or a per-instance token file). Never print it,
copy it into fixtures, or pass it on a command line. Scrub tokens, emails and internal hostnames
from every recorded response before committing it.

This repository is public. Specs, plans and fixtures refer to "the reference instance" and never
include its address, organisation name, app names or user emails. Raw recordings and the local
deny-list live in `fixtures-raw/`, which is git-ignored.

## Commits

One-line imperative subject naming the behaviour change, no body.
