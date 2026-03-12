# Rammy HR Chatbot — Integration Plan & Developer Guide

## Architecture Overview

```
Browser (HTML + chat.js)
        │  POST /api/chat  { message, history }
        ▼
Node.js Express Server  (port 3000)
  • Rate limiting (60 req/min)
  • CORS guard
  • Input validation
  • Forwards to Python
        │  POST /chat  { message, history }
        ▼
Python Flask Service  (port 5000)
  • PII detection
  • Small-talk routing
  • Chunk retrieval (BM25-style keyword scoring)
  • OpenAI GPT-4.1-mini call
        │
        ▼
OpenAI API  (gpt-4.1-mini)
```

The browser **never** talks directly to OpenAI. The API key lives only in the Python process.

---

## Project File Structure

```
rammy/
├── frontend/
│   ├── index.html          ← Existing HTML (unchanged)
│   ├── chat.js             ← UPDATED — wired to Node.js REST API
│   └── styling_Rev1.css    ← Existing CSS (unchanged)
│
├── server/
│   ├── server.js           ← NEW — Node.js Express REST API
│   ├── package.json        ← NEW — npm dependencies
│   └── .env.example        ← Copy to .env and fill in values
│
└── python/
    ├── chatbot_api.py       ← UPDATED — Flask service (optimized)
    ├── requirements.txt    ← NEW — pip dependencies
    └── .env.example        ← Copy to .env and fill in values
```

---

## Setup Instructions

### Step 1 — Python Backend

```bash
cd rammy/python

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Open .env and set OPENAI_API_KEY=sk-...

# Start the Flask service
python chatbot_api.py
# Running on http://127.0.0.1:5000
```

### Step 2 — Node.js Server

```bash
cd rammy/server

# Install dependencies
npm install

# Configure environment
cp .env.example .env
# Defaults are fine for local dev — no changes needed

# Start the server
npm start          # production
npm run dev        # development (auto-reload via nodemon)
# Running on http://localhost:3000
```

### Step 3 — Frontend

Open `frontend/index.html` in a browser via Live Server (VS Code) or any static server.
The default Live Server origin is `http://127.0.0.1:5500` — this matches the CORS setting in `.env.example`.

---

## API Reference

### POST /api/chat
Send a user message and receive Rammy's reply.

**Request body**
```json
{
  "message": "How do I update my address?",
  "history": [
    { "role": "user",      "content": "Hello" },
    { "role": "assistant", "content": "Hi! How can I help?" }
  ]
}
```

**Response**
```json
{ "reply": "You can update your address through Employee Self Service (ESS)..." }
```

**Error response**
```json
{ "error": "message is required" }
```

---

### POST /api/refresh
Triggers a background reload of all HR source pages.

**Response**
```json
{ "message": "Source refresh started in background." }
```

---

### GET /api/health
Liveness check — also pings the Python service.

**Response (healthy)**
```json
{ "status": "ok", "python": "reachable" }
```

**Response (Python down)**
```json
{ "status": "degraded", "python": "unreachable" }   (HTTP 503)
```

---

## Key Optimizations Made

### Python (`chatbot_api.py`)
| Issue | Fix |
|---|---|
| `client.responses.create` — invalid SDK method | Changed to `client.chat.completions.create` |
| Sources re-fetched on every startup (CLI) | Fetched once on startup, cached in memory, refreshed on demand |
| No HTTP interface | Wrapped in Flask with `/chat`, `/refresh`, `/health` endpoints |
| `psutil` performance monitor on every call | Removed — not meaningful in a server; use APM in production |
| Regex patterns compiled inside functions | Pre-compiled at module level |
| Hard-coded phrase boosts scattered in code | Extracted to `PHRASE_BOOSTS` list — easy to extend |
| `temperature` not set | Set to `0.3` for more consistent factual answers |
| `max_tokens` not set | Set to `300` to prevent runaway responses |

### Node.js (`server.js`)
- API key is never exposed to the browser — all OpenAI calls stay server-side
- Rate limiting prevents abuse (60 req/min per IP)
- CORS restricted to your frontend origin
- 30-second timeout on Python calls handles slow LLM responses gracefully
- History trimmed to 8 messages before forwarding

### Frontend (`chat.js`)
- Real `fetch()` call replaces simulated reply
- HTML-escaped output prevents XSS from API responses
- Input is disabled while waiting for a reply (prevents double-sends)
- Enter key submits via `form submit` event (more natural UX)
- `/refresh` command wired to the refresh endpoint
- Conversation history is maintained client-side and sent with every message

---

## Startup Order

Always start services in this order:
1. Python Flask service (`python chatbot_api.py`)
2. Node.js server (`npm start`)
3. Open the frontend in a browser

---

## Common Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| "chatbot backend is temporarily unavailable" | Python service not running | Start `chatbot_api.py` first |
| CORS error in browser console | `FRONTEND_ORIGIN` mismatch | Update `.env` in `server/` to match your browser URL |
| Empty replies from Rammy | No matching chunks found | Try `/refresh` or check that the HR URLs are reachable |
| `OPENAI_API_KEY` error on Python startup | Missing env var | Set key in `python/.env` |

---

## Next Steps (Recommended)

1. **Semantic search** — Replace keyword scoring with embeddings (OpenAI `text-embedding-3-small`) for much better retrieval accuracy.
2. **Scheduled refresh** — Add a cron job or `APScheduler` to refresh HR sources nightly.
3. **Session IDs** — Add a session token so the Node server can maintain per-user history server-side (removes the need to send history in every request).
4. **Authentication** — Add a simple API key header check in Node.js so only your frontend can call `/api/chat`.
5. **Deploy** — Python to a Render/Railway worker; Node to the same or a separate service; frontend to Netlify or Vercel.
