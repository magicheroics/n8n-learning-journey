# Inbox Manager Agent: Improvements Log

Three planned improvements to the [Inbox Manager Agent](inbox-manager-agent.md), what I changed, and how I checked it.

| # | Improvement | Status |
|---|-------------|--------|
| 1 | Switch replies to drafts | Done |
| 2 | Add a fallback category for emails that fit nothing | Done |
| 3 | Run all sample emails and record the results | In progress (1 of 6 recorded) |

---

## 1. Switch replies to drafts

### Problem

The workflow ended with a Gmail **Reply to a message** node, which sends the AI's reply immediately. Nobody reviews it first. An AI can promise the wrong thing, answer the wrong email, or reply to a message that shouldn't get a reply, and a sent email cannot be unsent.

### Change

I changed the final node from **Reply** to **Create a draft**, so the AI writes the reply and I decide whether to send it. This is a "human in the loop" design.

| Field | Value |
|-------|-------|
| Resource | Draft |
| Operation | Create |
| Subject | `Re: ` followed by the original email's subject |
| Email Type | Text |
| Message | The AI Agent's output (`{{ $json.output }}`) |
| To Email (option) | The original sender, dragged in from the trigger data |
| Thread ID (option) | The trigger's `threadId`, so the draft sits in the original conversation |

I dragged the sender and thread fields in from the Input panel after running **Fetch test event**, instead of typing the paths from memory.

### How I checked it

1. Sent a test email from a second address to the practice inbox.
2. Ran the workflow.
3. Looked in Gmail: the reply appeared in **Drafts**, unsent, and nothing left the account.

### Why it matters

Match the AI's freedom to the cost of a mistake:

| Task | Risk if wrong | Level of automation |
|------|---------------|---------------------|
| Adding a label | Very low | Fully automatic |
| Writing a draft | Low, because I review it | AI writes, human approves |
| Sending a reply | Medium to high | Human approves |
| Money, legal, or urgent issues | High | Alert a human |

### Related fix: filter the trigger

While testing, **Fetch test event** returned an email I had **sent**, because the Gmail Trigger had no filter. Sent mail could have been classified and answered. I added a **Search** filter of `-from:me` (Gmail search syntax for "not from me"), and a test email from a second address still came through.

---

## 2. Add a fallback category for emails that fit nothing

### Problem

The Text Classifier had four categories and had to choose one for every email, even when none fit. A forced guess could send an unusual email down the wrong path, for example a vague message about a legal matter getting a cheerful support reply.

### Change

I added a fifth category, **Needs Human Review**, with its own Gmail label. Emails in this category are labeled and the branch **stops there**. It is not connected to the AI Agent, so no draft is created.

**Steps**

1. Created the Gmail label `Needs Human Review`.
2. Added the category to the Text Classifier with this description:

```text
Emails that are unclear, unusual, or don't clearly fit any other category.
Also emails that mix several topics, contain legal threats, complaints about
staff, or requests for personal or account data, or anything where an
automatic reply could be wrong or risky. Do not use this for clear outages or
urgent incidents (those are High Priority). When in doubt between this
category and another, choose this one.
```

3. Duplicated an existing label node, switched its label to `Needs Human Review`, and connected the classifier's new output to it.
4. Left the new branch disconnected from the AI Agent.

Optional extra safety net: the Text Classifier's **When No Clear Match** setting can send anything unmatched to an **Other** output, which can go to the same label node.

### How I checked it

Sent an email with the subject **Quick question** and a vague body about a lawyer possibly getting in touch. Result: **no draft was created**, which is the intended behavior for this category.

### Why it matters

A classifier that must always pick a category will sometimes pick wrongly. Giving it an explicit "I'm not sure" option, and sending that option to a human, is safer than hoping the four categories cover everything.

---

## 3. Run all sample emails and record the results

### Method

The test emails are in [`samples/sample_emails.csv`](../samples/sample_emails.csv). The fallback sample is included as the sixth row.

1. Send one test email at a time from a second address to the practice inbox.
2. Run the workflow and wait for it to finish.
3. In Gmail, check which **label** the email received and whether a **draft** was created.
4. Record the result below and compare it with the expected label.

### Expected behavior

| Category | Label added | Draft created |
|----------|-------------|---------------|
| Customer Support | Yes | Yes |
| Finance/Billing | Yes | Yes (currently the same shared path) |
| High Priority | Yes | Yes (currently the same shared path) |
| Promotion | Yes | Yes (currently the same shared path) |
| Needs Human Review | Yes | **No** |

### Results

| # | Subject | Expected label | Actual label | Draft created? | Match? |
|---|---------|----------------|--------------|----------------|--------|
| 1 | Issue Logging Into Dashboard | Customer Support |  |  |  |
| 2 | Integration Not Syncing With CRM | Customer Support |  |  |  |
| 3 | Incorrect Charge on Invoice #4782 | Finance/Billing |  |  |  |
| 4 | System Outage Affecting All Users | High Priority |  |  |  |
| 5 | New Features + 25% Off Annual Plan | Promotion |  |  |  |
| 6 | Quick question | Needs Human Review |  | No | |

*Row 6: no draft was created. I still need to confirm the label in Gmail and fill in the rest of the table. Only record results from tests I actually ran.*

### What to look for

- **Wrong labels:** if the classifier picks the wrong category, the fix is usually a clearer category description, not a different model.
- **Overlap:** "System Outage Affecting All Users" could fit both Customer Support and High Priority. Check which one wins, and sharpen the High Priority description if needed (for example: "If an email is urgent and also fits another category, choose High Priority").
- **Fallback behavior:** it should catch unclear emails without grabbing clear ones. If it takes too many, tighten its description. If it takes too few, strengthen the "when in doubt" line.

---

## Still open

- [ ] Finish the test table above
- [ ] Give each category its own ending (High Priority notification, no reply for Promotion)
- [ ] Re-export the workflow so the repo matches the current version
