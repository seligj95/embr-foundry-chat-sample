# embr-foundry-chat-sample

A tiny chat app that demonstrates **Scenario 1** of the [Embr × Foundry POC](https://github.com/coreai-microsoft/embr/issues/300):

> An agent is orchestrated *inside* an Embr-hosted Python app using **Microsoft Agent Framework**. The Foundry project only provides the underlying LLM (the model deployment) — the agent loop, system prompt, tool-calling, and memory all live in this repo's own code.

```
┌──────────────────────────────────────────────┐
│              Embr-hosted container           │
│                                              │
│   ┌─────────────┐     ┌──────────────────┐   │
│   │   FastAPI   │────▶│  Microsoft       │   │
│   │   + HTML UI │     │  Agent Framework │   │
│   └─────────────┘     │  (agent loop +   │   │
│                       │   local tools)   │   │
│                       └────────┬─────────┘   │
└────────────────────────────────┼─────────────┘
                                 │ inference only
                                 ▼
                      ┌──────────────────────┐
                      │  Foundry project     │
                      │  → model deployment  │
                      │  (gpt-4o / ...)      │
                      └──────────────────────┘
```

Two toy tools are wired in (`get_weather`, `roll_dice`) so you can actually see tool-calling happen. Replace them with whatever your demo needs.

---

## Part 1 — Create the Foundry resources

You do this **once**, in the Azure AI Foundry portal. The app just needs a model endpoint + key.

### 1. Create a Foundry project

1. Go to [ai.azure.com](https://ai.azure.com) and sign in.
2. Click **+ New project**.
3. Pick an existing hub or let the portal create one. Region: anywhere that has the model you want (e.g., East US 2).
4. After the project is provisioned, open it.

### 2. Deploy a model

1. In the left nav choose **Models + endpoints**.
2. Click **+ Deploy model → Deploy base model**.
3. Pick `gpt-4o-mini` (cheap + fast for this demo) or `gpt-4o`.
4. Accept the default deployment name or give it one you'll remember (e.g., `gpt-4o-mini`).
5. Click **Deploy**.

### 3. Grab the three values you need

From your model deployment page in the Foundry portal, the **Target URI** will look like:

```
https://<resource>.services.ai.azure.com/api/projects/<project>/openai/v1/responses
```

You want the **base** of that URL (everything up to `/openai/v1`, dropping the `/responses` suffix). The OpenAI SDK will append the right path (`/chat/completions`) itself.

| Env var | Where to find it | Example |
|---|---|---|
| `FOUNDRY_BASE_URL` | Target URI with the trailing `/responses` removed | `https://jordan-embr.services.ai.azure.com/api/projects/proj-default/openai/v1` |
| `FOUNDRY_API_KEY` | **Keys + Endpoint** on the Foundry resource (click the key icon to reveal) | `abc123…` |
| `FOUNDRY_MODEL_DEPLOYMENT` | The deployment name you chose when you deployed the model | `gpt-5.4-mini-1` |

> Resource-scoped endpoints (`https://<resource>.openai.azure.com/openai/v1`) also work if you prefer. Both point at the same underlying inference, just via different auth/routing paths.

---

## Part 2 — Run locally

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# fill in the three values from Part 1

uvicorn app.main:app --reload --port 8000
```

Open [http://localhost:8000](http://localhost:8000). Try:

- *"What's the weather in Seattle and Tokyo?"* → should call `get_weather` twice and summarize.
- *"Roll 3d20 and tell me the highest."* → should call `roll_dice` three times.

---

## Part 3 — Deploy to Embr

### Prerequisite

The [Embr GitHub App](https://github.com/apps/embr-platform) must be installed on this repo.

### Deploy

```bash
# Push this repo to GitHub first (under your account)
gh repo create embr-foundry-chat-sample --source=. --public --push

# One-command deploy
embr quickstart deploy <your-user>/embr-foundry-chat-sample -i <installation-id>
```

`embr installations config` returns the installation ID if you don't know it.

### Configure the Foundry values

```bash
embr variables set FOUNDRY_BASE_URL https://<resource>.openai.azure.com/openai/v1
embr variables set FOUNDRY_API_KEY <your-key> --secret
embr variables set FOUNDRY_MODEL_DEPLOYMENT <deployment-name>
```

Trigger a redeploy so the new env vars take effect:

```bash
embr deployments create --restart
```

---

## Project layout

```
.
├── app/
│   ├── __init__.py
│   ├── agent.py          # Agent definition + local tools (get_weather, roll_dice)
│   ├── main.py           # FastAPI: /, /health, /api/chat, /api/config
│   └── static/index.html # Minimal chat UI
├── embr.yaml             # platform: python 3.12, port 8000
├── requirements.txt
├── .env.example
└── README.md
```

## Known limitations / gaps (feeding into the POC findings doc)

- **No auth on `/api/chat`.** V1 intentionally skipped auth so we could focus on the Embr ↔ Foundry wiring. Before shipping anything like this for real, put Entra in front of it.
- **Conversation history is in-memory.** Pod restart = lost context. Would want Redis / Cosmos in a real app.
- **API-key auth to Foundry.** Managed identity would be the secure story; not attempted in this POC.
- **No streaming.** `agent.run()` waits for the full response. Streaming via `agent.run_stream()` + SSE would be a small follow-up.
