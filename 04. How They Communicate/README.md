# Day 04 — How They Communicate: JSON-RPC 2.0, stdio, SSE & Streamable HTTP

You now know who the players are (Host, Client, Server) and what they expose (Tools, Resources, Prompts). Today's question: **how do they actually talk to each other?**

There are two distinct layers to this:
1. **The message format** — what does a message look like? (JSON-RPC 2.0)
2. **The transport** — how does the message travel between processes? (stdio / SSE / Streamable HTTP)

Understand both layers and MCP becomes completely transparent to you.

---

## The Two Layers of MCP Communication

```
┌──────────────────────────────────────────────────────────────────┐
│                    MCP COMMUNICATION STACK                       │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              LAYER 2: TRANSPORT                            │  │
│  │   HOW messages move between processes                      │  │
│  │                                                            │  │
│  │   stdio              SSE              Streamable HTTP      │  │
│  │   (local pipes)      (HTTP/events)    (modern HTTP)        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              ▲                                   │
│                              │  carries                          │
│                              ▼                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              LAYER 1: MESSAGE FORMAT                       │  │
│  │   WHAT messages look like                                  │  │
│  │                                                            │  │
│  │              JSON-RPC 2.0                                  │  │
│  │   (a simple JSON standard for requests & responses)        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Think of it like email:
- **JSON-RPC** is the format of the email (subject, body, headers must follow certain rules)
- **Transport** is how the email travels (SMTP, HTTP, a USB stick)

---

## Layer 1: JSON-RPC 2.0 — The Message Format

### What Is JSON-RPC?

**JSON-RPC 2.0** is a lightweight protocol for calling functions over a network. "RPC" stands for **Remote Procedure Call** — a way to call a function on a different machine (or process) as if it were a local function call.

MCP uses JSON-RPC 2.0 as its message envelope. Every single message that flows between a Client and a Server is a JSON-RPC message.

> **This is a common pattern you'll see everywhere.** JSON-RPC is used by Ethereum nodes (yes, blockchain), the Language Server Protocol (how VS Code understands your code), and many internal APIs. Learning it here gives you a transferable skill.

---

### The Three Types of Messages

JSON-RPC 2.0 defines exactly three message types:

```
┌──────────────────────────────────────────────────────────────────┐
│                   JSON-RPC 2.0 MESSAGE TYPES                     │
│                                                                  │
│  1. REQUEST     → "Do this and tell me the result"               │
│                   Has an id — expects a Response back            │
│                                                                  │
│  2. RESPONSE    → "Here is the result" (or "here is the error")  │
│                   Always paired with a Request by matching id    │
│                                                                  │
│  3. NOTIFICATION → "FYI, this happened" (fire and forget)        │
│                    No id — no Response expected                  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

### Type 1: Request

A Request asks the other side to do something and expects a result back.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/Users/you/app.py"
    }
  }
}
```

Breaking it down field by field:

| Field | Required? | What it means |
|-------|-----------|---------------|
| `"jsonrpc": "2.0"` | Yes | Declares the protocol version — always this exact string |
| `"id"` | Yes | A unique number (or string) for this request — the Response will include the same id so you can match them up |
| `"method"` | Yes | The function being called — like an endpoint name |
| `"params"` | No | Arguments for the method — structured as JSON |

The `id` is critical. When you have multiple requests in flight at the same time, the `id` is how both sides know which response belongs to which request.

---

### Type 2: Response

Every Request gets exactly one Response. The Response carries back either a successful result or an error.

**Successful response:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "print('hello world')\n"
      }
    ]
  }
}
```

