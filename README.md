# More Options for Jev

[简体中文：README.zh-CN.md](README.zh-CN.md)

Jev's native interface can support at most 255 options per call in a stable way. This repository organizes an extensible solution for situations with more than 255 options: it virtualizes and pages the candidate space so that Jev only sees a limited set of options at a time.

## Background and Goals

In real agent scenarios, tool, action, and parameter candidates expand rapidly with task context. A fixed limit of 255 options introduces two problems:

1. Candidates are truncated, and the correct option may never enter the current decision frame.
2. Hard clipping to fit the limit significantly reduces coverage and recoverability.

The goal of this module is to:

- Preserve the bounded nature of the Jev decision interface;
- Map a logically infinite candidate space into physically resident pages through paging;
- Support replayable, diagnosable decision flows with explicit state actions such as `PAGE`, `EXPAND`, and `REFINE`.

## Overview of the Approach: Virtual Option Space + Resident Options

```text
Open-world candidate space
        ↓ (virtualization)
Virtual pages (logical)
        ↓ (materialize current page)
Resident options (physical, visible to Jev)
        ↓
Jev decision + state transition
```

Core idea:

- **Physical pages**: the set of options the model currently actually sees.
- **Virtual pages**: addressable candidate directories in context that are not currently resident.
- **Paging actions**: when the current page does not contain a suitable candidate, the model can trigger a page change and materialize another virtual page into a new physical page.

This way, Jev does not need to process all candidates at once; it only makes a high-quality choice within a small window while preserving reachability across a larger space.

## Runtime Semantics (Recommended)

The following actions can be used as explicit state transitions for easier trace/replay:

- `PAGE`: switch to another candidate page.
- `EXPAND`: expand the current visible candidate coverage.
- `REFINE`: refine coarse candidates to field/fragment/token level.
- `INVALIDATE`: invalidate stale candidates after a revision change.
- `CLARIFY`: request clarification when the information is insufficient.
- `STOP`: finish or abort the current round.

## Relationship with the Jev Native Agent Project

This repository is a description and distillation of this capability module; the complete runtime prototype is located here:

- https://github.com/6Mikao9/jev-native-agent-with-extended-options

It is recommended to read the runtime and experimental documentation in that repository to understand the full context.

## Quick Start (from the Jev Native Agent project, Core / without helper models)

The following is the minimal runnable path that does not depend on a helper model, and is suitable for validating the paging/option protocol and execution boundaries:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -e .
jev-agent --workspace .\agent_workspace
```

You can also run:

```powershell
python -m jev_agent.cli --workspace .\agent_workspace
```

Common REPL commands:

- `:tools` view available tools
- `:plan file.search {"text":"TODO","path":"."}` generate a candidate plan
- `:approve` approve side-effecting execution

Notes:

- The Core mode uses a deterministic scripted chooser and is mainly intended for protocol and workflow smoke tests.
- No GPU, model weights, or API key are required.
- This path is meant to validate the runtime mechanism and does not represent the upper bound of real Jev end-to-end quality.

## Follow-up Extensions

If you need to connect a helper model or live Jev, continue in the upstream repository via the `.[models]` and `--live --key-stdin` paths. This repository currently focuses on the virtualization and paging of options beyond the 255-option limit.
