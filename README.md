<div align="center"> <h1>🌌 ARES AEGIS</h1> <h3>A Trading Operating System for Crypto Futures — Build · Validate · Deploy · Scale</h3> <p> <img src="https://img.shields.io/badge/Status-Pre--launch_(paper--only)-yellow.svg" alt="Status" /> <img src="https://img.shields.io/badge/Architecture-Event--Driven_Microservices-blue.svg" alt="Architecture" /> <img src="https://img.shields.io/badge/Venue-Binance_USD--M-f0b90b.svg" alt="Venue" /> <img src="https://img.shields.io/badge/Multi--tenant-SaaS-purple.svg" alt="Multi-tenant" /> <img src="https://img.shields.io/badge/i18n-vi_%7C_en_%7C_zh-lightgrey.svg" alt="i18n" /> </p>
<a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Ares%20Aegis%201%20image.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Ares%20Aegis%201%20image.bmp?raw=true" alt="Ares Aegis Live Terminal" width="900" style="max-width: 100%;"></a>

<p><em>Turn a trading idea into a running, risk-managed system — and own it, from the first rule to the marketplace.</em></p> </div>
📑 Table of Contents
What Ares Aegis Is
Project Status
The Strategy Lifecycle
System Architecture
Market Data & Fact Engine
Decision Layer
Execution & Risk
Platform: Builder, Marketplace, Community
AI Integration
Engineering Discipline
Technology Stack
Contact
🎯 What Ares Aegis Is
Ares Aegis is a multi-tenant Trading Operating System for crypto perpetual futures. It gives each user one environment to create a strategy, validate it on historical data, run it on paper in real time, deploy it live on their own exchange account, and — if they choose — publish it to a marketplace while keeping full ownership of the IP.

Ares Aegis does not sell signals and does not give financial advice. It sells infrastructure and an operating environment. Users connect their own exchange API keys; their capital stays on their own exchange account.

⚠️ Notice: This repository is a public showcase of the architecture, UI and capabilities. The production codebase is closed-source.

🚦 Project Status (September 2026)
Honesty is a design rule of this project, so the status comes first:

Area	Status
Platform	Pre-launch. Running on a live server for building and QA only — no real users, no real-money orders.
Trading modes	Paper and Live (there is no testnet mode). All current activity is paper.
Exchange	Binance USD-M Futures — the live execution path is complete end-to-end. Bybit adapter is in development (position mode, API-key permission checks and fee/min-notional rules must pass first).
Opening LIVE	Gated behind pre-launch blockers (price-freshness checks on SL/TP/sizing, wallet-balance derivation, and others) and a compliance sign-off (geo / KYC / ToS).
Copy-trade engine, AI Quant Lab, rental billing cron, event automation	Code exists; services are switched off until their launch gate.
🔁 The Strategy Lifecycle
Every subsystem serves one stage of the strategy lifecycle:

DATA / FACTS → CREATE → VALIDATE → PAPER → LIVE → MONITOR / RISK
                                                        ↓
                                     PUBLISH → MARKETPLACE (share · rent · copy)
Trust is earned by moving through three verified tiers, each backed by recorded telemetry rather than marketing claims:

Verified Backtest — historical performance against platform standards: minimum sample, regime coverage, data validity (rules may only use facts that were actually populated), drawdown and equity curve.
Verified Paper Trading — a real-time paper record, which is the real out-of-sample test.
Verified Live Trading — actual fill history.
A strategy's version history records every major change: what changed, when, and whether its risk or trading behaviour changed. The UI never shows a number or confidence level that has not reached the matching verified tier.

🏗️ System Architecture
Ares Aegis runs as Docker microservices that talk over Redis 7 (Streams with consumer groups for the signal pipeline, and Pub/Sub for control and alerts). PostgreSQL is the system of record, and DuckDB is the analytical warehouse.

