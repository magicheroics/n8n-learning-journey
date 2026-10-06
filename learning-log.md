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
