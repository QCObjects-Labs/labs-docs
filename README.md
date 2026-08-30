# QCObjects Labs — archived findings

Documentation of the **QCObjects discovery labs** (August 2026): what was
built, how it was verified, and — most importantly — **what we learned** the
hard way.

This repo is the central write-up. Each lab app lives in its own repo in this
organization, and one skill was published to the `qcobjects-skills` org.

## Lab series

| Lab | Repo | What we did | Status |
|-----|------|------------|--------|
| 1 — Hello world | [`hello-qcobjects`](../hello-qcobjects) | Load the QCObjects browser bundles as vendored scripts; minimal class-based component. | ✅ |
| 2 — Stock app template | [`qcobjects-app`](../qcobjects-app) | Checked out the official QCObjects app template (`mynewapp`) to study factory-standard layout. | ✅ |
| 3 — Verified web app | [`qcobjects-web`](../qcobjects-web) | From-scratch app on the 2.4 line; external `.tpl.html` component verified end-to-end in a headless browser. | ✅ |

## Companion artifact

- **Skill:** [`qcobjects-skills/scaffolding`](https://github.com/qcobjects-skills/scaffolding)
  — installable with `npx skills add qcobjects-skills/scaffolding`. Condenses
  the verified 2.4-line recipe and the render/blank-component diagnosis steps.

## Documents

- [FINDINGS.md](FINDINGS.md) — the consolidated findings learned across the
  labs (the read this for the knowledge).
- [APPENDIX-TOOLING.md](APPENDIX-TOOLING.md) — verification/tooling notes used
  during the labs (headless CDP, shadow-root checks, build recipes).