Service	Role
senses	Market-data ingestion over WebSocket (klines, aggregated trades), plus news and economic-calendar feeds. Fetches each stream once and fans it out to every user.
orderflow	Level-2 order-book tracking and order-flow / footprint features.
brain	The fact builder and decision layer: it turns raw data into normalized market facts and evaluates each user's strategies on them.
commander ×3	The execution fleet, sharded by node. It runs each position's lifecycle: entry, SL/TP, trailing, exits, and portfolio risk.
position-mode-watch	Polls the exchange for each account's position mode (one-way / hedge) so orders never go out in the wrong mode.
warehouse	The single writer that extracts Postgres → DuckDB every 90 s, plus a weekly factor-IC monitor.
backtest-worker	Asynchronous backtests on a read-only DuckDB snapshot.
api	The Flask REST + Socket.IO API.
dashboard	The React SPA.
system-monitor	Host and container metrics.
shadow · ailab · automation · rent-lifecycle	Built, but off until launch (see Status).
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Logs%20Backend.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Logs%20Backend.bmp?raw=true" alt="Backend logs" width="800" style="max-width: 100%;"></a> </div>
📡 Market Data & Fact Engine
Every decision (live, paper or backtest) reads the same canonical market facts, so the backtest measures the same thing the live bot trades on.

Fact snapshots are built on a fixed cadence, stamped with an epoch, and written to Postgres, then extracted into DuckDB. Read paths filter by epoch, so data from different fact definitions is never mixed.
Freshness and completeness are tracked per fact. A strategy may only use facts that were actually populated for the period it is tested on.
Live ⇄ backtest parity is treated as an invariant: backtest engine changes are signed off only when results match a reference run bit for bit.
Order-book & order flow (Numba-accelerated)
Temporal order book with @njit (Numba) kernels. It tracks depth over time, not just a single snapshot.
Iceberg / passive-absorption imbalance and liquidity-vacuum detection, i.e. thin zones where price can move fast.
Cancel-rate and aggressive-flow (CVD) metrics.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Ares%20Aegis%202.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Ares%20Aegis%202.bmp?raw=true" alt="Temporal Orderbook" width="800" style="max-width: 100%;"></a> </div>
🧠 Decision Layer
The brain evaluates each user's strategy against the fact snapshot. Strategies combine factors, weights, thresholds and VETO rules, and the built-in analyzers supply the facts they read:

Smart Money Concepts (SMC)
Order Blocks, Fair Value Gaps and premium/discount arrays.
Liquidity sweeps and Change of Character (CHoCH), with higher-timeframe trend context.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Ares%20Aegis%203.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Ares%20Aegis%203.bmp?raw=true" alt="SMC Structure" width="800" style="max-width: 100%;"></a> </div>
Ichimoku (Hosoda) Engine
27-state decision matrix built from price / cloud / line relationships.
Wave theory: N, V, P and Y wave structures.
Time theory: Kihon Suchi, Taito Suchi and projected Henka-bi (inflection dates).
Price projection: V, E, N and NT targets.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Ares%20Aegis%204.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Ares%20Aegis%204.bmp?raw=true" alt="Ichimoku Engine" width="800" style="max-width: 100%;"></a> </div>
Market measurables
Smoothed derivatives of price (Savitzky-Golay) and a market-phase tracker.
The earlier "physics" module (kinetic energy, friction, gravity fields) has been decommissioned: factors that fail validation are removed rather than kept for show.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Supreme%20Oracle.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Supreme%20Oracle.bmp?raw=true" alt="Signal confluence panel (early UI)" width="800" style="max-width: 100%;"></a> <p><sub>Early confluence panel (UI from May 2026).</sub></p> </div>
🛡️ Execution & Risk
Risk is managed centrally, so it protects the account even when a strategy is wrong.

Execution

Optimal Entry Engine, TWAP and smart limit-chase to reduce slippage.
Exchange-native algo orders for SL/TP, and position-mode awareness (one-way vs. hedge) for every order.
Paper exchange that simulates fills from order-book depth, using the same code path as live.
API keys are rejected (fail-closed) if they have withdrawal permission or no IP restriction.
Risk controls

Kelly-fraction sizing capped by a maximum fraction, and driven by an estimated win rate and the expected R:R.
Portfolio drawdown circuit breaker.
Trailing stops: ATR, Kijun-sen, Chandelier and PSAR, with automatic break-even.
Proactive exit, which closes a trade before its stop when the order flow or the structure turns against it.
Per-tier caps on live positions and live capital.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/TSLManager.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/TSLManager.bmp?raw=true" alt="Trade Management" width="800" style="max-width: 100%;"></a> </div>
🧩 Platform: Builder, Marketplace, Community
Strategy Builder (no-code)
A visual rule canvas with a factor picker. You combine facts, weights and VETO rules, then send the strategy straight to an asynchronous DuckDB backtest, where you get an equity curve, drawdown and per-trade diagnostics. Strategies are versioned.

