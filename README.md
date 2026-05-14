# E-Commerce Data Utilities

A TypeScript project providing query functions for a SQLite database, with Claude Code hooks for automated code quality enforcement.

> **Note:** This project includes examples from the Anthropic Academy lessons on Claude Code hooks.

## Getting Started

```bash
npm run setup
```

## Project Structure

```
.
├── src/
│   ├── main.ts               # Entry point — opens DB and applies schema
│   ├── schema.ts             # Schema creation functions
│   └── queries/              # All query modules (customer, order, product, etc.)
├── hooks/
│   ├── read_hook.js          # Blocks reads of .env files
│   ├── query_hook.js         # Detects duplicate query logic via Claude
│   └── tsc.js                # Runs TypeScript check after every file write
└── scripts/
    └── init-claude.js        # Bootstraps the Claude Code environment
```

## Claude Code Hooks

_From Anthropic Academy lessons_

### The Problem

When Claude modifies a function signature, it often doesn't update all the places where that function is called throughout the project. This leads to broken code that only surfaces at runtime — or not at all until a type error is noticed.

**Example:** Claude renames a parameter in `getOrderById(db, id)` but misses three other files that still call the old signature. No immediate error is shown, so the session ends with silently broken code.

### The Solution — TypeScript Typecheck Hook

The fix is a `PostToolUse` hook that runs the TypeScript compiler across the entire project after every file edit:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "./node_modules/.bin/tsc --noEmit"
          }
        ]
      }
    ]
  }
}
```

Running `tsc --noEmit` type-checks the whole project without producing output files. If Claude's edit breaks a call site anywhere in the codebase, the hook surfaces the error immediately — giving Claude the feedback it needs to fix the issue in the same session.

This project uses the TypeScript compiler API directly in `hooks/tsc.js` to achieve the same effect, parsing `tsconfig.json` and reporting all diagnostics back to Claude via `stderr` + exit code `2`.

> This pattern works for any statically typed language. For untyped languages, running your automated test suite as a post-edit hook achieves similar coverage.

### All Hooks in This Project

| Hook            | Trigger                 | Behavior                                                                                     |
| --------------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `read_hook.js`  | `Read` / `Grep` (pre)   | Blocks any attempt to read `.env` files                                                      |
| `query_hook.js` | `Write` / `Edit` (pre)  | Uses Claude to detect duplicate query logic in `src/queries/` and blocks redundant additions |
| `tsc.js`        | `Write` / `Edit` (post) | Runs full TypeScript typecheck on the project after every edit                               |

Prettier is also run automatically after every file write.
