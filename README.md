# Learning How mini ClawBot Works

This is my personal notebook for understanding how mini ClawBot is built. mini ClawBot is a Slack AI assistant that goes far beyond a basic chatbot. It can search through past Slack conversations, remember who you are across sessions, and take real actions on GitHub and Notion.

I went through this codebase step by step to understand how all three of these systems fit together inside a single bot.

---

## What mini ClawBot Actually Does

Before diving into the code, I first tried to understand what makes mini ClawBot different from a simple chatbot.

A basic chatbot works like this every single time:

```
User Message → LLM → Reply
```

It has no memory of past conversations. It has no knowledge of what happened before. It cannot take actions in the real world.

mini ClawBot works like this:

```
User Message
    │
    ├── Step 1: Recall what is known about this user from past sessions (Memory)
    ├── Step 2: Search old Slack messages for relevant context (RAG)
    └── Step 3: Load last 10 messages from this conversation (Session)
              │
              ▼
         LLM sees everything above + has 59 tools available
              │
    ├── 12 Slack tools  (search messages, send, schedule, reminders, memory ops)
    ├── 26 GitHub tools (repos, issues, PRs, file contents, code search)
    └── 21 Notion tools (pages, databases, search, create, update)
              │
              ▼
         Sends reply to Slack + saves new facts to memory in the background
```

That is the mental model I kept coming back to while reading the code.

---

## The 3 Systems I Studied

### System 1: RAG (How mini ClawBot Searches Slack History)

RAG stands for Retrieval Augmented Generation. This was the first system I tried to understand because it felt the most unfamiliar.

**The core idea:** Instead of asking the LLM to remember what was said in Slack (it cannot, it has no access), mini ClawBot indexes Slack messages into a local vector database. When a user asks something like "what did we discuss about the API last week?", it searches that database and hands the actual messages to the LLM as context.

**How indexing works (runs every 60 minutes in the background):**

```
Slack Channels
    → Fetch new messages
    → Convert each message to a vector using OpenAI text-embedding-3-small (1536 dimensions)
    → Store vectors + metadata in ChromaDB on local disk
```

**How search works (happens right before the LLM is called):**

```
User Query
    → Convert query to a vector using the same embedding model
    → Find the most similar vectors in ChromaDB using cosine similarity
    → Return the top matching Slack messages
    → Add those messages to the LLM context
```

mini ClawBot also checks the user's message for keywords like "discussed", "talked about", "mentioned", "said" before deciding whether to run the RAG search at all.

**Files I read to understand this:**

| File | What it does |
|------|-------------|
| [src/rag/indexer.ts](src/rag/indexer.ts) | Fetches Slack messages and creates embeddings on a schedule |
| [src/rag/vectorstore.ts](src/rag/vectorstore.ts) | Reads and writes to the ChromaDB vector database |
| [src/rag/retriever.ts](src/rag/retriever.ts) | Takes a query and returns matching messages |
| [src/rag/embeddings.ts](src/rag/embeddings.ts) | Handles the OpenAI embedding API calls |

**What clicked for me:** The reason RAG works is that meaning is preserved in vectors. Two sentences that say the same thing in different words will have similar vectors. That is why you can search by meaning rather than exact keywords.

---

### System 2: Memory (How mini ClawBot Remembers Users)

This system surprised me the most. mini ClawBot uses a cloud service called mem0 that automatically extracts facts from conversations and stores them per user ID.

**How memory gets saved (runs after every reply, in the background):**

```
Full conversation text
    → Sent to mem0 API
    → mem0 uses a small model internally to extract facts
    → Facts are stored and linked to this user's ID
```

Example facts mini ClawBot might extract:
- "User prefers concise responses"
- "User's GitHub username is myusername"
- "User is working on a recommendation system project"

**How memory gets used (runs before every LLM call):**

```
User's message + their user ID
    → Sent to mem0 as a search query
    → mem0 returns facts that are relevant to this message
    → Those facts are added to the LLM's system prompt
```

So if you told mini ClawBot your GitHub username two weeks ago, it will automatically use that username when you ask it to list your repos today, without you having to repeat yourself.

**Files I read to understand this:**

| File | What it does |
|------|-------------|
| [src/memory-ai/mem0-client.ts](src/memory-ai/mem0-client.ts) | All the mem0 API calls: add, search, delete memories |
| [src/memory-ai/index.ts](src/memory-ai/index.ts) | Re-exports memory functions for use in the agent |

**Users can also control their own memory by telling mini ClawBot:**

```
"What do you remember about me?"    → shows stored facts
"Remember that I prefer Python"     → saves a fact manually
"Forget about my old project"       → deletes a specific memory
"Forget everything about me"        → clears all memories
```

**What clicked for me:** Memory storage happens asynchronously after the response is sent. That means the user gets a fast reply and the memory extraction happens quietly in the background. Smart tradeoff.

---

### System 3: MCP (How mini ClawBot Uses GitHub and Notion)

MCP stands for Model Context Protocol. It is an open standard from Anthropic for connecting AI models to external tools without hardcoding each integration.

