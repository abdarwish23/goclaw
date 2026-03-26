# GoClaw Gateway Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                              GoClaw Multi-Tenant AI Agent Gateway                           │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌─────────────────────────────────────────┐     ┌──────────────────────┐
│   CHANNELS   │     │            GATEWAY CORE                 │     │    LLM PROVIDERS     │
│   (Input)    │     │                                         │     │                      │
│              │     │  ┌─────────────────────────────────┐    │     │  ┌────────────────┐  │
│  ┌────────┐  │     │  │    WebSocket RPC (v3)           │    │     │  │   Anthropic    │  │
│  │Telegram│──┼────►│  │    + HTTP API Server            │    │     │  │   (native)     │  │
│  └────────┘  │     │  └──────────────┬──────────────────┘    │     │  └────────────────┘  │
│  ┌────────┐  │     │                 │                       │     │  ┌────────────────┐  │
│  │Discord │──┼────►│  ┌──────────────▼──────────────────┐    │     │  │    OpenAI      │  │
│  └────────┘  │     │  │    Method Router + RBAC          │    │     │  │  (compat)      │  │
│  ┌────────┐  │     │  │    (owner/admin/operator/viewer) │    │     │  └────────────────┘  │
│  │WhatsApp│──┼────►│  └──────────────┬──────────────────┘    │     │  ┌────────────────┐  │
│  └────────┘  │     │                 │                       │────►│  │  DashScope     │  │
│  ┌────────┐  │     │  ┌──────────────▼──────────────────┐    │     │  │  (Qwen)        │  │
│  │ Feishu │──┼────►│  │    Agent Loop                    │    │     │  └────────────────┘  │
│  │ / Lark │  │     │  │    think → act → observe         │    │     │  ┌────────────────┐  │
│  └────────┘  │     │  │    auto-summarize at 75% ctx     │    │     │  │  OpenRouter    │  │
│  ┌────────┐  │     │  └──────────────┬──────────────────┘    │     │  └────────────────┘  │
│  │  Zalo  │──┼────►│                 │                       │     │  ┌────────────────┐  │
│  └────────┘  │     │  ┌──────────────▼──────────────────┐    │     │  │  Claude CLI    │  │
│  ┌────────┐  │     │  │    Scheduler (lane-based)        │    │     │  │  (stdio+MCP)   │  │
│  │  Web   │──┼────►│  │    main / subagent / cron        │    │     │  └────────────────┘  │
│  │Dashboard│ │     │  └──────────────┬──────────────────┘    │     │  ┌────────────────┐  │
│  └────────┘  │     │                 │                       │     │  │    ACP         │  │
│  ┌────────┐  │     │  ┌──────────┐  ┌▼─────────┐            │     │  │  (Console)     │  │
│  │HTTP API│──┼────►│  │   Cron   │  │Heartbeat │            │     │  └────────────────┘  │
│  │/v1/chat│  │     │  │  (at/    │  │(periodic │            │     └──────────────────────┘
│  └────────┘  │     │  │  every/  │  │ check-in,│            │
└──────────────┘     │  │  cron)   │  │ checklist│            │     ┌──────────────────────┐
                     │  └──────────┘  │ delivery)│            │     │     DATA LAYER       │
                     │                └──────────┘            │     │                      │
                     │  ┌─────────────────────────────────┐   │     │  ┌────────────────┐  │
                     │  │    Tool Registry                 │   │     │  │ PostgreSQL 18  │  │
                     │  │    fs, exec, web, memory,        │───┼────►│  │ + pgvector     │  │
                     │  │    subagent, MCP bridge           │   │     │  │ (multi-tenant) │  │
                     │  └─────────────────────────────────┘   │     │  └────────────────┘  │
                     │                                         │     │  ┌────────────────┐  │
                     └─────────────────────────────────────────┘     │  │  AES-256-GCM   │  │
                                                                     │  │  (API keys)    │  │
┌──────────────────────────────────────────────────────────────┐     │  └────────────────┘  │
│                        EXTENSIONS                            │     └──────────────────────┘
│                                                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │ MCP Bridge │  │  Skills    │  │  Docker    │             │
│  │ (server +  │  │  (BM25     │  │  Sandbox   │             │
│  │  client)   │  │   search)  │  │  (code)    │             │
│  └────────────┘  └────────────┘  └────────────┘             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  Browser   │  │    TTS     │  │ Knowledge  │             │
│  │ (Rod/CDP)  │  │ (OpenAI,   │  │   Graph    │             │
│  │            │  │  ElevenLab │  │            │             │
│  │            │  │  Edge,Mini)│  │            │             │
│  └────────────┘  └────────────┘  └────────────┘             │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │   i18n     │  │   OAuth    │  │  Tracing   │             │
│  │ (en/vi/zh) │  │            │  │  (OTel)    │             │
│  └────────────┘  └────────────┘  └────────────┘             │
└──────────────────────────────────────────────────────────────┘


