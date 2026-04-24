# Day 02 — Host, Client, Server Roles

MCP has three roles that every setup — no matter how simple or complex — always follows. Get this mental model right and everything else in MCP will click instantly.

---

## The Three Roles at a Glance

```
┌──────────────────────────────────────────────────────────────┐
│                          HOST                                │
│   (the app you open: Claude Desktop, Claude Code, Cursor)    │
│                                                              │
│   ┌──────────────────┐        ┌──────────────────┐          │
│   │     CLIENT 1     │        │     CLIENT 2     │   ...    │
│   │  (inside host)   │        │  (inside host)   │          │
│   └────────┬─────────┘        └────────┬─────────┘          │
└────────────┼──────────────────────────┼────────────────────┘
             │  MCP Protocol             │  MCP Protocol
             │                           │
             ▼                           ▼
    ┌─────────────────┐        ┌─────────────────┐
    │    SERVER A     │        │    SERVER B     │
    │  (GitHub MCP)   │        │  (Filesystem)   │
    └────────┬────────┘        └────────┬────────┘
             │                          │
             ▼                          ▼
        GitHub API               Your local files
```

One rule governs all of this:

> **1 Host → many Clients. 1 Client → exactly 1 Server.**

---

## Role 1: Host — The App You Open

The **Host** is the program you launch and interact with. It's the face of the whole system — you type into it, it shows you responses. But behind the scenes, it's also the manager that decides which servers to connect to and enforces permissions.

