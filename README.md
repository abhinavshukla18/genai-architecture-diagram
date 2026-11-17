# Architecture Diagram: Support Ticket Assistant

**Use case:** An assistant that classifies customer support tickets and drafts replies grounded in the company knowledge base. A human reviews and sends every reply.

```mermaid
flowchart LR
    subgraph OURS["OUR SYSTEM (we build and run)"]
        UI["User Interface<br/>(support agent dashboard)"]
        AG["Agent Loop<br/>(think-act cycle)"]
        RAG["RAG Pipeline<br/>(embed query, inject context)"]
        MC["MCP Client<br/>(tool discovery and calls)"]
        EV["Eval Collector<br/>(silent observer)"]
    end

    subgraph THIRD["THIRD-PARTY (we only call these)"]
        MS["MCP Server<br/>(knowledge base search tool)"]
        VDB[("Vector DB")]
        LLM["LLM API"]
    end

    NOTE["NOTE: Eval Collector sits beside the main flow.<br/>It only logs inputs, outputs, doc IDs, latency and scores.<br/>It never changes the response."]

    UI -->|"1. ticket text"| AG
    AG -->|"2. need KB context: query"| RAG
    RAG -->|"3. search_kb tool request"| MC
    MC -->|"4. MCP call"| MS
    MS -->|"5. query embedding"| VDB
    VDB -->|"6. top-k chunks + doc IDs"| MS
    MS -->|"7. tool result"| MC
    MC -->|"8. retrieved chunks"| RAG
    RAG -->|"9. context-injected prompt"| AG
    AG -->|"10. prompt"| LLM
    LLM -->|"11. category + draft reply with doc ID citations"| AG
    AG -->|"12. draft for human approval"| UI

    AG -.->|"logs: ticket, retrieved doc IDs, draft, latency"| EV
    MC -.->|"logs: tool calls and results"| EV
    EV --- NOTE
```

## Request trace (finger test)

User submits a ticket (1) → Agent decides it needs knowledge base context (2) → RAG pipeline builds the query (3) → MCP Client calls the MCP Server (4) → Vector DB search (5-6) → chunks return through MCP (7-8) → RAG injects context into the prompt (9) → Agent sends it to the LLM (10-11) → draft with citations returns to the UI for human approval (12). The Eval Collector logs along the way.

## Pillar choices

I used **RAG** so replies are grounded in real knowledge base documents and cite a document ID for every claim, instead of relying on the LLM's memory. **MCP** connects the agent to the knowledge base through a standard tool protocol, so the data source can be swapped without changing agent code. The **Agent Loop** decides when retrieval is needed, and the **Eval Collector** logs every run so we can measure citation accuracy and draft quality.
