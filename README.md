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

---

### Example 1 — PreToolUse: Block Sensitive File Reads

**The Problem**

Claude reads files to understand context while completing tasks. Left unchecked, it could read sensitive files like `.env` that contain API keys, database credentials, or other secrets — even unintentionally.

**The Solution**

A `PreToolUse` hook intercepts every `Read` and `Grep` call before it executes and blocks access to `.env` files:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node ./hooks/read_hook.js"
          }
        ]
      }
    ]
  }
}
```

`hooks/read_hook.js` reads the tool input from `stdin`, checks the target file path, and exits with code `2` (blocking the tool call) if the path includes `.env`:

```javascript
if (readPath.includes(".env")) {
  console.error("You cannot read the .env file");
  process.exit(2);
}
```

Claude receives the error message and moves on without ever seeing the file contents.

---

### Example 2 — PostToolUse: TypeScript Typecheck

**The Problem**

When Claude modifies a function signature, it often doesn't update all the places where that function is called throughout the project. This leads to broken code that only surfaces at runtime.

<img width="534" height="374" alt="DEBUG CONSOLE" src="https://github.com/user-attachments/assets/ffea0881-125c-4ed7-bdfc-48c4b89dc129" />

<img width="423" height="374" alt="settings local json" src="https://github.com/user-attachments/assets/10c2eccc-4404-4136-936f-46af2f797f0e" />

In this project, `src/schema.ts` exports:

```typescript
export async function createSchema(db: Database, verbose: boolean) {
```

If Claude changes this signature — say, removing the `verbose` parameter — it may not update `src/main.ts`, which calls `createSchema(db, false)`. The project silently breaks with no immediate feedback.

**The Solution**

<img width="660" height="398" alt="shashi@Mac queries_COMPLETED " src="https://github.com/user-attachments/assets/4e56521d-f2d9-4142-acc5-79212280179b" />

A `PostToolUse` hook runs the TypeScript compiler across the entire project after every file edit:

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

`tsc --noEmit` type-checks the whole project without producing output files. If Claude's edit breaks a call site anywhere in the codebase, the hook immediately surfaces the error — giving Claude the feedback it needs to fix the issue in the same session.

`hooks/tsc.js` in this project implements the same check using the TypeScript compiler API directly, parsing `tsconfig.json` and reporting all diagnostics back to Claude via `stderr` + exit code `2`.

> This pattern works for any statically typed language. For untyped languages, running your automated test suite as a post-edit hook achieves similar coverage.

<img width="549" height="401" alt="OUTPUT" src="https://github.com/user-attachments/assets/02a55e03-e5a8-4e48-b66d-733dd6de3ab3" />

---

### Example 3 — PreToolUse: Prevent Duplicate Queries

**The Problem**

Claude often adds new query functions without checking if similar ones already exist, leading to duplicate logic scattered across the codebase.

**The Solution**

A `PreToolUse` hook runs before every file write inside `src/queries/`. It sends the proposed change to Claude, which checks existing query files and blocks the edit if the functionality is already covered:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "node ./hooks/query_hook.js",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

If a duplicate is found, the hook exits with code `2` and returns specific feedback — which existing function to use instead. If the change is appropriate, it exits with `0` and the edit proceeds.

---

### All Hooks in This Project

| Hook            | Trigger                 | Behavior                                                                                     |
| --------------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `read_hook.js`  | `Read` / `Grep` (pre)   | Blocks any attempt to read `.env` files                                                      |
| `query_hook.js` | `Write` / `Edit` (pre)  | Uses Claude to detect duplicate query logic in `src/queries/` and blocks redundant additions |
| `tsc.js`        | `Write` / `Edit` (post) | Runs full TypeScript typecheck on the project after every edit                               |

Prettier is also run automatically after every file write.