**Examples of real Hosts:**
- Claude Desktop
- Claude Code (the CLI you're using right now)
- Cursor IDE
- VS Code with MCP extension
- Any custom app you build with the MCP SDK

**What the Host is responsible for:**

```
┌──────────────────────────────────────────────────────┐
│                  HOST RESPONSIBILITIES                │
│                                                      │
│  1. Run (or connect to) the AI model                 │
│  2. Read the config file — which servers to use?     │
│  3. Spawn a Client for each configured server        │
│  4. Show the AI's responses to the user              │
│  5. PERMISSION GATE — approve or deny tool calls     │
│     before they reach the server                     │
└──────────────────────────────────────────────────────┘
```

**Point 5 is critical.** The Host is the security layer. Before any tool executes, the Host checks:
- Is this allowed automatically?
- Does the user need to approve this?
- Should this be blocked entirely?

> **Analogy:** Think of the Host as an **operating system**. It doesn't do the work itself, but it manages resources, controls permissions, and coordinates everything happening inside it.

---

## Role 2: Client — The Translator Inside the Host

The **Client** is a small piece of code that lives *inside* the Host. You never interact with it directly — it's invisible infrastructure. Its only job is to be a well-behaved messenger between the Host (and the AI) and one specific Server.

**One Client = One Server. Always.**

```
┌──────────────────────────────────────────────────────────┐
│                HOST (Claude Desktop)                     │
│                                                          │
│   ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│   │   CLIENT 1   │   │   CLIENT 2   │   │  CLIENT 3  │  │
│   │              │   │              │   │            │  │
│   │ talks to     │   │ talks to     │   │ talks to   │  │
│   │ GitHub only  │   │ Filesystem   │   │ Slack only │  │
│   │              │   │ only         │   │            │  │
│   └──────┬───────┘   └──────┬───────┘   └─────┬──────┘  │
└──────────┼────────────────────┼────────────────┼─────────┘
           │                    │                │
           ▼                    ▼                ▼
       GitHub               Filesystem         Slack
       Server               Server             Server
```

**What the Client does:**
- Opens and maintains a connection to its assigned Server
- Sends the MCP handshake on startup ("who are you, what can you do?")
- Asks the server: tools? resources? prompts?
- Forwards tool-call requests from the AI to the Server
- Returns results back to the Host

**Why 1 Client per Server?**
It keeps connections clean and isolated. If the GitHub server crashes, only that Client is affected — the Filesystem and Slack connections keep working fine.

> **Analogy:** The Client is like a **browser tab**. Each tab is connected to one website. Closing or crashing one tab doesn't affect the others. They all live inside the same browser (Host), but they're independent.

---

## Role 3: Server — The Skill Provider

The **Server** is a separate program that exposes capabilities to the AI. This is where real work happens — reading files, querying databases, calling external APIs, sending messages.

```
┌──────────────────────────────────────────────────────────┐
│                     MCP SERVER                           │
│                                                          │
│  A standalone program that exposes:                      │
│                                                          │
│    TOOLS      → actions the AI can execute               │
│    RESOURCES  → data the AI can read                     │
│    PROMPTS    → reusable prompt templates                │
│                                                          │
│  The server has NO intelligence of its own.              │
│  It just provides capabilities and executes requests.    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Examples of real Servers:**

| Server | What it provides |
|--------|-----------------|
| `server-filesystem` | Read, write, search local files |
| `server-github` | Create issues, read PRs, manage repos |
| `server-postgres` | Run SQL queries against a database |
| `server-slack` | Send messages, read channel history |
| `server-brave-search` | Search the web |
| `server-puppeteer` | Control a browser, take screenshots |
| Your own custom server | Whatever you build |

**Servers are focused, not intelligent.** They don't decide anything. The AI decides to call a tool, the Client carries the request, the Server executes it and returns a result. That's it.

> **Analogy:** The Server is like a **kitchen in a restaurant**. It has the equipment and the recipes (tools). But it doesn't decide what to cook — the waiter (Client) brings the order from the customer (you, via the AI).

---

## How a Real Request Flows — Step by Step

You open Claude Desktop and type: **"What files are in my Downloads folder?"**

```
 YOU                 HOST                  CLIENT            SERVER
  │                   │                     │                  │
  │── "What files ───▶│                     │                  │
  │   in Downloads?"  │                     │                  │
  │                   │                     │                  │
  │                   │── AI thinks: ───────│                  │
  │                   │   "list_directory   │                  │
  │                   │    tool needed"     │                  │
  │                   │                     │                  │
  │                   │── Is this ──────────│                  │
  │                   │   allowed? (yes) ──▶│                  │
  │                   │                     │                  │
  │                   │                     │── tools/call ───▶│
  │                   │                     │   list_directory │
  │                   │                     │   path=~/Downloads
  │                   │                     │                  │
  │                   │                     │                  │── reads
  │                   │                     │                  │   folder
  │                   │                     │                  │
  │                   │                     │◀── result: ──────│
  │                   │                     │    [file1, file2] │
  │                   │                     │                  │
  │                   │◀── result ──────────│                  │
  │                   │                     │                  │
  │                   │── AI formats ───────│                  │
  │                   │   the response      │                  │
  │                   │                     │                  │
  │◀── "You have 8 ───│                     │                  │
  │    files in        │                     │                  │
  │    Downloads..."  │                     │                  │
```

Every MCP interaction — no matter how complex — is this same loop. The Host manages it, the Client carries messages, the Server executes.

---

## The Lifecycle of an MCP Connection

What happens from the moment you start Claude Desktop to when you close it:

```
STARTUP
  │
  ├── Host reads config file
  │     → "I need to connect to: filesystem, github, slack"
  │
  ├── Host spawns 3 Clients (one per server)
  │
  ├── Each Client connects to its Server
  │     → sends: initialize request
  │     ← receives: server name, version, capabilities
  │     → sends: initialized notification ("got it, ready")
  │
  ├── Each Client asks its Server:
  │     → tools/list      ← [tool1, tool2, ...]
  │     → resources/list  ← [resource1, ...]
  │     → prompts/list    ← [prompt1, ...]
  │
  ├── Host gives the AI the full inventory of ALL tools
  │     from ALL connected servers
  │
  │   (YOU ARE HERE — ready to use)
  │
DURING SESSION
  ├── You ask questions
  ├── AI decides which tools to call
  ├── Host checks permissions
  ├── Client sends request to correct Server
  ├── Server executes and responds
  └── AI uses result to answer you

SHUTDOWN
  └── Host closes all Client connections gracefully
```

---

## What the AI Actually Sees

The AI (language model) never directly sees or talks to servers. What it sees is a list of available tools — assembled by the Host from all connected servers:

```
WHAT THE AI SEES:

  Available tools:
  ┌──────────────────────────────────────────────────────────┐
  │  [from filesystem server]                                │
  │  - read_file(path)                                       │
  │  - write_file(path, content)                             │
  │  - list_directory(path)                                  │
  │                                                          │
  │  [from github server]                                    │
  │  - create_issue(owner, repo, title, body)                │
  │  - list_pull_requests(owner, repo)                       │
  │  - get_file_contents(owner, repo, path)                  │
  │                                                          │
  │  [from slack server]                                     │
  │  - send_message(channel, text)                           │
  │  - list_channels()                                       │
  └──────────────────────────────────────────────────────────┘

  The AI reads the tool names + descriptions and decides
  which ones to use — it has no idea which server they
  came from. That's the Host/Client's job to figure out.
```

---

## Comparison Table

| | Host | Client | Server |
|-|------|--------|--------|
| **What it is** | The app you open | Connector inside the Host | Standalone capability program |
| **Lives where** | Your computer (visible UI) | Inside the Host (invisible) | Local or remote process |
| **How many** | 1 (the app running) | 1 per Server | As many as configured |
| **Main job** | Run AI, manage UI, enforce permissions | Speak MCP to one Server | Expose tools, resources, prompts |
| **Has intelligence?** | Contains the AI | No — just a messenger | No — just executes |
| **Analogy** | Operating system | Browser tab | Website / API service |

---

## Common Confusions — Clear These Up Now

**"Is the Client the same as the Host?"**

No. The Host *contains* the Clients. Think of the Host as a building, and Clients as rooms in the building. One building, multiple rooms — each room connected to one external service.

**"Is the Server the AI?"**

No. The AI lives in (or is called by) the Host. The Server is a dumb, focused program — it has no reasoning ability. It just executes what it's told and returns a result.

**"Who decides which tool to call?"**

The **AI model** decides — based purely on reading tool descriptions. The Host then approves it, the Client delivers it, and the Server executes it.

**"Can a Client talk to multiple Servers?"**

No. One Client = one Server. This is a hard rule in the MCP spec. If you want 5 servers, you have 5 Clients.

**"What happens if a Server crashes?"**

Only that Client is affected. The Host and other Clients keep running normally. The Host may show an error for tools from that Server, but everything else continues working.

---

## Key Takeaway

> **Host = the app you use. Client = the invisible connector inside it. Server = the skill provider outside it.**
>
> The Host holds many Clients. Each Client speaks to exactly one Server. The Server exposes tools the AI can use.
>
> The AI decides what to call. The Host approves it. The Client delivers it. The Server executes it.

Once this picture is locked in, you'll find that everything else in MCP — transports, tool call flows, building servers — is just details layered on top of this structure.

---

[← Day 01: The Problem MCP Solves](../01.%20The%20Problem%20MCP%20Solves/) | [Next → Day 03: Tools, Resources & Prompts](../03.%20Tools%20Resources%20Prompts/)
