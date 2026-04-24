# Day 03 — What Servers Expose: Tools, Resources & Prompts

An MCP server doesn't just expose a generic API. It exposes exactly **three types of things** — called **primitives**. Every MCP server in the world is built from these three building blocks.

Understanding these three primitives tells you: what any server can do, how to use it, and how to build your own.

---

## The Big Picture

```
                          MCP SERVER
                 ┌────────────────────────────┐
                 │                            │
                 │  "What can I give you?"    │
                 │                            │
        ┌────────┴────────┬──────────────┬────┴────────┐
        │                 │              │             │
        ▼                 ▼              ▼             │
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│    TOOLS     │  │  RESOURCES   │  │   PROMPTS    │  │
│              │  │              │  │              │  │
│  DO things   │  │  READ things │  │  TEMPLATE    │  │
│  (actions)   │  │  (data)      │  │  things      │  │
└──────────────┘  └──────────────┘  └──────────────┘  │
                                                       │
       Model-controlled    App-controlled   User-controlled
```

**One rule to remember forever:**

```
Need to DO something?              → Tool
Need to READ stable data?          → Resource
Need a reusable prompt pattern?    → Prompt
```

---

## Primitive 1: Tools — Actions the AI Can Execute

### What Is a Tool?

A **Tool** is a function that lives on the MCP server and that the AI is allowed to call. Tools are the most important and most used primitive in MCP — nearly every server you encounter will be primarily a collection of tools.

Think of a Tool as a **button**. Each button does one specific thing when pressed:
- `read_file` button → reads a file and returns its contents
- `send_message` button → sends a Slack message
- `run_sql_query` button → runs a database query
- `create_github_issue` button → creates an issue on GitHub

The AI never executes code directly. It decides which button to press (and what inputs to give it). The Server handles all the execution.

---

### The Three Parts of Every Tool

Every tool has exactly three required parts:

```
┌────────────────────────────────────────────────────────────────┐
│                           TOOL                                 │
│                                                                │
│   name          "read_file"                                    │
│                 ↑ What the AI calls to invoke this tool        │
│                                                                │
│   description   "Read the full contents of a file from the     │
│                  filesystem. Returns the text content."        │
│                 ↑ What the AI reads to decide WHEN to use it   │
│                                                                │
│   inputSchema   { path: string (required) }                    │
│                 ↑ What arguments this tool needs               │
└────────────────────────────────────────────────────────────────┘
```

Here is what a complete tool definition looks like in JSON:

```json
{
  "name": "create_file",
  "description": "Create a new file at the given path with the given content. Use this when you need to write a new file. Fails if the file already exists — use write_file to overwrite an existing file.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "The path where the file should be created"
      },
      "content": {
        "type": "string",
        "description": "The text content to write into the new file"
      }
    },
    "required": ["path", "content"]
  }
}
```

---

### Why the Description Is Everything

The AI **never sees the code** inside a tool. It only reads the name and description. That's the only thing it uses to decide which tool to call.

```
USER: "Show me what's in my project folder"

AI sees these tools and reads their descriptions:
─────────────────────────────────────────────────────
  read_file          "Read the contents of a single file"
  write_file         "Write content to a file, overwriting if it exists"
  create_file        "Create a new file. Fails if it already exists."
  list_directory     "List all files and folders inside a directory"
  search_files       "Find files matching a name pattern"
─────────────────────────────────────────────────────

AI thinks: "Show me what's IN the folder... that's listing, not reading."
AI picks:   list_directory ← chose based on description alone
```

> **This is the single most important thing to understand about tools.** A vague description means the AI calls the wrong tool. A clear, specific description means the AI always picks correctly.
>
> When you build your own MCP server on Day 08, writing good descriptions will be your most important task.

---

### Input Schema Deep Dive

The `inputSchema` tells the AI exactly what arguments to pass. It follows **JSON Schema** format — an industry-standard way of describing the shape of data.

```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "The SQL SELECT query to run"
    },
    "limit": {
      "type": "integer",
      "description": "Maximum rows to return. Defaults to 100.",
      "default": 100
    },
    "include_column_names": {
      "type": "boolean",
      "description": "Whether to include column headers in the output"
    },
    "format": {
      "type": "string",
      "enum": ["table", "json", "csv"],
      "description": "Output format: table (default), json, or csv"
    }
  },
  "required": ["query"]
}
```