**Error response:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": "Parameter 'path' is required but was not provided"
  }
}
```

Notice: a Response has either `result` **or** `error` — never both.

**Standard error codes:**

| Code | Name | When it happens |
|------|------|-----------------|
| `-32700` | Parse error | The message was not valid JSON |
| `-32600` | Invalid request | Missing required field (e.g., no `jsonrpc`) |
| `-32601` | Method not found | The method name doesn't exist on this server |
| `-32602` | Invalid params | Arguments are wrong type or missing required ones |
| `-32603` | Internal error | Something went wrong server-side |

---

### Type 3: Notification

A Notification is a fire-and-forget message. It doesn't have an `id`, and no Response is expected.

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

MCP uses Notifications for lifecycle events — things like "I'm ready" or "something changed" that don't need a reply.

```
┌──────────────────────────────────────────────────────────────┐
│               NOTIFICATIONS IN THE MCP LIFECYCLE             │
│                                                              │
│  Client → Server: notifications/initialized                  │
│    "I received your capabilities. I'm ready to go."          │
│    (No response needed — server just notes it)               │
│                                                              │
│  Server → Client: notifications/tools/list_changed          │
│    "My tool list has changed — please re-fetch it."          │
│    (No response needed — client handles it on its own)       │
│                                                              │
│  Server → Client: notifications/resources/updated           │
│    "A resource you're subscribed to has new data."           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### The Complete MCP Initialization Exchange (JSON-RPC in Action)

Here is the full message sequence that happens every time a Client connects to a Server — raw JSON-RPC:

```
CLIENT → SERVER

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": { "listChanged": true },
      "sampling": {}
    },
    "clientInfo": {
      "name": "claude-code",
      "version": "1.0.0"
    }
  }
}

─────────────────────────────────────────────────────────────

SERVER → CLIENT

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": { "subscribe": true },
      "prompts": { "listChanged": true }
    },
    "serverInfo": {
      "name": "filesystem-server",
      "version": "2.1.0"
    }
  }
}

─────────────────────────────────────────────────────────────

CLIENT → SERVER (Notification — no id, no response expected)

{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

This three-message handshake happens on every new connection. After this, the Client asks for `tools/list`, `resources/list`, and `prompts/list` and the server is ready to serve.

---

### A Full Tool Call in JSON-RPC

Let's trace a complete tool call — from user input to AI response — at the raw message level:

```
USER: "How many lines in /src/app.py?"

─── STEP 1: AI decides to call read_file ──────────────────────

CLIENT → SERVER
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/src/app.py"
    }
  }
}

─── STEP 2: Server reads the file and responds ────────────────

SERVER → CLIENT
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "from flask import Flask\n\napp = Flask(__name__)\n\n@app.route('/')\ndef home():\n    return 'Hello'\n"
      }
    ],
    "isError": false
  }
}

─── STEP 3: AI reads the result and responds to user ──────────

AI: "The file /src/app.py has 8 lines."
```

The `id: 42` ties the response to the request. If five other tool calls were happening in parallel, each has its own `id` so the responses don't get mixed up.

---

## Layer 2: Transport — How Messages Travel

The transport is the mechanism that carries JSON-RPC messages between the Client and Server processes. MCP supports three transports, each designed for different deployment scenarios.

```
┌──────────────────────────────────────────────────────────────────────┐
│                        THREE TRANSPORTS                              │
│                                                                      │
│  stdio              SSE                    Streamable HTTP           │
│  (local only)       (remote, legacy)       (remote, modern)          │
│                                                                      │
│  Process A ──▶      Client ──▶ GET         Client ──▶ POST           │
│  stdin/stdout       Server ──▶ SSE events  Server ──▶ HTTP response  │
│  ◀── Process B      Client ──▶ POST        (bidirectional on demand) │
│                                                                      │
│  No network         Needs HTTP server      Needs HTTP server         │
│  Zero setup         2 connections          1 or 2 connections        │
│  Most common        Being phased out       The new standard          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Transport 1: stdio — Standard Input/Output

### What Is stdio?

`stdio` (standard input/output) is the simplest transport. The Client launches the MCP Server as a **child process** and communicates with it through the process's standard streams:

- `stdin` (standard input) — the Client writes messages **into** the Server
- `stdout` (standard output) — the Server writes messages **back** to the Client

