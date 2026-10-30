# Troubleshooting Log: Company Research Agent

Problems I hit while building the two workflows, what caused each one, and how I resolved it. For the full build, see [build-log.md](build-log.md).

## Summary

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | Could not find the text splitter node | The Default Data Loader splits text internally on its default setting | Left Text Splitting on **Simple** |
| 2 | Agent only returned information about the first company | The second company's chunks ranked below the first company's chunks, and the tool's limit cut them off | Raised the vector store tool's **Limit** from 5 to 10 |

---

## 1. Could not find the text splitter node

**Symptom:** I had been told to add a Recursive Character Text Splitter, but there was nowhere to attach it.

**Cause:** The text splitter connection on the **Default Data Loader** only appears when its **Text Splitting** setting is changed from **Simple** to **Custom**. On Simple, the loader splits the text itself, using a recursive character splitter with a chunk size of 1000 and an overlap of 200. The n8n documentation confirms these defaults. I had also been told the splitter plugs into the vector store, which was wrong: it plugs into the **loader**.

**Resolution:** Left the setting on **Simple**. It already used the values I wanted, and it meant one less node to debug.

**When to revisit:** Switch to **Custom** and add a Recursive Character Text Splitter node only if I need different chunk sizes, for example if answers about financial tables are wrong or answers lack surrounding context. Change one setting at a time and retest with the same questions.

**Lesson:** If a sub-node connection seems to be missing, check whether a setting on the parent node controls it.

---

## 2. Agent only answered with the first company's information

**Symptom:** After uploading a second company's report (Kenya Airways) into the same knowledge base, questions about Kenya Airways returned only Kenya Power (KPLC) information.

**How I investigated:**

1. I suspected the second upload had failed, for example because the PDF was a scanned image with no extractable text, or the run had errored or used a different memory key.
2. I asked a question using the company's full name and opened the execution log. In the vector store tool node, the five retrieved chunks were **all from the first company**, and none mentioned the second.
3. I raised the tool's **Limit** from 5 to 10 and asked again. The Kenya Airways answers appeared.

**Cause:** My first guess (the upload failed) was wrong. The second company's chunks **were in the store**. The search ranks every chunk in the store by how close its meaning is to the question and returns only the top N. With N = 5, five first-company chunks scored slightly higher than the best second-company chunks and took every slot. With N = 10, the second company's chunks reached the agent.

Reasons the first company's chunks may have scored higher:

- The question's meaning matched general content in both reports, and the company name is only a small part of what the search compares.
- The first report may have more chunks on the same topic.
- Some chunks (tables, number lists) embed less cleanly than plain paragraphs.

**Resolution:** Raised the **Limit** on the Simple Vector Store tool from **5 to 10**.

**Trade-offs of a higher limit:**

- More text is sent to the model on each question, which is a little slower and uses more of the model quota.
- More loosely related chunks can come through, so answers can blend sources. The system prompt rule to name the company and year for every figure matters more as a result.

**Further improvements I plan to try:**

1. Add to the system prompt: `When searching, always include the company name from the user's question in the search query.`
2. Use the full company name in questions ("Kenya Airways", not an abbreviation) and in the form's Company field, so labels stay consistent.
3. Move to a vector store that can **filter by metadata** (Qdrant), so a search can be limited to one company's chunks. This removes the ranking contest between companies, and the `company` labels I already store would make it possible.

**Lesson:** When results are missing, do not assume the data is missing. Open the execution log, read exactly which chunks the tool returned, and check whether the data is there but ranked too low. I also should have checked this before trusting my first theory.

---

## Things I checked that turned out not to be problems

- **Memory key and embeddings model matching between workflows:** these must be identical in both workflows (`company_knowledge` and the same Gemini embeddings model). They were fine, but a mismatch would give an empty or nonsense search, so they are the first thing to check when retrieval fails.
- **Binary field name in the form:** the file field is named after the form label (`Document`), and the Extract from File node's **Input Binary Field** must match it exactly.

## Debugging checklist for retrieval problems

1. Did the upload run succeed? Check the **Executions** tab: is the `text` field readable, and how many items did the Simple Vector Store output?
2. Do both workflows use the same **Memory Key** and the same **embeddings model**?
3. In the agent's execution log, open the vector store tool node and **read the retrieved chunks**. Which companies and years do they come from?
4. Is the **Limit** high enough for the number of documents stored?
5. Is the question specific, with the full company name?
6. Has n8n restarted since the upload? The Simple Vector Store keeps its data in memory, so a restart empties it.
