# Day 02 — Host, Client, Server Roles

Every MCP setup has **three characters** working together. If you understand these three, the rest of MCP makes sense instantly.

---

## The 3 Roles at a Glance

```
┌──────────────────────────────────────────────────────────┐
│                        HOST                              │
│   (the app you actually use — Claude Desktop, Cursor,    │
│    Claude Code, VS Code, etc.)                           │
│                                                          │
│   ┌────────────────┐      ┌────────────────┐             │
│   │    CLIENT 1    │      │    CLIENT 2    │   ...       │
│   │ (inside host)  │      │ (inside host)  │             │
│   └───────┬────────┘      └───────┬────────┘             │
└───────────┼────────────────────────┼─────────────────────┘
            │                        │
            │  MCP protocol          │  MCP protocol
            │                        │
            ▼                        ▼
    ┌───────────────┐        ┌───────────────┐
    │   SERVER A    │        │   SERVER B    │
    │ (GitHub MCP)  │        │ (Filesystem)  │
    └───────┬───────┘        └───────┬───────┘
            │                        │
            ▼                        ▼
       GitHub API              Your local files
```

One **Host** can hold many **Clients**. Each Client talks to exactly **one Server**. This is the golden rule.

---

## 1. Host — The App You Open

The **Host** is the program you launch on your computer. It's the face of the AI — what you type into and read replies from.

**Examples of Hosts:**
- Claude Desktop
- Claude Code (CLI)
- Cursor
- VS Code (with MCP extension)
- Any custom app built with the MCP SDK

**What the Host does:**
- Runs the LLM (or talks to one over API).
- Decides **which MCP servers to connect to** (from a config file).
- Spawns a **Client** for every server it wants to talk to.
- Shows the final answer to the user.

> Think of the Host as a **web browser**. The browser itself doesn't know about Gmail or YouTube — it just loads whatever site you point it at.

---

## 2. Client — The Translator Inside the Host

The **Client** is a small piece of code that lives **inside** the Host. You never see it directly. Its only job is to be a well-behaved messenger between the Host and one specific Server.

**What the Client does:**
- Opens a connection to **one** Server.
- Speaks the MCP protocol (JSON-RPC messages).
- Asks the Server: "What tools do you have? What resources? What prompts?"
- Forwards tool-call requests from the Host to the Server.
- Returns the Server's response back to the Host.

**Rule:** one Client = one Server. If the Host wants to connect to 5 servers, it spawns 5 Clients.

> Think of a Client as a **browser tab**. Each tab is connected to one website. Opening 5 tabs = 5 independent connections, but they all live inside the same browser.

---

## 3. Server — The Skill Provider

The **Server** is a separate program that exposes **capabilities** to the AI. This is where the real work happens — reading files, calling APIs, querying databases, sending Slack messages, etc.

**What a Server exposes:**
- **Tools** — actions the AI can take (`send_email`, `run_query`, `create_issue`)
- **Resources** — data the AI can read (files, DB rows, docs)
- **Prompts** — reusable prompt templates

**Examples of real Servers:**
- `@modelcontextprotocol/server-filesystem` — read/write local files
- `@modelcontextprotocol/server-github` — GitHub issues, PRs, repos
- `@modelcontextprotocol/server-postgres` — query a Postgres DB
- Custom ones you build yourself

> Think of a Server as a **website**. It offers specific features. The browser tab (Client) doesn't care *how* those features work internally — it just uses what the site exposes.

---

## How a Real Request Flows

You open Claude Desktop and type:

> "What files are in my Downloads folder?"

Here's what happens, step by step:

```
1. YOU type the question
       │
       ▼
2. HOST (Claude Desktop) sends it to the LLM
       │
       ▼
3. LLM thinks: "I need the filesystem server for this."
       │
       ▼
4. HOST tells the CLIENT for the filesystem server:
   "Call the tool `list_directory` with path=~/Downloads"
       │
       ▼
5. CLIENT sends that request over MCP to the SERVER
       │
       ▼
6. SERVER actually reads the folder, gets file list
       │
       ▼
7. SERVER sends the list back to the CLIENT
       │
       ▼
8. CLIENT hands it to the HOST
       │
       ▼
9. HOST feeds it to the LLM
       │
       ▼
10. LLM writes a friendly answer
       │
       ▼
11. YOU see: "You have 12 files in Downloads: ..."
```

Every MCP interaction — no matter how complex — is just this loop repeating.

---

## Quick Comparison Table

| Role   | Lives where           | How many | Main job                                |
|--------|-----------------------|----------|-----------------------------------------|
| Host   | Your computer (the app you open) | 1 at a time | Run the LLM, manage clients, show UI |
| Client | Inside the Host       | One per Server | Speak MCP to one specific Server |
| Server | Separate program (local or remote) | Many | Expose tools, resources, prompts |

---

## Common Confusions (Clear These Up Now)

**"Is the Client the same as the Host?"**
No. The Host *contains* Clients. One Host can have many Clients — one for each Server it connects to.

**"Is the Server the AI?"**
No. The AI (LLM) lives in/with the Host. The Server is a dumb, focused program that just exposes capabilities. It has no intelligence on its own.

**"Who decides which tool to call?"**
The **LLM** decides. The Host forwards that decision to the right Client, which talks to the right Server.

**"Can one Client talk to multiple Servers?"**
No. One Client = one Server. This keeps things clean and isolated.

---

## Key Takeaway

> **Host = the app. Client = the connector inside the app. Server = the skill provider outside the app.**
>
> Host holds many Clients. Each Client connects to one Server. Servers expose tools the AI can use.

Once this picture is clear, everything else in MCP — transports, tool calls, resources — is just details on top of this structure.

---

[← Day 01: The Problem MCP Solves](../01.%20The%20Problem%20MCP%20Solves/) | [Next → Day 03: Tools, Resources, Prompts](../03.%20Tools%20Resources%20Prompts/)
