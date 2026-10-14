# Troubleshooting: Gmail credential "needs to be reconnected" in n8n

**Environment:** n8n Community Edition in Docker on Ubuntu, Gmail OAuth2 credential backed by my own Google Cloud project (app in **Testing** mode)
**Affected node:** Gmail Trigger (the same credential is shared by the Gmail label and reply nodes)

---

## Summary

| | |
|---|---|
| **Issue** | The Gmail Trigger failed with *"The credential "Gmail account" needs to be reconnected"* |
| **Most likely cause** | Google expires OAuth refresh tokens after about 7 days while an app is in Testing mode, and the credential was about a week old |
| **Fix** | Disconnect the credential in n8n and sign in with Google again |
| **Verification** | **Fetch test event** on the Gmail Trigger returned a recent email from the mailbox |
| **Data loss** | None. The workflow and credential settings were unchanged |

---

## 1. The Issue

The Gmail Trigger node showed this error:

```
The credential "Gmail account" needs to be reconnected.
Access could not be refreshed because the connected account has revoked
access, the refresh token expired, or the account password or permissions
changed. Open the credential and reconnect it to continue.
```

The workflow had been running fine a week earlier, and I hadn't changed any Google or n8n settings.

**What the message means:** n8n could no longer get a new access token from Google. This is an authentication problem, not a network problem, and it is different from the earlier DNS error (see [troubleshooting-dns-in-docker.md](troubleshooting-dns-in-docker.md)).

---

## 2. Background: how the token lifecycle works

OAuth2 uses two kinds of tokens:

- **Access token:** short-lived (about an hour), sent with every API request
- **Refresh token:** long-lived, used by n8n to quietly get new access tokens without asking me to sign in again

The error appears when the refresh token stops working. The message lists the possible reasons:

1. The account revoked access
2. The refresh token expired
3. The account password or permissions changed

In my case I hadn't revoked anything or changed my password, so the likely reason was expiry. Google limits refresh tokens for apps in **Testing** mode (an unpublished OAuth consent screen with test users) to about **7 days**, and my credential was about a week old.

---

## 3. Diagnosis

### The credential screen looked healthy

Opening the credential showed a green **Account connected** banner, even though the error said it needed reconnecting.

**Lesson:** that banner only means tokens are stored. It does not prove they still work. The real test is making a call.

### Testing the connection

Opening the **Gmail Trigger** node and clicking **Fetch test event** runs a real request against the mailbox. This is the quickest way to check whether the credential works.

---

## 4. The fix

1. Open the **Gmail account** credential in n8n.
2. Click **Disconnect**. This clears the stored tokens.
3. Click **Sign in with Google** and choose the practice Gmail account.
4. If Google shows the "app isn't verified" warning, click **Advanced**, then **Go to (my app name)**, and allow the permissions.
5. Save the credential.

There is also a **Switch account** button, which re-runs the sign-in without clearing the tokens first. In my case **Disconnect** followed by a fresh sign-in is what worked.

### Verification

Clicking **Fetch test event** on the Gmail Trigger returned an email from the mailbox, which confirmed the credential was working again.

### If the reconnect itself fails

| Problem | Likely cause | Fix |
|---------|--------------|-----|
| Google blocks the sign-in | The account isn't listed as a test user | Add it under **Test users** on the OAuth consent screen (Audience page in Google Cloud) |
| `invalid_client` | Client ID or Client Secret is wrong | Re-paste both values from Google Cloud, copying instead of typing |
| Reconnect works but fails again later | Token expiry is recurring | See the prevention options below |

---

## 5. Prevention options

| Option | Effect | Tradeoff |
|--------|--------|----------|
| **Reconnect when it expires** | Takes about a minute each time | A weekly chore while the app stays in Testing mode |
| **Publish the app ("In production")** in Google Cloud | Refresh tokens generally stop expiring after 7 days | Google keeps showing the "unverified app" warning, and wider public use of Gmail permissions would need Google's verification process. Check the Publish screen before confirming, because Google's rules change |

For now, reconnecting when it expires is enough for a learning project.

---

## 6. Follow-up observation: the trigger had no filter

While verifying the fix, **Fetch test event** returned an email I had **sent**, not one I had received. That showed the Gmail Trigger has no filter, so it picks up every new message in the account, including my own sent mail.

In the Inbox Manager Agent, that means sent mail could be classified and passed to the **Reply to a message** node, which sends immediately. It could email a reply to someone I had just written to.

**Fix applied:** in the Gmail Trigger's **Filters**:

- **Label Names or IDs:** `INBOX`
- **Search:** `-from:me`

- [x] Add the filters
- [x] Send a test email from a second address and confirm the trigger picks it up
- [x] Confirm a sent email is no longer picked up

---

## 7. Lessons learned

1. **Read the full error.** It listed three possible causes, and checking each one against what I had actually done narrowed it quickly.
2. **A green "connected" status isn't proof.** Test the credential with a real call.
3. **Testing-mode OAuth apps have a lifecycle.** Tokens can expire after about a week, so expect to reconnect.
4. **A working trigger can still be wrong.** The test event showed the trigger had no filter, which is a design problem I would not have noticed from a green checkmark.
5. **Authentication errors and network errors look different.** DNS errors mean names can't be resolved. A "needs to be reconnected" or `401` error means the request reached Google and was refused.

---

## Quick reference

```text
Error: Credential needs to be reconnected
  1. Open the credential in n8n
  2. Disconnect
  3. Sign in with Google (Advanced -> Go to app, if warned) and allow access
  4. Save
  5. Gmail Trigger -> Fetch test event to verify

Sign-in blocked?   -> add the account under Test users in Google Cloud
invalid_client?    -> re-paste Client ID and Client Secret
Expires weekly?    -> normal in Testing mode; reconnect or publish the app
```
