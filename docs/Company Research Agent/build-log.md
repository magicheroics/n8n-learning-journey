# Build Log: Company Research Agent

A step-by-step record of how I built the two workflows, in the order I did them, including the decisions I made along the way. For the problems I hit and how I fixed them, see [troubleshooting.md](troubleshooting.md).

## 1. Goal and design

I wanted an AI agent that helps beginners understand shares and investing on the Nairobi Securities Exchange (NSE). Before building anything I wrote down what it should do:

1. Follow a clear system prompt that guides the user and gives research about the company involved.
2. Help produce a document explaining a share: its price range and whether it is worth looking into.
3. Help the user weigh whether an investment looks sensible.
4. Use the Chat Trigger as the entry point (open to better options).
5. State clearly that it is **not a financial advisor** and that users must do their own research.

### Design decisions after reviewing the plan

- **Point 3 changed.** An AI cannot reliably judge whether an investment is "wise", because it does not know the user's goals or risk tolerance. The agent instead presents **strengths and risks separately** and leaves the decision to the user. This is also consistent with point 5.
- **Facts must come from documents.** The agent may only state company figures that come from a tool result. If it cannot find something, it says so.
- **Disclaimer in the output itself,** not only in the prompt, so it cannot be skipped.
- **Read-only.** No trading or portfolio recommendations.

### Scope decision

A live market-data source was not part of this build. I chose to start with a **knowledge-base agent** that answers from documents I upload (annual reports, announcements, and later beginner guides). This teaches the new core skills (chunking, embeddings, retrieval) and is the foundation any later features would build on.

## 2. Tools used

| Tool | Role |
|---|---|
| n8n (Docker on my Ubuntu laptop) | Workflow automation |
| Google Gemini | Chat model and embeddings |
| Simple Vector Store | Knowledge base (kept in n8n's memory) |
| Company PDFs | The knowledge itself, downloaded by hand |

New concepts for me in this project: **chunks** (small pieces of text), **embeddings** (meaning as numbers), and **retrieval** (searching by meaning).

## 3. Workflow 1: Company Knowledge: Load Documents

**Job:** turn a PDF into a searchable knowledge base.

### Step 1: Form Trigger

Added **On form submission** with three fields: `Company` (text, required), `Year` (text), and `Document` (file, accepted type `.pdf`).

Test: I ran the node and submitted a short PDF. The Binary tab showed the file under the field name `Document`. That name matters in the next step.

### Step 2: Extract from File

Added **Extract from File**, Operation **Extract From PDF**, Input Binary Field `Document` (it must match the field name exactly).

Test: the output contained a `text` field with the readable content of the PDF.

### Step 3: Vector Store, embeddings, and loader

Added a **Simple Vector Store** in **Insert Documents** mode with Memory Key `company_knowledge`, then attached two sub-nodes:

- **Embeddings Google Gemini**, using my Gemini credential.
- **Default Data Loader**: Type of Data `JSON`, Data `{{ $json.text }}`, Text Splitting left on **Simple** (chunk size 1000, overlap 200), and two metadata entries:
  - `company` = `{{ $('On form submission').item.json.Company }}`
  - `year` = `{{ $('On form submission').item.json.Year }}`

### Step 4: Test

Ran the workflow with a test PDF. The Simple Vector Store succeeded and output **613 items**, meaning the document had been split into 613 chunks and each one embedded and stored.

### What each node does (and why it is needed)

| Node | Purpose | Without it |
|---|---|---|
| On form submission | Starts the workflow and supplies the file plus labels | No way to feed documents in or label them |
| Extract from File | Turns the PDF into plain text | Later nodes cannot read a PDF |
| Default Data Loader | Splits text into chunks and attaches metadata | One giant block that searches badly, with no source labels |
| Embeddings Google Gemini | Converts each chunk into numbers that represent meaning | Only exact-keyword matching would work |
| Simple Vector Store | Stores the chunks and their embeddings | Nothing left for the agent to search |

### About the labels (metadata)

`company` and `year` are tags stored with every chunk. They **do not affect how the search finds chunks**, because the search works on meaning. Their purpose is to record where each chunk came from, so the agent can cite its source, so different documents can be told apart, and so a store that supports filtering could narrow searches later.

## 4. Workflow 2: Company Research Agent

**Job:** a chat agent that searches the knowledge base and answers from what it finds.

1. **Chat Trigger** added as the entry point.
2. **AI Agent** added after it, with:
   - **Google Gemini Chat Model**, same credential as my earlier agents.
   - **Simple Memory** for conversation memory.
3. **Simple Vector Store as a tool** attached to the agent:
   - Operation Mode: **Retrieve Documents (As Tool for AI Agent)**
   - Name: `company_documents`
   - Description: `Search the uploaded company reports and announcements. Use this for any question about a company's finances, dividends, strategy, or risks.`
   - Memory Key: `company_knowledge`
   - Limit: started at 5 (later raised to 10, see [troubleshooting.md](troubleshooting.md))
   - **Embeddings Google Gemini** attached, using the same model as Workflow 1.
4. **System message** added to the agent:

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

### Testing

I tested with questions I could check against the PDF: profit for the year, whether a dividend was paid and how much, the risks mentioned, and a question the agent should refuse (a future share price). The agent worked in testing.

## 5. How the two workflows connect

They never trigger each other. They are linked only through the vector store:

- the **memory key** `company_knowledge` is the name of the "drawer" Workflow 1 writes into and Workflow 2 reads from, and
- the **embeddings model** must be the same in both, because the search compares meaning-numbers and numbers from different models are not comparable.

## 6. Adding a second company

I uploaded a second company's report (Kenya Airways) through the same form. Each upload adds its chunks to the same store, so the store now holds both companies mixed together. The agent searches the whole store and returns the closest chunks regardless of company, which led to the first real problem: see [troubleshooting.md](troubleshooting.md).

## 7. Other decisions

- **Instance-level MCP in n8n: left disabled.** It lets outside AI tools connect into my n8n instance. I do not need it, and it would add a new way into an instance that holds my credentials.
- **Source PDFs are not committed** to this repo (copyright), and no keys or credentials are included.

## 8. Known limitations

- The Simple Vector Store keeps data **in memory**, so restarting n8n or the container empties it and documents must be re-uploaded.
- Documents cannot be removed individually, only the whole store cleared.
- Uploading the same document twice duplicates its chunks.
- Financial tables may chunk badly, separating numbers from their labels.
- There is no filtering by company yet, so every question searches the whole store.

## 9. Next steps

- [ ] Add a beginner education layer (investor guides) with a `Document type` metadata field
- [ ] Move to Qdrant in Docker (with a volume) for persistent storage and filtering by company
- [ ] Generate a structured research document (overview, price range, dividends, risks, questions to research)
- [ ] Make the agent reachable from my phone (same-network test first, then my home server)
