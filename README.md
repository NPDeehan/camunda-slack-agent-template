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
  *(May not be needed if you are using a Camunda-provided development cluster — see Step 7.)*
- A **Slack workspace** where you can install a custom app.

---

## Step 1 — Add the BPMN to your Camunda project

1. Log in to [Camunda Web Modeler](https://modeler.camunda.io).
2. Click **Create new project** and give it a name.
3. Inside the project, click **Add file** → **Browse marketplace**. Search for **Camunda Slack Agent Starter** and select it. The BPMN will be added to your project automatically.

---

## Step 2 — Configure the model

Once the BPMN is loaded, make these three changes in Web Modeler before doing anything else. They must be unique to your deployment, especially if you are sharing a cluster with others.

### 2a. Set a unique Process Name and ID

1. Click on an empty area in the canvas (not on any element) to select the process itself.
2. In the properties panel, update **Name** to something descriptive and unique, e.g. `Alice Slack Agent`.
3. Update **ID** to a unique identifier using only letters, numbers, and hyphens, e.g. `alice-slack-agent`.

If two people deploy this template to the same cluster with the default ID `Camunda-Slack-Agent-Template`, their deployments will overwrite each other.

### 2b. Set Message Names

The **"Info From Slack"** start event and the **"Wait for Reply in Thread"** intermediate catch event each need their own unique message name. The two names must be **different** from each other.

1. Go to [uuidgenerator.net](https://www.uuidgenerator.net/) and generate two UUIDs.
2. Click the **"Info From Slack"** start event and paste the first UUID into its **Message Name** field.
3. Click the **"Wait for Reply in Thread"** intermediate catch event and paste the second UUID into its **Message Name** field.

### 2c. Set Webhook IDs

The Webhook ID becomes the last segment of the Camunda inbound webhook URL that Slack will send events to. Both events must share the **same** Webhook ID — this is what allows Slack replies to be routed to the correct process instance.

1. Go to [uuidgenerator.net](https://www.uuidgenerator.net/) and copy a UUID.
2. Click the **"Info From Slack"** start event and paste the UUID into its **Webhook ID** field.
3. Click the **"Wait for Reply in Thread"** intermediate catch event and paste the **same UUID** into its **Webhook ID** field.

---

## Step 3 — Update the secret placeholders in the model

The BPMN uses `HAWK` as a placeholder in all Slack-related secret names (e.g. `SLACK_HAWK_OATH_TOKEN`). Replace `HAWK` with your own identifier — your name, your team name, or any short label. This matters on shared clusters where multiple people may deploy this template, because secret names must be unique per cluster.

Choose your identifier now (e.g. `ALICE`, `MYTEAM`, `JOHN`) and use it consistently in every place below.

| Element | Field | Secret reference |
|---------|-------|-----------------|
| **"Show Answer in Slack"** service task (inside the agent sub-process) | Token | `{{secrets.SLACK_HAWK_OATH_TOKEN}}` |
| **"Info From Slack"** start event | Signing Secret | `{{secrets.SLACK_HAWK_SIGINING_SECRET}}` |
| **"Response"** intermediate catch event | Signing Secret | `{{secrets.SLACK_HAWK_SIGINING_SECRET}}` |

> **Tip:** Download the BPMN, do a find-and-replace of `HAWK` in any text editor, then re-upload to your project. All occurrences will be updated at once.

---

## Step 4 — Connect your cluster to the project

Before you can deploy, you need to link your Camunda cluster to the Web Modeler project.

1. Click the **back arrow** to return to the project view (the list of files in your project).
2. In the left sidebar, click **Connected clusters**.
3. In the dropdown, select the cluster you want to deploy to.

The cluster is now set as the deployment target for every BPMN in this project.

---

## Step 5 — Deploy the process

Deploy now — before creating the Slack app — so that Camunda generates the inbound webhook URL you will need in the next step.

1. Open `camunda-slack-agent-template.bpmn` in Web Modeler.
2. Click **Deploy** in the top-right corner and confirm.

> **Expected warnings:** You will see warnings about secrets that do not exist yet (`SLACK_…`, `AWS_…`). This is normal — the secrets will be added in the following steps. The deployment will still succeed.

### Find and copy the webhook URL

Once deployed, retrieve the webhook URL that Slack will use to send events to Camunda.

1. Click the **"Info From Slack"** start event.
2. In the properties panel, open the **Webhook** tab.
3. Copy the full URL — it will look like:
   ```
   https://<region>-<cluster-id>.connectors.camunda.io/inbound/<your-webhook-UUID>
   ```

Keep this URL handy — you will paste it into the Slack app manifest in the next step.

---

## Step 6 — Create your Slack app from the manifest

This repo includes a `slack-app-manifest.json` file that configures the app name, bot scopes, and event subscriptions automatically. You just need to fill in two placeholders before creating the app.

### 6a. Fill in the manifest placeholders

The manifest is reproduced below. Replace the two placeholders before moving on:

| Placeholder | Replace with |
|-------------|-------------|
| `[NAME_OF_YOUR_APP]` *(appears twice)* | Your chosen bot name, e.g. `My Camunda Agent` |
| `[YOUR_REQUEST_URL]` | The webhook URL you copied in Step 5 |

```json
{
    "display_information": {
        "name": "[NAME_OF_YOUR_APP]"
    },
    "features": {
        "bot_user": {
            "display_name": "[NAME_OF_YOUR_APP]",
            "always_online": false
        }
    },
    "oauth_config": {
        "scopes": {
            "bot": [
                "app_mentions:read",
                "chat:write",
                "channels:history"
            ]
        },
        "pkce_enabled": false
    },
    "settings": {
        "event_subscriptions": {
            "request_url": "[YOUR_REQUEST_URL]",
            "bot_events": [
                "app_mention",
                "message.channels"
            ]
        },
        "org_deploy_enabled": false,
        "socket_mode_enabled": false,
        "token_rotation_enabled": false,
        "is_mcp_enabled": false
    }
}
```

### 6b. Create the app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and sign in.
2. Click **Create New App**.
3. Select **From an app manifest**.
4. Select your workspace and click **Next**.
5. Choose the **JSON** tab, delete the placeholder content, and paste the updated manifest JSON from above.
6. Click **Next**, review the summary of permissions and events, then click **Create**.

### 6c. Copy the Signing Secret and add it to Camunda

1. On the **Basic Information** page, scroll to **App Credentials**.
2. Click **Show** next to **Signing Secret** and copy it.

> **Add to your Camunda cluster now** — Log in to [Camunda Console](https://console.camunda.io) → your cluster → **Secrets** tab → **Create new secret**:
> - **Name:** `SLACK_[YOURNAME]_SIGINING_SECRET`
> - **Value:** the Signing Secret you just copied

### 6d. Install the app and copy the Bot Token

1. In the left sidebar click **OAuth & Permissions**.
2. Click **Install to Workspace** and approve the permissions.
3. After installing, the page shows a **Bot User OAuth Token** starting with `xoxb-`. Copy it.

> **Add to your Camunda cluster now** — Console → your cluster → **Secrets** → **Create new secret**:
> - **Name:** `SLACK_[YOURNAME]_OATH_TOKEN`
> - **Value:** the `xoxb-…` token you just copied

---

## Step 7 — Configure AWS credentials *(skip if on a Camunda dev cluster)*

> **Using a Camunda-provided development cluster?** Dev clusters often come with Bedrock model access pre-configured at the cluster level. Try running the agent first — if it fails to invoke the model, come back here and add the AWS secrets manually.

If you are using your own cluster or the agent cannot reach Bedrock, you need an IAM user or role with `bedrock:InvokeModel` permission on `us.anthropic.claude-sonnet-4-6`. See the AWS guide [Creating IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) if you need to create one (choose **Application running outside AWS** as the use case).

> **Add to your Camunda cluster** — Console → your cluster → Secrets → Create new secret. Create all three:
> - **Name:** `AWS_REGION` — **Value:** your Bedrock-enabled region, e.g. `us-east-1`
> - **Name:** `AWS_ACCESS_KEY` — **Value:** the IAM access key id
> - **Name:** `AWS_SECRET_KEY` — **Value:** the IAM secret access key

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
| `slack-app-manifest.json` | Slack app manifest — fill in the two placeholders and use it to create your Slack app in one step. |
