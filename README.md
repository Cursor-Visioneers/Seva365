# Seva365 — Nila (Sri Lanka GIC Assistant)

**Seva365** is the workspace that bundles **Nila**, a Government Information Centre (GIC) AI assistant for Sri Lanka. Citizens can ask about civil documents, agriculture, taxes, education, and related government services in **English**, **Sinhala**, and **Tamil** — via text chat, live voice, video avatar, and offline kiosk modes.

This repository is a **monorepo-style workspace**: four independent projects live side by side. Each has its own Git history and remote under [Cursor-Visioneers](https://github.com/Cursor-Visioneers). Use this README as the map; each subfolder has its own README for deep setup.

---

## Table of contents

1. [How the pieces fit together](#how-the-pieces-fit-together)
2. [Repository layout](#repository-layout)
3. [Nila-backend](#nila-backend)
4. [Nila-frontend](#nila-frontend)
5. [Nila-Agents](#nila-agents)
6. [Nila-Offline](#nila-offline)
7. [Which app should I run?](#which-app-should-i-run)
8. [Environment variables (overview)](#environment-variables-overview)
9. [Typical local dev stack](#typical-local-dev-stack)
10. [Knowledge base & content](#knowledge-base--content)
11. [Subproject READMEs](#subproject-readmes)
12. [License](#license)

---

## How the pieces fit together

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Seva365 workspace                                  │
├─────────────────┬─────────────────┬─────────────────┬───────────────────────┤
│  Nila-frontend  │   Nila-Agents   │   Nila-Offline  │     Nila-backend      │
│  (dashboard UI) │ (live agents UI)│ (offline PWA)   │  (FastAPI + RAG API)  │
└────────┬────────┴────────┬────────┴────────┬────────┴───────────┬───────────┘
         │                 │                 │                    │
         │  REST / WS      │  Next API       │  self-contained    │
         └─────────────────┴─────────────────┘                    │
                           │                                        │
                           └────────────────────────────────────────┘
                                    POST /api/chat
                                    WS   /api/avatar/live/ws
                                    WS   /api/live/eleven/ws
```

| Project | Role | Primary stack |
|---------|------|----------------|
| **Nila-backend** | Central API: Supabase vector RAG, multilingual chat, Beyond Presence avatar live, ElevenLabs voice | FastAPI, OpenAI, Gemini, Supabase |
| **Nila-frontend** | Marketing site + citizen dashboard (chat, AI agent, resources, offline links) | Next.js 16, React 19, Tailwind, shadcn/ui |
| **Nila-Agents** | Standalone live-agent experience: EN/TA → OpenAI, SI → Gemini, avatar + TTS | Next.js 15, LiveKit, Beyond Presence |
| **Nila-Offline** | Kiosk-friendly **offline-first** chat: local markdown RAG, Gemini, PWA cache | Next.js 16, Gemini, IndexedDB, next-pwa |

**Data flow (production-style):**  
`Nila-frontend` dashboard pages call **`Nila-backend`** for chat and avatar live sessions. **`Nila-Agents`** can run alone (its own `/api/*` routes proxy LLMs and avatar). **`Nila-Offline`** does not require the Python API for demos — it uses local markdown + optional Gemini.

---

## Repository layout

```
Seva365/
├── README.md                 # This file — workspace overview
│
├── Nila-backend/             # FastAPI server (port 8000)
│   ├── main.py
│   ├── routers/
│   ├── lib/
│   ├── content/
│   ├── static/
│   ├── scripts/
│   ├── docs/
│   ├── frontend/nila-avatar/
│   └── README.md
│
├── Nila-frontend/            # Next.js dashboard + landing
│   ├── README.md
│   └── nila-FE/            # Actual Next.js app
│       ├── app/
│       ├── components/
│       ├── lib/
│       ├── hooks/
│       └── styles/
│
├── Nila-Agents/              # Next.js live agents (EN / SI / TA)
│   ├── app/
│   ├── context/
│   ├── lib/
│   ├── scripts/
│   ├── public/
│   └── README.md
│
└── Nila-Offline/             # Offline PWA variant
    ├── README.md
    └── nila-avatar/          # Actual Next.js app
        ├── app/
        ├── components/
        ├── content/
        ├── lib/
        ├── hooks/
        └── public/
```

Each subfolder is a **separate Git repository** (cloned into this workspace). Remotes:

| Folder | GitHub remote |
|--------|----------------|
| `Nila-backend` | https://github.com/Cursor-Visioneers/Nila-backend |
| `Nila-frontend` | https://github.com/Cursor-Visioneers/Nila-frontend |
| `Nila-Agents` | https://github.com/Cursor-Visioneers/Nila-Agents |
| `Nila-Offline` | https://github.com/Cursor-Visioneers/Nila-Offline |

---

## Nila-backend

**Purpose:** Authoritative backend for RAG-grounded chat, resource extraction (forms, offices, laws), and live voice/avatar integrations.

**Tech:** Python 3, FastAPI, Uvicorn, Supabase (pgvector), OpenAI embeddings + GPT-4o (EN/TA), Google Gemini (Sinhala), ElevenLabs, Beyond Presence + LiveKit.

**Default URL:** `http://localhost:8000` · **API docs:** `http://localhost:8000/docs`

### Folder structure

```
Nila-backend/
├── main.py                      # FastAPI app, CORS, router mounts, static UI routes
├── seed_content.py              # Embed & upload content/**/*.md → Supabase
├── requirements.txt
├── .env.example
│
├── routers/                     # HTTP + WebSocket route handlers
│   ├── chat.py                  # POST /api/chat — main text chat
│   ├── avatar.py                # Avatar setup, ask, LiveKit session, TTS
│   ├── avatar_live.py           # Live avatar WS + OpenAI-compatible RAG for Bey
│   ├── live_elevenlabs.py       # ElevenLabs ConvAI bridge (/api/live/eleven/*)
│   ├── live_openai.py           # OpenAI Realtime (/api/live/en/ws)
│   ├── live.py                  # Gemini live WebSocket
│   ├── voice.py                 # Turn-based voice agent
│   ├── status.py                # Dashboard / health stats
│   ├── resources.py             # Resource-related endpoints
│   ├── reindex.py               # Webhook reindex (n8n)
│   └── n8n_convert.py           # HTML → Markdown pipeline
│
├── lib/                         # Shared business logic
│   ├── rag.py                   # Embeddings + Supabase vector search
│   ├── chat_service.py          # run_chat() — RAG + LLM (shared entry)
│   ├── language_detector.py     # EN / SI / TA detection
│   ├── resource_extractor.py    # Forms, offices, laws from chunks
│   ├── bey_presence.py          # Beyond Presence API (agents, calls, external LLM)
│   ├── bey_call_poller.py       # Poll call transcripts → RAG (local mode)
│   ├── avatar_live_sessions.py  # Push resources to live WS clients
│   ├── openai_chat_stream.py    # SSE stream for Bey external LLM
│   ├── elevenlabs_convai.py     # ElevenLabs live helpers
│   ├── gemini_client.py         # Gemini chat
│   ├── gemini_live.py           # Gemini live speech
│   ├── openai_client.py         # OpenAI chat
│   ├── openai_realtime.py       # OpenAI Realtime
│   ├── voice_stt.py             # Speech-to-text helpers
│   └── live_resources.py        # Live session resource updates
│
├── content/                     # Markdown knowledge (seeded to Supabase)
│   ├── en/                      # English topics (e.g. birth-certificate.md)
│   ├── si/                      # Sinhala
│   ├── ta/                      # Tamil
│   └── synced/                  # Synced / pipeline output
│
├── static/                      # Built-in test UIs (served at /static, /live-eleven, etc.)
│   ├── live-eleven.html         # English live voice (Supabase RAG)
│   ├── avatar-beyond.html       # Avatar live (WebSocket + LiveKit)
│   ├── avatar-test.html
│   ├── chat-live.html
│   ├── gemini-live.html
│   ├── live-en.html
│   ├── voice-agent.html
│   ├── voice-chat.html
│   └── js/livekit-bey.js        # LiveKit + microphone helpers
│
├── scripts/
│   ├── start-public-tunnel.sh   # cloudflared or ngrok → :8000 (for Bey voice RAG)
│   └── complete-voice-rag-setup.sh
│
├── docs/
│   ├── FRONTEND_AVATAR_LIVE.md  # Avatar live WebSocket + resource box contract
│   └── FRONTEND_LIVE_ELEVEN.md  # Live ElevenLabs frontend guide
│
└── frontend/nila-avatar/        # Minimal React reference (App.jsx); optional dist mount
    └── src/App.jsx
```

### Built-in test pages

| Path | Description |
|------|-------------|
| `/` | Health JSON |
| `/docs` | Swagger UI |
| `/chat` | Multi-turn chat UI |
| `/live-eleven` | English live voice — full Supabase RAG |
| `/avatar` | Avatar live — WebSocket + LiveKit + resources |
| `/live-en` | OpenAI Realtime (experimental) |
| `/live` | Gemini Live |
| `/voice` | Turn-based voice |
| `/test` | Avatar/voice smoke tests |

### Key API endpoints

| Method | Path | Use |
|--------|------|-----|
| `POST` | `/api/chat` | Text chat + `resources[]` |
| `GET` | `/api/status` | Status badges |
| `WS` | `/api/avatar/live/ws` | Avatar live session |
| `POST` | `/api/avatar/setup` | Create/fix Bey agent + external LLM |
| `POST` | `/api/avatar/openai/v1/chat/completions` | Called by Beyond Presence for voice RAG |
| `WS` | `/api/live/eleven/ws` | ElevenLabs live English voice |
| `POST` | `/api/reindex` | Re-seed from content (webhook secret) |

See **[Nila-backend/README.md](Nila-backend/README.md)** for full env vars, Supabase SQL, tunnel setup, and troubleshooting.

---

## Nila-frontend

**Purpose:** Public-facing **marketing site** and **citizen dashboard** — onboarding, chat UI, live AI agent (avatar), resources, history, offline info, and support pages. Talks to **Nila-backend** for real chat and avatar live sessions.

**Tech:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, Radix/shadcn UI, Framer Motion, LiveKit client, React Three Fiber (orb visuals).

**App root:** `Nila-frontend/nila-FE/` (run commands from there).

### Folder structure

```
Nila-frontend/
├── README.md
└── nila-FE/
    ├── app/
    │   ├── page.tsx                    # Landing: hero, features, trust, CTA
    │   ├── layout.tsx                  # Root layout, fonts, theme
    │   ├── globals.css
    │   ├── get-started/
    │   │   └── page.tsx                # Onboarding / sign-up flow
    │   └── dashboard/
    │       ├── page.tsx                # Dashboard home (activity, quick links)
    │       ├── chat/page.tsx           # Text chat with Nila (backend API)
    │       ├── ai-agent/page.tsx       # Live avatar: WS + LiveKit + resource box
    │       ├── resources/page.tsx      # Civic resources browser
    │       ├── history/page.tsx        # Conversation history
    │       ├── offline/page.tsx        # Offline / PWA guidance
    │       └── support/page.tsx        # Help & support
    │
    ├── components/
    │   ├── navbar.tsx, footer.tsx, hero-section.tsx, …   # Marketing sections
    │   ├── dashboard/
    │   │   ├── sidebar.tsx             # Dashboard navigation
    │   │   └── avatar-orb.tsx          # 3D orb visual
    │   ├── avatar/
    │   │   └── resource-box.tsx        # Forms / offices / laws panel (avatar live)
    │   └── ui/                         # shadcn/ui primitives (button, dialog, …)
    │
    ├── lib/
    │   ├── api.ts                      # Backend client (chat, avatar live status, proxy paths)
    │   ├── livekit.ts                  # LiveKit room helpers
    │   ├── nila-urls.ts                # External links (e.g. deployed Nila-Agents URL)
    │   ├── civic-resources.ts          # Static / curated resource data
    │   └── utils.ts                    # cn() and shared utilities
    │
    ├── hooks/
    │   ├── useAvatarLive.ts            # WebSocket + LiveKit + resources state machine
    │   ├── use-mobile.ts
    │   └── use-toast.ts
    │
    ├── styles/
    │   └── globals.css
    │
    ├── public/                         # Static assets
    ├── components.json                 # shadcn config
    ├── .env.example
    └── package.json
```

### Dashboard routes

| Route | Description |
|-------|-------------|
| `/` | Marketing landing page |
| `/get-started` | User onboarding |
| `/dashboard` | Main hub after login / setup |
| `/dashboard/chat` | Text chat → `POST /api/chat` on backend |
| `/dashboard/ai-agent` | Live video avatar → `/api/avatar/live/ws` + LiveKit |
| `/dashboard/resources` | Government resources |
| `/dashboard/history` | Past sessions |
| `/dashboard/offline` | Points users to offline / kiosk experience |
| `/dashboard/support` | Support content |

### Backend integration

- Browser HTTP uses same-origin **`/api/nila/*`** (proxied in `next.config` to `NILA_BACKEND_URL`).
- WebSockets use **`NEXT_PUBLIC_NILA_API_URL`** (must be reachable from the browser).
- Copy `.env.example` → `.env.local` and set `NILA_BACKEND_URL` / `NEXT_PUBLIC_NILA_API_URL` to `http://127.0.0.1:8000` when developing locally.

---

## Nila-Agents

**Purpose:** **Working live agents** for Sinhala, English, and Tamil in one Next.js app — unified chat, language-specific LLM routing, Beyond Presence avatar panel, Google TTS / ElevenLabs, and kiosk mode. Can be deployed standalone (e.g. Vercel) without running the full dashboard.

**Tech:** Next.js 15, React 19, Tailwind v4, LiveKit, next-pwa, Beyond Presence, OpenAI + Gemini via internal API routes.

**Package name:** `nila-role-c` · **Default dev:** `http://localhost:3000`

### Language → engine routing

| Language | LLM | Notes |
|----------|-----|--------|
| English (`EN`) | OpenAI | `gpt-4.1-mini` (configurable) |
| Tamil (`TA`) | OpenAI | Same as EN |
| Sinhala (`SI`) | Gemini | `gemini-2.0-flash` |

Routing logic lives in `lib/chatRouter.js`.

### Folder structure

```
Nila-Agents/
├── app/
│   ├── page.jsx                        # Main app: avatar + unified chat + language selector
│   ├── layout.jsx
│   ├── globals.css
│   ├── kiosk/
│   │   └── page.jsx                    # Kiosk mode (5 min idle reset)
│   ├── components/                     # (under app/ in this repo)
│   │   ├── AvatarPanel.jsx             # Beyond Presence / LiveKit avatar
│   │   ├── UnifiedChat.jsx             # Chat transcript + input
│   │   ├── LanguageSelector.jsx
│   │   ├── EngineBadge.jsx             # Shows Gemini vs OpenAI
│   │   └── ChatDemo.jsx
│   └── api/
│       ├── chat/route.js               # Chat completion proxy
│       ├── avatar/route.js             # Avatar actions (e.g. TTS speak)
│       ├── llm/
│       │   ├── route.js                # LLM router entry
│       │   ├── gemini/route.js         # Sinhala
│       │   └── openai/[lang]/route.js  # EN / TA
│       ├── session/sync/route.js       # Session sync
│       └── calls/[callId]/messages/route.js
│
├── context/
│   └── ConversationContext.jsx         # Shared conversation state
│
├── lib/
│   ├── chatRouter.js                   # SI → Gemini, EN/TA → OpenAI
│   ├── openai.js, gemini.js            # Provider clients
│   ├── elevenlabs.js, googleTts.js     # Voice output
│   ├── llmProxyHandler.js, llmAuth.js  # Secure proxy to providers
│   ├── nilaPrompts.js                  # System prompts per language
│   ├── sessionStore.js                 # Session persistence
│   └── conversationUtils.js
│
├── scripts/
│   ├── preCacheQueries.js              # npm run precache
│   ├── register-bp-gemini.mjs          # npm run bp:register-gemini
│   └── register-bp-agents.mjs          # npm run bp:register-agents
│
├── public/
│   ├── manifest.json                   # PWA manifest
│   └── fonts/NotoSansSinhala.woff2      # Sinhala typography
│
├── .env.example
├── next.config.mjs
└── package.json
```

### npm scripts

| Command | Description |
|---------|-------------|
| `npm run dev` | Development server |
| `npm run build` / `start` | Production |
| `npm run precache` | Pre-cache common queries |
| `npm run bp:register-gemini` | Register Bey agent with Gemini |
| `npm run bp:register-agents` | Register per-language Bey agents |

### Quick start

```bash
cd Nila-Agents
npm install
cp .env.example .env.local   # add API keys
npm run dev
```

---

## Nila-Offline

**Purpose:** **Offline-first**, kiosk-friendly GIC assistant — no Supabase required. Uses **local markdown RAG** (keyword search), **Google Gemini** when online, **IndexedDB** cache (~87 questions), rich built-in replies, and **PWA** install for field / low-connectivity demos.

**Tech:** Next.js 16 (webpack for PWA), React 19, Gemini, next-pwa, idb, optional Three.js/VRM avatar components.

**App root:** `Nila-Offline/nila-avatar/`

### Folder structure

```
Nila-Offline/
├── README.md                           # High-level offline product docs
└── nila-avatar/
    ├── app/
    │   ├── page.tsx                    # Chat UI, cache warm-up, offline toggle
    │   ├── layout.tsx
    │   ├── globals.css
    │   └── api/chat/route.ts           # Gemini + local RAG
    │
    ├── components/
    │   ├── AssistantMessage.tsx        # Plain-text bubbles, links, sections
    │   ├── NilaAvatar.tsx, LocalAvatar3D.tsx, OfflineAvatar.tsx
    │   ├── BeyondPresenceOffline.tsx
    │   └── avatar/AvatarModel.tsx      # 3D model (GLB/VRM)
    │
    ├── content/en/                     # 14+ English topic markdown files
    │   ├── birth-certificate.md
    │   ├── national-id.md, passport.md, driving-license.md
    │   ├── marriage-certificate.md, tax-sri-lanka.md, education-sri-lanka.md
    │   └── agriculture-*.md          # Ministry, DOA, 1920 helpline, etc.
    │
    ├── lib/
    │   ├── rag.ts                      # Keyword search over markdown (no vector DB)
    │   ├── offlineCache.ts             # IndexedDB + OFFLINE_QUERIES list
    │   ├── knowledgeReplies.ts         # Rich offline fallback answers
    │   ├── formatMessage.ts            # Strip markdown, order sections
    │   ├── offlineSpeech.ts
    │   ├── avatarConfig.ts, beyOfflineClips.ts
    │   └── …
    │
    ├── hooks/
    │   └── useOnlineStatus.ts
    │
    ├── public/
    │   ├── avatar/                     # avatar.glb, nila-human.glb, nila.vrm
    │   ├── bey-offline/                # Offline Bey clip placeholders
    │   └── sw.js, workbox-*.js         # Service worker (production build)
    │
    └── package.json
```

### Offline features

| Feature | Details |
|---------|---------|
| **Warm-up cache** | ~87 pre-cacheable questions → IndexedDB |
| **Built-in answers** | `knowledgeReplies.ts` — steps, phones, URLs without network |
| **PWA** | Enabled in **production** build only (disabled in `dev`) |
| **Kiosk** | `?kiosk=true` — larger UI, 5-minute idle session reset |
| **Test offline** | Toggle or `?offline=true` |

### Topics (English knowledge base)

Civil documents (birth, NIC, passport, licence, marriage), agriculture (ministry, DOA, 1920, subsidies, export crops, livestock), tax (IRD, TIN), education (enrollment, O/L, A/L, UGC).

See **[Nila-Offline/nila-avatar/README.md](Nila-Offline/nila-avatar/README.md)** for env vars, cache versioning, and extending `content/en/`.

---

## Which app should I run?

| Goal | Run |
|------|-----|
| Full RAG + Supabase + production chat API | **Nila-backend** (`uvicorn` on :8000) |
| Citizen dashboard + live avatar against API | **Nila-backend** + **Nila-frontend** (`nila-FE`) |
| Standalone trilingual live agents + avatar | **Nila-Agents** only |
| Offline kiosk / PWA demo without Python | **Nila-Offline** (`nila-avatar`) |
| Quick API smoke test | **Nila-backend** → open `/docs` or `/chat` |

---

## Environment variables (overview)

### Nila-backend (`.env`)

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | Embeddings + EN/TA chat |
| `GEMINI_API_KEY` | Sinhala chat |
| `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` | Vector RAG |
| `ELEVENLABS_API_KEY`, `ELEVENLABS_AGENT_ID` | Live voice / TTS |
| `BEYOND_PRESENCE_API_KEY`, `BEY_AGENT_ID` | Avatar live |
| `NILA_PUBLIC_BASE_URL` | Public HTTPS URL for Bey → your RAG (tunnel) |
| `BEY_LLM_API_SECRET` | Bearer token for external LLM endpoint |

### Nila-frontend (`nila-FE/.env.local`)

| Variable | Purpose |
|----------|---------|
| `NILA_BACKEND_URL` | Server-side proxy target |
| `NEXT_PUBLIC_NILA_API_URL` | Browser + WebSocket base URL |

### Nila-Agents (`.env.local`)

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY`, `GEMINI_API_KEY` | LLMs |
| `BEY_API_KEY`, `BEY_AGENT_ID_*` | Beyond Presence per language |
| `ELEVENLABS_API_KEY`, `ELEVENLABS_VOICE_ID_*` | TTS voices |
| `PUBLIC_APP_URL` | Public app URL for callbacks |

### Nila-Offline (`nila-avatar/.env.local`)

| Variable | Purpose |
|----------|---------|
| `GEMINI_API_KEY` | Online AI replies (optional; KB fallback works) |
| `BEYOND_PRESENCE_API_KEY`, `BP_PERSONA_ID` | Optional live avatar |
| `ELEVENLABS_API_KEY`, `N8N_WEBHOOK_SECRET` | Optional integrations |

---

## Typical local dev stack

**Terminal 1 — API**

```bash
cd Nila-backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
# Configure keys, run Supabase SQL, then:
python seed_content.py
uvicorn main:app --reload --port 8000
```

**Terminal 2 — Dashboard**

```bash
cd Nila-frontend/nila-FE
npm install
cp .env.example .env.local
# NILA_BACKEND_URL=http://127.0.0.1:8000
npm run dev
```

**Optional — Live agents only**

```bash
cd Nila-Agents
npm install && npm run dev
```

**Optional — Offline kiosk**

```bash
cd Nila-Offline/nila-avatar
npm install && npm run dev
# For PWA: npm run build && npm start
```

---

## Knowledge base & content

| Location | Format | Used by |
|----------|--------|---------|
| `Nila-backend/content/{en,si,ta}/` | Markdown → Supabase vectors | Backend RAG (`seed_content.py`) |
| `Nila-Offline/nila-avatar/content/en/` | Markdown, keyword RAG | Offline app only |

To add backend topics: edit markdown under `Nila-backend/content/`, run `python seed_content.py`, or `POST /api/reindex` with webhook secret.

To add offline topics: edit `Nila-Offline/nila-avatar/content/en/`, update `OFFLINE_QUERIES` in `lib/offlineCache.ts`, optionally `knowledgeReplies.ts`, bump `DB_VERSION`, re-warm cache.

---

## Subproject READMEs

| Document | Contents |
|----------|----------|
| [Nila-backend/README.md](Nila-backend/README.md) | Supabase schema, avatar live, tunnels, API reference, troubleshooting |
| [Nila-Offline/README.md](Nila-Offline/README.md) | Offline product overview |
| [Nila-Offline/nila-avatar/README.md](Nila-Offline/nila-avatar/README.md) | PWA, cache, URL params, extending KB |
| [Nila-Agents/README.md](Nila-Agents/README.md) | Quick start for live agents |
| [Nila-backend/docs/FRONTEND_AVATAR_LIVE.md](Nila-backend/docs/FRONTEND_AVATAR_LIVE.md) | Avatar WebSocket protocol for frontends |
| [Nila-backend/docs/FRONTEND_LIVE_ELEVEN.md](Nila-backend/docs/FRONTEND_LIVE_ELEVEN.md) | ElevenLabs live voice integration |

---

## License

GIC / Visioneers buildathon project. Each subrepository may carry its own license terms — check individual repos before redistribution.
