# Learning log

Short entries on what I did, what went wrong, and what I took from it. Newest entries go at the top.

---

## Entry 1: October 2026 (getting started)

**Goal:** go from "what is n8n?" to a working AI agent.

**What I did**
- Saw an AI WhatsApp chatbot built in n8n and decided to learn how it works
- Installed n8n with Docker on Ubuntu using the free Community Edition
- Started following Nate Herk's Inbox Agent course and freeCodeCamp's Zero to Hero course
- Set up a free Gemini API key and a Gmail OAuth2 credential
- Built, published, and exported my first agent, a simple email-writing chatbot

**What went wrong (and the fixes)**
- `sss_cache` warning after adding myself to the docker group: harmless, verified with `getent group docker`
- Gemini `503 high demand` error: the preview model was overloaded, so I switched models
- Google Cloud billing error `OR_BACR2_59`: billing isn't needed for the Gmail API, so I created the project directly
- OAuth `invalid_client`: I had mistyped the Client Secret; copy-paste fixed it
- Webhook `404 not registered`: I used the production URL before publishing the workflow

**What I learned**
- n8n is mostly low-code; understanding APIs, JSON, and webhooks matters more than programming languages
- OAuth separates *who the app is* (Client ID and Secret) from *what the user allows* (scopes and approval)
- Errors are information: the status code (`401`, `404`, `429`, `503`) tells you whose side the problem is on
- Test URLs and production URLs behave differently in n8n

**Next:** finish the Inbox Manager Agent, then improve the email agent so it only sends when asked.

## Entry 2: Inbox Manager Agent (prototype)

**Goal:** understand how an agent reacts to incoming email on its own, instead of waiting for a chat message.

**What I did**
- Built Gmail Trigger → Text Classifier → label → AI Agent → reply, based on Nate Herk's tutorial
- Wrote label definitions and five fake test emails to drive it
- Got the Customer Support branch working end to end

**What I noticed**
- Only one of the four categories has actions attached, so the others still go nowhere
- The agent had pinned data on it, which can hide whether the model actually ran
- The reply node sends immediately, so drafts would be safer while testing

## Entry 3: Tracing what n8n does on the network

**Goal:** understand what happens on the network when an n8n node calls Google.

**What happened**
- A workflow failed away from home with "The DNS server returned an error." My laptop's DNS worked, but the n8n container was still using my home network's DNS servers. I fixed it by recreating the container with `--dns 1.1.1.1 --dns 8.8.8.8` and a localhost-only port binding. Full write-up: [troubleshooting-dns-in-docker.md](docs/troubleshooting-dns-in-docker.md)
- The Text Classifier hit a Gemini `503` on a preview model again. I switched to a stable model and turned on Retry On Fail.

**What I learned**
- `gmail.googleapis.com` is the Gmail API host and `generativelanguage.googleapis.com` is the Gemini API host. Every Gmail or Gemini node call is a request to one of them.
- Each call follows the same chain: DNS lookup, TCP and TLS on port 443, an HTTPS request carrying my OAuth token, then a JSON response with a status code.
- Failures at different layers look different: DNS error (name resolution), `503` (server overloaded), `401` (not authenticated).
- DNS only controls outbound name lookup. Who can reach n8n is controlled by the port binding (`127.0.0.1`) and the login, not by DNS.

**Exercise**

```bash
nslookup gmail.googleapis.com
curl -i https://gmail.googleapis.com/gmail/v1/users/me/messages
```

Result:
- `nslookup` returned 8 IPv4 and 8 IPv6 addresses, answered by my current network's DNS server (not my home one), which shows the laptop adapts to each network while the container uses fixed public DNS.
- `curl` returned `HTTP/2 401` with a `www-authenticate: Bearer` header and a JSON error: `status: UNAUTHENTICATED`, reason `CREDENTIALS_MISSING`, location `Authorization` header. The request reached Gmail's list-messages method, but with no OAuth token Google refused it. This is the same token n8n attaches to every Gmail node call.
- `401` means "who are you?" while `403` means "I know you, but you're not allowed."

**Next:** wire up the remaining inbox agent branches and record the test results.

**What I learned**
- Polling triggers, items, classification, structured output, routing, and workflow vs agent
- Overlapping keywords in category definitions can confuse the classifier
- Email text is untrusted input, so prompts should treat it as data

**Next:** wire the other branches, add a High Priority notification, and record test results.
