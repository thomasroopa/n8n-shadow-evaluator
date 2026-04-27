# Setup Guide — The Shadow Evaluator

Full installation and configuration reference. The [README](./README.md) covers the high-level run steps; this document covers everything else.

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| n8n | ≥ 1.30.0 | Self-hosted (Docker recommended) |
| Docker + Docker Compose | ≥ 24.0 | For the recommended setup |
| Caddy | ≥ 2.7 | Reverse proxy + automatic HTTPS |
| Node.js | ≥ 18 LTS | Only needed if running n8n without Docker |
| A domain name | — | Required for Caddy's automatic SSL |
| OpenAI API access | GPT-4 | `gpt-4-turbo-preview` or `gpt-4` |
| Google AI API access | Gemini 2.5 Flash | Via [Google AI Studio](https://aistudio.google.com) |
| Gmail account | — | For OAuth2 delivery credential |

---

## 1. Clone the Repository

```bash
git clone https://github.com/<your-username>/shadow-evaluator.git
cd shadow-evaluator
```

---

## 2. Start n8n with Docker Compose

A `docker-compose.yml` is provided. It starts n8n with a persistent volume so your workflows, credentials, and execution history survive restarts.

```bash
docker compose up -d
```

Verify n8n is running:

```bash
docker compose logs -f n8n
# Should end with: "n8n ready on 0.0.0.0, port 5678"
```

n8n is now available at `http://localhost:5678`. Do **not** expose port 5678 directly to the internet — Caddy handles public traffic.

---

## 3. Configure Caddy

Replace `your-domain.com` in `caddy/Caddyfile` with your actual domain:

```
your-domain.com {
    # Serve the HTML submission form
    root * /var/www/shadow-evaluator
    file_server

    # Proxy webhook calls through to n8n
    reverse_proxy /webhook/* localhost:5678 {
        header_up Host {host}
        header_up X-Real-IP {remote}
    }
}
```

Copy the frontend file to the web root Caddy expects:

```bash
sudo mkdir -p /var/www/shadow-evaluator
sudo cp frontend/index.html /var/www/shadow-evaluator/index.html
```

Start Caddy:

```bash
sudo caddy start --config caddy/Caddyfile
```

Caddy will automatically obtain and renew an SSL certificate for your domain via Let's Encrypt. First startup requires port 80 and 443 to be open and DNS to be resolving.

---

## 4. Import the Workflow into n8n

1. Open n8n at `https://your-domain.com` (or `http://localhost:5678` for local testing)
2. Navigate to **Workflows → Import from File**
3. Select `exports/shadow-evaluator-workflow.json`
4. The workflow will import in an inactive state — do not activate it yet

---

## 5. Configure Credentials

All credentials are managed inside n8n. Navigate to **Settings → Credentials** and create the following:

### OpenAI API Key

1. Create a new credential: **OpenAI API**
2. Paste your API key from [platform.openai.com](https://platform.openai.com/api-keys)
3. Name it `OpenAI — Shadow Evaluator`

### Google Gemini API Key

1. Create a new credential: **Google Gemini (PaLM) API**
2. Paste your API key from [Google AI Studio](https://aistudio.google.com/app/apikey)
3. Name it `Gemini — Shadow Evaluator`

### Gmail OAuth2

1. Create a new credential: **Gmail OAuth2 API**
2. Follow the n8n OAuth2 setup flow — you will need a Google Cloud project with the Gmail API enabled
3. Authorise with the Gmail account that will send approved content

After creating each credential, open the imported workflow and assign the relevant credential to each AI and Gmail node.

---

## 6. Environment Variables

If you are running n8n without Docker or want to override defaults, set these in your environment or in the `.env` file at the project root:

```env
# n8n configuration
N8N_HOST=your-domain.com
N8N_PORT=5678
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/

# Optional: set a timezone for execution logs
GENERIC_TIMEZONE=Europe/London

# Optional: basic auth for the n8n editor UI (recommended in production)
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=<strong-password>
```

> ⚠️ Never commit real API keys or passwords. Add `.env` to `.gitignore`.

---

## 7. Activate and Test

1. In n8n, open the imported workflow and click **Activate** (toggle in the top-right)
2. Open `https://your-domain.com` in a browser
3. Fill in the form with a test topic and submit
4. Watch the execution in n8n under **Executions**
5. Check that the Gmail delivery node fires on success

A successful first-pass execution should complete in under 100ms. A single-revision execution will take 600–700ms.

---

## Workflow Configuration Reference

These values are set directly on the n8n nodes, not via environment variables:

| Node | Parameter | Default | Notes |
|---|---|---|---|
| Content Creator | Temperature | `0.8` | Range 0.7–1.0. Lower = more consistent but less creative |
| Content Creator | Max Tokens | `2000` | Increase to 4000 for long-form content |
| The Judge | Temperature | `0.0` | Do not change — deterministic scoring depends on this |
| IF Node | Threshold | `8` | Minimum acceptable score. Adjust to raise or lower the quality bar |
| Counter | Max attempts | `3` | Increase with caution — each additional attempt adds ~$0.01 cost |
| Webhook | Timeout | `300s` | Should accommodate 3 full revision cycles |

---

## Troubleshooting

**Caddy reports a certificate error on first start**

Ensure ports 80 and 443 are open in your firewall and that your domain's DNS A record resolves to the server's public IP. Caddy needs to complete an ACME HTTP-01 challenge.

**n8n webhook returns 404**

The workflow must be **active** (not just saved). Check the toggle in the n8n editor. Also confirm the webhook path in the node matches the path in your Caddyfile (`/webhook/shadow-evaluator`).

**The Judge returns malformed JSON and the workflow crashes**

The Structured Output Parser handles minor issues automatically (missing commas, unmatched quotes). If it still fails, the Judge's response likely contains prose before the JSON object. Tighten the Judge's system prompt to: *"Return ONLY a valid JSON object. No preamble, no explanation, no markdown."*

**Workflow hits max attempts on every execution**

The quality threshold (8/10) may be too high for the current prompts, or the Creator's system prompt is underspecified. Try: (1) reducing the threshold to 7, (2) adding more specific requirements to the Creator's system prompt, or (3) switching the Creator model to `gpt-4` instead of `gpt-4-turbo-preview`.

**Executions from different users are contaminating each other**

This should not happen if the UUID node is running correctly. Open the UUID Generator node, confirm it uses `$workflow.staticData.counter = 0` (not a local variable), and that it fires before the Creator on every trigger.

**Gmail delivery fails with OAuth2 error**

OAuth2 tokens expire. In n8n, go to **Settings → Credentials**, open the Gmail credential, and click **Reconnect**. Re-authorise with your Google account.

---

## Security Notes

- The webhook endpoint is public by default. For production, add an API key check at the Caddy layer or use n8n's built-in header authentication on the Webhook node.
- n8n's editor UI should not be publicly accessible. Restrict it to `localhost` or a VPN, or enable basic auth via the environment variables above.
- Do not log or store the content of submitted requirements if they contain sensitive information — n8n stores full execution data by default. This can be disabled under **Settings → Log Level**.