**The idea:** Instead of writing GitHub API code directly inside mini ClawBot, you run a separate GitHub MCP server process. mini ClawBot talks to that process over stdin/stdout using JSON-RPC. The server handles all the actual API calls.

**How it looks at startup:**

```
mini ClawBot starts
    → Spawns a GitHub MCP server process (via npx)
    → Spawns a Notion MCP server process (via npx)
    → Asks each: "what tools do you have?" (tools/list)
    → Registers all discovered tools alongside the built-in Slack tools
    → LLM now sees all 59 tools total
```

**When a tool is called:**

```
LLM decides to call github_create_issue
    → mini ClawBot parses the tool name: server = "github", tool = "create_issue"
    → Sends a JSON-RPC request to the GitHub MCP server process
    → Server calls the real GitHub API
    → Returns the result back to mini ClawBot
    → mini ClawBot feeds the result back to the LLM
    → LLM generates the next response
```

**Example JSON-RPC message mini ClawBot sends to the GitHub server:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "owner": "your-username",
      "repo": "your-repo",
      "title": "Fix login timeout bug",
      "body": "As discussed on Oct 5..."
    }
  }
}
```

**Files I read to understand this:**

| File | What it does |
|------|-------------|
| [src/mcp/client.ts](src/mcp/client.ts) | Spawns MCP server processes and routes tool calls to them |
| [src/mcp/config.ts](src/mcp/config.ts) | Loads which MCP servers to start from config |
| [src/mcp/tool-converter.ts](src/mcp/tool-converter.ts) | Converts MCP tool definitions into OpenAI function calling format |

**What clicked for me:** MCP is just a standardized way to run tools as separate processes. The benefit is that adding a new tool (say, a Jira integration) means pointing to a new MCP server. The core of mini ClawBot stays untouched.

---

## How mini ClawBot Handles One Message (The Full Flow)

Once I understood the three systems, I traced through what happens when a single message arrives:

```
Step 1: Slack sends the message event to mini ClawBot
        mini ClawBot checks if the user is allowed to interact
        It adds a 👀 reaction so the user knows it received the message

Step 2: Memory retrieval
        mini ClawBot queries mem0 with the user ID and message
        Gets back any relevant stored facts about this user

Step 3: RAG check
        mini ClawBot looks for keywords suggesting the user wants to search history
        If found: embeds the query, searches ChromaDB, retrieves matching Slack messages

Step 4: Assemble the full LLM context
        System prompt
        + Memory facts (if any)
        + RAG results (if any)
        + Last 10 messages from this session
        + Current user message
        + All 59 tool definitions

Step 5: Call the LLM (GPT-4o)
        If the LLM returns tool calls: execute them, add results, call LLM again
        This loop continues until the LLM gives a final text response

Step 6: Send the reply to Slack
        Remove the 👀 reaction
        Post the response (threaded if needed)

Step 7: Store new memories (background, async)
        Send the conversation to mem0
        mem0 extracts and stores any new facts
```

This is all wired together in [src/agents/agent.ts](src/agents/agent.ts) and [src/channels/slack.ts](src/channels/slack.ts).

---

## Project Structure (How the Code Is Organised)

```
Slack-ClawdBot/
├── src/
│   ├── index.ts                 # Boots everything in order: DB → RAG → Memory → MCP → Slack
│   ├── config/
│   │   └── index.ts             # Reads and validates all environment variables using Zod
│   ├── channels/
│   │   └── slack.ts             # Listens for Slack events and triggers the agent
│   ├── agents/
│   │   └── agent.ts             # The brain: assembles context, runs the tool loop, returns response
│   ├── memory/
│   │   └── database.ts          # SQLite database for sessions and conversation history
│   ├── memory-ai/
│   │   ├── index.ts             # Memory system exports
│   │   └── mem0-client.ts       # mem0 cloud API integration
│   ├── rag/
│   │   ├── index.ts             # RAG system exports
│   │   ├── vectorstore.ts       # ChromaDB read/write
│   │   ├── embeddings.ts        # OpenAI embedding API calls
│   │   ├── indexer.ts           # Background job: fetch Slack messages and embed them
│   │   └── retriever.ts         # Semantic search at query time
│   ├── mcp/
│   │   ├── index.ts             # MCP system exports
│   │   ├── client.ts            # Spawns MCP servers and routes tool calls
│   │   ├── config.ts            # MCP server configuration loader
│   │   └── tool-converter.ts    # Translates MCP tool format into OpenAI function format
│   ├── tools/
│   │   ├── slack-actions.ts     # Wrappers for Slack API calls
│   │   └── scheduler.ts         # Cron job manager for scheduled messages
│   └── utils/
│       └── logger.ts            # Winston logger
├── docs/
│   ├── ARCHITECTURE.md          # Deeper architecture notes
│   ├── RAG.md                   # RAG system details
│   ├── MEMORY.md                # Memory system details
│   └── MCP.md                   # MCP integration details
├── scripts/
│   ├── setup-db.ts              # Sets up the SQLite schema
│   └── run-indexer.ts           # Manually triggers a Slack re-index
├── .env.example                 # Template showing all required environment variables
├── mcp-config.example.json      # Template for MCP server configuration
├── Dockerfile
├── docker-compose.yml
└── package.json
```

---

## Running mini ClawBot Yourself

If you want to run mini ClawBot to see it in action, here is what you need.

### What you need first

- Node.js 18+
- A Slack workspace with admin access
- An OpenAI API key
- GitHub Personal Access Token (for GitHub tools)
- Notion Integration Token (for Notion tools)
- mem0 API key (for memory)

### Setup

```bash
# Install dependencies
npm install