## Data Flow

    User Message
         │
         ▼
    ┌─────────┐    connect     ┌──────────┐   authenticate   ┌──────────┐
    │ Channel │───────────────►│ WS/HTTP  │─────────────────►│  RBAC    │
    └─────────┘                │ Server   │                  │  Router  │
                               └──────────┘                  └────┬─────┘
                                                                  │
                                    ┌─────────────────────────────▼──────┐
                                    │         Agent Loop                 │
                                    │                                    │
                                    │  1. Load context (SOUL.md, etc.)   │
                                    │  2. Think (LLM call)               │
                                    │  3. Act (tool calls)               │
                                    │  4. Observe (tool results)         │
                                    │  5. Repeat until done              │
                                    │  6. Auto-summarize if >75% ctx     │
                                    │                                    │
                                    └──────────┬─────────────────────────┘
                                               │
                              ┌────────────────┼────────────────┐
                              ▼                ▼                ▼
                        ┌──────────┐    ┌──────────┐    ┌──────────┐
                        │   LLM    │    │  Tools   │    │  Memory  │
                        │ Provider │    │ (fs,web, │    │(pgvector)│
                        │          │    │  exec,   │    │          │
                        │          │    │  MCP...) │    │          │
                        └──────────┘    └──────────┘    └──────────┘


## Multi-Tenant Isolation

    ┌─────────────────────────────────────────────────┐
    │                   Master Tenant                  │
    │   owner → full cross-tenant access               │
    ├─────────────────────────────────────────────────┤
    │  Tenant A              │  Tenant B               │
    │  ┌─────────────────┐   │  ┌─────────────────┐   │
    │  │ agents           │   │  │ agents           │   │
    │  │ sessions         │   │  │ sessions         │   │
    │  │ skills           │   │  │ skills           │   │
    │  │ cron jobs        │   │  │ cron jobs        │   │
    │  │ heartbeats       │   │  │ heartbeats       │   │
    │  │ memory (vectors) │   │  │ memory (vectors) │   │
    │  │ LLM providers    │   │  │ LLM providers    │   │
    │  └─────────────────┘   │  └─────────────────┘   │
    │                        │                         │
    │  Roles:                │  Roles:                 │
    │  admin/operator/viewer │  admin/operator/viewer   │
    └─────────────────────────────────────────────────┘

    Every table has tenant_id column → complete data isolation


## Heartbeat System

    ┌──────────────────────────────────────────────────┐
    │              Heartbeat Ticker                     │
    │                                                  │
    │  1. Poll agent_heartbeats WHERE due              │
    │  2. Check active_hours + timezone                │
    │  3. Load HEARTBEAT.md checklist                  │
    │  4. Run agent loop (isolated session)            │
    │  5. If response < ack_max_chars → suppress       │
    │  6. Otherwise → deliver to channel/chat_id       │
    │  7. Log run (status, tokens, duration)           │
    │  8. Schedule next_run_at + stagger offset        │
    │                                                  │
    │  Config: interval (min 5m), retries, light_ctx   │
    └──────────────────────────────────────────────────┘


## Deployment (Docker Compose)

    ┌─────────────────────────────────────────────────┐
    │                Docker Network                    │
    │                                                  │
    │  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
    │  │ goclaw   │  │ postgres │  │  goclaw-ui   │  │
    │  │ :18790   │  │ :5432    │  │  :3000 (web) │  │
    │  │ (Go)     │◄─┤ pgvector │  │  (nginx+SPA) │  │
    │  └──────────┘  └──────────┘  └──────────────┘  │
    │       ▲                            │            │
    │       └────────────────────────────┘            │
    │           WebSocket + HTTP proxy                │
    └─────────────────────────────────────────────────┘

    Env vars: GOCLAW_GATEWAY_TOKEN, GOCLAW_ENCRYPTION_KEY, POSTGRES_PASSWORD
    Restart policy: unless-stopped (survives crashes + reboots)