Breaking each field down:

| JSON Schema Field | What it means |
|-------------------|---------------|
| `"type": "object"` | The input is always a JSON object for MCP tools |
| `"properties"` | Defines each accepted argument |
| `"type": "string"` | This argument must be text |
| `"type": "integer"` | This argument must be a whole number |
| `"type": "boolean"` | This argument must be `true` or `false` |
| `"type": "array"` | This argument is a list |
| `"enum": [...]` | Only these specific values are allowed |
| `"description"` | The AI reads this to understand what this argument does |
| `"required": [...]` | These arguments MUST be provided |
| `"default"` | Used if the argument isn't provided |

---

### A Complete Tool Interaction — Step by Step

```
USER: "How many users signed up in the last 7 days?"

STEP 1 — AI SEES THE TOOL LIST
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   run_sql_query(query, limit)     → "Execute a SQL SELECT query"
   list_tables()                   → "List all tables in the database"
   describe_table(table_name)      → "Show columns for a table"

STEP 2 — AI DECIDES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   "Count users from the last 7 days — that needs SQL"
   → picks run_sql_query

STEP 3 — AI CONSTRUCTS THE CALL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   run_sql_query({
     query: "SELECT COUNT(*) FROM users
             WHERE created_at > NOW() - INTERVAL '7 days'",
     limit: 1
   })

STEP 4 — HOST CHECKS PERMISSIONS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Is this a SELECT? Yes → auto-allow

STEP 5 — CLIENT SENDS TO SERVER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   MCP message: tools/call → run_sql_query

STEP 6 — SERVER RUNS THE QUERY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Connects to DB, executes, returns rows

STEP 7 — RESULT FLOWS BACK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   { "count": 342 }

STEP 8 — AI RESPONDS TO YOU
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   "342 users signed up in the last 7 days."
```

---

### Tool Response Format

Every tool returns a **content array**:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Contents of app.py:\n\nprint('hello world')\n"
    }
  ]
}
```

Content can also include images:

```json
{
  "content": [
    {
      "type": "image",
      "data": "<base64-encoded-image>",
      "mimeType": "image/png"
    }
  ]
}
```

If something goes wrong, the tool flags it with `isError`:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Error: File not found at /Users/you/missing.py"
    }
  ],
  "isError": true
}
```

The `isError: true` flag is important — it tells the AI "this failed" so it can try a different approach, rather than treating an error message as valid data.

---

### Parallel Tool Calls — AI Can Call Multiple at Once

The AI isn't limited to one tool call per response. It can call multiple tools simultaneously:

```
USER: "Read my app.py and also list my /src folder"

AI calls simultaneously:
  → read_file({ path: "app.py" })
  → list_directory({ path: "/src" })

Both requests go to the server in parallel.
Both results come back.
AI combines them into one response.
```

This is more efficient than sequential calls — especially important when tools take time to run (e.g., database queries, API calls).

---

### Real-World Tool Examples from Popular Servers

**Filesystem Server:**

| Tool | What It Does |
|------|-------------|
| `read_file` | Read contents of a single file |
| `write_file` | Write content to a file (overwrites if exists) |
| `create_directory` | Create a new folder |
| `list_directory` | List all files and folders at a path |
| `move_file` | Move or rename a file |
| `search_files` | Find files matching a name pattern |
| `get_file_info` | Get metadata: size, permissions, modified date |
| `read_multiple_files` | Read several files in a single call |

**GitHub Server:**

| Tool | What It Does |
|------|-------------|
| `create_issue` | Create a new issue in a repo |
| `list_issues` | List all issues with filters |
| `create_pull_request` | Open a new PR |
| `create_branch` | Create a new branch |
| `get_file_contents` | Read any file from a repo |
| `search_repositories` | Search GitHub for repos |
| `add_issue_comment` | Comment on an existing issue |
| `merge_pull_request` | Merge a PR |

**Slack Server:**

| Tool | What It Does |
|------|-------------|
| `send_message` | Send a message to a channel |
| `reply_to_thread` | Reply in a thread |
| `get_channel_history` | Read recent messages in a channel |
| `list_channels` | List all available channels |
| `upload_file` | Upload a file to a channel |

---

## Primitive 2: Resources — Data the AI Can Read

### What Is a Resource?

