# Company Research Agent (n8n + Gemini + Vector Store)

An AI research assistant that answers questions about companies listed on the Nairobi Securities Exchange (NSE) using documents I upload (annual reports, results announcements, and beginner investing guides). It is built in n8n with Google Gemini and a vector store knowledge base.

> **This project provides information and education only. It is not financial advice.** The AI can be wrong, so always verify figures against the original documents and do your own research.

## Documentation

| File | What it covers |
|---|---|
| [build-log.md](build-log.md) | Step-by-step record of how both workflows were built, with settings and design decisions |
| [troubleshooting.md](troubleshooting.md) | Problems I hit, their causes, and how I resolved them |

## Why I built this

My long-term idea is an AI agent that helps beginners understand shares and the basics of investing. The first version of that idea is a knowledge-base agent that answers from real documents, so its answers come from sources I can check instead of from the AI's memory.

## What I learned from earlier projects, and what is new here

| Project | Skills used |
|---|---|
| Simple email agent | Chat Trigger, AI Agent, Gemini, Simple Memory, Gmail tool |
| Inbox Manager Agent | Gmail Trigger, Text Classifier, branching, labels, AI-written drafts |
| **Company Research Agent (this project)** | Form Trigger, file extraction, chunking, embeddings, vector store, retrieval as an agent tool |

The new ideas in this project are **chunking**, **embeddings**, and **retrieval**. The agent setup itself (trigger, AI Agent, model, memory, tool) is the same pattern as my earlier agents.

## Architecture

Two workflows share one knowledge base. They never trigger each other. They are linked only by the vector store's **memory key** (`company_knowledge`) and the **embeddings model**, which must match in both.

```
Workflow 1: Company Knowledge: Load Documents

  Form Trigger ──> Extract from File ──> Simple Vector Store (Insert Documents)
                                              ▲            ▲
                          Embeddings Google Gemini    Default Data Loader
                                                      (chunking + metadata)

Workflow 2: Company Research Agent

  Chat Trigger ──> AI Agent
                     ├── Google Gemini Chat Model
                     ├── Simple Memory
                     └── Tool: Simple Vector Store (Retrieve as Tool)
                              └── Embeddings Google Gemini
```

## Workflow 1: Load documents

| Node | Purpose | Key settings |
|---|---|---|
| On form submission | Starts the workflow and receives the PDF plus labels | Fields: Company (text, required), Year (text), Document (file, `.pdf`) |
| Extract from File | Converts the PDF into plain text | Operation: Extract From PDF; Input Binary Field: `Document` |
| Default Data Loader | Splits the text into chunks and attaches metadata | Type of Data: JSON; Data: `{{ $json.text }}`; Text Splitting: Simple (chunk size 1000, overlap 200); metadata: `company`, `year` |
| Embeddings Google Gemini | Turns each chunk into numbers that represent its meaning | Gemini credential |
| Simple Vector Store | Stores the chunks and their embeddings | Mode: Insert Documents; Memory Key: `company_knowledge` |



## Workflow 2: Research agent

| Node | Purpose | Key settings |
|---|---|---|
| Chat Trigger | Where the user types a question | Default |
| AI Agent | Decides when to search and writes the answer | System message below |
| Google Gemini Chat Model | The language model | Same credential as my email agents |
| Simple Memory | Remembers the conversation | Default |
| Simple Vector Store (tool) | Searches the knowledge base | Mode: Retrieve Documents (As Tool for AI Agent); Name: `company_documents`; Memory Key: `company_knowledge`; Limit: 10 (started at 5, see troubleshooting) |
| Embeddings Google Gemini | Embeds the question for searching | **Must be the same model as Workflow 1** |

### System prompt

```
You are a research assistant that helps beginners understand companies listed on the Nairobi Securities Exchange.

Rules:
- For any fact or figure about a company, use the company_documents tool and answer only from what it returns.
- If the tool returns nothing relevant, say "I couldn't find this in the documents." Never guess numbers.
- Say which company and year each figure comes from.
- Explain jargon in simple words, with a short example.
- Do not tell the user to buy or sell. Present strengths and risks separately.
- End answers about a company with: "This is information, not financial advice. Please do your own research."
```

## How the pieces fit together

1. **Extraction:** a PDF is a print format, so the text has to be pulled out first.
2. **Chunking:** a long report is cut into small pieces so a search returns the relevant paragraph, not the whole document.
3. **Embeddings:** each chunk is converted to numbers capturing its meaning, so a question can match a chunk even when the wording differs.
4. **Vector store:** holds the chunks so they can be searched by meaning.
5. **Retrieval as a tool:** the agent searches the store, reads the best 5 chunks, and writes its answer from them. It works like an open-book exam: the AI does not "know" the report, it reads the relevant pages first.

## Testing

Questions I used to check the agent (answers verified against the source PDF):

| Question | Expected behaviour | Result |
|---|---|---|
| What company is in your documents? | Names the company from the metadata | _to fill in_ |
| What was the company's profit for the year? | Correct figure, with company and year | _to fill in_ |
| Did it pay a dividend, and how much per share? | Correct figure, with source | _to fill in_ |
| What will the share price be next year? | Refuses to guess | _to fill in_ |



## Limitations

- **The Simple Vector Store lives in n8n's memory.** Restarting n8n or the Docker container empties it, so documents must be uploaded again.
- **Documents can only be removed all at once** (Clear Store), not one at a time.
- **Financial tables may chunk badly**, separating a number from its label. If figures are wrong, try a larger chunk size using a custom text splitter.
- **Uploading the same document twice duplicates its chunks.**
- **No filtering by company.** Every question searches the whole store, so with several companies loaded the results can mix. Using full company names in questions and a higher result limit helps; filtering needs a store like Qdrant.
- The agent is only as good as the documents loaded, and old guides may contain outdated rules or fees.

## Troubleshooting summary

| Problem | Cause | Fix |
|---|---|---|
| Couldn't find the text splitter node | The Default Data Loader splits internally on its default **Simple** setting; the splitter connection only appears on **Custom** | Left it on Simple (1000 characters, 200 overlap) |
| Agent only answered with the first company's information after I added a second company | The second company's chunks were stored but ranked below the first company's, and the tool's limit of 5 cut them off | Raised the vector store tool's **Limit** from 5 to 10 |

Full details are in [troubleshooting.md](troubleshooting.md).

## Next steps

- [ ] Add a beginner education layer (NSE and Capital Markets Authority investor guides) with a `Document type` metadata field
- [ ] Persistent storage with Qdrant in Docker (with a volume), plus filtering searches by company
- [ ] Generate a structured research document (overview, price range, dividends, risks, questions to research)
- [ ] Make the agent reachable from my phone (same-network test first, then Tailscale on my home server)

## Repository contents

```
company-research-agent/
├── README.md
├── build-log.md
├── troubleshooting.md
├── workflows/
│   ├── load-documents.json
│   └── research-agent.json
└── screenshots/
```

**Not included on purpose:** the source PDFs (copyrighted, or may contain personal details) and any API keys or credentials. Annual reports can be downloaded from each company's investor relations page.