```
┌─────────────────────────────────────────────────────────────────┐
│                      stdio TRANSPORT                            │
│                                                                 │
│   HOST PROCESS (Claude Desktop)                                 │
│   ┌──────────────────────────────────┐                          │
│   │   MCP CLIENT                     │                          │
│   │                                  │                          │
│   │   writes JSON-RPC ──────────────▶│ stdin  ┌──────────────┐  │
│   │                                  │        │  MCP SERVER  │  │
│   │   reads JSON-RPC  ◀──────────────│ stdout │  (child      │  │
│   │                                  │        │   process)   │  │
│   │                                  │ stderr │              │  │
│   │   (logs only, not messages) ◀────│        └──────────────┘  │
│   └──────────────────────────────────┘                          │
│                                                                 │
│   Each message is one line of JSON, terminated by newline \n   │
└─────────────────────────────────────────────────────────────────┘
```

### How stdio Works Step by Step

```
1. Host reads config:
   {
     "mcpServers": {
       "filesystem": {
         "command": "npx",
         "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you"]
       }
     }
   }

2. Host spawns child process:
   npx -y @modelcontextprotocol/server-filesystem /Users/you

3. Client connects to that process's stdin/stdout

4. Messages flow as newline-delimited JSON:

   Client  ──────▶  stdin  ──────▶  Server
   {"jsonrpc":"2.0","id":1,"method":"initialize",...}\n

   Server  ──────▶  stdout  ─────▶  Client
   {"jsonrpc":"2.0","id":1,"result":{...}}\n

5. stderr is reserved for logs/debug output — never for protocol messages
```

### When to Use stdio

```
✅  Server runs on the SAME machine as the Host (local setup)
✅  You're configuring Claude Desktop, Claude Code, Cursor, VS Code
✅  You want the simplest possible setup — no web server needed
✅  You're building your first MCP server — start here
✅  The server should start and stop with the Host app

❌  The server runs on a remote machine or cloud
❌  Multiple hosts need to share the same server
❌  The server needs to serve browser-based clients
```

### Why stdio Is the Default

stdio is the most common MCP transport in practice because most MCP usage today is local:

```
Your machine
├── Claude Desktop ──── (Client) ──── filesystem server (stdio)
│                  ──── (Client) ──── github server (stdio)
│                  ──── (Client) ──── postgres server (stdio)
└── All servers live on the same machine — stdio is perfect
```

No ports to open. No authentication. No HTTP server to run. Just a command to launch.

---

## Transport 2: SSE — Server-Sent Events

### What Is SSE?

**Server-Sent Events** (SSE) is a web standard that lets a server push a continuous stream of events to a browser or client over HTTP. It's a one-way channel: Server → Client only.

Because MCP needs bidirectional communication (Client sends requests, Server sends responses), the SSE transport uses **two separate HTTP connections**:

```
┌──────────────────────────────────────────────────────────────────┐
│                         SSE TRANSPORT                            │
│                                                                  │
│  MCP CLIENT                            MCP SERVER (HTTP)         │
│                                                                  │
│  Connection 1: GET /sse  ────────────────────────────────────▶   │
│               ◀─────────────── SSE event stream (always open)    │
│               ◀─────────────── Server pushes responses here      │
│                                                                  │
│  Connection 2: POST /message  ──────────────────────────────▶   │
│               Client sends requests here (one POST per message)  │
│                                                                  │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                                  │
│  Client: "Hey server, I'm ready to receive events"              │
│          (opens persistent GET /sse connection)                  │
│                                                                  │
│  Client: POST /message → {"method":"initialize",...}            │
│  Server: event: message                                          │
│          data: {"id":1,"result":{...}}                           │
│          (pushed back through the open SSE stream)               │
└──────────────────────────────────────────────────────────────────┘
```

### What an SSE Event Looks Like

SSE has its own text format. A single event over the SSE stream looks like this:

```
event: message
data: {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05",...}}

```

Two key parts:
- `event:` — the event type (MCP always uses `message`)
- `data:` — the JSON-RPC message payload
- A blank line terminates the event