A **Resource** is a data endpoint that a server makes available for reading. Unlike tools, resources don't execute actions — they expose stable, accessible data.

```
Tools     → AI performs an action (write, send, create, delete)
Resources → AI reads data (configs, schemas, docs, logs)
```

Resources are addressed with **URIs** — identifiers that look like URLs but use the `resource://` scheme:

```
resource://config/app-settings
resource://database/schema
resource://docs/api-reference
resource://logs/latest
resource://cache/users
```

---

### Resource Structure

Every resource has these fields:

```
┌──────────────────────────────────────────────────────┐
│                     RESOURCE                         │
│                                                      │
│  uri           resource://db/schema                  │
│                ↑ unique address to identify it       │
│                                                      │
│  name          "Database Schema"                     │
│                ↑ human-readable label                │
│                                                      │
│  description   "Tables, columns, and data types      │
│                 for all tables in the database"      │
│                ↑ what this data contains             │
│                                                      │
│  mimeType      "application/json"                    │
│                ↑ what format the data comes in       │
└──────────────────────────────────────────────────────┘
```

Example resource list from a database server:

```json
[
  {
    "uri": "resource://db/schema",
    "name": "Database Schema",
    "description": "All tables, columns, and data types in the database",
    "mimeType": "application/json"
  },
  {
    "uri": "resource://db/stats",
    "name": "Database Statistics",
    "description": "Row counts, table sizes, and index information",
    "mimeType": "application/json"
  },
  {
    "uri": "resource://db/query-logs",
    "name": "Recent Query Logs",
    "description": "The last 100 queries executed, with durations",
    "mimeType": "text/plain"
  }
]
```

---

### Resource vs Tool — When to Use Which?

