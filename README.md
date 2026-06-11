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

When a user `@mentions` the Slack bot, Camunda starts a process instance. The AI agent runs inside a BPMN ad-hoc sub-process and can call a set of **tools** — BPMN elements (service tasks, script tasks, timer events) that the agent chooses at runtime. The agent posts its answer back into the Slack thread via the outbound Slack connector, then the process waits for either a follow-up reply or a timeout before looping or finishing.

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
  *(May not be needed if you are using a Camunda-provided development cluster — see Step 5.)*
- A **Slack workspace** where you can install a custom app.

---

## Step 1 — Add the BPMN to your Camunda project

1. Log in to [Camunda Web Modeler](https://modeler.camunda.io).
2. Click **Create new project** and give it a name.
3. Add the BPMN using either of the two options below.

**Camunda Marketplace**

Inside the project, click **Add file** → **Browse marketplace**. Search for **Camunda Slack Agent Starter** and select it. The BPMN will be added to your project automatically.


---
> Do not deploy yet. Complete Steps 2 through 5 first — configure the model, update the secret placeholders, and collect your credentials — before deploying.

---

## Step 2 — Configure the model

Once the BPMN is loaded, make these three changes in Web Modeler before doing anything else. They must be unique to your deployment, especially if you are sharing a cluster with others.

### 2a. Set a unique Process Name and ID

1. Click on an empty area in the canvas (not on any element) to select the process itself.
2. In the properties panel, update **Name** to something descriptive and unique, e.g. `Alice Slack Agent`.
3. Update **ID** to a unique identifier using only letters, numbers, and hyphens, e.g. `alice-slack-agent`.

If two people deploy this template to the same cluster with the default ID `Camunda-Slack-Agent-Template`, their deployments will overwrite each other.

### 2b. Set Message Names

The **"Info From Slack"** start event and the **"Wait for Reply in Thread"** intermediate catch event both require a unique message name. Camunda uses this name internally to route inbound Slack events to the correct process.

1. Go to [uuidgenerator.net](https://www.uuidgenerator.net/) and generate two UUIDs — one for each event. They must be **different**.
2. Click the **"Info From Slack"** start event and paste the first UUID into its **Message Name** field.
3. Click the **"Wait for Reply in Thread"** intermediate catch event and paste the second UUID into its **Message Name** field.

### 2c. Set Webhook IDs

The Webhook ID becomes the last segment of the Camunda inbound webhook URL that Slack will send events to. It must be unique to your deployment.

1. Go to [uuidgenerator.net](https://www.uuidgenerator.net/) and copy a UUID.
2. Click the **"Info From Slack"** start event and paste the UUID into its **Webhook ID** field.
3. Click the **"Wait for Reply in Thread"** intermediate catch event and paste the **same UUID** into its **Webhook ID** field.

Both events must share the **identical** Webhook ID — this is what tells Camunda to route Slack replies to the correct process instance.

---

## Step 3 — Update the secret placeholders in the model

The BPMN uses `HAWK` as a placeholder in all Slack-related secret names (e.g. `SLACK_HAWK_OATH_TOKEN`). You need to replace `HAWK` with your own identifier — your name, your team name, or any short label — before you deploy. This matters on shared clusters where multiple people may deploy this template, because secret names must be unique per cluster.

Choose your identifier now (e.g. `ALICE`, `MYTEAM`, `JOHN`) and use it consistently in every place below.

The `HAWK` placeholder appears in the following locations — make sure you have updated all of them before moving on:

| Element | Field | Secret reference |
|---------|-------|-----------------|
| **"Show Answer in Slack"** service task (inside the agent sub-process) | Token | `{{secrets.SLACK_HAWK_OATH_TOKEN}}` |
| **"Info From Slack"** start event | Signing Secret | `{{secrets.SLACK_HAWK_SIGINING_SECRET}}` |
| **"Wait for Reply in Thread"** intermediate catch event | Signing Secret | `{{secrets.SLACK_HAWK_SIGINING_SECRET}}` |

> **Tip:** If you prefer to edit outside Web Modeler, download the BPMN, do a find-and-replace of `HAWK` in any text editor (all occurrences will be updated at once), then re-upload to your project.

---

## Step 4 — Create and configure your Slack app

You need a Slack app installed into your workspace. The app acts as the bot that users mention and that posts replies on behalf of the agent.

In the coming steps you'll be asked to create secrets - No need to do it now, but you will be told to add secrets to Camunda as you go through these steps. Here is how to get to the Secrets page — you will need it several times:

> **How to add a secret:** Log in to [Camunda Console](https://console.camunda.io) → click your cluster name → click the **Secrets** tab → click **Create new secret** → enter the name and value → click **Create**.

### 4a. Create the app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and sign in with your Slack account.
2. Click **Create New App**.
3. Select **From scratch**.
4. Enter an app name (e.g. *My Camunda Agent*) and select the workspace to install it into.
5. Click **Create App**.

### 4b. Add Bot Token Scopes

1. In the left sidebar click **OAuth & Permissions**.
2. Scroll down to **Scopes → Bot Token Scopes**.
3. Click **Add an OAuth Scope** and add the following:

| Scope | Required? | Why it is needed |
|-------|-----------|-----------------|
| `app_mentions:read` | Yes | Allows Slack to deliver `@mention` events to Camunda so the process starts. |
| `chat:write` | Yes | Allows the agent to post messages back into the Slack thread. |
| `channels:history` | Yes | Allows the inbound connector to receive threaded replies in public channels. |


### 4c. Install the app and add the Bot Token to Camunda

1. Scroll back to the top of the **OAuth & Permissions** page.
2. Click **Install to Workspace** and approve the permissions.
3. After installing, the page shows a **Bot User OAuth Token** starting with `xoxb-`. Copy it.

> **Add to your Camunda cluster now** — Console → your cluster → Secrets → Create new secret:
> - **Name:** `SLACK_[YOURNAME]_OATH_TOKEN`
> - **Value:** the `xoxb-…` token you just copied

### 4d. Copy the Signing Secret and add it to Camunda

1. In the left sidebar click **Basic Information**.
2. Scroll to **App Credentials**.
3. Copy the **Signing Secret**.

> **Add to your Camunda cluster now** — Console → your cluster → Secrets → Create new secret:
> - **Name:** `SLACK_[YOURNAME]_SIGINING_SECRET`
> - **Value:** the Signing Secret you just copied

---

## Step 5 — Configure AWS credentials *(skip if on a Camunda dev cluster)*

> **Using a Camunda-provided development cluster?** Dev clusters often come with Bedrock model access pre-configured at the cluster level. Try deploying and running the agent first — if it fails to invoke the model, come back here and add the AWS secrets manually.

If you are using your own cluster or the agent cannot reach Bedrock, you need an IAM user or role with `bedrock:InvokeModel` permission on `us.anthropic.claude-sonnet-4-6`. See the AWS guide [Creating IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) if you need to create one (choose **Application running outside AWS** as the use case).

Once you have your credentials:

> **Add to your Camunda cluster** — Console → your cluster → Secrets → Create new secret. Create all three:
> - **Name:** `AWS_REGION` — **Value:** your Bedrock-enabled region, e.g. `us-east-1`
> - **Name:** `AWS_ACCESS_KEY` — **Value:** the IAM access key id
> - **Name:** `AWS_SECRET_KEY` — **Value:** the IAM secret access key

---

## Step 6 — Deploy the BPMN


Now that the secret placeholders are updated and all secrets exist in the cluster, you are ready to deploy.

1. Open `camunda-slack-agent-template.bpmn` in Web Modeler.
2. Click **Deploy** in the top-right corner.
3. Select your cluster and click **Deploy**.

---

## Step 7 — Connect Slack Event Subscriptions

Now that the process is deployed, Camunda has created the inbound webhook endpoint. You need to give Slack this URL so it knows where to send events.

### 7a. Find the webhook URL

The URL has the format:

```
https://<region>-<cluster-id>.connectors.camunda.io/inbound/<your-webhook-UUID>
```

where `<your-webhook-UUID>` is the UUID you set in Step 2c.

To find the exact URL:

1. Open the deployed BPMN in Web Modeler.
2. Click the **Info From Slack** start event.
3. In the properties panel, open the **Webhook** tab — the full URL is shown there. Copy it.

### 7b. Enable Event Subscriptions in Slack

1. Return to your app at [api.slack.com/apps](https://api.slack.com/apps).
2. In the left sidebar click **Event Subscriptions**.
3. Toggle **Enable Events** on.
4. Paste the webhook URL into the **Request URL** field.
   Slack immediately sends a `url_verification` challenge — the BPMN responds to it automatically, so the URL should show a green **Verified** tick within a few seconds.
5. Under **Subscribe to bot events**, click **Add Bot User Event** and add:

| Event | Required? | Why it is needed |
|-------|-----------|-----------------|
| `app_mention` | Yes | Fires the process start event when a user `@mentions` the bot. |
| `message.channels` | Yes | Delivers threaded replies to the intermediate catch event in public channels. |


6. Click **Save Changes** at the bottom.
7. Slack will prompt you to **reinstall the app** to apply the new scopes — click the link and approve again.

---

## Step 8 — Test it

### Slack

1. Invite the bot into a channel if it is not there yet: type `/invite @your-app-name` in any channel.
2. Mention the bot with a question:
   ```
   @my-camunda-agent In 30 seconds can you respond to me with the current time in Jakarta?
   ```
   This message is a great first test because it exercises two built-in capabilities at once — the timer wait and the current time tool.
3. You should see a process instance appear in Camunda **Operate** and a reply arrive in the Slack thread.
4. Reply in the thread to ask a follow-up — the process is waiting for it.

### Tasklist (no Slack required)

1. Open [Camunda Tasklist](https://tasklist.camunda.io) and click **Start process**.
2. Find the `Get Question About tech Stuff` start event and submit a question.
3. The agent will create a user task with the answer — open it to read the reply and optionally submit a follow-up.

---

## Built-in BPMN capabilities

The template includes two built-in agent tools that are implemented entirely in BPMN — no external service required.

### Wait for user reply

After the agent posts a message to Slack, the process sits on an **event-based gateway** and waits for one of two things:

- A **threaded reply** arrives via the Slack inbound intermediate catch event. The reply is correlated to the correct process instance using Slack's `thread_ts` value as the correlation key. The agent picks up the new message and loops.
- A **4-minute timeout** fires if no reply arrives. The process ends cleanly.

This lets the agent hold a multi-turn conversation in a Slack thread without any polling or custom code.

### Wait for a duration

The agent has access to a **Wait for Next Communication** tool, which is a BPMN timer intermediate event. When the agent calls this tool it passes an [ISO 8601 duration](https://en.wikipedia.org/wiki/ISO_8601#Durations) string (e.g. `PT30M` for 30 minutes, `P1D` for one day). The process pauses for exactly that duration using Camunda's native timer, then automatically resumes.

This is useful for agents that check back later, send reminders, or implement scheduled follow-ups — with no external scheduler or custom code.

---

## Customising the template

### Swap in your own agent logic

The agent's system prompt is on the **AI Agent (ad-hoc sub-process)** element. Click it in Web Modeler and update the `System Prompt` input to describe what your agent should do.

### Add or remove tools

Any BPMN element inside the ad-hoc sub-process is a potential tool. Add a new service task (HTTP connector, script task, etc.) and give it a clear **Documentation** description — that description becomes the tool description the LLM sees. Remove sample tools you do not need by selecting them and pressing Delete.

### Change the model

The AI Agent task currently points at `us.anthropic.claude-sonnet-4-6` on AWS Bedrock. You can change this to any other Bedrock model, or switch the provider entirely to OpenAI, Anthropic direct, or Azure OpenAI by changing the **Provider type** input on the same task.

### Add a second channel

Add a new start event for the inbound side (e.g. a Microsoft Teams or email inbound connector) and a new service task inside the ad-hoc sub-process as the "send reply" tool. The agent will discover it automatically from its documentation description.

---

## File reference

| File | Purpose |
|------|---------|
| `camunda-slack-agent-template.bpmn` | The main process definition — AI agent, Slack connectors, timer events, and conversation flow. |
