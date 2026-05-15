# EZ Pilot

Autonomous AI agent platform. Self-evolving, multi-model, browser-capable.

## Pages

| File | Purpose |
|---|---|
| `index.html` | Landing page (public) |
| `signin.html` | Sign in / sign up / payment gate |
| `ez-pilot.html` | Main agent app (requires auth) |

## Auth Flow

```
signin.html → Firebase Auth (email/phone/Google/Microsoft)
    ↓ success
  Admin or subscribed? → redirect to ez-pilot.html
  Not subscribed?      → payment gate → Stripe → ez-pilot.html
    ↓
ez-pilot.html loads → session guard checks localStorage
  No session? → redirect back to signin.html
  Valid session? → show app
    ↓
Logout → clear session → redirect to signin.html
```

## Engine (Python/FastAPI)

```bash
cd engine
pip install -r requirements.txt
playwright install chromium
cp .env.example .env     # fill in API keys
python main.py            # → http://localhost:5000
```

### Architecture (follows hard rules)

```
engine/
├── main.py                 ← 3 lines. app = create_app()
├── config.py               ← Re-export from services/config
├── tools/                  ← Pure utilities. No deps. No state.
│   ├── helpers.py
│   ├── crypto.py
│   ├── text.py
│   └── validation.py
├── connectors/             ← Dumb API wrappers. No logic.
│   ├── ai.py               ← 11 providers (Anthropic, OpenAI, Gemini, Grok, Mistral, OpenRouter, Ollama, Gemma, Llama3, HuggingFace)
│   ├── messaging.py         ← WhatsApp, Telegram, Slack, Discord
│   ├── redis_conn.py        ← Redis connection (fallback: in-memory)
│   ├── firebase.py          ← Firebase Admin token verify
│   ├── supabase_conn.py     ← Supabase JWT verify + REST
│   └── credential_pool.py   ← Multi-key rotation (Hermes pattern)
├── models/
│   └── schemas.py           ← All dataclasses + Pydantic models
├── skills/
│   └── registry.py          ← 16 skills + BM25 ranker
├── services/               ← ALL business logic
│   ├── app_factory.py       ← FastAPI app creation + lifespan
│   ├── orchestrator.py      ← 4-layer pipeline (Router→Planner→Executor→Synthesizer)
│   ├── verifier.py          ← 3-layer validation (Structural+Semantic+Execution)
│   ├── memory.py            ← Redis conversation cache + PoEH vector search
│   ├── logic_engine.py      ← Topology snapshots, stress points, causal tags
│   ├── corrector.py         ← Hermes ErrorClassifier (12 types → 6 recovery actions)
│   ├── context.py           ← Hermes ContextEngine (token tracking + compression)
│   ├── cost.py              ← Per-model cost tracking + plan limits
│   ├── optimizer.py         ← Prompt enhancement per skill
│   ├── browser.py           ← Claude Sonnet + Playwright (entropy/heal/judge)
│   ├── career.py            ← Job search + application drafting
│   ├── evolver.py           ← Skill evolution + benchmarks
│   ├── queue.py             ← Redis-backed async task queue
│   ├── tools.py             ← Tool execution with permissions
│   ├── auth.py              ← Dual Firebase + Supabase auth
│   └── config.py            ← pydantic-settings env validation
├── routes/                 ← Thin. Validate → call service → return.
│   ├── chat.py
│   ├── agents.py
│   ├── career.py
│   ├── webhooks.py
│   └── health.py
├── workers/
│   └── queue.py
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── setup.py
├── lint_deps.py            ← Architecture rule enforcer
├── requirements.txt
└── .env.example
```

### Dependency Flow (enforced by lint_deps.py)

```
tools → connectors → services → models → skills → routes → main.py
  ↑ nothing   ↑ tools only   ↑ all below  ↑ nothing  ↑ models  ↑ services+models
```

### Quick Commands

```bash
make setup    # Auto-setup (install deps, create .env, pull models)
make start    # Development server
make prod     # Production (4 workers)
make docker   # Docker deployment
make lint     # Check architecture rules
make check    # Validate providers via health endpoint
```

## API Endpoints

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/health` | No | Provider + skill status |
| POST | `/api/chat` | Yes | AI chat (orchestrator pipeline) |
| POST | `/api/task` | Yes | Task execution |
| POST | `/api/agent/browse` | Max | Browser agent |
| POST | `/api/career/search` | Yes | Job search |
| POST | `/api/career/apply` | Yes | Draft application |
| GET | `/api/logic/stats` | Yes | Logic engine stats |
| GET | `/api/logic/traces` | Admin | Topology snapshots |
| POST | `/api/evolver/benchmark` | Admin | Run benchmark suite |
| POST | `/api/evolver/evolve` | Admin | Trigger skill evolution |
| POST | `/api/webhooks/whatsapp` | No | WhatsApp webhook |
| POST | `/api/webhooks/telegram` | No | Telegram webhook |
| POST | `/api/webhooks/slack` | No | Slack webhook |
| POST | `/api/webhooks/discord` | No | Discord webhook |

## Config

Firebase: `ez-pilot-c321c` (auth + session)
Supabase: `wqimmpagmvymvewnhtyz.supabase.co` (optional DB)
Redis: conversation cache + PoEH + task queue + logic engine traces
