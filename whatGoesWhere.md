# What Goes Where

This document defines where each kind of thinQL content belongs.

## Root Files

### `README.md`

Explains the project, the goals, and the basics.

Use this file for:
- What thinQL is.
- Why thinQL exists.
- The project goals.
- The basic API shape.
- The high-level usage model.
- Links or references to the LLM instructions and implementation notes.

Do not use this file for:
- Language-specific implementation design.
- Detailed C implementation notes.
- Detailed Go implementation notes.
- Internal design discussion that belongs in `implement.md`.

### `ThinQL.llm.md`

Contains the actual instructions about how to use thinQL.

Use this file for:
- LLM-facing usage rules.
- How to generate code that uses thinQL.
- Usage examples.
- Query shape rules.
- Positional mapping rules from the user/code-generation perspective.
- Transaction usage rules.
- Prohibitions such as no ORM behavior, no query builders, and no dynamic SQL assemblers.

Do not use this file for:
- Internal implementation design.
- Driver internals.
- C memory ownership design discussion.
- Go reflection design discussion.

### `implement.md`

Contains project-level implementation design discussion.

Use this file for:
- Shared implementation principles.
- Cross-language implementation decisions.
- Design topics that affect every implementation.
- Current implementation direction.
- Open project-level implementation questions.

Do not use this file for:
- C-only implementation details.
- Go-only implementation details.
- Usage examples that belong in `ThinQL.llm.md`.
- General project explanation that belongs in `README.md`.

## Language Directories

### `c/`

Contains the C implementation.

Use this directory for:
- C source files.
- C header files.
- C-specific implementation design notes.

Expected files:
- `c/thinQL.c`
- `c/thinQL.h`
- `c/implement.md`

### `c/implement.md`

Contains C-specific implementation design discussion.

Use this file for:
- C public API design.
- C memory ownership rules.
- C string buffer rules.
- C null handling.
- C driver handle design.
- C struct mapping design.
- C transaction state design.
- C error/result structs.
- C implementation tradeoffs.

Do not use this file for:
- General thinQL goals.
- Go implementation design.
- LLM usage instructions.

### `go/`

Contains the Go implementation.

Use this directory for:
- Go source files.
- Go-specific implementation design notes.

Expected files:
- `go/thinQL.go`
- `go/implement.md`

### `go/implement.md`

Contains Go-specific implementation design discussion.

Use this file for:
- Go public API design.
- Go package-level state discussion.
- Go reflection scanning design.
- Go transaction state design.
- Go database/sql integration.
- Go error/result behavior.
- Go implementation tradeoffs.

Do not use this file for:
- General thinQL goals.
- C implementation design.
- LLM usage instructions.

## Workshop

### `workshop/`

Contains previous material, research material, and working documents that are not the current primary source of truth.

Use this directory for:
- Archived docs.
- Previous drafts.
- Images.
- Workshop notes.
- Material that may be mined or reorganized later.

### `workshop/docs/`

Contains the previous docs tree.

Use this directory for:
- Previous versions of docs.
- Source material that has been copied or moved into the current root docs.
- License file, if intentionally kept there.

Do not treat this directory as the primary current documentation unless explicitly stated.

## Current Source Of Truth

The current primary docs are:
- `README.md`
- `ThinQL.llm.md`
- `implement.md`
- `c/implement.md`
- `go/implement.md`

The current implementation locations are:
- `c/`
- `go/`

The workshop docs are retained for reference.