### When to Use SSE

```
✅  Server runs on a REMOTE machine (different from the Host)
✅  Multiple clients need to share one server
✅  The server is a web service (REST API + MCP support)
✅  You're serving browser-based MCP clients
✅  You need the server to run continuously (not spawned per session)

❌  New projects — prefer Streamable HTTP instead (SSE is legacy)
❌  You need to stream large responses efficiently
```

> **Important:** SSE was the original remote transport in MCP, but it has a fundamental limitation — it requires two persistent connections and can't efficiently stream large responses back to the client. The MCP spec has moved on to **Streamable HTTP** as the modern standard. If you're building a remote server today, use Streamable HTTP.

---

## Transport 3: Streamable HTTP — The Modern Standard

### What Is Streamable HTTP?

Streamable HTTP is MCP's newest transport (introduced mid-2025). It solves the limitations of SSE by using a **single HTTP connection** that can optionally upgrade to a stream when needed.

The core idea: use plain HTTP POST for most requests, but if the server needs to stream a large response or push multiple events, it can switch that same connection to SSE for the duration of that response.

```
┌─────────────────────────────────────────────────────────────────┐
│                    STREAMABLE HTTP TRANSPORT                     │
│                                                                  │
│  MCP CLIENT                         MCP SERVER (HTTP)           │
│                                                                  │
│  ── SIMPLE REQUEST/RESPONSE ────────────────────────────────    │
│                                                                  │
│  POST /mcp ──────────────────────────────────────────────────▶  │
│  body: {"jsonrpc":"2.0","id":1,"method":"tools/list"}           │
│                                                                  │
│  ◀─────────────────────────────────────────── 200 OK           │
│  body: {"jsonrpc":"2.0","id":1,"result":{...}}                  │
│                                                                  │
│  ── STREAMING RESPONSE (when needed) ───────────────────────    │
│                                                                  │
│  POST /mcp ──────────────────────────────────────────────────▶  │
│  body: {"method":"tools/call","params":{"name":"long_task"}}    │
│                                                                  │
│  ◀─────────────────────────────────── 200 OK (Content-Type:    │
│                                          text/event-stream)     │
│  ◀─────────── event: message                                    │
│               data: {"progress": "25%"}                         │
│  ◀─────────── event: message                                    │
│               data: {"progress": "50%"}                         │
│  ◀─────────── event: message                                    │
│               data: {"result": "done"}                          │
│                                                                  │
│  ONE endpoint. Server decides: plain response or stream.        │
└─────────────────────────────────────────────────────────────────┘
```

### Streamable HTTP vs SSE — What Changed

| | SSE Transport | Streamable HTTP |
|--|--------------|-----------------|
| **Connections needed** | 2 (persistent GET + POST per message) | 1 (single POST, upgrades to stream if needed) |
| **Server complexity** | Must manage persistent SSE connection | Simpler — standard HTTP endpoint |
| **Streaming support** | Server → Client only | Bidirectional streaming possible |
| **Stateless support** | No — always stateful | Yes — can work statelessly |
| **Status in MCP spec** | Legacy (still supported) | Current standard |
| **Build new servers with?** | No | Yes |

### Session Management in Streamable HTTP

Since Streamable HTTP can be stateless (each HTTP request is independent), MCP uses a **session ID** to link multiple requests into one logical session:

```
Client → Server: POST /mcp
  Headers: { "Mcp-Session-Id": "session-abc123" }
  Body:    { "method": "initialize", ... }

Server → Client: 200 OK
  Headers: { "Mcp-Session-Id": "session-abc123" }  ← same session
  Body:    { "result": { ... } }

All subsequent requests include the same Mcp-Session-Id.
The server uses it to look up the session state.
```

### When to Use Streamable HTTP

```
✅  Building a NEW remote MCP server — use this
✅  Server runs in the cloud (deployed to Railway, Render, AWS, etc.)
✅  You need efficient streaming for long-running tools
✅  You want a simple, standard HTTP endpoint
✅  Multiple clients share one server

❌  Local server on the same machine as Host → use stdio instead
```