# Copy the environment template
cp .env.example .env
```

Fill in `.env`:

```env
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_APP_TOKEN=xapp-your-app-token
SLACK_USER_TOKEN=xoxp-your-user-token

OPENAI_API_KEY=sk-your-openai-key
DEFAULT_MODEL=gpt-4o

MEM0_API_KEY=m0-your-mem0-key
MEMORY_ENABLED=true

GITHUB_PERSONAL_ACCESS_TOKEN=ghp_your-github-token
NOTION_API_TOKEN=secret_your-notion-token

RAG_ENABLED=true
```

### Run

```bash
npm run dev        # Development with hot reload
npm run build && npm start   # Production
docker-compose up -d         # Docker
```

### What a healthy startup looks like

```
✅ Database initialized
✅ Vector store initialized
✅ Background indexer started
✅ Memory system initialized
✅ MCP initialized: github, notion
✅ Task scheduler started
✅ Slack app started
```

---

## All 59 Tools mini ClawBot Has Access To

### Slack Tools (12, built in)

| Tool | Purpose |
|------|---------|
| `search_knowledge_base` | Semantic search over indexed Slack messages |
| `send_message` | Send a message to a channel or user |
| `get_channel_history` | Fetch recent messages from a channel |
| `schedule_message` | Schedule a one-time future message |
| `schedule_recurring_message` | Set up a recurring message |
| `set_reminder` | Set a Slack reminder |
| `list_channels` | List all channels mini ClawBot can see |
| `list_users` | List all workspace users |
| `get_my_memories` | Show what mini ClawBot remembers about you |
| `remember_this` | Save a fact to memory manually |
| `forget_about` | Delete a specific memory |
| `forget_everything` | Clear all memories for your user |

### GitHub Tools via MCP (26)

Search repos, get repo info, list and create issues, list and create pull requests, read file contents from repos, search code, and more.

### Notion Tools via MCP (21)

Search pages and databases, read page content, create and update pages, query and create databases, and more.

---

## Concepts I Now Understand Better

After going through this codebase, these ideas became much clearer to me:

**Tool use / function calling:** The LLM does not execute tools itself. It returns a JSON object saying "call this function with these arguments." mini ClawBot executes it and feeds the result back. This loop is the engine behind most agentic AI systems.

**Vector similarity search:** Two texts with similar meaning have similar vector representations. That is why RAG can find relevant Slack messages even when the user's query uses different words.

**Async memory writes:** mini ClawBot stores memories after the reply is sent, not before. This means the user gets a fast response. Memory extraction is a background task.

**MCP as a plugin system:** MCP is basically a standardized way to run tools as separate processes. It decouples the bot from the tool implementations. You could swap out or add MCP servers without touching mini ClawBot's core logic.

**Zod for config validation:** mini ClawBot validates all environment variables at startup using Zod schemas. If something is missing or wrong, it fails fast with a clear error instead of breaking mid-conversation.

**Barrel `index.ts` files:** Each subsystem has an `index.ts` that re-exports only what the rest of the app needs. This keeps internal implementation details hidden and makes imports clean.

---

## What I Want to Study Next

- [ ] Read [src/agents/agent.ts](src/agents/agent.ts) more carefully to understand exactly how the tool loop is implemented
- [ ] Understand the SQLite session schema in [src/memory/database.ts](src/memory/database.ts)
- [ ] Read [docs/RAG.md](docs/RAG.md) for deeper notes on the RAG pipeline
- [ ] Try swapping GPT-4o for Claude (Anthropic) as the primary model
- [ ] Understand the pairing code system that controls who can DM mini ClawBot
- [ ] Experiment with changing `RAG_MIN_SIMILARITY` to see how it affects result quality

---

## Resources

- [Slack Bolt.js Docs](https://slack.dev/bolt-js/) - The framework mini ClawBot uses to listen for Slack events
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling) - How the tool loop works at the API level
- [mem0 Documentation](https://docs.mem0.ai/) - The memory service mini ClawBot uses
- [Model Context Protocol Docs](https://modelcontextprotocol.io/) - The MCP standard
- [ChromaDB Docs](https://docs.trychroma.com/) - The vector database mini ClawBot uses for RAG

---

## Credits

mini ClawBot was originally built by [Vizuara AI Labs](https://www.youtube.com/@vizuara). I studied this codebase to learn how production-grade Slack bots are built using RAG, long-term memory, and tool orchestration. All credit for the original architecture and implementation goes to the Vizuara team.


---

This is a personal learning notebook. 
