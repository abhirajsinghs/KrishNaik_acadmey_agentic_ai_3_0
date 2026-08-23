# 🔌 MCP Class 1: Why MCP Had to Exist
*📝 Disclaimer: AI-Generated notes compiled from the full Class 1 (MCP module) transcript — "Model Context Protocol," Agentic AI & AgentOps Specialization Bootcamp 3.0 with Cloud, Krish Naik Academy.*
### 📋 Agentic AI & AgentOps Specialization Bootcamp 3.0 with Cloud | Krish Naik Academy

**🎙️ Mentor:** Mayank Aggarwal
**⏱️ Duration:** ~5 hours | **📅 Session:** Day 1 of the MCP module (23 Aug 2026)

---

## 📰 Quick Updates

- CVs: please send them following the format Mayank has asked for before, not just a plain CV — several people have been sending them the wrong way.
- The course content page is being updated with 5–7 hours of new material this week.
- MCP is a big topic — Mayank expects it to take **3–4 classes** to cover in full depth (protocol internals, building your own servers/clients, dockerizing, and publishing a server as a package). After MCP wraps up, the plan is to jump back into LangChain, then move on to multi-agent systems.
- If you haven't finished revising the LangChain classes yet, this is a good two-week window to catch up before things move fast again.

Click [here](https://ai-automation-with-mayank.netlify.app/#home) to see the detailed notes about MCP

---

## 🧠 From Black Box to Hands: Why MCP Had to Exist

Mayank framed the whole topic as a story rather than a spec, and it's worth following in order because MCP only makes sense in light of the problem before it.

It starts with ChatGPT's launch on November 30, 2022 — roughly 1 million users in a week, ~100 million within a few months. Before that, Mayank pointed out, machines mostly only understood fixed inputs like buttons; ChatGPT was the first mass moment where a machine could take natural language in and give natural language back. But underneath, the very first ChatGPT was just an LLM — a next-token predictor. He likes to describe an LLM at that stage as **"just a brain"**: it could write you a great leave-application email, but it had no hands. It couldn't actually *send* the email.

That gap — a brain with no hands — is what triggered the whole chain of developments MCP sits at the end of.

**Step 1: Function calling (mid-2023).** OpenAI shipped the ability to bind tools/functions to an LLM: give it a description of an action (e.g. "send email"), and the model could tell you *which* function to call and with what arguments — though the model itself never executes anything; it just requests the call. This felt like magic at first: no more manual copy-pasting between ChatGPT and your inbox.

**Step 2: The scaling problem.** The catch showed up once this pattern scaled past a single hobby project. Mayank's example: a company running three AI assistants (general chatbot, coding assistant, analytics assistant) each touching ~20 tools across a handful of applications — that's dozens of tools, each with its own auth, its own data format, its own error handling, and its own risk of breaking whenever the underlying API changes. Multiply that across every company doing the same integration work independently, and — as he put it — **"we found a shorter way, but now we need a complete, complete thing to make the shortcut running."** The tool-calling era solved the problem at individual scale but recreated it as a maintenance problem at company scale.

**Step 3: Anthropic's answer.** Anthropic's response was to introduce a *protocol* — not another wrapper, but a shared language, the way HTTP is a shared language for the web — so that an application could expose "here are my tools, resources, and prompts" once, and any AI host could speak to it the same way. That protocol is MCP: Model Context Protocol.

```mermaid
flowchart LR
    A[ChatGPT launches:<br/>LLM as a black box] --> B[Function calling era:<br/>bind tools directly to the LLM]
    B --> C[Scaling problem:<br/>duplicate code, auth, and<br/>maintenance across every team]
    C --> D[MCP: a shared protocol —<br/>the provider maintains the server,<br/>any AI host can connect a client]

    style C fill:#fecaca,stroke:#ef4444
    style D fill:#bbf7d0,stroke:#22c55e
```

One line Mayank repeated to anchor the whole idea:

> *"MCP is not about giving our LLM more intelligence. It is about giving it an ability to see more."*

That's also, he noted, exactly why "context" is in the name — the M is model, the C is context, and the P (protocol) is the new piece that makes context-sharing standardized instead of ad hoc.

---

## 📖 What MCP Actually Is

Quoting the pattern he displayed on screen: MCP is an **open-source standard for connecting AI applications to external systems** — data sources, local files, databases, tools, search engines, calculators, workflows, and specialized prompts.

He deliberately avoided the common "USB-C port" analogy, saying it oversimplifies things. His framing instead: an MCP server is fundamentally a **collection of tools** (used ~99% of the time), wrapped around APIs that already existed, plus two other pieces:

- **Tools** — actions the AI can call (send email, query a DB, etc.) — same idea as the "function calling" tools from before MCP, just standardized.
- **Resources** — not covered in depth yet; flagged as something for a future session.
- **Prompts** — also deferred; several students asked about this and Mayank confirmed it would get its own proper explanation once they build a server that uses one, rather than trying to explain it in the abstract.

These three are referred to as **MCP primitives**.

---

## ⚖️ Before vs. After MCP

| | Before MCP (raw tools/functions) | After MCP |
|---|---|---|
| **Defining an integration** | Easy for a single API — any LLM can generate a quick Python wrapper for, say, Gmail's send API | Easy to *connect* — just point your host at the server |
| **Maintenance** | Every team/company writes and maintains its own wrapper code — violates DRY at scale | Maintenance sits with whoever runs the MCP server (usually the provider) |
| **Change management** | If Gmail changes its API, every integrator has to update their own code | The MCP server absorbs the change; connected clients don't need to touch anything |
| **Security** | Every tool needs its own auth handling, owned by you | Still needs auth, but it's centralized at the server |

Mayank's summary of the core trade-off: at 500 teams all wrapping the same "send email" API independently, you haven't saved effort — you've just relocated it. MCP's value isn't a new capability so much as who's responsible for keeping the integration working.

---

## 🏗️ Architecture: Host, Client, and Server

MCP follows a client-server pattern with one added layer — the **host**, which is the actual application a person uses (Claude Desktop, VS Code, Cursor, any LangChain/LangGraph/AutoGen-based app). The host doesn't talk to an MCP server directly; it starts up an **MCP client** internally, and that client does the talking.

Mayank's analogy for this — and he corrected himself live while giving it, which is worth keeping because the corrected version is the one that stuck:

> *"Mobile phone is your host, and network is your server"* — with the SIM card as the client in between, since a phone can't reach the network without one.

The client and server communicate using **JSON-RPC 2.0**, not plain REST — when asked "is MCP just a REST client?", Mayank's answer was: at a very high level you can think of it that way, but it's a distinct protocol with its own transport rules.

There are two transport types:
- **STDIO** (standard input/output) — for servers that run locally on your own machine.
- **Streamable HTTP** — for remote/hosted servers. This replaced the older **HTTP + SSE** transport, which was deprecated in the MCP spec released in March 2025 (though the practical cutoff for individual servers varies).

```mermaid
sequenceDiagram
    participant H as Host (e.g. Claude Desktop)
    participant C as MCP Client
    participant S as MCP Server

    H->>C: Starts up, connects to configured server
    C->>S: list_tools() request
    S-->>C: Tool list + schemas (name, description, input schema)
    C-->>H: "Here's what this server can do"

    Note over H: User asks the AI to send an email
    H->>C: Model decides to call "send_email" with args
    C->>S: call_tool("send_email", args)
    S-->>C: Result
    C-->>H: Result passed back to the model
```

A subtlety Mayank flagged live with a real example: connecting even two MCP servers to Claude already added a large chunk of tokens to a single "hi" message — because the **entire tool list from every connected server is sent up front**, on the first message, so the model knows what's available before it can decide what to call. He showed this directly in Claude's context inspector (custom tools alone were tens of thousands of tokens with just two servers attached). This is a real cost, and he pointed to **middleware/interceptors** as the mechanism for trimming the tool list down to only what's relevant for a given request rather than loading everything by default — something covered earlier in the course's middleware material and revisited here in the MCP context.

---

## 💻 Live Demo: A Local MCP Server, End to End

Mayank built and connected a minimal local MCP server live, using **FastMCP**, to make the host/client/server flow concrete rather than theoretical:

1. **Wrote a small server** exposing a couple of plain-function tools (a greeting tool, then arithmetic tools like add/subtract) — no special MCP boilerplate needed beyond FastMCP's decorator; it picks up the function's docstring as the tool description automatically.
2. **Tested it with the MCP Inspector**, watching requests go client → server in real time (e.g. sending an "execute tool" call and seeing the response come back).
3. **Registered it as a local STDIO server in VS Code**, using:
   ```bash
   uv run python server.py
   ```
   as the launch command, with the working directory set to the folder containing `server.py`. Once added, VS Code's own `mcp.json` records the server, and a new chat session in VS Code can immediately see and call its tools.
4. **Connected the same kind of server to Claude Desktop** via Manage Connectors → Add custom connector, pointing at a remote MCP server URL (or the same local-server pattern) — showing that the same server works across hosts without any changes, which was really the point of the whole demo.

He also flagged, in response to a student's use case, that a locally-running server can be shared with a small team by port-forwarding or tunneling it (not the most secure option) — full team-wide hosting is the "next class" territory, along with pushing a server as an installable PyPI package.

---

## 🗺️ What's Next

- **Next weekend:** a proper deep-dive cleanup of MCP — resources and prompts explained properly (not just tools), authentication patterns, and live-building + publishing an MCP server to PyPI so anyone can install and use it.
- After MCP wraps: back to **LangChain**, then **multi-agent systems**.
- At least two cloud-based projects are planned across major cloud providers later in the course.

---

## 💬 Live Q&A Highlights

| Question | Answer |
|---|---|
| How is MCP different from just calling a REST API directly? | Functionally the end result can be the same, but with a raw API *you* maintain the wrapper, auth, and error handling in every project that uses it. With MCP, whoever owns the server maintains it once, and every AI host just connects a client. |
| Is MCP basically a REST client? | At a big-picture level you can think of it that way, but it's a distinct protocol built on JSON-RPC 2.0, not literally REST. |
| Won't connecting many MCP servers/tools blow up context or confuse the agent? | Yes — the full tool list from every connected server is sent on the first message. Mayank showed this live (two servers already added tens of thousands of tokens). A good tool description helps, and middleware/interceptors can filter down to only relevant tools instead of loading everything by default. |
| Can an MCP server "extend" another one, like inheritance? | Not in a code sense — but a server *can* contain an MCP client internally that forwards certain requests on to another MCP server. This is possible but not a common pattern. |
| How is an MCP server actually hosted/deployed? | However you'd host any other server — a plain running process, a Docker container, or a cloud service like Cloud Run/AKS. MCP doesn't mandate a deployment method. |
| Can tool access be restricted per user role (e.g. read-only vs. read-write)? | In principle yes — since the client requests the tool list from the server, the server can check the caller's auth token/scope before deciding which tools to return. It's not something MCP prescribes out of the box; you'd build that check yourself. |
| If a company already has 25–30 working APIs, why build an MCP server on top? | Because AI hosts speak MCP's protocol natively. Without a server, every team that wants AI to use those APIs has to write and maintain its own tool wrapper — the exact duplication problem MCP exists to remove. |
| How is MCP different from RAG? | They're unrelated mechanisms for giving a model context — RAG retrieves and injects relevant text into a prompt; MCP is a protocol for letting a model call tools/actions on external systems. Not to be confused, and not typically compared directly. |
| Is MCP an orchestrator? | No — it's a connection/communication protocol between a host and external tools, not something that plans or sequences agent behavior. |
| When was the older HTTP+SSE transport deprecated? | The MCP spec update deprecating it in favor of Streamable HTTP was released in March 2025, though the practical cutover date varies server by server. |

---

## ✅ Action Items After Class

- [ ] If LangChain revision isn't done yet, use this window before MCP ramps up further
- [ ] Set up FastMCP locally and try building a tiny server (2–3 simple tools) to get comfortable with the pattern shown in class
- [ ] Test your server with the MCP Inspector before wiring it into VS Code or Claude Desktop
- [ ] If prepping for interviews touching MCP auth, read up on JWT tokens and OAuth flows — Mayank pointed to this as the background knowledge worth having, even though the course will cover an authentication layer for MCP servers directly in an upcoming session

---