This is a common point of confusion. Here's the clearest decision framework:

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  The data is STATIC or STABLE (doesn't change often)?         │
│  Reading it has NO SIDE EFFECTS?                               │
│  It's a WELL-KNOWN endpoint (like a config or schema)?         │
│                              → Use a RESOURCE                  │
│                                                                │
│  The operation DOES SOMETHING (creates, deletes, queries)?     │
│  It needs ARGUMENTS to run?                                    │
│  The result DEPENDS on what you ask for?                       │
│                              → Use a TOOL                      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

Concrete examples:

| What you want | Resource or Tool? | Why |
|---------------|-------------------|-----|
| Read the database schema | Resource | Static, no args needed |
| Run a SQL query | Tool | Dynamic, needs a query arg |
| Read API documentation | Resource | Static reference data |
| Call an external API | Tool | Requires execution |
| Get the current config | Resource | Stable, no side effects |
| Send an email | Tool | Side effect (sends something) |
| Read a static log file | Resource | Read-only, stable |
| Clear a log file | Tool | Side effect (modifies something) |

---

### Who Controls Resources?

```
┌──────────────────────────────────────────────────────────────┐
│  Tools     → AI model decides when to call them              │
│              (model-controlled)                              │
│                                                              │
│  Resources → Application / developer decides what to expose  │
│              (application-controlled)                        │
│                                                              │
│  Tools are autonomous — the AI triggers them on its own.     │
│  Resources are passive — they're available if accessed,      │
│  but the AI doesn't "call" them the way it calls tools.      │
└──────────────────────────────────────────────────────────────┘
```

---

### Dynamic Resource Templates

Resources can also be **parametrized** — the URI contains a variable:

```
resource://users/{user_id}
resource://repos/{owner}/{repo}/issues
resource://products/{product_id}/reviews
```

This is called a **Resource Template**. The server defines the pattern, the client fills in the variable when accessing it. This lets one resource definition handle many different data endpoints.

Example:
```
resource://users/42        → returns data for user ID 42
resource://users/100       → returns data for user ID 100
```

---

## Primitive 3: Prompts — Reusable Prompt Templates

### What Is a Prompt?

A **Prompt** is a server-defined, reusable template that takes optional arguments and returns a fully-formed prompt message. Think of it as a **professional prompt written by the server author**, ready for you to invoke with just a few parameters.

**Without a Prompt (manual):**
```
You type every time:
"Please review the following Python code for security vulnerabilities.
For each issue found, provide the line number, explain why it's dangerous,
rate the severity as Low/Medium/High/Critical, and provide a corrected
version. Pay special attention to SQL injection, XSS, and insecure
deserialization."
```

**With a Prompt named `security-review`:**
```
You select: security-review
Arguments:  language=python
Done. The server builds the full prompt for you.
```

The server author did the prompt engineering once. You (and anyone using that server) benefit from it every time.

---

### Prompt Structure

```
┌──────────────────────────────────────────────────────────┐
│                        PROMPT                            │
│                                                          │
│  name          "security-review"                         │
│                ↑ identifier                              │
│                                                          │
│  description   "Perform a thorough security review of    │
│                 code, finding vulnerabilities and rating  │
│                 their severity"                          │
│                ↑ what this template does                 │
│                                                          │
│  arguments     [{ name: "language", required: true },    │
│                 { name: "focus", required: false }]      │
│                ↑ variables the user fills in             │
└──────────────────────────────────────────────────────────┘
```

A full prompt definition in JSON:

```json
{
  "name": "security-review",
  "description": "Review code for security vulnerabilities with severity ratings",
  "arguments": [
    {
      "name": "language",
      "description": "Programming language of the code to review",
      "required": true
    },
    {
      "name": "focus",
      "description": "Specific vulnerability types to focus on (e.g. sql-injection, xss)",
      "required": false
    }
  ]
}
```

---

### What a Prompt Returns

When you invoke a prompt with its arguments, the server builds and returns a list of ready-to-use messages:

```json
{
  "description": "Security review for Python code",
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "Please review the following Python code for security vulnerabilities.\nFocus specifically on: SQL injection\n\nFor each issue you find:\n1. State the line number\n2. Explain why it is dangerous\n3. Rate severity: Low / Medium / High / Critical\n4. Provide a corrected version\n\nCode to review:\n{code}"
      }
    }
  ]
}
```

The server constructed that entire structured prompt. You just provided `language=python` and `focus=sql-injection`.

---

### More Prompt Examples

```json
[
  {
    "name": "explain-error",
    "description": "Explain an error message in plain English and suggest how to fix it",
    "arguments": [
      { "name": "error", "description": "The full error message or stack trace", "required": true },
      { "name": "language", "description": "Programming language", "required": false }
    ]
  },
  {
    "name": "write-tests",
    "description": "Generate unit tests for a given function or class",
    "arguments": [
      { "name": "function_name", "description": "Name of the function to test", "required": true },
      { "name": "framework", "description": "Test framework: pytest, jest, mocha", "required": false }
    ]
  },
  {
    "name": "summarize-pr",
    "description": "Summarize a pull request for non-technical stakeholders",
    "arguments": [
      { "name": "pr_number", "description": "GitHub PR number", "required": true }
    ]
  }
]
```

---

### Who Controls Prompts?

```
┌──────────────────────────────────────────────────────────────┐
│  Tools     → AI model triggers them autonomously             │
│  Resources → Application exposes them for reading            │
│  Prompts   → USER explicitly selects them                    │
│              (user-controlled)                               │
│                                                              │
│  The AI does NOT automatically invoke prompts.               │
│  You choose which prompt to use and when.                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Side-by-Side Comparison — All Three Primitives

| | Tools | Resources | Prompts |
|-|-------|-----------|---------|
| **What it is** | An executable action | A readable data endpoint | A reusable prompt template |
| **Direction** | AI calls it | AI reads from it | User selects it |
| **Who triggers** | AI model | Application | User |
| **Has side effects?** | Yes (can write, send, delete) | No (read-only) | No |
| **Needs arguments?** | Usually yes | Sometimes (templates) | Sometimes |
| **Most common?** | ✅ Very common | Less common | Rarely used |
| **Protocol call** | `tools/call` | `resources/read` | `prompts/get` |
| **Example** | `send_slack_message` | `resource://config` | `code-review` |

---

## Real Scenario: All Three Primitives Working Together

Imagine a **Database MCP Server** that exposes all three:

```
DATABASE MCP SERVER
│
├── TOOLS (you want the AI to DO something)
│   ├── run_query(sql, limit)             → execute a SELECT statement
│   ├── run_migration(sql)                → run a schema migration
│   └── export_table(table_name, format)  → export table as CSV or JSON
│
├── RESOURCES (stable data for the AI to READ)
│   ├── resource://db/schema              → all tables and column types
│   ├── resource://db/stats               → row counts, sizes, index info
│   └── resource://db/query-history       → last 50 queries run
│
└── PROMPTS (templates for common tasks)
    ├── analyze-query-performance         → template for profiling slow queries
    └── generate-migration                → template for writing schema migrations
```

**A real conversation using all three:**

```
USER: "Something is making our app slow. Can you check the database?"

1. AI reads RESOURCE: resource://db/stats
   → sees one table has 50M rows with no index

2. AI reads RESOURCE: resource://db/query-history
   → spots a repeated expensive JOIN query

3. AI calls TOOL: run_query({ sql: "EXPLAIN ANALYZE SELECT ..." })
   → confirms the query is doing a full table scan

4. USER selects PROMPT: analyze-query-performance
   → server generates a structured performance analysis prompt

5. AI calls TOOL: run_migration({ sql: "CREATE INDEX ..." })
   → fixes the slow query

6. AI calls TOOL: run_query({ sql: "EXPLAIN ANALYZE ..." })
   → confirms query now uses the index
```

Resources helped the AI understand the situation. Tools let it act. The prompt gave structure to the analysis.

---

## How the Server Announces Its Primitives

When an MCP Client first connects to a Server, it asks three questions:

```
Client → Server: "What tools do you have?"      tools/list
Client → Server: "What resources do you have?"  resources/list
Client → Server: "What prompts do you have?"    prompts/list

Server responds with its full inventory for each.
```

After this exchange, the Host gives the AI the complete list of tools. The AI can now use any of them. Resources and prompts are also available on demand.

```
┌──────────────────────────────────────────────────────────┐
│  CONNECTION SEQUENCE                                     │
│                                                          │
│  1. Client connects                                      │
│  2. initialize / initialized handshake                   │
│  3. Client requests tools/list   → gets tool list        │
│  4. Client requests resources/list → gets resource list  │
│  5. Client requests prompts/list → gets prompt list      │
│  6. AI is fully briefed — ready to use the server        │
└──────────────────────────────────────────────────────────┘
```

---

## Common Confusions — Clear These Up Now

**"Do all servers need all three primitives?"**

No. Most servers only implement Tools. Resources and Prompts are optional. A server with 3 tools and no resources or prompts is perfectly valid — and is the most common type.

**"Can't I just make everything a tool?"**

Technically yes, but it's bad design. A `get_schema()` tool that just returns static data is less clear than a `resource://db/schema` resource. Use the right primitive for the job — it makes your server easier to understand and use.

**"Is a Prompt the same as a system prompt?"**

No. A system prompt is the instructions you give the AI at the start of a session (like "you are a helpful assistant"). An MCP Prompt is a server-exposed template that generates a specific message for a specific task. They're completely different things.

**"Who decides when to call a tool?"**

The **AI model** decides, based solely on reading tool descriptions. You don't call tools directly — you describe what you want, and the AI figures out which tool to use.

**"Can the AI call resources the same way it calls tools?"**

No. Tools are called via `tools/call`. Resources are fetched via `resources/read`. Different protocol methods, different behavior. Resources don't execute logic — they just return data.

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│                 THREE PRIMITIVES — QUICK CARD                │
│                                                              │
│  TOOL       Action. AI calls it. Has inputs/outputs.         │
│             Has side effects. Most common primitive.         │
│             → send, create, delete, run, write               │
│                                                              │
│  RESOURCE   Data. App exposes it. Always read-only.          │
│             Addressed by URI. No side effects.               │
│             → config, schema, docs, logs, stats              │
│                                                              │
│  PROMPT     Template. User selects it. Builds a prompt.      │
│             Takes optional arguments. Returns messages.      │
│             → code-review, explain-error, write-tests        │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Takeaway

> MCP servers expose exactly **three types of things**: **Tools** (actions), **Resources** (data), and **Prompts** (templates).
>
> **Tools** are the core of MCP — they're what lets AI actually do things in the world. Resources give AI passive access to stable data. Prompts let server authors share reusable, expert-crafted prompt patterns.
>
> When you build your own server on Day 08, you will implement `list_tools()` and `call_tool()` handlers — and now you know exactly what those handlers are expected to provide.

---

[← Day 02: Host, Client, Server Roles](../02.%20Host%20Client%20Server%20Roles/) | [Next → Day 04: How They Communicate](../04.%20How%20They%20Communicate/)
