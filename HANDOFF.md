# HANDOFF — casehub-blocks-ui

**Date:** 2026-06-30
**Project:** `/Users/mdproctor/claude/casehub/blocks-ui`
**Workspace:** `/Users/mdproctor/claude/public/casehub/blocks-ui`

---

## Last Session

**Scaffold created — Yarn workspace, TypeScript, 3 stub components, workspace bootstrapped from parent session.**

## Immediate Next Step

Begin component design. First session should:

1. **Read casehub-pages source** — understand component API, dataset contracts, `registerPanel`, `hostPanel`, `pages-event` system
2. **Read peer app dashboards** — see what claudony, devtown, aml already render to identify shared patterns
3. **Design dataset contracts** — what data shapes do case-timeline, trust-score-panel, and channel-activity need?
4. **Implement case-timeline first** — most universally needed component (every app has cases)
5. **Test standalone** — each component must work in a test harness before embedding via pages

## What's Left

Nothing yet — greenfield.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | case-timeline component | M | Med | Dataset: case status, milestones, agent activity |
| — | trust-score-panel component | M | Med | Dataset: Bayesian Beta scores from ledger |
| — | channel-activity component | M | Med | Dataset: qhorus message stream, commitments |