---

## Full Comparison: All Three Transports

```
┌───────────────────────────────────────────────────────────────────────┐
│                   TRANSPORT COMPARISON TABLE                          │
├────────────────────┬──────────────┬──────────────┬────────────────────┤
│                    │    stdio     │     SSE      │  Streamable HTTP   │
├────────────────────┼──────────────┼──────────────┼────────────────────┤
│ Server location    │ Same machine │ Remote       │ Remote             │
│ Setup complexity   │ None         │ Medium       │ Low–Medium         │
│ Needs HTTP server  │ No           │ Yes          │ Yes                │
│ Connections        │ 1 (pipes)    │ 2 persistent │ 1 per request      │
│ Streaming          │ No           │ Server→Client│ Bidirectional      │
│ Can be stateless   │ No           │ No           │ Yes                │
│ MCP spec status    │ Current      │ Legacy       │ Current            │
│ Most common use    │ Local tools  │ Old remote   │ New remote APIs    │
│ Typical config     │ CLI command  │ URL endpoint │ URL endpoint       │
│ Who starts server  │ Host spawns  │ Always runs  │ Always runs        │
│ Auth support       │ Implicit     │ HTTP headers │ HTTP headers/OAuth │
├────────────────────┼──────────────┼──────────────┼────────────────────┤
│ USE WHEN           │ Local setup  │ Legacy only  │ New remote server  │
└────────────────────┴──────────────┴──────────────┴────────────────────┘
```

---

## What a Complete Communication Flow Looks Like

Putting both layers together — here is every message that flows when you ask Claude Desktop (using the filesystem server via stdio) to read a file:

```
YOU TYPE: "Read my README.md file"

═══ INITIALIZATION (already done at startup) ══════════════════

Client → Server (via stdin):
  {"jsonrpc":"2.0","id":1,"method":"initialize","params":{...}}

Server → Client (via stdout):
  {"jsonrpc":"2.0","id":1,"result":{"capabilities":{...}}}

Client → Server (Notification):
  {"jsonrpc":"2.0","method":"notifications/initialized"}

Client → Server:
  {"jsonrpc":"2.0","id":2,"method":"tools/list"}

Server → Client:
  {"jsonrpc":"2.0","id":2,"result":{"tools":[
    {"name":"read_file","description":"...","inputSchema":{...}},
    {"name":"write_file","description":"...","inputSchema":{...}},
    ...
  ]}}

═══ YOUR QUESTION (live request) ══════════════════════════════

HOST sends user message to AI model.

AI thinks: "read the README.md — I need read_file"

Client → Server:
  {
    "jsonrpc": "2.0",
    "id": 47,
    "method": "tools/call",
    "params": {
      "name": "read_file",
      "arguments": { "path": "/Users/you/README.md" }
    }
  }

Server → Client:
  {
    "jsonrpc": "2.0",
    "id": 47,
    "result": {
      "content": [{"type":"text","text":"# My Project\n\n..."}],
      "isError": false
    }
  }

AI reads the file content and forms a response.

HOST shows you: "Here's your README.md: ..."
```

Every MCP interaction in the world follows this same structure. The primitives change (tools, resources, prompts), the transport changes (stdio, HTTP), but the JSON-RPC envelope stays the same.

---

## MCP Message Methods — Reference

Here are all the JSON-RPC `method` names you'll encounter:

```
LIFECYCLE
  initialize                          Handshake — Client → Server
  notifications/initialized           "Ready" signal — Client → Server

CAPABILITY DISCOVERY  
  tools/list                          Get all tools the server exposes
  resources/list                      Get all resources
  prompts/list                        Get all prompts

EXECUTION
  tools/call                          Execute a tool
  resources/read                      Fetch a resource's data
  prompts/get                         Retrieve a prompt with arguments

CHANGE NOTIFICATIONS (Server → Client)
  notifications/tools/list_changed    Tool list has changed, re-fetch
  notifications/resources/list_changed
  notifications/resources/updated     A subscribed resource has new data
  notifications/prompts/list_changed
```

