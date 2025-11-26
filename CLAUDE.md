# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`zuif` is a Zig library and executable project. The project follows standard Zig package conventions with:
- A library module (`zuif`) defined in `src/root.zig`
- An executable that uses the library module, defined in `src/main.zig`

## Important Constraints

**Zig Version**: This project uses Zig 0.15.x (minimum 0.15.2). This version has significant differences from previous versions. Be aware of API changes and new language features when working with this codebase.

**Language Requirement**: ALL code, comments, documentation, and markdown files MUST be written in English. This is a strict requirement for consistency across the codebase.

**Documentation Structure**:
- Plans must be placed in the `plans/` directory
- Specifications must be placed in the `spec/` directory
  - The `spec/` folder contains 18 detailed specification documents (00-overview.md through 17-tests.md)
  - See [README.md](README.md) for the complete specifications index
  - See [PROJECT-STATUS.md](PROJECT-STATUS.md) for current project status

## Current Project Phase

**SPECIFICATION PHASE ONLY**

Until further notice, the project is in the specification phase. This means:

1. **DO NOT** begin full code implementation
2. **FOCUS ON** refining and completing the specification documents in `spec/`
3. **CODE EXAMPLES** in specifications are allowed and encouraged where they help clarify the design
4. **IMPLEMENTATION GUIDES** within spec documents are acceptable to illustrate how components should work

The goal is to have complete, well-thought-out specifications before starting the implementation phase. All specification work should be done in the `spec/` directory.

## Build Commands

Build the project (outputs to `zig-out/bin/zuif`):
```bash
zig build
```

Run the executable:
```bash
zig build run
```

Pass arguments to the executable:
```bash
zig build run -- arg1 arg2
```

Run all tests (runs both module and executable tests in parallel):
```bash
zig build test
```

Run tests with fuzzing:
```bash
zig build test --fuzz
```

## Architecture

The project is split into two modules as defined in `build.zig`:

1. **Library module (`zuif`)**: Entry point at `src/root.zig`
   - Contains reusable business logic
   - Can be imported by other Zig packages
   - Exposed as a public module for consumers

2. **Executable module**: Entry point at `src/main.zig`
   - Imports the `zuif` library module with `@import("zuif")`
   - Contains the `main` function and CLI logic
   - Not exposed to external consumers

The build system creates two separate test executables:
- One for the library module (`mod_tests`)
- One for the executable module (`exe_tests`)

Both test suites run in parallel when you execute `zig build test`.

## Ongoing Work (TODO: Remove when complete)

**Architectural Analysis in Progress**

We are reviewing and debating the critical problems identified in `ARCHITECTURAL-ANALYSIS.md`.

**Current status:**
- ✅ 1.1 Memory Corruption (VDOM Diffing) - RESOLVED
- ✅ 1.2 Circular Dependencies (Context) - FALSE POSITIVE
- ✅ 1.3 SSR Adopt Fragility - RESOLVED
- ✅ 1.4 CSS Dynamic Composition - RESOLVED
- ⏳ **1.5 WASM Target Incompatibility - PENDING**

**Next session:** Continue discussion on problem 1.5 (WASM Target Incompatibility). The analysis and options are documented in the file. Key questions to resolve:
1. How much code really needs to be platform-specific?
2. Is the Cmd/Effects system sufficient abstraction?
3. How to handle server-only features?
4. Should there be a `ServerCmd` extending `Cmd`?

**NOTE TO SELF**: Remove this entire "Ongoing Work" section once the architectural analysis is complete and all decisions are finalized in `ARCHITECTURAL-ANALYSIS.md`.
