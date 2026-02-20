# 🤖 Manga Translator × n8n × Telegram Bot — Setup Guide

Automate manga translation: send a `.zip` of manga pages to your Telegram Bot, get back a translated `.zip` automatically.

---

## Architecture

```
Telegram User
    │ sends .zip
    ▼
Telegram Bot  ──►  n8n Workflow  ──►  POST /batch-zip  ──►  Manga Translator
                                                                    │
Telegram User  ◄──  translated .zip  ◄──  n8n  ◄──────────────────┘
```

---

## Step 1 — Start the Manga Translator Server

The translator must be running in `web` mode:

```bash
# Navigate to project root
cd e:\Project\manga-image-translator-web-ui

# Without GPU (slower but works everywhere)
python -m manga_translator --mode web -v

# With GPU (recommended, much faster)
python -m manga_translator --mode web -v --use-gpu

# Server starts at: http://127.0.0.1:5003
```

> **Verify the `/batch-zip` endpoint is available:**
> ```bash
> curl -s http://127.0.0.1:5003/queue-size
> # Expected: {"size": 0}
> ```

---

## Step 2 — Expose the Server to n8n

### Option A: Self-hosted n8n (runs on same machine)
No extra steps needed — use `http://localhost:5003` in the n8n workflow.

### Option B: n8n Cloud or n8n on a different machine
Use **ngrok** to expose the local server:

```bash
# Install ngrok: https://ngrok.com/download
ngrok http 5003
# You'll get a URL like: https://abc123.ngrok-free.app
# Use this URL instead of localhost:5003 in the n8n workflow
```

---

## Step 3 — Create a Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot` and follow the prompts
3. Copy your **Bot Token** (looks like `7123456789:AABB...`)
4. **Optional:** Allow the bot to receive group messages:
   - Send `/setprivacy` → select your bot → choose `Disable`

---

## Step 4 — Configure n8n

### Add Telegram Credentials
1. In n8n → **Credentials** → **Add Credential** → search `Telegram`
2. Paste your **Bot Token**
3. Save — note the **Credential ID** shown in the URL

### Set Environment Variable (for the workflow)
In n8n → **Settings** → **Environment Variables**, add:
```
TELEGRAM_BOT_TOKEN = 7123456789:AABB...   (your bot token)
```

---

## Step 5 — Import the n8n Workflow

1. In n8n → **Workflows** → **Add Workflow** → **Import from File**
2. Select `n8n_telegram_manga_workflow.json` from this project folder
3. In each **Telegram** node, update the credential to your saved Telegram credential
4. In the **POST to /batch-zip** node, update the URL:
   - Self-hosted: `http://localhost:5003/batch-zip`
   - ngrok: `https://YOUR_SUBDOMAIN.ngrok-free.app/batch-zip`
5. Customize translation parameters in `POST to /batch-zip` → Body:

| Parameter | Default | Options |
|-----------|---------|---------|
| `target_lang` | `ENG` | `IND`, `CHS`, `CHT`, `KOR`, `FRA`, `DEU`, `ESP`, etc. |
| `translator` | `sugoi` | `sugoi`, `deepl`, `gpt3.5`, `m2m100`, `none` |
| `size` | `M` | `S` (fastest), `M`, `L`, `X` (most accurate) |
| `detector` | `default` | `default`, `ctd` (better for dense text) |

6. **Activate** the workflow (toggle switch top-right)

---

## Step 6 — Set Up the Webhook

After activating, n8n will show a **Webhook URL** for the Telegram Trigger. Register it with Telegram:

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<YOUR_N8N_WEBHOOK_URL>"
```

Or n8n handles this automatically when you use the Telegram Trigger node with a credential.

---

## Step 7 — Test It!

1. Open Telegram, start a chat with your bot
2. Send a `.zip` file containing manga pages (JPG/PNG/WEBP)
3. Bot replies: *"📥 Got your manga! Translation started..."*
4. Wait for processing (depends on page count and hardware)
5. Bot sends back: *"✅ Translation complete!"* with the translated ZIP

---

## Translators Reference

| Translator | Requires API Key | Offline | Best For |
|------------|-----------------|---------|----------|
| `sugoi` | No | ✅ | Japanese → English (fastest offline) |
| `m2m100` | No | ✅ | Multi-language (slower) |
| `deepl` | `DEEPL_AUTH_KEY` | ❌ | High quality, many languages |
| `gpt3.5` | `OPENAI_API_KEY` | ❌ | Best quality, expensive |
| `google` | No | ❌ | Currently disabled |

Set API keys in a `.env` file at the project root:
```env
DEEPL_AUTH_KEY=your-key-here
OPENAI_API_KEY=sk-your-key-here
```

---

## Performance Expectations

| Setup | Pages / Minute |
|-------|---------------|
| CPU only + sugoi | ~1–2 pages/min |
| GPU (RTX 3060+) + sugoi | ~5–15 pages/min |
| GPU + deepl/gpt3.5 | ~5–10 pages/min (API rate limited) |

> A 200-page volume = ~15–200 minutes depending on setup.
> n8n workflow timeout is set to **2 hours** by default.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Bot doesn't respond | Check webhook is registered; n8n workflow is active |
| "No supported images" error | ZIP must contain `.jpg`, `.png`, or `.webp` files |
| Translation hangs forever | Restart the translator server; check GPU memory |
| ZIP returned is empty | Check `result/` folder; look at server console logs |
| Timeout error in n8n | Increase timeout in `POST to /batch-zip` node options |
