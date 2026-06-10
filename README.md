# Camunda Slack Agent — Starter Template

![Process diagram](img/Screenshot%202026-06-10%20140656.png)

This is a **ready-to-deploy starter template** for building a conversational AI agent on Camunda 8 with Slack as the primary communication channel. If you have never built a Camunda agent before and want a working starting point — this is it.

The template ships with:

- **Pre-configured Slack connectors** for both sending messages and receiving replies in a thread.
- A **Camunda Tasklist channel** as a secondary interface (useful for testing without Slack).
- Built-in BPMN-native abilities to **wait for a user reply** or **pause for a period of time** before continuing — no custom code required.
- An **AI agent** backed by Claude Sonnet 4.6 on AWS Bedrock, wired up inside an ad-hoc sub-process.

You replace the sample domain logic with your own, keep what you need, and delete the rest.

---

## How it works

When a user `@mentions` the Slack bot, Camunda starts a process instance. The AI agent runs inside a BPMN ad-hoc sub-process and can call a set of **tools** — BPMN elements (service tasks, user tasks, script tasks, timer events) that the agent chooses at runtime. The agent posts its answer back into the Slack thread via the outbound Slack connector, then the process waits for either a follow-up reply or a timeout before looping or finishing.

```
User @mentions bot in Slack
        │
        ▼
Slack inbound connector fires process start event
        │
        ▼
┌───────────────────────────────────────────┐
│  AI Agent (ad-hoc sub-process)            │
│  Model: Claude Sonnet 4.6 on Bedrock      │
│                                           │
│  Available tools:                         │
│  ├─ Post answer to Slack thread           │
│  ├─ Get current time / date               │
│  └─ Wait for a set duration              │
└───────────────────────────────────────────┘
        │
        ▼
Agent posts reply → process waits for thread reply (up to 4 min)
        │
        ├─ User replies → agent loops with follow-up
        └─ Timeout → process ends
```

The same process can also be started from **Camunda Tasklist** by submitting a form — handy for quickly testing the agent without touching Slack.

---

## Prerequisites

Before starting you will need:

- A **Camunda 8 SaaS** account with a cluster on **version 8.9 or later**.
  Sign up at [camunda.com](https://camunda.com) if you do not have one.
- An **AWS account** with Amazon Bedrock enabled and access to the model `us.anthropic.claude-sonnet-4-6` in your chosen region.
  *(Skip this if you are using a Camunda-provided development cluster — see Step 2.)*
- A **Slack workspace** where you can install a custom app.

---

## Step 1 — Create and configure the Slack app

You need a Slack app installed into your workspace. The app acts as the bot that users mention and that posts replies on behalf of the agent.

### 1a. Create the app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and sign in with your Slack account.
2. Click **Create New App**.
3. Select **From scratch**.
4. Enter an app name (e.g. *My Camunda Agent*) and select the workspace to install it into.
5. Click **Create App**.

### 1b. Add Bot Token Scopes

The app needs permission to read mentions and post messages.

1. In the left sidebar click **OAuth & Permissions**.
2. Scroll down to **Scopes → Bot Token Scopes**.
3. Click **Add an OAuth Scope** and add each of the following:

| Scope | Required? | Why it is needed |
|-------|-----------|-----------------|
| `app_mentions:read` | Yes | Allows Slack to deliver `@mention` events to Camunda so the process starts. |
| `chat:write` | Yes | Allows the agent to post messages back into the Slack thread. |
| `channels:history` | Yes | Allows the inbound connector to receive threaded replies in public channels. |
| `groups:history` | Optional | Same as above for private channels. Only add if you want the bot to work there. |
| `im:history` | Optional | Same as above for direct messages. |
| `mpim:history` | Optional | Same as above for multi-person DMs. |

### 1c. Install the app and copy the Bot Token

1. Scroll back to the top of the **OAuth & Permissions** page.
2. Click **Install to Workspace** and approve the permissions.
3. After installing, the page shows a **Bot User OAuth Token** starting with `xoxb-`.
4. Copy this token — you will add it to Camunda as a secret. See Step 3 for the exact secret name to use.

### 1d. Copy the Signing Secret

1. In the left sidebar click **Basic Information**.
2. Scroll to **App Credentials**.
3. Copy the **Signing Secret** — you will add it to Camunda as a secret. See Step 3 for the exact secret name to use.

> **Note:** Event Subscriptions (pointing Slack at the Camunda webhook URL) is covered in Step 5 after you have deployed the process and have the actual URL.

---

## Step 2 — Collect your AWS credentials

> **Using a Camunda-provided development cluster?** Development clusters provisioned by Camunda often come with model access pre-configured, meaning the `AWS_REGION`, `AWS_ACCESS_KEY`, and `AWS_SECRET_KEY` secrets may already be set at the cluster level and you can skip this step entirely. Try deploying and testing first — if the agent fails to invoke the model, come back here and set these secrets manually.

If you are using your own cluster or the agent cannot reach Bedrock, you need an IAM user or role with the following permission:

- `bedrock:InvokeModel` on `arn:aws:bedrock:<region>::foundation-model/anthropic.claude-sonnet-4-6` (and the cross-region variant `us.anthropic.claude-sonnet-4-6`).

You will need:

- **AWS Region** — the region where you have Bedrock enabled, e.g. `us-east-1`.
- **Access Key ID** — from the IAM user's security credentials.
- **Secret Access Key** — from the same IAM user.

If you are not sure how to create an IAM user, follow the AWS guide: [Creating IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html). When creating access keys, choose **Application running outside AWS** as the use case.

---

## Step 3 — Add secrets to Camunda

Camunda connectors reference secrets by name using `{{secrets.SECRET_NAME}}` placeholders. Secrets are stored in the Camunda Console and injected at runtime — they never appear in the BPMN file itself.

### How to add a secret

1. Log in to [Camunda Console](https://console.camunda.io).
2. Click on your cluster name to open the cluster details.
3. Click the **Secrets** tab.
4. Click **Create new secret**, enter the name and value, then click **Create**.

Repeat for each secret below.

### Personalising the secret names

The BPMN uses `HAWK` as a placeholder name in all Slack secret references (e.g. `SLACK_HAWK_OATH_TOKEN`). Before creating your secrets, **do a find-and-replace of `HAWK` with your own identifier** — your name, your team name, or any short label that makes these secrets recognisable to you. This matters especially on shared clusters where multiple people might deploy this template at the same time.

To rename: open `camunda-slack-agent-template.bpmn` in a text editor, find all occurrences of `HAWK`, replace them with your chosen identifier (e.g. `ALICE`), and save. Then create your secrets using the renamed values below.

### Required secrets

| Secret name | Where to find the value | What it is used for |
|-------------|------------------------|---------------------|
| `SLACK_[YOURNAME]_OATH_TOKEN` | Slack app → **OAuth & Permissions** → Bot User OAuth Token (`xoxb-…`) | Authenticates the outbound Slack connector so it can post messages as your bot. |
| `SLACK_[YOURNAME]_SIGINING_SECRET` | Slack app → **Basic Information** → Signing Secret | Used by the inbound Slack connector to verify that incoming webhook requests really came from Slack and have not been tampered with. |
| `SLACK_[YOURNAME]_WEBHOOK_ID` | A short URL-safe string **you choose**, e.g. `my-camunda-agent` | Becomes the last segment of the Camunda inbound webhook URL that you will give to Slack. It does not come from Slack — you invent it and then use it in both places. Use only letters, numbers, and hyphens. |
| `AWS_REGION` | Your AWS console, e.g. `us-east-1` | Tells the Bedrock connector which AWS region to call. May not be needed on a Camunda dev cluster — see Step 2. |
| `AWS_ACCESS_KEY` | AWS IAM → your user → Security credentials → Access keys | The access key id for the IAM credentials used to invoke Bedrock. May not be needed on a Camunda dev cluster — see Step 2. |
| `AWS_SECRET_KEY` | Shown once when you create the access key in AWS IAM | The secret access key that pairs with `AWS_ACCESS_KEY`. May not be needed on a Camunda dev cluster — see Step 2. |

---

## Step 4 — Deploy the BPMN

1. Log in to [Camunda Web Modeler](https://modeler.camunda.io).
2. Create a new project (or open an existing one).
3. Upload `camunda-slack-agent-template.bpmn` from this repository.
4. Open the file and click **Deploy** in the top-right corner.
5. Select your cluster and click **Deploy**.

---

## Step 5 — Connect Slack Event Subscriptions

Now that the process is deployed, Camunda has created the inbound webhook endpoint that Slack will send events to.

### 5a. Find the webhook URL

The URL has the format:

```
https://<region>-<cluster-id>.connectors.camunda.io/inbound/<your-SLACK_HAWK_WEBHOOK_ID>
```

To find the exact URL:

1. Open the deployed BPMN in Web Modeler.
2. Click the **Info From Slack** start event (the Slack inbound connector).
3. In the properties panel on the right, open the **Webhook** tab.
4. The full URL is shown there — copy it.

Alternatively, go to **Console → Connectors** and look for the inbound connector for this process.

### 5b. Enable Event Subscriptions in Slack

1. Return to your app at [api.slack.com/apps](https://api.slack.com/apps).
2. In the left sidebar click **Event Subscriptions**.
3. Toggle **Enable Events** on.
4. Paste the webhook URL into the **Request URL** field.
   Slack immediately sends a `url_verification` challenge. The BPMN is already configured to respond automatically — the URL should show a green **Verified** tick within a few seconds.
5. Under **Subscribe to bot events**, click **Add Bot User Event** and add:

| Event | Required? | Why it is needed |
|-------|-----------|-----------------|
| `app_mention` | Yes | Fires the process start event when a user `@mentions` the bot. |
| `message.channels` | Yes | Delivers threaded replies to the inbound intermediate catch event in public channels. |
| `message.groups` | Optional | Same for private channels. Only add if you want the bot to work there. |
| `message.im` | Optional | Same for direct messages. |
| `message.mpim` | Optional | Same for multi-person DMs. |

6. Click **Save Changes** at the bottom.
7. Slack will prompt you to **reinstall the app** to apply the new event scopes — click the link and approve again.

---

## Step 6 — Test it

### Slack

1. Open Slack and go to any channel your bot has been added to.
   If the bot is not in any channel yet, type `/invite @your-app-name` in a channel.
2. Mention the bot with a question:
   ```
   @my-camunda-agent what can you help me with?
   ```
3. You should see a process instance appear in Camunda **Operate** and a reply arrive in the Slack thread.
4. Reply in the thread to ask a follow-up — the process is waiting for it.

### Tasklist (alternative, no Slack required)

1. Open [Camunda Tasklist](https://tasklist.camunda.io).
2. Click **Start process**.
3. Find the `Get Question About tech Stuff` form start event and submit a question.
4. When the agent answers, a user task appears in Tasklist — open it to read the answer and optionally submit a follow-up.

---

## Built-in BPMN capabilities

The template includes two built-in agent tools that are implemented entirely in BPMN — no external service required.

### Wait for user reply

After the agent posts a message to Slack, the process does not end immediately. It sits on an **event-based gateway** and waits for one of two things:

- A **threaded reply** from the user arrives via the Slack inbound intermediate catch event. The reply is correlated to the correct process instance using Slack's `thread_ts` value (the timestamp of the original thread) as the correlation key. The agent then picks up the new message and loops.
- A **4-minute timeout** fires if no reply arrives. The process ends cleanly.

This means the agent can hold a multi-turn conversation in a Slack thread without any polling or custom code.

### Wait for a duration

The agent has access to a **Wait for Next Communication** tool, which is a BPMN timer intermediate event. When the agent calls this tool it passes an [ISO 8601 duration](https://en.wikipedia.org/wiki/ISO_8601#Durations) string (e.g. `PT30M` for 30 minutes, `P1D` for one day). The process pauses for exactly that duration using Camunda's native timer, then automatically resumes.

This is useful for building agents that check back later, send reminders, or implement scheduled follow-ups — again, with no external scheduler or custom code.

---

## Customising the template

### Swap in your own agent logic

The agent's system prompt is in the **AI Agent (ad-hoc sub-process)** element. Open it in Web Modeler and update the `System Prompt` input to describe what your agent should do.

### Add or remove tools

Any BPMN element inside the ad-hoc sub-process is a potential tool. Add a new service task (HTTP connector, script task, etc.), give it a clear **Documentation** description in Web Modeler — that description becomes the tool description the LLM sees. Remove the sample tools you do not need by selecting them and pressing Delete.

### Change the model

The AI Agent task's **Model** property currently points at `us.anthropic.claude-sonnet-4-6` on AWS Bedrock. You can change this to any other Bedrock model, or switch the provider entirely to OpenAI, Anthropic direct, or Azure OpenAI by changing the **Provider type** input on the same task.

### Add a second channel

Add a new start event for the inbound side (e.g. a Microsoft Teams or email inbound connector) and a new service task inside the ad-hoc sub-process as the "send reply" tool. The agent will discover the new tool from its documentation description and use it automatically when a conversation arrives on that channel.

---

## File reference

| File | Purpose |
|------|---------|
| `camunda-slack-agent-template.bpmn` | The main process definition — AI agent, Slack connectors, timer events, and conversation flow. |
