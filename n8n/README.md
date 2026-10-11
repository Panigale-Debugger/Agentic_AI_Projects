# Project 2: AI Support Agents in n8n

Two mini projects built in [n8n](https://n8n.io) that show two common patterns for AI customer support:

| # | Workflow | Pattern | One-line summary |
|---|----------|---------|------------------|
| A | **RAG Support Assistant** | Retrieval-Augmented Generation | Answers customer questions *only* from a stored knowledge base |
| B | **Multi-Agent Support Desk** | Supervisor + specialist agents | A supervisor routes each question to a billing, technical or shipping specialist |

## Folder structure

```
project-2-n8n-support-agents/
├── README.md
├── screenshots/
│   ├── rag-workflow.png
│   ├── rag-chat-demo.png
│   ├── multi-agent-workflow.png
│   └── multi-agent-chat-demo.png
└── workflows/
    ├── RAG_Support_Assistant.json
    └── Multi-Agent_Support_Desk.json
```

## How to import and run

1. Open n8n and create a new workflow.
2. Use the menu **⋯ → Import from file** and select a JSON file from `workflows/`.
3. Attach your own **OpenAI credentials** to every OpenAI node (chat model and embeddings). The exported files reference a managed "Gateway credits" credential that will not exist on another instance.
4. Open the chat panel (or the "Open chat" button) and start asking questions.

---

## A. RAG Support Assistant

### What it does
A support chatbot that does not rely on the model's own knowledge. For every question it first searches a small support knowledge base and answers **only** from what it retrieves. If the answer is not there, it says it does not know and suggests contacting support. This reduces hallucinated policies.

### Architecture

The workflow has two separate flows on one canvas.

**Flow 1: Ingestion (run once, manually)**

```
Load Knowledge Base (Manual Trigger)
        │
Sample Support Articles (Code node)
        │
Store Articles (In-Memory Vector Store, insert)
   ├── Embeddings (Ingest)  – OpenAI embeddings
   └── Article Loader       – Default Data Loader
```

- **Load Knowledge Base**: manual trigger that starts ingestion.
- **Sample Support Articles**: a JavaScript Code node holding five articles, each emitted as one item (`"Title: content"`):
  - Refund policy
  - Reset password
  - Shipping times
  - Cancel subscription
  - Contact hours
- **Article Loader** turns each item into a document, **Embeddings (Ingest)** converts it into a vector, and **Store Articles** saves it in an in-memory vector store under the key `support_kb`. `clearStore` is enabled, so each run replaces the previous contents.

**Flow 2: Question answering (chat)**

```
Ask a Question (Chat Trigger)
        │
Support RAG Assistant (AI Agent)
   ├── Chat Model            – OpenAI chat model
   ├── Conversation Memory   – window of last 6 messages
   └── search_knowledge_base – Vector Store (retrieve-as-tool, key: support_kb)
            └── Embeddings (Query)
```

- The agent's system prompt forces it to **always call `search_knowledge_base` first**, answer **only** from retrieved articles, and fall back to "I don't know, please contact support" otherwise.
- The vector store is exposed to the agent as a **tool**, so the agent decides how to phrase the search query.
- The query embeddings must use the same embedding model as ingestion so the vectors are comparable.
- **Conversation Memory** lets the user ask follow-ups such as "and how long does that take?".

### Example questions
| Question | Expected source |
|----------|-----------------|
| "Can I get my money back after 40 days?" | Refund policy (account credit after 30 days) |
| "My reset link isn't working" | Reset password (link valid for 1 hour) |
| "When does express shipping arrive?" | Shipping times (1-2 business days) |
| "What is the capital of France?" | Not in the knowledge base, so the bot declines |

### Limitations
- The **in-memory vector store is not persistent**: data is lost when n8n restarts, so run Flow 1 again. For production, swap it for a persistent store (Pinecone, Qdrant, Supabase, etc.).
- Articles are hard-coded in a Code node for the demo; a real setup would load them from Google Drive, Notion, a help centre, or files.

### Screenshots
![RAG workflow](screenshots/rag-workflow.png)
![RAG chat demo](screenshots/rag-chat-demo.png)

---

## B. Multi-Agent Support Desk

### What it does
A supervisor agent receives the customer's message and delegates it to specialist agents. Each specialist has its own focused prompt and policies. For mixed questions (e.g. "I want a refund and where is my parcel?") the supervisor calls several specialists and merges their answers into one friendly reply.

### Architecture

```
Customer Message (Chat Trigger)
        │
Support Supervisor (AI Agent)
   ├── Supervisor Model      – OpenAI chat model
   ├── Conversation Memory   – window of last 6 messages
   ├── billing_specialist    (Agent Tool + Billing Model)
   ├── technical_specialist  (Agent Tool + Technical Model)
   └── shipping_specialist   (Agent Tool + Shipping Model)
```

### The agents

| Agent | Handles | Knowledge baked into its prompt |
|-------|---------|---------------------------------|
| **Support Supervisor** | Routing and combining answers. Told **not** to answer from its own knowledge. | n/a |
| **billing_specialist** | Refunds, invoices, payment failures, subscription changes, cancellations | Full refund within 30 days, then account credit. Refunds take 5 business days. Cancel anytime, access until period end, no partial refunds. |
| **technical_specialist** | Login problems, password resets, errors, bugs, how-to questions | Gives numbered troubleshooting steps. Reset links expire after 1 hour. Escalates to a human engineer if unresolved. |
| **shipping_specialist** | Delivery, shipping times, tracking, lost packages | Standard 3-5 business days, express 1-2. Orders after 2pm ship next business day. Tracking is emailed on shipment. |

### How it works
1. The customer sends a message through the chat trigger.
2. The supervisor reads each specialist's **tool description** to decide who should handle it.
3. The supervisor calls the specialist tool(s). The request is passed using n8n's `$fromAI("request", ...)`, so the supervisor writes the sub-question itself, including relevant details.
4. Each specialist, running on its own model with its own system prompt, returns an answer.
5. The supervisor combines the results into one concise reply. Conversation memory keeps context across turns.

### Why this design?
- **Separation of concerns**: each specialist has a small, focused prompt, which is easier to test and less error-prone than one giant prompt.
- **Easy to extend**: adding a new department (e.g. "returns") means adding one more Agent Tool and Model.
- **Parallel topics in one message**: mixed questions are handled in a single reply.

### Example questions
| Question | Agents used |
|----------|-------------|
| "I was charged twice, can I get a refund?" | billing |
| "I can't log in and my reset link expired." | technical |
| "I ordered yesterday at 4pm, when will it arrive?" | shipping |
| "Cancel my subscription and tell me where my order is." | billing + shipping |

### Limitations
- Policies are written directly into the prompts, so they must be updated by hand. Connecting the specialists to the RAG knowledge base from Project A would be a natural improvement.
- Specialists cannot look up real orders or accounts; there are no external data tools yet.

### Screenshots
![Multi-agent workflow](screenshots/multi-agent-workflow.png)
![Multi-agent chat demo](screenshots/multi-agent-chat-demo.png)

---

## Tech stack
- **n8n**: workflow automation, AI Agent, Agent Tool, Chat Trigger, Window Buffer Memory
- **OpenAI**: chat model and text embeddings
- **n8n In-Memory Vector Store**: for the RAG knowledge base
- **JavaScript** (Code node): sample article data
