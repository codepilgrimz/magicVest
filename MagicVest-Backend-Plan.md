# MagicVest Backend — Complete Functional Documentation

## Table of Contents

1. [Context & Goals](#1-context--goals)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Tech Stack](#3-tech-stack)
4. [External Services & API Keys — Signup Checklist](#4-external-services--api-keys--signup-checklist)
5. [Repo & Folder Structure](#5-repo--folder-structure)
6. [Feature-by-Feature Functional Breakdown](#6-feature-by-feature-functional-breakdown)
7. [REST API Catalog](#7-rest-api-catalog)
8. [Socket.IO Event Contract](#8-socketio-event-contract)
9. [MongoDB Schema](#9-mongodb-schema)
10. [Background Workers (BullMQ)](#10-background-workers-bullmq)
11. [External Data Sources — Per Field](#11-external-data-sources--per-field)
12. [Authentication Flows](#12-authentication-flows)
13. [File Uploads (R2)](#13-file-uploads-r2)
14. [Environment Variables](#14-environment-variables)
15. [Local Development Setup](#15-local-development-setup)
16. [Deployment (Production)](#16-deployment-production)
17. [Seed Strategy](#17-seed-strategy)
18. [Frontend Integration (follow-up scope)](#18-frontend-integration-follow-up-scope)
19. [Implementation Timeline](#19-implementation-timeline)
20. [Critical Files](#20-critical-files)
21. [Verification Plan](#21-verification-plan)

---

## 1. Context & Goals

The current project at [magicvest-app/src/MagicVest.jsx](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx) is a **7,337-line single-file React app with no backend and no network calls**. All data is mocked inline (TOKENS, RUGS, SIGNALS, WHALE_WALLETS, SCAMMERS, LP_POOLS, NOTIFICATIONS, SIMULATED_WALLETS, etc.). There is no persistence — connected wallets, watchlists, custom wallets, and auth state are lost on refresh.

**This project builds a production-shaped Node.js backend that replaces every mock with real data.**

### Scope
- REST API for CRUD and queries (auth, profile, watchlist, custom wallets, scammer reports, etc.).
- Socket.IO for real-time price ticks, rug alerts, whale moves, LP events, notifications, and social-hype updates.
- MongoDB Atlas for persistence with Mongoose ODM.
- BullMQ workers for price ingestion, rug scanning, whale polling, and social hype.
- Seed script so the existing UI works against the real backend from day one.
- External integrations: **CoinGecko, DexScreener, Moralis, GoPlus Security, Helius, Stripe, Cloudflare R2, Google OAuth**.

### Out of scope (this document)
- Frontend refactor — a separate follow-up pass wires the UI to the real endpoints.
- Mobile app.
- Admin dashboard (skeleton admin routes only).

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CLIENT (React / Vite)                           │
│             magicvest-app/src/MagicVest.jsx + src/api/*                  │
└─────────────┬──────────────────────────────────┬────────────────────────┘
              │ HTTPS (REST /api/v1/*)            │ WSS (Socket.IO)
              │                                    │
┌─────────────▼──────────────────────────────────▼────────────────────────┐
│                    API SERVICE  (Express 5 + TypeScript)                 │
│  auth · tokens · signals · whales · lp · holdings · rugs · scammers      │
│  social · notifications · watchlist · billing · uploads · onboarding     │
└──────┬──────────────────────┬─────────────────────────────────┬──────────┘
       │                       │                                 │
       │  Mongoose             │  BullMQ enqueue                 │  io.emit
       │                       │  Redis pub/sub                  │
       ▼                       ▼                                 │
┌──────────────────┐  ┌──────────────────┐                      │
│  MongoDB Atlas   │  │  Redis (Upstash) │                      │
│  18 collections  │  │  queues, cache,  │                      │
│  Time Series x5  │  │  socket adapter, │                      │
│  Change Streams  │  │  SIWE nonce      │                      │
└───────┬──────────┘  └──────────────────┘                      │
        │ Change Streams                                         │
        └──────────────► io.emit to rooms ◄──────────────────────┘
                                ▲
┌───────────────────────────────┴─────────────────────────────────────────┐
│              WORKER SERVICE  (same codebase, WORKER_ROLE=worker)         │
│  priceIngestor · holdersRefresh · pairDiscovery · rugScanner             │
│  whalePoller · socialHype · notificationsFanout · email                  │
└──┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬────┘
   │          │          │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼          ▼          ▼
┌──────┐  ┌────────┐  ┌───────┐  ┌───────┐  ┌────────┐  ┌────────┐  ┌──────┐
│ Coin │  │  Dex   │  │Moralis│  │GoPlus │  │ Helius │  │Stripe  │  │  R2  │
│Gecko │  │Screener│  │ (EVM) │  │  Sec  │  │  (SOL) │  │(billing│  │(files│
└──────┘  └────────┘  └───────┘  └───────┘  └────────┘  └────────┘  └──────┘
```

**Key principles:**
- **DB is the source of truth.** Workers write to Mongo; Change Streams fan out to sockets. Price ticks bypass DB (too noisy).
- **Two processes, one codebase.** API handles HTTP/WS; Worker runs BullMQ consumers. Deployed as separate Railway services sharing env.
- **Redis is multi-purpose**: BullMQ queues, rate limit store, SIWE nonce cache, Socket.IO adapter (so worker emits reach any API instance).
- **All external API calls go through `src/integrations/*` wrappers** with `bottleneck` rate-limits and `opossum` circuit breakers.

---

## 3. Tech Stack

| Layer | Choice | Reason |
|---|---|---|
| Runtime | Node.js 20 LTS | Current LTS, native `--env-file`, stable |
| Language | **TypeScript** (strict, `noUncheckedIndexedAccess`) | Type safety across Mongoose + Socket.IO + BullMQ payloads |
| Web framework | **Express 5** | User's choice; larger ecosystem, straightforward |
| DB | **MongoDB Atlas** with Mongoose | User's choice; schema flexibility for crypto data, Time Series for ticks |
| Real-time | **Socket.IO** with Redis adapter | User's choice; rooms/namespaces fit the feature set |
| Cache / queue / pub-sub | **Redis** (Upstash) | One system for BullMQ + cache + rate limit + socket adapter + SIWE nonces |
| Jobs | **BullMQ** | Retries, delayed jobs, repeatable jobs, bull-board UI |
| Validation | **zod** | Runtime + compile-time types from one source |
| Logging | **pino** + pino-pretty in dev | Fast structured JSON |
| Error tracking | **Sentry** | Free tier is enough for MVP |
| Files | **Cloudflare R2** (S3-compatible), presigned PUT | Free 10 GB, zero egress |
| Billing | **Stripe** | Card on file + subscription |
| Auth | JWT access + refresh, bcrypt, passport-google, SIWE | See §12 |
| Email | **Resend** | 100/day free, good DX |
| Deploy | **Railway** | Free tier, long-lived sockets, simple two-service setup |
| Dev runner | **tsx watch** | Faster than nodemon+ts-node, handles ESM |
| Test | **vitest** | Fast, modern, zero config |

---

## 4. External Services & API Keys — Signup Checklist

### Required to start backend development

| # | Service | Purpose | Free tier | Env vars | Signup |
|---|---|---|---|---|---|
| 1 | **MongoDB Atlas** | Primary DB (18 collections + Time Series) | M0 free 512 MB | `MONGODB_URI` | mongodb.com/cloud/atlas → create cluster → Database Access → Network Access (0.0.0.0/0 for dev) → copy connection string |
| 2 | **Upstash Redis** | BullMQ + cache + socket adapter + SIWE nonce | Free 10k cmds/day | `REDIS_URL` | upstash.com → create database → copy TLS URL |
| 3 | **DexScreener** | Prices, volume, liquidity, mcap, pair age (primary feed) | Free, ~300 req/min | `DEXSCREENER_API_KEY` (optional; higher rate limits if you email them) | docs.dexscreener.com |
| 4 | **CoinGecko** | Price fallback + historical sparkline bootstrap | Demo plan free, 30 calls/min | `COINGECKO_API_KEY` | coingecko.com/en/api/pricing → Demo plan |
| 5 | **Moralis** | ERC20 holders count, EVM token metadata, Streams webhooks for EVM whale tx | Free 40k compute units/day | `MORALIS_API_KEY`, `MORALIS_STREAMS_SECRET` | moralis.io → Web3 APIs → Web3 Streams → copy API key; Streams → create stream pointed at `/api/v1/_webhooks/moralis` |
| 6 | **GoPlus Security** | Rug check flags (honeypot, sellBlocked, mintable, proxy, hiddenOwner, LP locked, taxes) | Free 30 req/min; more with App Key | `GOPLUS_APP_KEY`, `GOPLUS_APP_SECRET` | gopluslabs.io → Get App Key |
| 7 | **Helius** | Solana RPC + webhooks for Solana whale tx tracking | Free 100k credits/day | `HELIUS_API_KEY` | helius.dev → create project |
| 8 | **Cloudflare R2** | Avatar + KYC file storage (S3-compatible, presigned PUT) | Free 10 GB + unlimited egress | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `R2_PUBLIC_URL` | cloudflare.com → R2 → create bucket → API Tokens → Read/Write |
| 9 | **Google Cloud Console** | Google OAuth client for social login | Free | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` | console.cloud.google.com → APIs & Services → Credentials → OAuth 2.0 Client → Web application → Authorized redirect URI `http://localhost:3000/api/v1/auth/google/callback` (add prod URL later) |

### Required before launching to users (not for dev)

| # | Service | Purpose | Free tier | Env vars | Signup |
|---|---|---|---|---|---|
| 10 | **Stripe** | Subscription billing (Pro monthly/yearly), card on file, webhooks | Free; 2.9% + 30¢ per charge | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` | stripe.com → Developers → API keys, Webhooks → listen to `customer.subscription.*` |
| 11 | **Sentry** | Backend error tracking, release health | 5k errors/mo free | `SENTRY_DSN` | sentry.io → new Node project |
| 12 | **Resend** | Transactional email (verify email, password reset, billing receipts) | 3k emails/mo, 100/day free | `RESEND_API_KEY`, `EMAIL_FROM` | resend.com → API Keys; verify your domain |
| 13 | **Railway** | API + worker deploy + logs | $5 credit/mo; ~$20/mo actual MVP | (PaaS env UI) | railway.app → new project from GitHub → two services (api, worker) |

### Deferred (feature flag, implement later)

| Service | Used by | Notes |
|---|---|---|
| **Twitter / X API Basic** ($100/mo) | Social Hype feature beyond seed | Expensive — seed/mock at MVP |
| **Telegram Bot API** | Social Hype | Free but needs bot infra |
| **Etherscan Pro** ($199/mo) | Not needed — Moralis covers ground cheaper | Skip |
| **ClamAV** worker | KYC document virus scan | Only if you expand uploads beyond avatars |

### Minimum set to run `npm run dev` locally
**#1 through #9** above, all on free tiers. Stripe/Sentry/Resend/Railway are deferred until you're ready to deploy or implement billing.

**Estimated monthly cost:**
- Dev: **$0** (everything in free tiers)
- Production soft launch (1–5k users): **~$30–50/mo**
- Growth phase (10k+ users): **~$150–300/mo** (Atlas M10, Moralis paid, CoinGecko Analyst, Resend paid)

---

## 5. Repo & Folder Structure

```
d:/Work/magicvest project/magicvest project/magicvest-app_2/
├── magicvest-app/            # existing frontend — untouched by backend work
└── magicvest-backend/        # NEW
```

### Backend folder tree

```
magicvest-backend/
├── src/
│   ├── app.ts                       # Express app assembly, middleware, route mount
│   ├── server.ts                    # HTTP + Socket.IO boot, graceful shutdown
│   ├── config/
│   │   ├── env.ts                   # zod-validated env loader
│   │   ├── db.ts                    # Mongoose connect + Change Streams helper
│   │   ├── redis.ts                 # ioredis singleton
│   │   └── logger.ts                # pino
│   ├── modules/                     # feature-first
│   │   ├── auth/                    # routes, controller, service, schemas, strategies/{password,google,siwe}
│   │   ├── users/                   # profile, preferences
│   │   ├── tokens/                  # scanner, safe picks, search, profile
│   │   ├── signals/                 # radar
│   │   ├── whales/                  # system + custom tracked wallets
│   │   ├── contracts/               # contract scanner (async job)
│   │   ├── lp/                      # LP monitor
│   │   ├── holdings/                # connected wallets + portfolio
│   │   ├── rugs/                    # rug alerts
│   │   ├── scammers/                # DB + user reports
│   │   ├── social/                  # hype + sentiment feed
│   │   ├── notifications/
│   │   ├── watchlist/
│   │   ├── billing/                 # Stripe
│   │   ├── onboarding/
│   │   └── uploads/                 # R2 presign
│   ├── models/                      # Mongoose schemas (18 — see §9)
│   ├── sockets/
│   │   ├── index.ts                 # io init, JWT auth middleware
│   │   ├── namespaces/              # market, alerts, notifications, social
│   │   ├── changeStreams.ts         # Mongo → socket fan-out
│   │   └── backfill.ts              # reconnect replay via lastEventAt
│   ├── middleware/                  # authenticate, requireSubscription, errorHandler, requestId, rateLimit
│   ├── workers/
│   │   ├── index.ts                 # boots when WORKER_ROLE=worker
│   │   ├── priceIngestor.ts         # DexScreener primary + CoinGecko fallback, every 10s
│   │   ├── holdersRefresh.ts        # Moralis hourly
│   │   ├── pairDiscovery.ts         # DexScreener new-pair scanner, every 30s
│   │   ├── rugScanner.ts            # GoPlus + Moralis heuristics
│   │   ├── whalePoller.ts           # Moralis Streams (EVM) + Helius webhooks (Solana)
│   │   ├── socialHype.ts            # Twitter/Telegram placeholder
│   │   ├── notificationsFanout.ts
│   │   ├── emailWorker.ts
│   │   └── queues.ts                # BullMQ queue definitions
│   ├── integrations/                # thin SDK wrappers
│   │   ├── coingecko.ts
│   │   ├── dexscreener.ts
│   │   ├── moralis.ts
│   │   ├── goplus.ts
│   │   ├── helius.ts
│   │   ├── stripe.ts
│   │   ├── r2.ts
│   │   └── resend.ts
│   ├── lib/                         # jwt, nonce, pagination, cache, errors, bottleneck
│   └── types/express.d.ts           # augments req.user
├── seeds/
│   ├── seed.ts                      # orchestrator: drops + re-inserts in dep order
│   └── data/                        # 14 entity files, canonical-typed
├── scripts/migrate.ts
├── tests/
├── docker-compose.yml               # Mongo + Redis + bull-board for local dev
├── Dockerfile
├── tsconfig.json                    # strict: true, noUncheckedIndexedAccess: true
├── .env.example
├── .env.development
├── .env.production                  # gitignored, lives in Railway
├── package.json
└── README.md
```

**npm scripts:**
- `dev` → `tsx watch src/server.ts`
- `dev:worker` → `WORKER_ROLE=worker tsx watch src/workers/index.ts`
- `build` → `tsc`
- `start` → `node dist/server.js`
- `seed` → `tsx seeds/seed.ts`
- `test` → `vitest`
- `lint` → `eslint src`

---

## 6. Feature-by-Feature Functional Breakdown

For each UI page/feature in MagicVest.jsx, this section documents **what the user sees, what endpoints serve it, which sockets update it live, which workers feed the data, and which external APIs are involved**.

---

### 6.1 Authentication (email/password, Google, Web3 wallet)

**UI** — [MagicVest.jsx:1700-1839](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L1700) `AuthModal`: login/signup tabs, email + password, Google button, 5 wallet provider buttons.

**Endpoints**
- `POST /api/v1/auth/register { email, password, name }` → bcrypt hash, insert `users`, issue access (15m) + refresh (30d) tokens.
- `POST /api/v1/auth/login { email, password }` → bcrypt compare, issue tokens.
- `POST /api/v1/auth/refresh` → reads refresh token from httpOnly cookie, rotates it, returns new access token.
- `POST /api/v1/auth/logout` → revokes refresh token family, clears cookie.
- `GET /api/v1/auth/google` → 302 to Google OAuth (PKCE flow).
- `GET /api/v1/auth/google/callback?code=...` → exchange, upsert user, issue tokens, redirect to FE.
- `POST /api/v1/auth/wallet/nonce { address }` → generate random nonce, store in Redis 5min TTL.
- `POST /api/v1/auth/wallet/verify { address, signature, nonce, chain }` → verify with `ethers.verifyMessage` (EVM) or `tweetnacl` (Solana), upsert user, issue tokens.
- `POST /api/v1/auth/forgot-password { email }` → Resend email with signed reset link.

**Storage** — access token in React state (memory); refresh token in `httpOnly` `Secure` `SameSite=Lax` cookie.

**Collections** — `users`, `refreshTokens`.

**External APIs** — Google OAuth, Resend.

**Socket.IO** — On login, FE opens `/notifications` namespace with JWT in `socket.handshake.auth.token`. Server auto-joins `user:<userId>` room.

---

### 6.2 Onboarding

**UI** — `OnboardingFlow` (not shown in detail; invoked at [MagicVest.jsx:6942](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L6942)). Steps: experience level, chains, concerns, email confirm.

**Endpoint** — `POST /api/v1/onboarding/complete { exp, chains[], concerns[], xp }`.

**Collection** — writes `users.onboarding`.

---

### 6.3 Scanner (main dashboard)

**UI** — [MagicVest.jsx:2292](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L2292) `ScannerPage`. Shows token list (price, age, vol, holders, liq, score, rug, pump, mcap, sparkline), live rug alerts panel, new tokens counter.

**Endpoints**
- `GET /api/v1/tokens?filter=all|safe|radar&sort=-score&cursor=...` → paginated list from `tokens` collection.
- `GET /api/v1/tokens/search?q=PEPE` → by symbol or partial address.
- `GET /api/v1/tokens/:chain/:addr/history?range=24h|7d|30d` → sparkline from `priceTicks` Time Series.
- `GET /api/v1/rug-alerts?limit=5` → recent rug alerts for the side panel.

**Socket.IO rooms joined** — `global:scanner` (throttled `price:batch` every 2s), `global:rug-alerts`.

**Workers feeding it** — `priceIngestor` (10s cadence), `holdersRefresh` (1h), `rugScanner` (new pair + 6h re-scan), `pairDiscovery` (30s).

**External APIs** — DexScreener (primary), CoinGecko (fallback), Moralis (holders), GoPlus (rug flags).

---

### 6.4 Safe Picks

**UI** — [MagicVest.jsx:2676](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L2676) `SafePicksPage`. AI-verified safe tokens with badges.

**Endpoint** — `GET /api/v1/tokens?filter=safe` → `tokens` where `score >= 80`, sorted by score desc.

**Socket.IO** — `global:scanner` room reuses `price:batch`.

---

### 6.5 Signals / Radar

**UI** — [MagicVest.jsx:2843](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L2843) `SignalsPage`. AI-generated pump signals with badges (HIDDEN GEM, TOP PICK, HOT SIGNAL).

**Endpoints**
- `GET /api/v1/signals?hot=true&sort=-score&cursor=...`.
- `GET /api/v1/signals/:id` → detail view.

**Socket.IO room** — `global:signals` → `signal:new` events.

**Worker** — `rugScanner` creates signals when a token crosses a threshold (high score + rising pump + verified). Signals have `expiresAt` TTL (24–72h).

---

### 6.6 Watchlist

**UI** — [MagicVest.jsx:2949](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L2949) `WatchlistPage`. User's tracked tokens with alert toggle. Star icon appears on every token row in Scanner, Safe Picks, Radar, Profile pages (handled by `toggleWatchlist` at [MagicVest.jsx:1902](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L1902)).

**Endpoints**
- `GET /api/v1/watchlist` → list with populated token data.
- `POST /api/v1/watchlist { tokenId }` → add.
- `DELETE /api/v1/watchlist/:tokenId` → remove.
- `PUT /api/v1/watchlist/:tokenId/alert { enabled, thresholds? }` → toggle alerts.

**Socket.IO** — Client emits `watchlist:subscribe`; server auto-joins `token:<sym>:<chain>` rooms for each watched token, so the client receives `price:tick` per token.

**Alert mechanism** — When `priceIngestor` detects `price > thresholds.priceAbove` for a watched token, it enqueues `notifications-fanout` → writes `notifications` doc → Change Stream → `notification:new` in `user:<userId>` room.

---

### 6.7 Whale Tracker

**UI** — [MagicVest.jsx:3048](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L3048) `WhalePage`. Shows system whales + user's custom wallets, portfolio inspector, activity feed, behavior signals.

**Endpoints**
- `GET /api/v1/whales?scope=system|user&tag=smart_money&risk=low` → list.
- `GET /api/v1/whales/:id` → detail with embedded portfolio and recent activity.
- `GET /api/v1/whales/:id/activity?cursor=...` → full activity from `walletActivityLog` Time Series.
- `POST /api/v1/whales/custom { addr, chain, label, tag, notes }` → add user-scoped wallet.
- `PATCH /api/v1/whales/custom/:id` → edit label/notes.
- `DELETE /api/v1/whales/custom/:id` → remove.
- `PUT /api/v1/whales/:id/notifications { enabled }` → per-wallet notification toggle.

**Socket.IO** — Client emits `whale:subscribe { addr }` → joins `wallet:<addr>` room → receives `whale:move` events.

**Workers**
- `whalePoller` registers Moralis Streams (EVM) or Helius webhooks (Solana) for each tracked wallet address.
- Incoming webhook → parse tx → write `walletActivityLog` → Change Stream → emit `whale:move`.
- Portfolio is recomputed when significant position changes (buy/sell above threshold).

**External APIs** — Moralis Streams (EVM), Helius webhooks (Solana).

---

### 6.8 Contract Scanner (AI)

**UI** — [MagicVest.jsx:3407](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L3407) `ContractScannerPage`. User pastes a contract address → animated progress → results: red flags, security checks, AI confidence, holder distribution.

**Endpoints**
- `POST /api/v1/contracts/scan { chain, addr }` → enqueues `rug-scan` job → returns `{ jobId }`.
- `GET /api/v1/contracts/scan/:jobId` → `{ status:"pending"|"done"|"failed", result? }`.
- `GET /api/v1/contracts/:chain/:addr` → cached scan (24h TTL).

**Worker** — `rugScanner` job:
1. Fetch DexScreener pair data (liq, pairCreatedAt, txs).
2. Call GoPlus `/api/v1/token_security/{chain}?contract_addresses=...` → parse flags.
3. Call Moralis for holders count + top holders distribution.
4. Compute composite `score` (weighted: 30% GoPlus risk, 20% LP lock, 20% holders distribution, 15% liquidity, 10% age, 5% social).
5. Write `rugAlerts` if `severity >= warning`.
6. Update `tokens.rug`, `tokens.score`.

**External APIs** — GoPlus Security, Moralis, DexScreener.

---

### 6.9 LP Monitor

**UI** — [MagicVest.jsx:3717](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L3717) `LPMonitorPage`. LP pool health, lock status, lock provider, event timeline.

**Endpoints**
- `GET /api/v1/lp-pools?chain=ETH&search=PEPE` → list.
- `GET /api/v1/lp-pools/:chain/:addr` → detail with lock info.
- `GET /api/v1/lp-pools/:chain/:addr/events?cursor=...` → `lpEvents` Time Series.

**Socket.IO room** — `global:lp-monitor` → `lp:event` events.

**Worker** — `rugScanner` detects LP add/remove/lock/unlock events from on-chain data (Moralis for EVM, Helius for Solana) and writes `lpEvents`. Change Stream fans out.

**External APIs** — Moralis, Helius, GoPlus (for lock verification).

---

### 6.10 Holdings (connected wallets)

**UI** — [MagicVest.jsx:4258](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L4258) `HoldingsPage`. Connect Phantom/MetaMask/Rabby, or manual address entry, view portfolio, P&L, tx history.

**Endpoints**
- `POST /api/v1/wallets/connect { provider, addr, chain, signature }` → verify signature, insert `connectedWallets`.
- `POST /api/v1/wallets/manual { addr, chain, label }` → address-only, read-only mode.
- `DELETE /api/v1/wallets/:id` → disconnect.
- `GET /api/v1/wallets/:id/portfolio` → embedded `holdings[]` with live P&L (price joined from `tokens`).
- `GET /api/v1/wallets/:id/transactions?cursor=...` → `walletTxs` Time Series.

**Worker** — When a wallet is connected, `whalePoller` enrolls the address in Moralis Streams / Helius. Incoming tx → write `walletTxs` → recompute `connectedWallets.holdings[]` snapshot → append daily point to `historyDaily[]`.

**External APIs** — Moralis Streams (EVM), Helius (Solana).

---

### 6.11 Rug Alerts

**UI** — [MagicVest.jsx:5040](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L5040) `RugAlertsPage`. Historical archive of rugs with full detail: timeline, security checks matrix, dev wallet, AI confidence, price impact.

**Endpoints**
- `GET /api/v1/rug-alerts?severity=critical&cursor=...` → cursor pagination.
- `GET /api/v1/rug-alerts/:id` → detail with embedded `timeline[]` and `checks{}`.
- `GET /api/v1/rug-alerts/stats` → aggregate (`$facet` query): `rugsToday`, `totalLost24h`, `honeypots`, `avgTime`, `savedUsers`, `alertsSent`.

**Socket.IO room** — `global:rug-alerts` → `rug:new` events.

**Worker** — `rugScanner` writes `rugAlerts` → Change Stream → broadcast to `global:rug-alerts` + targeted `user:<userId>` rooms whose `prefToggles.rug === true`.

**External APIs** — GoPlus (primary), Moralis (dev wallet lookup), DexScreener (price impact).

---

### 6.12 Scammer Database

**UI** — [MagicVest.jsx:5342](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L5342) `ScammerDBPage`. Browse 2,412+ flagged wallets, filter by type/risk/chain, view scammer profile (activity, connections, flags), submit report.

**Endpoints**
- `GET /api/v1/scammers?risk=critical&type=Rug%20Pull&cursor=...`
- `GET /api/v1/scammers/:addr` → detail with embedded activity/connections/flags.
- `POST /api/v1/scammers/report { addr, type, token?, note, evidence? }` → writes `scammerReports` with status `pending`, rate-limited to 5/hour/user.
- `GET /api/v1/scammers/stats` → aggregate counts.
- Admin: `PATCH /api/v1/admin/reports/:id { status }`.

**Moderation flow** — Reports default `pending`. AI worker auto-promotes high-confidence reports to `investigating`. Admin manually confirms to `confirmed`, which promotes the addr into `scammers` collection.

**External APIs** — none directly; enrichment via GoPlus (malicious address flag) and Moralis (tx history).

---

### 6.13 Social Hype

**UI** — [MagicVest.jsx:6598](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L6598) `SocialHypePage`. Trending tokens by mentions, sentiment feed by source, influencer verdicts.

**Endpoints**
- `GET /api/v1/hype/trending?src=Twitter&trend=viral&sort=-hype` → `socialHype` sorted.
- `GET /api/v1/hype/:sym` → per-token breakdown.
- `GET /api/v1/sentiment/feed?cursor=...` → `sentimentFeed` Time Series.
- `GET /api/v1/influencers` → curated list.

**Socket.IO room** — `global:hype` → `hype:update` every 60s.

**Worker** — `socialHype` worker (placeholder for MVP; seed data drives UI until Twitter/Telegram keys are purchased).

---

### 6.14 Notifications

**UI** — [MagicVest.jsx:6780](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L6780) `NotificationsPage`. Filterable alert center: all/unread/critical/price/whale/security.

**Endpoints**
- `GET /api/v1/notifications?filter=unread&cursor=...`.
- `PUT /api/v1/notifications/:id/read`.
- `PUT /api/v1/notifications/read-all`.
- `PUT /api/v1/users/me/preferences { rug, whale, price, security, ai, community }`.

**Socket.IO** — Auto-joined `user:<userId>` room receives `notification:new` as soon as `notifications` is inserted (Change Stream pattern).

**TTL** — Critical notifications kept 365d, non-critical 30d via `expireAt` index.

---

### 6.15 Profile / Settings

**UI** — User menu around [MagicVest.jsx:7150](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L7150). Avatar upload, name, subscription plan (Pro monthly/yearly), card on file, cancel subscription.

**Endpoints**
- `GET /api/v1/users/me` → full profile.
- `PATCH /api/v1/users/me { name?, avatarIdx?, avatarKey? }`.
- `POST /api/v1/uploads/avatar/sign { mime, size }` → returns R2 presigned PUT URL + key. FE uploads directly to R2, then `PATCH /users/me { avatarKey }`.
- `GET /api/v1/billing/subscription`.
- `POST /api/v1/billing/plan { planId }` → Stripe subscription create/update.
- `PUT /api/v1/billing/card { stripePaymentMethodId }`.
- `POST /api/v1/billing/cancel`.
- `POST /api/v1/billing/webhook` → Stripe events (subscription.updated, invoice.paid, etc.).

**External APIs** — Cloudflare R2, Stripe, Resend (billing receipts).

---

### 6.16 Token Profile (deep-dive page)

**UI** — [MagicVest.jsx:5642](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx#L5642) `ProfilePage`. Collapsible sections: security, sentiment, influencer, on-chain, hype, red flags, whale movements.

**Endpoint** — `GET /api/v1/tokens/:chain/:addr/profile` → aggregated response joining `tokens`, latest `rugAlerts`, `socialHype`, `lpPools`, recent whale moves for the token.

**Socket.IO** — Token-specific: client emits `token:subscribe { sym, chain }` → joins `token:<sym>:<chain>` room → receives `price:tick`.

---

## 7. REST API Catalog

All routes prefixed with `/api/v1`.

| Module | Method + Path | Auth | Purpose |
|---|---|---|---|
| **auth** | POST `/auth/register` | public | email/password signup |
| | POST `/auth/login` | public | email/password login |
| | POST `/auth/refresh` | cookie | rotate refresh token, issue new access token |
| | POST `/auth/logout` | auth | revoke refresh token family |
| | GET `/auth/google` | public | start Google OAuth |
| | GET `/auth/google/callback` | public | OAuth exchange |
| | POST `/auth/wallet/nonce` | public | get SIWE nonce |
| | POST `/auth/wallet/verify` | public | verify signature, issue tokens |
| | POST `/auth/forgot-password` | public | email reset link |
| **users** | GET `/users/me` | auth | full profile |
| | PATCH `/users/me` | auth | name, avatar |
| | PATCH `/users/me/preferences` | auth | notification prefs |
| **onboarding** | POST `/onboarding/complete` | auth | exp/chains/xp |
| **uploads** | POST `/uploads/avatar/sign` | auth | R2 presigned PUT |
| **tokens** | GET `/tokens` | auth | scanner list (filter/sort/cursor) |
| | GET `/tokens/search?q=` | auth | search |
| | GET `/tokens/:chain/:addr` | auth | single token |
| | GET `/tokens/:chain/:addr/profile` | auth | deep-dive aggregate |
| | GET `/tokens/:chain/:addr/history?range=` | auth | sparkline / chart |
| **signals** | GET `/signals` | auth | radar list |
| | GET `/signals/:id` | auth | detail |
| **watchlist** | GET `/watchlist` | auth | user's watchlist |
| | POST `/watchlist` | auth | add token |
| | DELETE `/watchlist/:tokenId` | auth | remove |
| | PUT `/watchlist/:tokenId/alert` | auth | toggle alerts |
| **whales** | GET `/whales` | auth | list (scope/tag/risk filter) |
| | GET `/whales/:id` | auth | detail |
| | GET `/whales/:id/activity` | auth | activity feed (cursor) |
| | POST `/whales/custom` | auth | add user-scoped wallet |
| | PATCH `/whales/custom/:id` | auth | edit |
| | DELETE `/whales/custom/:id` | auth | remove |
| | PUT `/whales/:id/notifications` | auth | toggle notifications |
| **contracts** | POST `/contracts/scan` | auth | enqueue scan job |
| | GET `/contracts/scan/:jobId` | auth | job status |
| | GET `/contracts/:chain/:addr` | auth | cached result |
| **lp** | GET `/lp-pools` | auth | list |
| | GET `/lp-pools/:chain/:addr` | auth | detail |
| | GET `/lp-pools/:chain/:addr/events` | auth | event feed |
| **holdings** | POST `/wallets/connect` | auth | connect via signature |
| | POST `/wallets/manual` | auth | address-only connect |
| | DELETE `/wallets/:id` | auth | disconnect |
| | GET `/wallets/:id/portfolio` | auth | holdings + P&L |
| | GET `/wallets/:id/transactions` | auth | tx log cursor |
| **rugs** | GET `/rug-alerts` | auth | list |
| | GET `/rug-alerts/:id` | auth | detail |
| | GET `/rug-alerts/stats` | auth | aggregate stats |
| **scammers** | GET `/scammers` | auth | list |
| | GET `/scammers/:addr` | auth | detail |
| | POST `/scammers/report` | auth | submit report |
| | GET `/scammers/stats` | auth | aggregates |
| **social** | GET `/hype/trending` | auth | trending tokens |
| | GET `/hype/:sym` | auth | per-token hype |
| | GET `/sentiment/feed` | auth | sentiment feed |
| | GET `/influencers` | auth | curated list |
| **notifications** | GET `/notifications` | auth | list (filter/cursor) |
| | PUT `/notifications/:id/read` | auth | mark read |
| | PUT `/notifications/read-all` | auth | mark all read |
| **billing** | GET `/billing/subscription` | auth | current plan |
| | POST `/billing/plan` | auth | change plan |
| | PUT `/billing/card` | auth | update card |
| | POST `/billing/cancel` | auth | cancel |
| | POST `/billing/webhook` | Stripe sig | Stripe events |
| **webhooks** | POST `/_webhooks/moralis` | Moralis sig | EVM wallet tx events |
| | POST `/_webhooks/helius` | Helius sig | Solana wallet tx events |
| **admin** | PATCH `/admin/reports/:id` | admin | moderate report |
| | GET `/admin/reports` | admin | pending reports |
| **health** | GET `/health` | public | `{ ok: true }` |

### Conventions
- Error envelope: `{ error: { code, message, details?, requestId } }`.
- **Cursor pagination** for time-series (signals, notifications, whale activity, rug alerts): `?cursor=<b64>&limit=50` → `{ data, nextCursor }`.
- **Offset pagination** for stable lists (scammers, LP pools, tokens): `?page=1&limit=20` → `{ data, pagination }`.
- Dates: ISO 8601 UTC. Money: canonical numbers (USD). Formatting is FE responsibility.
- All bodies validated with `zod` schemas attached to each route.
- `X-Request-Id` propagated into pino logs.
- Rate limiting via `express-rate-limit` + Redis: 100/min global, 5/min for auth, 5 reports/hour/user.

---

## 8. Socket.IO Event Contract

### Namespaces
| Namespace | Auth | Purpose |
|---|---|---|
| `/market` | optional | price ticks (public data) |
| `/alerts` | required | rug, LP, signals (public but need to know user for preferences) |
| `/notifications` | required | per-user private |
| `/social` | optional | hype + sentiment |

### Rooms
| Room | Joined | Events delivered |
|---|---|---|
| `user:<userId>` | auto on handshake | `notification:new` |
| `token:<sym>:<chain>` | client emits `token:subscribe` | `price:tick` |
| `wallet:<addr>` | client emits `whale:subscribe` | `whale:move` |
| `global:scanner` | scanner page open | `price:batch` (2s throttle) |
| `global:rug-alerts` | rug alerts page | `rug:new` |
| `global:lp-monitor` | LP page | `lp:event` |
| `global:signals` | radar page | `signal:new` |
| `global:hype` | hype page | `hype:update` (60s) |

### Server → Client events
```ts
price:tick          { sym, chain, price: Number, pct24h: Number, t: Number }
price:batch         { t: Number, items: [{ sym, chain, price, pct24h, vol24h }] }
rug:new             { id, sym, addr, chain, type, severity, desc, lostUsd,
                      aiConfidence, devAddr, verified, t }
whale:move          { walletAddr, action, sym, amtUsd, priceImpactPct, txHash, t }
notification:new    { id, type, title, desc, sym, severity,
                      refCollection, refId, createdAt }
lp:event            { sym, event, severity, type, detail, amtUsd, t }
signal:new          { id, sym, chain, score, rug, pump, priceAt, pctAt,
                      why, badge, hot, t }
hype:update         { t, items: [{ sym, src, mentions, hype, deltaPct, trend }] }
event:ack-backfill  { since, count }
```

### Client → Server events
```ts
token:subscribe      { sym, chain }
token:unsubscribe    { sym, chain }
watchlist:subscribe  // server auto-joins rooms for user's entire watchlist
watchlist:unsubscribe
whale:subscribe      { addr }
whale:unsubscribe    { addr }
rooms:join           { room }
rooms:leave          { room }
notification:read    { id }
```

### Handshake (JWT auth)
```ts
const socket = io(WSS_URL + "/notifications", {
  auth: { token: accessJwt, lastEventAt: localStorage.lastEventAt },
  transports: ["websocket"],
});

// server
io.of("/notifications").use((socket, next) => {
  const { userId } = verifyJwt(socket.handshake.auth.token);
  socket.data.userId = userId;
  socket.data.lastEventAt = socket.handshake.auth.lastEventAt;
  socket.join(`user:${userId}`);
  next();
});
```

### Reconnect backfill (60-second window)
1. Client persists `lastEventAt` to localStorage on every event.
2. On reconnect, includes in handshake.
3. Server queries `rugAlerts`, `lpEvents`, `signals`, `notifications`, `walletActivityLog` for `createdAt > lastEventAt AND createdAt > now-60s`.
4. Replays events in order, emits `event:ack-backfill { since, count }`.
5. If offline > 60s, FE shows "reconnected" banner and refetches via REST.

### Producer pattern (Mongo → socket)
- **MongoDB Change Streams** watch `rugAlerts`, `lpEvents`, `signals`, `notifications`, `walletActivityLog`. The sockets service subscribes to these streams and broadcasts to appropriate rooms.
- **Workers never emit sockets directly** — they only write to Mongo. Keeps invariants in the DB.
- **Price ticks bypass DB** (too noisy): `priceIngestor` → Redis cache + direct `io.of('/market').to('token:...').emit(...)`.
- **Redis adapter** (`@socket.io/redis-adapter`) so worker process emits reach users connected to any API instance.

---

## 9. MongoDB Schema

Canonical types; all display formatting stays on the frontend. `$facet` aggregations compute stats on read.

| # | Collection | Purpose & key fields | Critical indexes |
|---|---|---|---|
| 1 | `users` | auth + profile + `subPlan` + `onboarding{exp,chains,xp}` + `prefToggles` + `avatarKey` (R2) | `{email:1}` UNIQUE |
| 2 | `refreshTokens` | hashed refresh tokens with `userId`, `deviceId`, `expiresAt` | `{userId:1}`, `{expiresAt:1}` TTL |
| 3 | `tokens` | one doc per deployed contract. `sym`, `addr`, `chain`, `price`, `pct24h`, `vol24h`, `holders`, `liq`, `mcap`, `rug/pump/score` 0-100, `verified`, `lpLocked`, `renounced`, `dep`, `deployedAt` | `{chain:1, addr:1}` UNIQUE, `{sym:1}`, `{score:-1}` |
| 4 | `priceTicks` | **Time Series**, timeField `t`, meta `{tokenId,sym,chain}`, 90-day retention | implicit meta |
| 5 | `watchlists` | one doc per (user, token). `alert`, `alertThresholds`, `addedAt` | `{userId:1, tokenId:1}` UNIQUE |
| 6 | `trackedWallets` | unified system whales + user custom. `scope:"system"\|"user"`, `ownerId` (nullable), `addr`, `chain`, `label`, `tag`, `risk`, `portfolio[]`, `activity[]` (last 20) | `{scope:1, addr:1}`, `{ownerId:1}` sparse |
| 7 | `walletActivityLog` | **Time Series** per-wallet whale moves, 1-year. Feeds `whale:move` | meta `{walletAddr}` |
| 8 | `socialHype` | per (tokenId, src). `mentions`, `hype`, `deltaPct`, `trend` | `{tokenId:1, src:1}` UNIQUE, `{hype:-1}` |
| 9 | `sentimentFeed` | **Time Series**, 30-day retention | meta `{src}` |
| 10 | `signals` | AI signals, `hot`, `badge`, `why`, `expiresAt` TTL 24–72h | `{hot:1, score:-1}`, `{expiresAt:1}` TTL |
| 11 | `influencers` | small reference table | `{name:1}` UNIQUE |
| 12 | `scammers` | curated DB. `addr`, `alias`, `type`, `risk`, `stolenUsd`, `victims`, `method`, `status`, `activity[]`, `connections[]`, `flags[]` | `{addr:1}` UNIQUE, `{risk:1, lastActive:-1}` |
| 13 | `scammerReports` | user + AI submissions. `status`, `votes`, `reportAddr` | `{status:1, createdAt:-1}`, `{reportAddr:1}` |
| 14 | `lpPools` | per token LP state. `locked`, `lockPct`, `lockEnd`, `provider`, `health`, `trend` | `{tokenId:1}` UNIQUE, `{health:1, alerts:-1}` |
| 15 | `lpEvents` | **Time Series**, 180-day. `event`, `severity`, `type` | meta `{tokenId}` |
| 16 | `rugAlerts` | embedded `timeline[]` + `checks{}` + `devAddr`, `aiConfidence` | `{severity:1, createdAt:-1}`, `{tokenId:1}`, `{devAddr:1}` |
| 17 | `notifications` | per user. `type`, `severity`, `read`, `refCollection`/`refId`, `expireAt` (critical +365d, else +30d) | `{userId:1, read:1, createdAt:-1}`, `{expireAt:1}` TTL |
| 18 | `connectedWallets` | user's web3 wallets + holdings snapshot + 90-day `historyDaily[]` | `{userId:1, addr:1}` UNIQUE |
| + | `walletTxs` | **Time Series**, user tx log, drives holdings recompute | meta `{walletId,userId}` |

**Design decisions locked:**
- `sym` is NOT globally unique — real PK is `(chain, addr)`.
- `watchlists` uses **one doc per (user,token)** not an array — atomic ops scale past 16MB.
- System whales + user custom wallets share one collection with a `scope` discriminator — same socket room, one index set.
- `notifications` uses a TTL index for free server-side archival.
- Holdings are a snapshot recomputed from `walletTxs` on ingest, not on every read.

---

## 10. Background Workers (BullMQ)

Workers run in a separate process (`WORKER_ROLE=worker`) so API latency is independent of scanning load. `bottleneck` rate-limits outbound calls per integration; `opossum` circuit breaker returns cached data with `stale:true` when a feed is down.

| Queue | Cadence | Source | Writes to |
|---|---|---|---|
| `prices` | every 10s repeatable | DexScreener primary, CoinGecko fallback | `tokens.price/pct24h/vol24h/liq/mcap/pump`, `priceTicks`, Redis cache |
| `holders-refresh` | every 1h per token | Moralis `/erc20/:addr/owners` | `tokens.holders` |
| `pair-discovery` | every 30s | DexScreener new-pair endpoint | `tokens` (new rows), triggers `rug-scan` |
| `rug-scan` | on-demand + 6h re-scan | GoPlus Security + DexScreener + Moralis | `rugAlerts.checks{}`, `tokens.rug/score`, `notifications` fanout |
| `whale-poll` | Moralis Streams (EVM) + Helius webhooks (SOL) + 60s poll fallback | Moralis, Helius | `walletActivityLog`, `trackedWallets.activity`, `notifications` |
| `social-hype` | every 60s | placeholder (Twitter/Telegram deferred) | `socialHype`, `sentimentFeed` |
| `notifications-fanout` | on demand | internal | `notifications` + WS emit |
| `email` | on demand | Resend | transactional email |

---

## 11. External Data Sources — Per Field

For a scanner row (e.g. PEENPE/ETH at $18.56, 32,987 holders, $340M mcap):

| Field | Source | Call pattern |
|---|---|---|
| `price`, `pct24h`, `vol24h`, `liq`, `mcap` | **DexScreener** primary, CoinGecko fallback | `/latest/dex/tokens/{addr}`, polled by `priceIngestor` every 10s |
| `deployedAt` (age) | **DexScreener** `pair.pairCreatedAt` | captured once on discovery |
| `holders` | **Moralis** `/erc20/{address}/owners?limit=1` (reads total from meta) | polled hourly |
| sparkline / history | **Our own `priceTicks`** Time Series; CoinGecko `market_chart` for bootstrap | — |
| `rug` 0-100 + `checks{}` | **GoPlus** `/api/v1/token_security/{chain}` | called on new pair + 6h re-scan |
| `score` (MAGICSCORE) | **Computed in-house** composite | in `rugScanner` after feeds land |
| `pump` 0-100 | **Computed in-house** from DexScreener `txns.h1`, `priceChange.h1/h6/h24` | in `priceIngestor` |
| EVM whale tx | **Moralis Streams** webhook | pushed to `/api/v1/_webhooks/moralis` |
| Solana whale tx | **Helius** webhook | pushed to `/api/v1/_webhooks/helius` |
| Social hype (MVP) | **Seed + mock** | real Twitter/Telegram after MVP |

---

## 12. Authentication Flows

### Email / Password
1. `POST /auth/register { email, password, name }` → bcrypt hash (cost 12) → insert `users`.
2. Server issues access token (15m) + refresh token (30d).
3. Access → returned in JSON body (FE keeps in memory). Refresh → `Set-Cookie` httpOnly Secure SameSite=Lax.
4. Refresh tokens stored **hashed** in `refreshTokens` with `userId`, `deviceId`, `expiresAt`. Rotated on every use; reuse-detection revokes the whole family.

### Google OAuth (Authorization Code + PKCE)
1. `GET /auth/google` → 302 to Google with PKCE code challenge.
2. Google → `GET /auth/google/callback?code=...` → server exchanges code, fetches `userinfo`.
3. Upsert `users` by email; link `googleId`.
4. Issue JWT pair; redirect to FE with success.

### Web3 wallet (SIWE pattern)
1. `POST /auth/wallet/nonce { address }` → random 32-byte nonce, Redis `SET EX 300`, return nonce.
2. FE asks wallet to sign the message: `"MagicVest wants to sign in as {addr}: {nonce}"`.
3. `POST /auth/wallet/verify { address, signature, nonce, chain }` → `ethers.verifyMessage` (EVM) or `tweetnacl` (Solana). Nonce deleted immediately (one-shot). Upsert `users.walletAddresses[]`.
4. Issue JWT pair.

### Token storage on frontend
- **Access token** → in-memory React state only. 15-minute lifetime.
- **Refresh token** → httpOnly Secure SameSite=Lax cookie. Only the `/auth/refresh` endpoint reads it.
- **CSRF** → double-submit token header on mutating requests.
- On app load: FE calls `POST /auth/refresh` → if cookie valid, mints a new access token.

### Middleware
- `authenticate` → verifies `Authorization: Bearer <jwt>`, sets `req.user`.
- `authenticateOptional` → same but no 401.
- `requireSubscription('pro')` → checks `req.user.subPlan`.
- `requireAdmin` → for moderation.
- Socket: `io.use(authMiddleware)` → reads handshake auth token, auto-joins `user:<userId>`.

---

## 13. File Uploads (R2)

**Pattern: presigned PUT (not proxied uploads).**

1. FE: `POST /api/v1/uploads/avatar/sign { mime: "image/png", size: 1048576 }`.
2. Backend validates mime (jpg/png), size (≤ 2MB for avatars, ≤ 10MB for KYC). Generates R2 presigned URL (5 min expiry) with a random key.
3. Response: `{ uploadUrl, key, publicUrl }`.
4. FE PUTs the file directly to R2.
5. FE: `PATCH /users/me { avatarKey: key }` → backend updates `users.avatarKey`.
6. When FE renders profile, it builds the public URL as `${R2_PUBLIC_URL}/${avatarKey}` (or fetches a fresh signed GET for private KYC files).

**Why not Multer/multipart:** presigned offloads bandwidth from our API; R2 handles the upload directly. Multer stays in the repo as a fallback only for server-side image processing if needed later.

---

## 14. Environment Variables

All zod-validated at boot — app crashes if any required value is missing.

```
# Core
NODE_ENV=development|production
PORT=3000
LOG_LEVEL=info
WORKER_ROLE=api|worker|both

# Data stores
MONGODB_URI=mongodb+srv://...
REDIS_URL=rediss://...

# Auth
JWT_ACCESS_SECRET=<random 64 bytes>
JWT_REFRESH_SECRET=<random 64 bytes>
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=30d
COOKIE_DOMAIN=.magicvest.app
FRONTEND_ORIGIN=https://magicvest.app

# Google OAuth
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=https://api.magicvest.app/api/v1/auth/google/callback

# Crypto data
COINGECKO_API_KEY=
DEXSCREENER_API_KEY=                # optional; raises rate limit
HELIUS_API_KEY=
MORALIS_API_KEY=
MORALIS_STREAMS_SECRET=
GOPLUS_APP_KEY=
GOPLUS_APP_SECRET=

# Files
R2_ACCOUNT_ID=
R2_ACCESS_KEY_ID=
R2_SECRET_ACCESS_KEY=
R2_BUCKET=magicvest-uploads
R2_PUBLIC_URL=https://files.magicvest.app

# Billing
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Email
RESEND_API_KEY=re_...
EMAIL_FROM=no-reply@magicvest.app

# Observability
SENTRY_DSN=
```

---

## 15. Local Development Setup

### Prereqs
- Node.js 20 LTS
- Docker Desktop (for local Mongo + Redis + bull-board)
- Git

### Steps

```bash
# 1. Clone and install
cd "d:/Work/magicvest project/magicvest project/magicvest-app_2"
# (this plan creates magicvest-backend/ on implementation)
cd magicvest-backend
npm install

# 2. Start infra
docker compose up -d       # Mongo, Redis, bull-board on ports 27017 / 6379 / 3100

# 3. Env
cp .env.example .env.development
# Fill values from §4 signup checklist. Minimum for dev:
#   MONGODB_URI=mongodb://localhost:27017/magicvest
#   REDIS_URL=redis://localhost:6379
#   JWT_ACCESS_SECRET / JWT_REFRESH_SECRET (openssl rand -hex 64)
#   COINGECKO_API_KEY, MORALIS_API_KEY, GOPLUS_APP_KEY, HELIUS_API_KEY

# 4. Seed
npm run seed
# → inserts ~200 docs: 1 demo user + tokens, whales, signals, LP pools,
#   scammers, notifications, 24h of synthetic priceTicks

# 5. Run API + worker (two terminals)
npm run dev                # terminal 1: API on :3000
npm run dev:worker         # terminal 2: BullMQ consumers

# 6. Smoke check
curl http://localhost:3000/health          # → { ok: true }
curl http://localhost:3000/api/v1/tokens?limit=5   # → seeded tokens

# 7. bull-board (job queue UI)
# http://localhost:3100 → watch prices, rug-scan, whale-poll queues
```

### Demo user (pre-seeded)
- Email: `minatotanakawork@gmail.com`
- Password: set to `demodemo` in seed (change in `seeds/data/demoUser.ts`)
- Has: 5 watchlist items, 1 connected wallet (simulated Phantom), 3 custom tracked wallets, 18 notifications.

### Connecting the frontend locally
- Add to `magicvest-app/.env.local`: `VITE_API_URL=http://localhost:3000/api/v1` and `VITE_WS_URL=ws://localhost:3000`.
- Frontend integration is a separate follow-up task (see §18).

---

## 16. Deployment (Production)

### Services
- **Railway** — two services from one GitHub repo:
  - `api` — start command `node dist/server.js`, `WORKER_ROLE=api`, public domain, `/health` probe.
  - `worker` — start command `node dist/workers/index.js`, `WORKER_ROLE=worker`, no public domain.
  - Both share env via Railway's shared variables.
- **MongoDB Atlas** — M0 free for dev/staging; M10 ($57/mo) for production once holders > 5k.
- **Upstash Redis** — serverless, scales to zero in off-hours.
- **Cloudflare R2** — `magicvest-uploads` bucket; CORS policy allows `FRONTEND_ORIGIN`.
- **Stripe** — production keys + webhook pointed to `https://api.magicvest.app/api/v1/billing/webhook`.
- **Moralis Streams** — webhook endpoint: `https://api.magicvest.app/api/v1/_webhooks/moralis`; HMAC verification with `MORALIS_STREAMS_SECRET`.
- **Helius** — webhook endpoint: `https://api.magicvest.app/api/v1/_webhooks/helius`.

### Why not serverless (Vercel/Netlify)
- Socket.IO needs long-lived TCP connections.
- BullMQ workers need a long-running process.
- Railway/Fly.io/Render all handle this natively.

### Observability
- **pino** → stdout → Railway log aggregator.
- **Sentry** → backend errors, release health.
- **bull-board** → deployed as a separate Railway service with auth (admin-only) for ops.

### Rollout order
1. Deploy `api` with DB seeded; validate REST + auth flows.
2. Deploy `worker`; confirm `prices` queue is consuming.
3. Register Moralis Streams + Helius webhooks.
4. Point frontend at the prod API URL.
5. Enable Stripe webhooks; test subscription creation in Stripe test mode first.

---

## 17. Seed Strategy

Hand-authored Mongoose seed at `magicvest-backend/seeds/seed.ts`. One file per entity in `seeds/data/`, each exporting canonical-typed arrays translated once from the mock arrays in [MagicVest.jsx](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx) at lines 162, 182, 189, 195, 209, 281, 289, 297, 324, 383, 392, 412, 445, 550, 580.

**Why not a JSX parser or JSON dump:** the mocks are display strings (`"$3,380"`, `"+22.4%"`, `"14m ago"`) with inline icon imports. AST parsing is brittle. A hand-authored seed converts strings → `Number`/`Date` once, validates against schemas, and stays clearly versioned.

**Seed order:**
1. `users` (1 demo user `minatotanakawork@gmail.com`).
2. `tokens` (generates ObjectIds; keeps a `symToId` map for later refs).
3. `trackedWallets`, `scammers`, `influencers`, `lpPools` (ref tokens).
4. `signals`, `rugAlerts`, `lpEvents`, `socialHype`, `sentimentFeed`, `recentReports`.
5. `watchlists`, `notifications`, `connectedWallets` (ref both user and tokens).
6. 24h of synthetic `priceTicks` per token.

Gated by `NODE_ENV !== 'production'`. Total ~200 docs, 2–3s on Atlas M0.

---

## 18. Frontend Integration (follow-up scope)

This plan **does not modify the frontend file**. After the backend is running and seeded, a separate pass will:

- Create `magicvest-app/src/api/client.ts` — `fetch` wrapper with base URL + `credentials:'include'` + access-token header + 401 → refresh retry.
- Create `magicvest-app/src/api/socket.ts` — Socket.IO singleton with namespace factory and `lastEventAt` persistence to localStorage.
- Replace top-of-file `const TOKENS = [...]`, `const RUGS = [...]`, etc. (lines 162–669) with hooks that call the client.
- Wire `AuthModal` (lines 1700–1839) to real auth endpoints.
- Persist `wlItems`, `connectedWallets`, `customWallets` via real endpoints instead of `useState`.
- Wire socket events into live-updating components (scanner ticker, rug alerts panel, notifications badge, whale activity feed).

The seed step is designed so the UI can integrate incrementally — any page whose mocks are replaced with API calls immediately works against the real backend.

---

## 19. Implementation Timeline

**7 days, aggressive — full-time focus, AI-assisted coding, no context switching.**

| Day | Milestone | Deliverables |
|---|---|---|
| **1** | Bootstrap + Auth + Schema + Seed | Express 5 + TS strict + Mongoose + Redis + env validation + pino + `/health`. Auth module complete (register/login/refresh/logout, bcrypt + JWT with refresh rotation, `authenticate` middleware). All 18 Mongoose schemas defined. `seeds/seed.ts` + 14 data files running — DB has ~200 seeded docs. Frontend is now unblocked to integrate. |
| **2** | REST CRUD for all entities | Routes + controllers + zod schemas for: tokens (scanner/search/profile/history), signals, watchlist, whales (system + custom), LP pools, rug alerts, scammers (+ report submission), notifications, social hype, sentiment feed, influencers. Cursor pagination helper in `lib/pagination.ts` reused across all list endpoints. Error envelope + requestId middleware + X-Request-Id logs. |
| **3** | Socket.IO + Change Streams | `io` server + Redis adapter + JWT handshake middleware + 4 namespaces (`/market`, `/alerts`, `/notifications`, `/social`) + room helpers. Change Stream watchers on `rugAlerts`, `lpEvents`, `signals`, `notifications`, `walletActivityLog` → fan out to rooms. Reconnect backfill (`lastEventAt`) working. Manual `POST /_test/trigger-notif` dev-only route. |
| **4** | BullMQ + priceIngestor + rugScanner | BullMQ queues (`prices`, `holders-refresh`, `pair-discovery`, `rug-scan`, `whale-poll`, `social-hype`, `notifications-fanout`, `email`). Worker process boots via `WORKER_ROLE=worker`. `priceIngestor` pulls DexScreener every 10s (CoinGecko fallback), writes `tokens` + `priceTicks`, emits `price:batch` to `global:scanner`. `rugScanner` calls GoPlus + Moralis, writes `rugAlerts` + `tokens.score`. bull-board admin UI mounted on `/admin/queues` (basic auth). |
| **5** | OAuth + SIWE + Uploads + Moralis/Helius webhooks | Google OAuth (Authorization Code + PKCE) via `passport-google-oauth20`. SIWE wallet auth (nonce in Redis + `ethers.verifyMessage` EVM / `tweetnacl` Solana). R2 presigned PUT uploads for avatar (`POST /uploads/avatar/sign`). `holdersRefresh` worker (Moralis hourly). `whalePoller` webhook endpoints `/_webhooks/moralis` + `/_webhooks/helius` with HMAC verification, writing `walletActivityLog`. |
| **6** | Billing + Admin + Rate limits | Stripe integration (`/billing/*` routes + `/billing/webhook` for subscription lifecycle). `requireSubscription('pro')` gate on paid features. Scammer report moderation admin routes + `requireAdmin` middleware. `express-rate-limit` + Redis store applied per route group (5/min auth, 5/hr reports, 100/min global). Resend-based email worker (verification, password reset, billing receipts). |
| **7** | Deploy + Observability + E2E smoke | Sentry wired up (server + worker). Dockerfile + `railway.json` for two services (api + worker). Deploy to Railway with Atlas M0 + Upstash Redis + R2 bucket. Register Moralis Streams + Helius webhooks at production URLs. Run verification plan §21 end-to-end. Document any deferred items in README. |

### What's deferred past Day 7 (not blockers)
- **Social Hype worker** — seed data drives UI; wire Twitter/Telegram later.
- **Contract scanner async job UI polish** — synchronous GoPlus call works for MVP.
- **Admin dashboard UI** — admin routes exist; full dashboard is post-MVP.
- **KYC uploads** — only avatar uploads shipped; KYC schema ready, routes stubbed.
- **Full test suite** — critical-path vitest tests written; full coverage post-MVP.

### Assumptions that make 7 days feasible
1. All API keys in §4 ready on Day 1 (MongoDB, Redis, Moralis, GoPlus, Helius, CoinGecko, R2, Google OAuth).
2. No infra detours (no custom Dockerfiles beyond a standard Node 20 image, no Kubernetes, no self-hosted Redis).
3. Heavy use of boilerplate: feature module template (route + controller + service + schema + test) reused across 14 modules.
4. Mock data seed is the integration-testing substrate — frontend integrates continuously instead of waiting.
5. Frontend integration is **parallel work** (separate follow-up scope in §18), not in this 7-day budget.

### Risk flags
- **GoPlus rate limits** may throttle `rugScanner` on new pair bursts — `bottleneck` queue at 20/min + 24h Redis cache on results is mandatory from Day 4.
- **Moralis Streams webhook delivery** can lag in free tier — keep 60s polling fallback in `whalePoller`.
- **Atlas M0 connection cap (500)** could bite if both API and worker open many connections — use a shared Mongoose connection pool with `maxPoolSize: 10` per process.
- **First-time Chromium/Puppeteer install** for any PDF-generating features will fail behind strict firewalls — not needed for core backend, but noted.

---

## 20. Critical Files

**To be created (plan anchors):**
- [magicvest-backend/src/app.ts](magicvest-backend/src/app.ts) — Express + middleware + route mount
- [magicvest-backend/src/server.ts](magicvest-backend/src/server.ts) — HTTP + Socket.IO server, graceful shutdown
- [magicvest-backend/src/config/env.ts](magicvest-backend/src/config/env.ts) — zod env validation
- [magicvest-backend/src/models/](magicvest-backend/src/models/) — 18 Mongoose schemas
- [magicvest-backend/src/modules/auth/auth.service.ts](magicvest-backend/src/modules/auth/auth.service.ts) — password + JWT + SIWE
- [magicvest-backend/src/sockets/index.ts](magicvest-backend/src/sockets/index.ts) — io init + JWT handshake + namespaces
- [magicvest-backend/src/sockets/changeStreams.ts](magicvest-backend/src/sockets/changeStreams.ts) — Mongo → socket fan-out
- [magicvest-backend/src/sockets/backfill.ts](magicvest-backend/src/sockets/backfill.ts) — reconnect replay
- [magicvest-backend/src/workers/priceIngestor.ts](magicvest-backend/src/workers/priceIngestor.ts) — DexScreener + CoinGecko poller
- [magicvest-backend/src/workers/holdersRefresh.ts](magicvest-backend/src/workers/holdersRefresh.ts) — Moralis ERC20 owners poll
- [magicvest-backend/src/workers/rugScanner.ts](magicvest-backend/src/workers/rugScanner.ts) — GoPlus → `checks{}` → composite score
- [magicvest-backend/src/workers/whalePoller.ts](magicvest-backend/src/workers/whalePoller.ts) — Moralis Streams (EVM) + Helius (Solana)
- [magicvest-backend/src/integrations/goplus.ts](magicvest-backend/src/integrations/goplus.ts) — GoPlus client + flag→score mapper
- [magicvest-backend/src/integrations/moralis.ts](magicvest-backend/src/integrations/moralis.ts) — Moralis REST + Streams webhook handler
- [magicvest-backend/seeds/seed.ts](magicvest-backend/seeds/seed.ts) — seed orchestrator

**Source of truth for entity shapes (do not modify):**
- [magicvest-app/src/MagicVest.jsx](d:/Work/magicvest project/magicvest project/magicvest-app_2/magicvest-app/src/MagicVest.jsx) — mock arrays at lines 162, 182, 189, 195, 209, 281, 289, 297, 324, 383, 392, 412, 445, 550, 580

---

## 21. Verification Plan

### Local dev bring-up
1. `cd magicvest-backend && docker compose up -d` (Mongo + Redis + bull-board).
2. `cp .env.example .env.development` and fill in keys.
3. `npm install && npm run seed` — verify ~200 docs inserted, log counts per collection.
4. `npm run dev` (API on :3000) + `npm run dev:worker` (second terminal).
5. `curl http://localhost:3000/health` → `{ ok: true }`.

### Auth & REST smoke
6. `POST /api/v1/auth/register` then `/auth/login` → receive access token + refresh cookie.
7. `GET /api/v1/tokens?filter=safe` (authenticated) → seeded tokens with `score >= 80`.
8. `POST /api/v1/watchlist { tokenId }` then `GET /api/v1/watchlist` → item present.
9. `POST /api/v1/scammers/report { ... }` → returns `{ status: "pending" }`.

### Socket.IO
10. Connect with JWT in handshake auth → `/notifications` namespace → server emits `notification:new` from a manual dev-only `POST /_test/trigger-notif` → client receives it.
11. Emit `token:subscribe` for a seeded symbol → `priceIngestor` tick → client receives `price:tick`.
12. Disconnect 30s, reconnect with `lastEventAt` → receive `event:ack-backfill` and any missed notifications.

### Workers
13. Check `priceTicks` collection grows ~1 doc per token per 10s; `tokens.holders` updates hourly.
14. Block DexScreener in hosts file → CoinGecko fallback kicks in → `GET /tokens` keeps serving live data.
15. `POST /api/v1/contracts/scan` for a known scam token → `rugScanner` calls GoPlus → `rugAlerts.checks{}` populated (honeypot/mintable/proxy/sellBlocked/hiddenOwner).
16. Use Moralis Streams dashboard to fire a simulated EVM tx for a tracked wallet → webhook hits `/api/v1/_webhooks/moralis` → `walletActivityLog` insert → `whale:move` emitted to `wallet:<addr>` room.

### Seed parity with UI
17. Point frontend at `http://localhost:3000/api/v1` via a minimal API client → scanner page renders with the same 18 tokens currently hardcoded at line 162.

### Production deploy
18. Railway: API service reports healthy; worker service reports consuming `prices`, `holders-refresh`, `rug-scan`, `whale-poll` queues.
19. Sentry receives a deliberately thrown error from `/api/v1/_debug/error` (dev-only).
20. Atlas shows read/write ops on expected collections and TTL index pruning `notifications` after 30d.
21. Moralis Streams webhook endpoint registered and receives events; GoPlus calls stay within free-tier quotas (logged via bottleneck stats).
