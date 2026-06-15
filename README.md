# Bobby — Production Deployment Guide

## Architecture

```
[Clients] → [FastAPI / Uvicorn] → [Claude API]  (chat, vision, code)
                    ↓                 [OpenAI]   (image generation)
              [PostgreSQL]        ← full message history (durable)
              [Redis]             ← hot cache, last 40 turns (fast)
```

**Smart history management:** Users with thousands of messages never blow up
the context window. Bobby automatically summarises old turns into a compact
memory block and sends only the recent verbatim turns to Claude.

---

## Option A — Railway (recommended, easiest)

1. Push this repo to GitHub.

2. Go to [railway.app](https://railway.app) → New Project → Deploy from GitHub.

3. Add two plugins from the Railway dashboard:
   - **PostgreSQL** (click + New → Database → PostgreSQL)
   - **Redis** (click + New → Database → Redis)

4. Set environment variables in Railway → Variables:
   ```
   ANTHROPIC_API_KEY   = sk-ant-...
   OPENAI_API_KEY      = sk-...
   ```
   `DATABASE_URL` and `REDIS_URL` are injected automatically by Railway plugins.

5. Deploy. Bobby is live at your Railway URL. ✅

---

## Option B — Render

1. Push repo to GitHub.

2. New Web Service → connect repo → set:
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT --workers 2`

3. Add a **PostgreSQL** database (Render dashboard → New → PostgreSQL).
   Copy the Internal Database URL into your env as `DATABASE_URL`.

4. Add a **Redis** instance (Render dashboard → New → Redis).
   Copy the Internal Redis URL into your env as `REDIS_URL`.

5. Set `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` in Environment.

6. Deploy. ✅

---

## Option C — Local development

```bash
cp .env.example .env
# Fill in your API keys in .env

docker compose up
```

Bobby runs at http://localhost:8000.

---

## API Reference

### Chat
```bash
curl -X POST https://your-app.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"user_id": "alice", "message": "Write me a Python quicksort"}'
```

### Chat with image
```bash
curl -X POST https://your-app.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"user_id": "alice", "message": "What bugs do you see?", "image_source": "https://..."}'
```

### Streaming chat (SSE)
```bash
curl -N -X POST https://your-app.railway.app/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"user_id": "alice", "message": "Tell me a story"}'
```

### Generate image
```bash
curl -X POST https://your-app.railway.app/generate-image \
  -H "Content-Type: application/json" \
  -d '{"user_id": "alice", "prompt": "a cozy cabin in the snow, digital art"}'
```

### Check message count
```bash
curl https://your-app.railway.app/history/alice/count
```

### Clear a user's history
```bash
curl -X DELETE https://your-app.railway.app/history/alice
```

### Health check
```bash
curl https://your-app.railway.app/health
```

---

## Scaling notes (50K users)

| Concern | Solution |
|---|---|
| API response time | Streaming endpoint (`/chat/stream`) so users see tokens immediately |
| DB connections | asyncpg pool (max 20) — handles thousands of concurrent requests |
| Redis eviction | 2-hour TTL on inactive sessions — cold users re-warm from Postgres |
| Context window | Auto-summarisation kicks in at 30 turns — scales to unlimited history |
| Horizontal scaling | Stateless FastAPI workers — add more Railway replicas freely |
| Cost | Haiku for summarisation (~10× cheaper than Sonnet) keeps costs sane |
