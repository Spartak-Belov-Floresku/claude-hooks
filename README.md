# Claude Code Hooks — Educational Repository

This project teaches **Claude Code Hooks** through a realistic, working codebase. An e-commerce SQLite database and a set of TypeScript query functions serve as the environment that Claude Code operates on, while the hooks intercept and augment those interactions in real time.

By working through this repository you will learn:

- How to write hooks that fire on specific Claude Code events
- How to read and interpret the stdin payload your hook receives
- How to allow, block, or provide feedback using exit codes
- How to share hook configurations across machines safely
- How to use the Claude Agent SDK inside a hook for AI-powered enforcement

---

## Prerequisites

| Requirement | Version |
|---|---|
| Node.js | 18 or later |
| npm | 8 or later |
| [`jq`](https://jqlang.org/) | any recent version — used by the logging hooks |

---

## Quick Start

```bash
# 1. Clone the repository and enter the project directory
git clone <repo-url>
cd queries

# 2. Install dependencies and generate your local hooks config
npm run setup

# 3. Open the project in Claude Code
claude .
```

After step 2 you will have a `.claude/settings.local.json` file with absolute paths resolved for your machine. Open the project in Claude Code and the hooks will begin firing automatically.

---

## How It All Fits Together

```
Claude Code (the AI)
      │
      │  wants to write / edit a file
      ▼
PreToolUse hook ──► hooks/query_hook.js
      │                Uses the Claude Agent SDK to check whether
      │                the new query duplicates an existing one.
      │                Blocks the write (exit 2) if duplication found.
      │
      │  write proceeds
      ▼
 File is written
      │
      ▼
PostToolUse hook ──► Prettier (auto-formats the file)
                 ──► hooks/tsc.js (TypeScript type check)
                         Blocks with error output (exit 2) if types fail.
```

The e-commerce database and `src/queries/` files are the surface that Claude works on. The hooks are the interesting part — they fire every time Claude touches a file, enforcing quality automatically.

---

## Project Structure

```
.claude/
  settings.example.json   # Shared hook config with $PWD placeholders
  settings.local.json     # Generated on setup — machine-specific absolute paths

hooks/
  query_hook.js           # PreToolUse: AI review for query duplication
  tsc.js                  # PostToolUse: TypeScript type checking
  read_hook.js            # Additional hook example

scripts/
  init-claude.js          # Replaces $PWD placeholders → settings.local.json

src/
  main.ts                 # Entry point — opens DB and creates schema
  schema.ts               # Schema creation functions
  queries/
    customer_queries.ts
    product_queries.ts
    order_queries.ts
    analytics_queries.ts
    inventory_queries.ts
    promotion_queries.ts
    review_queries.ts
    shipping_queries.ts

sdk.ts                    # Standalone Claude Agent SDK usage example
```

---

## Claude Code Hooks

### Available Hook Events

| Event | When it fires |
|---|---|
| `PreToolUse` | Before Claude calls a tool — can inspect input and block the call |
| `PostToolUse` | After a tool completes — can inspect both input and response |
| `Notification` | When Claude needs permission for a tool, or after 60 s of idle |
| `Stop` | When Claude finishes responding |
| `SubagentStop` | When a subagent (shown as a "Task" in the UI) finishes |
| `PreCompact` | Before a compaction operation (manual or automatic) |
| `UserPromptSubmit` | When the user submits a prompt, before Claude processes it |
| `SessionStart` | When starting or resuming a session |
| `SessionEnd` | When a session ends |

### Hook Exit Codes

Your hook command communicates back to Claude Code through its exit code:

| Exit code | Meaning |
|---|---|
| `0` | Success — allow the action to proceed |
| `1` | Error in the hook itself — Claude Code logs it but proceeds |
| `2` | Block — Claude Code stops the action and shows any stderr output as feedback to Claude |

Writing to `stderr` before exiting with `2` lets you tell Claude exactly what went wrong, which it will use to self-correct.

---

### Hooks in This Project

#### `hooks/query_hook.js` — AI-powered query duplication check

**Trigger:** `PreToolUse` on `Write | Edit | MultiEdit` for files inside `src/queries/`

This hook uses the **Claude Agent SDK** to have a second Claude instance review the proposed file content before it is written. If the new query function duplicates logic already present in the queries directory, the hook outputs a specific explanation to stderr and exits with `2`, blocking the write.

```js
// Simplified core logic
const prompt = `You are reviewing a proposed change to a database query file.
Identify if any new query functions duplicate existing ones in ./src/queries.
If yes, explain which existing function to use instead.
If no, say "Changes look appropriate."`;

for await (const message of query({ prompt })) { ... }

if (!result.includes("Changes look appropriate")) {
  console.error(`Query duplication detected:\n\n${result}`);
  process.exit(2); // block the write
}
```

This demonstrates using one Claude instance to govern another — a powerful pattern for AI-assisted code quality enforcement.

#### `hooks/tsc.js` — TypeScript type checking

**Trigger:** `PostToolUse` on `Write | Edit | MultiEdit` for `.ts` / `.tsx` files

After Claude writes or edits a TypeScript file, this hook runs the TypeScript compiler programmatically against `tsconfig.json` (with `noEmit: true`). If there are type errors, it prints them to stderr and exits with `2`, blocking further progress until Claude fixes the types.

```js
const typeChecks = runTypeCheck("./tsconfig.json");
if (typeChecks) {
  console.error(typeChecks);
  process.exit(2); // block — type errors found
}
```

#### Logging hooks (`jq`)

Both `PreToolUse` and `PostToolUse` include a catch-all `"matcher": "*"` logging hook:

```json
{
  "matcher": "*",
  "hooks": [{ "type": "command", "command": "jq . > pre-log.json" }]
}
```

These write the raw stdin payload to `pre-log.json` / `post-log.json` on every tool use. They are invaluable when writing new hooks — inspect those files to see exactly what data your command receives.

---

### Understanding Hook stdin

The stdin payload your hook receives varies by event type **and** by which tool was called. This is the trickiest part of hook development.

**`PostToolUse` on the `TodoWrite` tool:**

```json
{
  "session_id": "9ecf22fa-edf8-4332-ae85-b6d5456eda64",
  "transcript_path": "/path/to/transcript",
  "hook_event_name": "PostToolUse",
  "tool_name": "TodoWrite",
  "tool_input": {
    "todos": [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }]
  },
  "tool_response": {
    "oldTodos": [],
    "newTodos": [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }]
  }
}
```

**`Stop` hook:**

```json
{
  "session_id": "af9f50b6-f042-4773-b3e2-c3a4814765ce",
  "transcript_path": "/path/to/transcript",
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

Notice the shape changes completely between hook types, and `tool_input` contents vary by which tool fired. Always inspect the log files or use the debugging pattern below before writing hook logic.

---

### Debugging Hooks

Use a catch-all logging hook to capture the raw payload before writing any real logic:

```json
"PostToolUse": [
  {
    "matcher": "*",
    "hooks": [
      { "type": "command", "command": "jq . > post-log.json" }
    ]
  }
]
```

Trigger Claude Code to use any tool, then open `post-log.json` to see exactly what your hook command would have received. Change `PostToolUse` to `PreToolUse`, `Stop`, `Notification`, etc. to inspect those payloads. This project already has these hooks wired up — the log files will appear after your first Claude Code interaction.

---

### Settings Configuration

After running `npm run setup` you will see two files in `.claude/`:

- **`settings.example.json`** — checked into version control; hook script paths use a `$PWD` placeholder
- **`settings.local.json`** — generated on your machine; `$PWD` is replaced with the real absolute path

**Why absolute paths?**

The Claude Code documentation recommends absolute paths for hook scripts to prevent path interception and binary planting attacks. The problem is that absolute paths are machine-specific, making a shared `settings.json` impossible.

**How `init-claude.js` solves it:**

`settings.example.json` uses `$PWD` as a stand-in:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [{ "type": "command", "command": "node $PWD/hooks/query_hook.js" }]
      }
    ]
  }
}
```

When you run `npm run setup`, `scripts/init-claude.js`:

1. Reads `settings.example.json`
2. Replaces every `$PWD` with `process.cwd()` (your machine's absolute path)
3. Validates the result is valid JSON
4. Writes it to `.claude/settings.local.json`

Everyone on the team runs `npm run setup` once; everyone gets correct absolute paths with no manual editing.

---

## Hook Security Best Practices

The Claude Code documentation lists the following security recommendations for hook scripts. All five apply directly to the hooks in this project.

### 1. Validate and sanitize inputs

Never trust the stdin payload blindly. Always confirm that expected fields exist and have the correct type before using them.

```js
// Bad — crashes if tool_input or file_path is missing
const filePath = hookData.tool_input.file_path;

// Good — guard before use
const filePath = hookData?.tool_input?.file_path;
if (!filePath || typeof filePath !== "string") {
  process.exit(0); // nothing actionable — allow
}
```

### 2. Always quote shell variables

In shell-based hooks, an unquoted variable is vulnerable to word splitting and glob expansion.

```bash
# Bad — breaks on file paths that contain spaces
jq . > $FILE

# Good
jq . > "$FILE"
```

The `jq` logging hooks in this project pipe stdin directly (`jq . > pre-log.json`) with no shell variable interpolation, which sidesteps this issue entirely.

### 3. Block path traversal

If your hook operates on file paths from stdin, a `..` segment in the path can escape the directory you intend to constrain.

```js
const normalizedPath = path.resolve(filePath);
const allowedDir   = path.resolve(process.cwd(), "src/queries");

if (!normalizedPath.startsWith(allowedDir + path.sep)) {
  process.exit(0); // outside allowed directory — skip
}
```

`hooks/query_hook.js` already contains this exact guard before forwarding any file contents to Claude for review.

### 4. Use absolute paths

Always use full absolute paths for hook script commands in `settings.json`. Relative paths or bare script names can be intercepted by a malicious binary earlier in `$PATH`.

```json
// Vulnerable — relies on $PATH resolution
{ "command": "node hooks/query_hook.js" }

// Safe — unambiguous
{ "command": "node /home/user/myproject/hooks/query_hook.js" }
```

This is the exact reason the `$PWD` placeholder system exists in this project. See [Settings Configuration](#settings-configuration) for how it works.

### 5. Skip sensitive files

Hook scripts that read or forward file contents — such as `query_hook.js`, which sends proposed code to a Claude subagent — must avoid sending secrets.

```js
const SENSITIVE = [".env", ".git/", "id_rsa", "credentials", ".pem"];

if (SENSITIVE.some(pattern => filePath.includes(pattern))) {
  process.exit(0); // skip — potentially sensitive file
}
```

Files to watch: `.env` files, anything inside `.git/`, private keys, certificate files, and any file excluded by `.gitignore`.

---

## The E-Commerce Database

The database is the realistic codebase that Claude operates on during the exercises. Understanding its structure helps you follow along when hooks fire in response to Claude editing query files.

### Schema

| Domain | Tables |
|---|---|
| Customers | `customers`, `addresses`, `customer_segments`, `customer_activity_log` |
| Products | `products`, `categories` |
| Inventory | `inventory`, `warehouses` |
| Orders | `orders`, `order_items` |
| Other | `reviews`, `promotions` |

See `src/schema.ts` for full table definitions.

### Query Modules

All query functions live in `src/queries/` and follow a consistent pattern — they accept a `Database` instance as the first argument and return a `Promise`.

**`customer_queries.ts`**

| Function | Description |
|---|---|
| `getCustomerByEmail(db, email)` | Customer with default shipping address and last order date |
| `fetchActiveCustomers(db, daysInactive?)` | Customers who ordered within N days (default 90) |
| `findCustomersBySegment(db, segmentName)` | Customers in a segment with lifetime value |
| `getCustomerProfile(db, customerId)` | Full profile: addresses, order count, last 5 products |
| `searchCustomersByName(db, firstName?, lastName?)` | Partial-match search by name |
| `listCustomersWithReviews(db)` | Customers who have left at least one review |

**`analytics_queries.ts`**

| Function | Description |
|---|---|
| `calculateCustomerLifetimeValue(db, customerId)` | Total spend, order count, avg order value, preferred categories |
| `getSalesByCategory(db, startDate, endDate)` | Sales per category with top product and segment breakdown |
| `findRepeatCustomers(db, minOrders?)` | Customers with N+ orders and avg days between orders |
| `getProductPerformance(db, productId)` | Sales, review metrics, inventory turnover, segment data |
| `calculateSegmentMetrics(db, segmentName)` | Revenue, orders/customer, top products, preferred states |
| `findTrendingProducts(db, days?)` | Products with accelerating sales over the last N days |

Additional modules: `product_queries.ts`, `order_queries.ts`, `inventory_queries.ts`, `promotion_queries.ts`, `review_queries.ts`, `shipping_queries.ts`.

### Usage Example

```typescript
import { open } from "sqlite";
import sqlite3 from "sqlite3";
import { getCustomerByEmail } from "./src/queries/customer_queries";

const db = await open({ filename: "ecommerce.db", driver: sqlite3.Database });
const customer = await getCustomerByEmail(db, "user@example.com");
```

---

## Claude Agent SDK

The Claude Agent SDK lets you **run Claude Code programmatically** — from code or from the command line — rather than only through the interactive terminal UI.

### Key Properties

| Property | Detail |
|---|---|
| **Same Claude Code** | Not a trimmed-down model — you get the full Claude Code experience with all built-in tools |
| **Inherits settings** | A subagent launched inside a directory automatically inherits that directory's `.claude/settings.local.json`, including all configured hooks |
| **Read-only by default** | The SDK instance has read-only file system permissions unless explicitly granted more — a useful safety boundary when using Claude as a reviewer |
| **Useful in pipelines** | Designed for embedding Claude as a step in larger automations: CI pipelines, hook scripts, build tools, quality gates |

### Available Interfaces

The SDK is available in three forms.

> **Package rename:** Older course material and documentation may reference `@anthropic-ai/claude-code`. That package has been **renamed to `@anthropic-ai/claude-agent-sdk`**. This project uses the new name. The `sdk.ts` file includes a comment noting this change.

**CLI — non-interactive prompt mode:**

```bash
claude -p "Look for duplicate queries"
```

The `-p` flag sends a single prompt and exits without opening an interactive session, making it suitable for scripts and automation.

**TypeScript:**

```typescript
import { query, SDKMessage } from "@anthropic-ai/claude-agent-sdk";

const prompt = "Look for duplicate queries";

for await (const message of query({ prompt })) {
  console.log(message);
}
```

**Python:**

```python
import anyio
from claude_code_sdk import query

async def main():
    prompt = "Look for duplicate queries"
    async for message in query(prompt=prompt):
        print(message)

anyio.run(main)
```

### How This Project Uses It

Both `hooks/query_hook.js` and `sdk.ts` use this pattern. The hook spawns a second Claude instance scoped to `src/queries/`, passes it a review prompt, then reads the `result` message to decide whether to block or allow the write. Because the subagent runs with **read-only access by default**, it can inspect files without any risk of unintended modifications.

Run the standalone example:

```bash
npm run sdk
```

---

## Scripts

| Command | What it does |
|---|---|
| `npm run setup` | Installs dependencies and generates `.claude/settings.local.json` |
| `npm run sdk` | Runs `sdk.ts` via tsx — demonstrates the Claude Agent SDK |