---

## Common Confusions — Clear These Up Now

**"JSON-RPC vs REST — what's the difference?"**

REST uses different HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) and URLs (`/users/42/posts`) to express different operations. JSON-RPC always uses `POST` to one endpoint and puts the operation name in the JSON body (`"method": "tools/call"`). REST is resource-oriented; JSON-RPC is function-call-oriented.

**"Why does SSE use two connections?"**

SSE is server-to-client only — HTTP was designed for client-initiated requests. To send data *from* the client (MCP requests) while also receiving a continuous stream (server responses), SSE needs a separate POST endpoint for client messages. Streamable HTTP avoids this by letting a single POST response upgrade to a stream.

**"Can I mix transports? Like stdio for some servers and HTTP for others?"**

Yes — and this is common. Claude Desktop might connect to a local filesystem server via stdio and a remote web-search server via Streamable HTTP simultaneously. The Host manages both connections through separate Clients.

**"Is JSON-RPC the same as REST API?"**

No. They're both JSON-over-HTTP (for remote transport) but REST organizes actions around resources and URLs, while JSON-RPC organizes actions around named methods. MCP uses JSON-RPC because MCP is fundamentally about calling functions (tools), not manipulating resources.

**"What happens if a message is lost in stdio?"**

stdio is a local pipe between two processes on the same machine — it's reliable. Packets don't get dropped the way they can over a network. If the server crashes, the pipe closes and the Client gets an error immediately.

**"Does the `id` have to be a number?"**

No — JSON-RPC 2.0 says the `id` can be a string, number, or `null`. MCP implementations typically use incrementing integers for simplicity, but the spec allows any of these forms.

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│              LAYER 1: JSON-RPC 2.0 MESSAGES                      │
│                                                                  │
│  REQUEST       id + method + params → expects Response           │
│  RESPONSE      id + result (or error) → matches a Request        │
│  NOTIFICATION  method only → no id, no response expected         │
│                                                                  │
│  All messages: {"jsonrpc":"2.0", ...}                            │
├──────────────────────────────────────────────────────────────────┤
│              LAYER 2: TRANSPORT                                   │
│                                                                  │
│  stdio          Local. Child process. stdin/stdout pipes.        │
│                 Use for: any local server (the default)          │
│                                                                  │
│  SSE            Remote. 2 HTTP connections. Legacy.              │
│                 Use for: existing servers only                   │
│                                                                  │
│  Streamable HTTP  Remote. 1 HTTP endpoint. Modern standard.      │
│                   Use for: all new remote servers                │
├──────────────────────────────────────────────────────────────────┤
│              KEY METHOD NAMES                                     │
│                                                                  │
│  initialize               First message, always                  │
│  notifications/initialized  "I'm ready" acknowledgment           │
│  tools/list               Discover what tools exist              │
│  tools/call               Execute a tool                         │
│  resources/read           Read a resource's data                 │
│  prompts/get              Get a prompt template                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaway

> MCP communication has two layers: **what** (JSON-RPC 2.0) and **how** (transport).
>
> **JSON-RPC** is a dead-simple standard: every message is either a Request (wants a Response), a Response (answers a Request), or a Notification (fire and forget). All MCP messages — from the initial handshake to every tool call — follow this format.
>
> **The transport** decides how messages physically travel: `stdio` for local (most common today), `SSE` for remote (legacy), and **Streamable HTTP** for remote (the modern standard going forward).
>
> Understanding these two layers means you can read any MCP log, debug any connection issue, and implement either a client or a server confidently — because you know exactly what messages should be flowing and through what channel.

---

[← Day 03: Tools, Resources & Prompts](../03.%20Tools%20Resources%20Prompts/) | [Next → Day 05: Real MCP Server In Action](../05.%20Real%20MCP%20Server%20In%20Action/)