<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/DynamicRuleBuilder.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/DynamicRuleBuilder.bmp?raw=true" alt="Strategy Builder" width="800" style="max-width: 100%;"></a> </div>
Strategy Marketplace
Creators publish validated strategies. Other users can subscribe to or rent them, and the platform takes a commission on rentals. A 1:1 leader → copier copy-trade engine is built and switched off until launch. Creators keep their IP. The marketplace is an extension of the lifecycle, not its core.

Tiers & referral
Access runs on tiers: Guest, Tier 1, Tier 2, Tier 3 and Admin. The platform checks what a user may do in one central place, based on what they own, their tier, compliance status and risk limits. A two-level referral programme pays a share of tier fees.

Communication Hub
One inbox for system notifications, bug reports, feature requests, announcements, strategy discussions, creator broadcasts, direct messages and AI support.

Account & safety
2FA and a security center.
A wallet for platform fees.
Guided onboarding.
Compliance gating: geo policy, KYC and versioned Terms of Service that users must re-accept when they change.
The UI is fully localized in Vietnamese, English and Chinese.
🤖 AI Integration
AI assistant & market briefings. Google Gemini and xAI Grok turn market facts into readable briefings and answer support questions in the Communication Hub. All AI calls go through a server-side proxy with per-tier quotas; no model keys ship to the browser.
AI Strategy Architect. It helps draft a strategy from a plain-language description, and the result then goes through the normal validation path.
AI Quant Lab (offline, currently off). A research pipeline on DuckDB + Polars using XGBoost / LightGBM / CatBoost, with Optuna tuning and SHAP explanations. It is not part of the live decision path until a model clears the same verified tiers as any strategy.
<div align="center"> <a target="_blank" rel="noopener noreferrer" href="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/blob/main/Ares%20Aegis%205.bmp?raw=true"><img src="https://github.com/cuongnpv-create/-Ares-Aegis-Institutional-Trading-Platform/raw/main/Ares%20Aegis%205.bmp?raw=true" alt="Ares Aegis UI" width="800" style="max-width: 100%;"></a> </div>
📐 Engineering Discipline
The platform handles other people's money, so the process is designed to catch mistakes before they reach production:

A 7-article engineering constitution. It covers a single source of truth, entities with declared lifecycles, capability over role, auditable risky actions, page archetypes, domain leading implementation, and machine enforcement. A rule that no machine can enforce is not a rule.
Review gates. Changes that touch money paths, protected areas or API contracts go through independent review roles (architecture, quant, data integrity, compliance, product/UX). Every ruling is recorded in an append-only ledger.
Risky actions such as delete, liquidate, force-stop, withdraw and disabling 2FA all go through one pipeline: capability check → confirmation → immutable audit log.
CI guards check i18n coverage, UI and route rules, fact epoch filtering, and zero TypeScript errors as a baseline.
🛠️ Technology Stack
Category	Technologies
Backend	Python 3.12, Flask 3 + Flask-SocketIO (eventlet), asyncio services, Pydantic 2
Numerics	Numba (@njit), NumPy, Pandas, SciPy
Messaging & state	Redis 7 (Streams + consumer groups, Pub/Sub, AOF)
Storage	PostgreSQL 15 (system of record), DuckDB (analytics warehouse), Polars
Exchange connectivity	CCXT / CCXT Pro (async WebSockets) — Binance USD-M; Bybit in development
AI / ML	Gemini & Grok (server-side proxy), XGBoost, LightGBM, CatBoost, Optuna, SHAP
Frontend	React 18, TypeScript 5.8, Vite 6, Tailwind CSS 4, Zustand, Socket.IO client
Infrastructure	Docker Compose, Nginx Proxy Manager (HTTPS)
🤝 Contact
We are open to conversations with prop trading firms, funds and investors who are interested in strategy infrastructure.

Founder / Lead Architect: Cuong Nguyen
Telegram: @cuongnpv
Email: Cuongnpv@gmail.com
⚖️ Disclaimer: Ares Aegis is trading infrastructure and an analytical tool. Nothing in this repository is financial, investment or trading advice. Trading leveraged crypto derivatives carries a high risk and can result in the loss of all capital. Backtest and paper results do not guarantee future performance. The platform is pre-launch and is not yet available to the public.
