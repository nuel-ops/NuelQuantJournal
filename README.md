NUELTRADES AI - all source files (package-lock.json omitted; npm install regenerates it).
Each file starts with a line: ===== FILE: <path> =====
Create each file at that path inside a folder named nueltrades-ai, then run: npm install && npm run db:migrate && npm run dev

===== FILE: .env.example =====
# PostgreSQL connection string. Create the database first.
DATABASE_URL="postgresql://nueltrades:CHANGE_ME@localhost:5432/nueltrades"

# Where uploaded chart images are stored (private; served only through authenticated routes).
# Must be a persistent volume in production.
UPLOAD_DIR="./storage/uploads"

# Max upload size in megabytes.
MAX_UPLOAD_MB="8"

# Public base URL of the app (used for password reset links).
APP_URL="http://localhost:3000"

# Optional AI (trade reviews and coaching). Without both values the AI features show "not configured".
# Server-side only; never exposed to the browser. AI_MODEL must be a model ID your Anthropic account can use.
ANTHROPIC_API_KEY=""
AI_MODEL=""

===== FILE: .gitignore =====
node_modules
.next
storage
.env
*.tsbuildinfo
next-env.d.ts

===== FILE: Dockerfile =====
FROM node:22-slim AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-slim AS run
WORKDIR /app
ENV NODE_ENV=production PORT=3000 UPLOAD_DIR=/data/uploads
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/drizzle ./drizzle
COPY --from=build /app/scripts ./scripts
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./package.json
VOLUME /data
EXPOSE 3000
# Apply migrations, then start. Fails loudly if the database is unreachable.
CMD ["sh", "-c", "npx tsx scripts/migrate.ts && node server.js"]

===== FILE: README.md =====
# NUELTRADES AI (phase 1)

Trading journal and analytics platform: Next.js 15 + TypeScript, PostgreSQL via Drizzle ORM, private on-disk chart storage.

## What works (tested)
**MT5 sync (server side tested, EA not yet run on a live terminal):** per-account revocable connector keys (hashed), heartbeat and position/deposit ingestion with idempotent de-duplication, open positions updated when they close, login/currency mismatch flagged as "Action required", status shown as Connected only after a verified heartbeat and Disconnected after 15 minutes of silence, sync log per account. The read-only Expert Advisor source is in `connectors/mt5/` (see its README); it has **not been compiled or run against a real MT5 terminal here**, so test it on a demo account first.

Registration/login/logout, per-user data isolation, accounts (live/demo/prop with prop rules), separate Real/Demo/Backtest/Prop/Strategy/custom journals, manual trades, Notion-style pages (folders, templates, tags, search, calendar, favorite, duplicate, archive, markdown import/export, autosave), chart screenshots and TradingView links, central chart library with filters, idempotent CSV statement import with preview and column mapping, decimal-safe analytics, user-defined risk thresholds with warnings, data export, account deletion.

## Also built (tested)
- **TradingView webhook + Signal center:** per-user revocable secret, JSON alert receiver (`/api/webhooks/tradingview`), validation, rate limiting, duplicate/replay protection (24h window), stored signals compared with your real trades. It stores only what each alert sends.
- **AI trade review and AI coach:** server-side calls to the Anthropic Messages API. Set `ANTHROPIC_API_KEY` and `AI_MODEL`; without them the UI says AI is not configured and nothing is generated. Statistics are computed in code; the model only explains them; the coach refuses to answer with fewer than 10 closed trades. **The real provider call has not been exercised here (tests use a test double), so try it once with your key.**

## NOT built (be aware)
- **Other brokers and MT4:** no direct integration (Exness, FBS, HFM expose no public trade-history API; use MT5 + the connector, or CSV import). Those accounts are labelled "Manual tracking" or "Manual import only".
- **AI chart-image analysis (vision), SMT divergence engine, automated backtesting, PDF reports, HTML statement import, Google sign-in.**
- **Password-reset email:** no email provider is bundled. An administrator issues a link: `npm run admin:reset-link -- user@example.com`.
- Rate limiting is in-memory (single instance). Weekly/monthly drawdown and prop-limit tracking are not computed. Statement times are read as UTC.
- Journal board/gallery views and custom fields are not implemented (tree, list and calendar are).

## Run locally
Requires Node 22+ and PostgreSQL 14+.
```
cp .env.example .env        # set DATABASE_URL
npm install
npm run db:migrate
npm run dev                 # http://localhost:3000
```
## Test / build
```
npm test            # isolated in-process PostgreSQL (PGlite) with the real migration
npm run typecheck
npm run build && npm start
```
## Deploy
`docker compose up --build` (set `POSTGRES_PASSWORD` in `.env`). Migrations run on start. Mount `/data/uploads` on a persistent volume and run behind HTTPS (cookies are Secure in production). The Docker build has not been run in this environment.

## Backup
`pg_dump $DATABASE_URL > backup.sql` plus a copy of the uploads volume; restore with `psql` and by restoring the volume. Back up both together.

## Troubleshooting
- 503 "database unavailable": check `DATABASE_URL` and that Postgres is reachable; retrying is safe.
- Uploads fail with 413/415: files must be PNG/JPEG/WebP under `MAX_UPLOAD_MB`.
- Import shows "Map these required columns": pick the matching columns in step 2.

See `docs/API.md` and `docs/ANALYTICS.md`.

===== FILE: connectors/mt5/NuelTradesConnector.mq5 =====
//+------------------------------------------------------------------+
//| NuelTradesConnector.mq5                                          |
//| READ-ONLY connector: sends account info and trade history to     |
//| your NUELTRADES AI server. It never places, modifies or closes   |
//| orders and never reads or sends your trading password.           |
//+------------------------------------------------------------------+
#property copyright "NUELTRADES AI"
#property version   "1.00"
#property strict

input string ApiUrl          = "https://your-server.example.com"; // Base URL, no trailing slash
input string ApiKey          = "";                                // Connector key from the Accounts page (starts with ntc_)
input int    SyncIntervalSec = 60;                                // How often to sync
input int    HistoryDays     = 90;                                // History window sent on every sync (re-sending is safe: the server de-duplicates)

#define BATCH 200

struct PosAgg
  {
   ulong    id;
   string   symbol;
   bool     isBuy;
   double   volIn, volOut;
   datetime tIn, tOut;
   double   pIn, pOut, sl, tp;
   double   profit, commission, swap;
   bool     hasIn, hasOut;
  };

//--- helpers ---------------------------------------------------------
string Iso(datetime serverTime)
  {
   // History times are broker-server time; convert to UTC for the server.
   long offset = (long)(TimeTradeServer() - TimeGMT());
   string s = TimeToString((datetime)((long)serverTime - offset), TIME_DATE | TIME_SECONDS); // 2025.03.03 08:00:00
   StringReplace(s, ".", "-");
   StringReplace(s, " ", "T");
   return s + "Z";
  }

string Num(double v) { return DoubleToString(v, 8); }

string Esc(string s)
  {
   StringReplace(s, "\\", " ");
   StringReplace(s, "\"", " ");
   StringReplace(s, "\n", " ");
   StringReplace(s, "\r", " ");
   return s;
  }

bool Post(const string path, const string json, string &response)
  {
   char data[], result[];
   string rh;
   int n = StringToCharArray(json, data, 0, WHOLE_ARRAY, CP_UTF8);
   if(n > 0) ArrayResize(data, n - 1); // drop terminating zero
   string headers = "Content-Type: application/json\r\nAuthorization: Bearer " + ApiKey + "\r\n";
   ResetLastError();
   int code = WebRequest("POST", ApiUrl + path, headers, 15000, data, result, rh);
   if(code == -1)
     {
      int err = GetLastError();
      PrintFormat("NuelTrades: WebRequest failed (error %d). Add %s to Tools > Options > Expert Advisors > Allow WebRequest for listed URL.", err, ApiUrl);
      return false;
     }
   response = CharArrayToString(result, 0, WHOLE_ARRAY, CP_UTF8);
   if(code < 200 || code >= 300)
     {
      PrintFormat("NuelTrades: server rejected %s (HTTP %d): %s", path, code, response);
      return false;
     }
   return true;
  }

//--- sync steps --------------------------------------------------------
bool Heartbeat()
  {
   string json = StringFormat("{\"login\":\"%I64d\",\"server\":\"%s\",\"currency\":\"%s\",\"balance\":%s,\"equity\":%s,\"platform\":\"MT5\"}",
                              AccountInfoInteger(ACCOUNT_LOGIN), Esc(AccountInfoString(ACCOUNT_SERVER)), AccountInfoString(ACCOUNT_CURRENCY),
                              Num(AccountInfoDouble(ACCOUNT_BALANCE)), Num(AccountInfoDouble(ACCOUNT_EQUITY)));
   string resp;
   return Post("/api/connector/heartbeat", json, resp);
  }

string PositionJson(const PosAgg &p, bool closed)
  {
   string s = StringFormat("{\"ticket\":\"%I64u\",\"symbol\":\"%s\",\"direction\":\"%s\",\"lots\":%s,\"openedAt\":\"%s\",\"entryPrice\":%s,\"stopLoss\":%s,\"takeProfit\":%s,\"profit\":%s,\"commission\":%s,\"swap\":%s",
                           p.id, Esc(p.symbol), p.isBuy ? "BUY" : "SELL", Num(p.volIn), Iso(p.tIn), Num(p.pIn), Num(p.sl), Num(p.tp),
                           Num(p.profit), Num(p.commission), Num(p.swap));
   if(closed)
      s += StringFormat(",\"closedAt\":\"%s\",\"exitPrice\":%s", Iso(p.tOut), Num(p.pOut));
   return s + "}";
  }

bool SendBatch(const string positions, const string flows)
  {
   string json = "{\"positions\":[" + positions + "],\"cashFlows\":[" + flows + "]}";
   string resp;
   bool ok = Post("/api/connector/positions", json, resp);
   if(ok) Print("NuelTrades: synced -> ", resp);
   return ok;
  }

bool SyncHistory()
  {
   datetime from = TimeCurrent() - (datetime)(HistoryDays * 86400);
   if(!HistorySelect(from, TimeCurrent() + 86400))
     {
      Print("NuelTrades: HistorySelect failed");
      return false;
     }
   PosAgg aggs[];
   string flowJson[];
   int total = HistoryDealsTotal();

   for(int i = 0; i < total; i++)
     {
      ulong tk = HistoryDealGetTicket(i);
      if(tk == 0) continue;
      long type = HistoryDealGetInteger(tk, DEAL_TYPE);

      if(type == DEAL_TYPE_BALANCE)
        {
         int n = ArraySize(flowJson);
         ArrayResize(flowJson, n + 1);
         flowJson[n] = StringFormat("{\"ticket\":\"%I64u\",\"amount\":%s,\"occurredAt\":\"%s\",\"comment\":\"%s\"}", tk,
                                    Num(HistoryDealGetDouble(tk, DEAL_PROFIT)), Iso((datetime)HistoryDealGetInteger(tk, DEAL_TIME)),
                                    Esc(HistoryDealGetString(tk, DEAL_COMMENT)));
         continue;
        }
      if(type != DEAL_TYPE_BUY && type != DEAL_TYPE_SELL) continue; // skip credit, bonus, charge, etc.

      ulong pid = (ulong)HistoryDealGetInteger(tk, DEAL_POSITION_ID);
      int idx = -1;
      for(int k = 0; k < ArraySize(aggs); k++) if(aggs[k].id == pid) { idx = k; break; }
      if(idx < 0)
        {
         idx = ArraySize(aggs);
         ArrayResize(aggs, idx + 1);
         aggs[idx].id = pid; aggs[idx].symbol = ""; aggs[idx].isBuy = true;
         aggs[idx].volIn = 0; aggs[idx].volOut = 0; aggs[idx].tIn = 0; aggs[idx].tOut = 0;
         aggs[idx].pIn = 0; aggs[idx].pOut = 0; aggs[idx].sl = 0; aggs[idx].tp = 0;
         aggs[idx].profit = 0; aggs[idx].commission = 0; aggs[idx].swap = 0;
         aggs[idx].hasIn = false; aggs[idx].hasOut = false;
        }
      long entry = HistoryDealGetInteger(tk, DEAL_ENTRY);
      double vol = HistoryDealGetDouble(tk, DEAL_VOLUME);
      aggs[idx].profit     += HistoryDealGetDouble(tk, DEAL_PROFIT);
      aggs[idx].commission += HistoryDealGetDouble(tk, DEAL_COMMISSION) + HistoryDealGetDouble(tk, DEAL_FEE);
      aggs[idx].swap       += HistoryDealGetDouble(tk, DEAL_SWAP);
      if(entry == DEAL_ENTRY_IN)
        {
         if(!aggs[idx].hasIn)
           {
            aggs[idx].hasIn = true;
            aggs[idx].symbol = HistoryDealGetString(tk, DEAL_SYMBOL);
            aggs[idx].isBuy = (type == DEAL_TYPE_BUY);
            aggs[idx].tIn = (datetime)HistoryDealGetInteger(tk, DEAL_TIME);
            aggs[idx].pIn = HistoryDealGetDouble(tk, DEAL_PRICE);
            aggs[idx].sl = HistoryDealGetDouble(tk, DEAL_SL);
            aggs[idx].tp = HistoryDealGetDouble(tk, DEAL_TP);
           }
         aggs[idx].volIn += vol;
        }
      else if(entry == DEAL_ENTRY_OUT || entry == DEAL_ENTRY_OUT_BY || entry == DEAL_ENTRY_INOUT)
        {
         aggs[idx].hasOut = true;
         aggs[idx].volOut += vol;
         aggs[idx].tOut = (datetime)HistoryDealGetInteger(tk, DEAL_TIME);
         aggs[idx].pOut = HistoryDealGetDouble(tk, DEAL_PRICE);
        }
     }

   // Fully closed positions only. Partially closed ones are sent once they are fully closed (or while still open, below).
   string batch = "";
   int inBatch = 0;
   bool allOk = true;
   for(int k = 0; k < ArraySize(aggs); k++)
     {
      if(!aggs[k].hasIn || !aggs[k].hasOut) continue;
      if(aggs[k].volOut + 0.0000001 < aggs[k].volIn) continue; // still partially open
      batch += (inBatch > 0 ? "," : "") + PositionJson(aggs[k], true);
      if(++inBatch >= BATCH) { allOk = SendBatch(batch, "") && allOk; batch = ""; inBatch = 0; }
     }

   // Currently open positions (so they show as OPEN and update when they close).
   for(int j = 0; j < PositionsTotal(); j++)
     {
      ulong t = PositionGetTicket(j);
      if(t == 0 || !PositionSelectByTicket(t)) continue;
      PosAgg p;
      p.id = (ulong)PositionGetInteger(POSITION_IDENTIFIER);
      p.symbol = PositionGetString(POSITION_SYMBOL);
      p.isBuy = (PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY);
      p.volIn = PositionGetDouble(POSITION_VOLUME);
      p.tIn = (datetime)PositionGetInteger(POSITION_TIME);
      p.pIn = PositionGetDouble(POSITION_PRICE_OPEN);
      p.sl = PositionGetDouble(POSITION_SL);
      p.tp = PositionGetDouble(POSITION_TP);
      p.profit = 0; p.commission = 0; p.swap = PositionGetDouble(POSITION_SWAP);
      batch += (inBatch > 0 ? "," : "") + PositionJson(p, false);
      if(++inBatch >= BATCH) { allOk = SendBatch(batch, "") && allOk; batch = ""; inBatch = 0; }
     }

   // Cash flows (deposits/withdrawals) go in their own batches.
   string flows = "";
   int nf = 0;
   for(int f = 0; f < ArraySize(flowJson); f++)
     {
      flows += (nf > 0 ? "," : "") + flowJson[f];
      if(++nf >= BATCH) { allOk = SendBatch("", flows) && allOk; flows = ""; nf = 0; }
     }
   if(inBatch > 0 || nf > 0) allOk = SendBatch(batch, flows) && allOk;
   return allOk;
  }

//--- events --------------------------------------------------------------
int OnInit()
  {
   if(StringLen(ApiKey) < 20 || StringFind(ApiKey, "ntc_") != 0)
     {
      Alert("NuelTradesConnector: paste your connector key (starts with ntc_) into the ApiKey input.");
      return INIT_PARAMETERS_INCORRECT;
     }
   if(StringFind(ApiUrl, "https://") != 0)
     {
      Alert("NuelTradesConnector: ApiUrl must start with https://");
      return INIT_PARAMETERS_INCORRECT;
     }
   EventSetTimer(MathMax(15, SyncIntervalSec));
   OnTimer(); // first sync immediately
   return INIT_SUCCEEDED;
  }

void OnDeinit(const int reason) { EventKillTimer(); }

void OnTimer()
  {
   if(!Heartbeat()) return; // do not send history if the key/account check failed
   SyncHistory();
  }
//+------------------------------------------------------------------+

===== FILE: connectors/mt5/README.md =====
# NuelTradesConnector (MT5)

Read-only Expert Advisor. It sends account balance/equity, closed and open positions, and deposits/withdrawals to your server. It places no orders and never touches your trading password; the only secret is a revocable connector key that can write to ONE NUELTRADES account.

**Status: source written but NOT compiled or run against a live MT5 terminal in this environment.** The server side is tested; compile and test this file on a demo account first and report any compiler errors.

## Setup
1. In NUELTRADES AI > Accounts, open the account and click **Set up MT5 connector**. Copy the key (shown once).
2. MT5: File > Open Data Folder > MQL5 > Experts, copy `NuelTradesConnector.mq5`, then compile in MetaEditor (F7).
3. MT5: Tools > Options > Expert Advisors > tick **Allow WebRequest for listed URL** and add your server's https URL.
4. Drag the EA onto any chart, set `ApiUrl` and `ApiKey`.
5. The account turns **Connected** only after the server accepts a heartbeat from the terminal. If the terminal's login or currency differs from the account you set up, it shows **Action required** with the reason. If the terminal stops reporting for 15 minutes it shows **Disconnected**.

## Behaviour and limits
- Every sync re-sends the last `HistoryDays` days; the server de-duplicates by position id, so this is safe. Closed trades are never rewritten.
- Positions that are partially closed are sent when fully closed (and while open, as open).
- Times are converted from broker server time to UTC using `TimeTradeServer() - TimeGMT()`.
- Rotate or revoke the key from the Accounts page at any time.
- MT4 is not supported (different API); use CSV import.

===== FILE: docker-compose.yml =====
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: nueltrades
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD in .env}
      POSTGRES_DB: nueltrades
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nueltrades"]
      interval: 5s
      retries: 10
  app:
    build: .
    depends_on:
      db: { condition: service_healthy }
    environment:
      DATABASE_URL: postgresql://nueltrades:${POSTGRES_PASSWORD}@db:5432/nueltrades
      APP_URL: ${APP_URL:-http://localhost:3000}
      MAX_UPLOAD_MB: ${MAX_UPLOAD_MB:-8}
    ports: ["3000:3000"]
    volumes: [uploads:/data/uploads]
volumes:
  pgdata:
  uploads:

===== FILE: docs/ANALYTICS.md =====
# Analytics definitions

Only closed trades count; open trades are listed separately. All money math uses decimal.js (no floats).

- **net** = profit + commission + swap (commission and swap are signed as brokers report them)
- **win / loss / breakeven**: net > 0 / < 0 / = 0. **Win rate** = wins / closed trades
- **grossProfit / grossLoss**: sum of positive nets / absolute sum of negative nets
- **profitFactor** = grossProfit / grossLoss (n/a when there are no losses)
- **expectancy** = net profit / closed trades
- **avgR** = mean(net / riskAmount) over trades with a recorded risk amount (coverage is reported)
- **maxDrawdown**: largest peak-to-trough fall of the cumulative closed-trade net curve. It ignores floating P&L and starting balance, so it is a trade-results drawdown, not an equity drawdown
- Deposits and withdrawals never enter trading profit. Recorded balance = starting balance + deposits - withdrawals + closed-trade net
- Groups with fewer than 30 closed trades are flagged "small sample" and always rank below larger samples; a 95% Wilson interval is shown for win rate
- Analytics require a scope (journal, journal kind or account). Real, demo, backtest and prop results are never blended by default

===== FILE: docs/API.md =====
# API

JSON over HTTPS. Auth is an HttpOnly, SameSite=Lax session cookie (`nt_session`, 30 days, hashed server-side). Every route except register/login/logout/reset-confirm requires a session and scopes all queries to the signed-in user; another user's IDs return 404. Writes from a foreign Origin are rejected (403). Errors: `{ "error": { "code", "message", "details?" } }` with 400 validation, 401 unauthenticated, 403, 404, 409 conflict, 413/415 upload, 429 rate limit, 503 database unavailable.

| Method | Path | Purpose |
|---|---|---|
| POST | /api/auth/register, /login, /logout | Sessions (register 10/h/IP, login 8/15min/IP+email) |
| GET | /api/auth/me | Current user |
| POST | /api/auth/reset/confirm | Set password from a reset token (single use, 1h, revokes sessions) |
| POST | /api/auth/reset/request | Returns 501: no email provider bundled (see README) |
| GET/PATCH/DELETE | /api/profile | Profile, timezone; DELETE needs password and cascades all data |
| GET/POST | /api/accounts; GET/PATCH/DELETE /api/accounts/:id | Live/demo/prop accounts (+ prop rules), recorded balance |
| GET/POST | /api/journals; GET/PATCH/DELETE /api/journals/:id | Journals (delete blocked while they hold trades) |
| GET/POST | /api/trades; GET/PATCH/DELETE /api/trades/:id; GET /api/trades/export | Trades, filters, CSV export |
| GET/POST | /api/strategies | Strategies |
| GET/POST | /api/pages; GET/PATCH/DELETE /api/pages/:id; POST /:id/duplicate; GET /:id/export; POST /import; GET /:id/attachments | Notion-style pages, markdown import/export, autosave target |
| GET/POST | /api/charts; PATCH/DELETE /api/charts/:id; GET /api/charts/:id/file | Screenshot library (PNG/JPEG/WebP, magic-byte checked, SHA-256 deduplicated) |
| POST | /api/charts/links; DELETE /api/charts/attachments/:id | http(s) chart URLs; detach |
| POST | /api/imports/preview, /commit; GET /api/imports | CSV statement import (preview writes nothing; commit is idempotent) |
| GET | /api/analytics | Stats and breakdowns for a scope |
| GET/PUT | /api/risk | Thresholds and today's warnings |
| GET | /api/export | Full JSON export of your data |
| GET/POST/DELETE | /api/accounts/:id/connector | Connector status and sync log; create/rotate key (shown once); revoke |
| POST | /api/connector/heartbeat | Connector auth (`Authorization: Bearer ntc_...`): verifies login/currency, records balance/equity, marks Connected |
| POST | /api/connector/positions | Connector auth: up to 500 positions + cash flows per call, idempotent, 120 requests/min/key |
| POST | /api/webhooks/tradingview | Alert webhook; secret in JSON body (`secret`) or `?key=`. 201 new, 200 duplicate, 422 stale/future, 401 bad secret |
| GET/POST/DELETE | /api/signals/secret | Webhook secret status; create/rotate (shown once); revoke |
| GET | /api/signals | Received signals with matched trades |
| GET | /api/ai/status | Whether AI is configured |
| GET/POST | /api/ai/trades/:id | Stored AI reviews; run a new review (20 AI calls/hour/user) |
| POST | /api/ai/coach | Coaching answer from computed stats (needs 10+ closed trades) |

===== FILE: drizzle.config.ts =====
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: { url: process.env.DATABASE_URL ?? "postgresql://localhost/nueltrades" },
});

===== FILE: drizzle/0000_init.sql =====
CREATE TYPE "public"."account_kind" AS ENUM('LIVE', 'DEMO', 'PROP');--> statement-breakpoint
CREATE TYPE "public"."cash_flow_type" AS ENUM('DEPOSIT', 'WITHDRAWAL');--> statement-breakpoint
CREATE TYPE "public"."connection_state" AS ENUM('MANUAL_TRACKING', 'MANUAL_IMPORT_ONLY', 'CONNECTING', 'CONNECTED', 'SYNCING', 'DISCONNECTED', 'ACTION_REQUIRED', 'ERROR');--> statement-breakpoint
CREATE TYPE "public"."drawdown_method" AS ENUM('STATIC', 'TRAILING');--> statement-breakpoint
CREATE TYPE "public"."import_status" AS ENUM('COMPLETED', 'FAILED');--> statement-breakpoint
CREATE TYPE "public"."journal_kind" AS ENUM('REAL', 'DEMO', 'BACKTEST', 'PROP', 'STRATEGY', 'CUSTOM');--> statement-breakpoint
CREATE TYPE "public"."trade_direction" AS ENUM('BUY', 'SELL');--> statement-breakpoint
CREATE TYPE "public"."trade_status" AS ENUM('OPEN', 'CLOSED');--> statement-breakpoint
CREATE TABLE "audit_events" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text,
	"action" text NOT NULL,
	"entity" text,
	"entity_id" text,
	"ip" text,
	"meta" jsonb DEFAULT '{}'::jsonb NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "balance_snapshots" (
	"id" text PRIMARY KEY NOT NULL,
	"account_id" text NOT NULL,
	"balance" numeric(20, 8) NOT NULL,
	"equity" numeric(20, 8) NOT NULL,
	"taken_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "cash_flows" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"account_id" text NOT NULL,
	"type" "cash_flow_type" NOT NULL,
	"amount" numeric(20, 8) NOT NULL,
	"occurred_at" timestamp with time zone NOT NULL,
	"note" text,
	"source_id" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "chart_attachments" (
	"id" text PRIMARY KEY NOT NULL,
	"image_id" text,
	"link_id" text,
	"trade_id" text,
	"page_id" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "chart_images" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"storage_key" text NOT NULL,
	"content_type" text NOT NULL,
	"size_bytes" integer NOT NULL,
	"sha256" text NOT NULL,
	"original_name" text,
	"caption" text,
	"tags" text[] DEFAULT '{}'::text[] NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "chart_images_storage_key_unique" UNIQUE("storage_key")
);
--> statement-breakpoint
CREATE TABLE "chart_links" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"url" text NOT NULL,
	"caption" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "connector_keys" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"account_id" text NOT NULL,
	"token_hash" text NOT NULL,
	"prefix" text NOT NULL,
	"last_used_at" timestamp with time zone,
	"revoked_at" timestamp with time zone,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "connector_keys_token_hash_unique" UNIQUE("token_hash")
);
--> statement-breakpoint
CREATE TABLE "import_batches" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"account_id" text NOT NULL,
	"journal_id" text NOT NULL,
	"file_name" text NOT NULL,
	"file_sha256" text NOT NULL,
	"format" text NOT NULL,
	"status" "import_status" DEFAULT 'COMPLETED' NOT NULL,
	"total_rows" integer DEFAULT 0 NOT NULL,
	"imported" integer DEFAULT 0 NOT NULL,
	"duplicates" integer DEFAULT 0 NOT NULL,
	"failed" integer DEFAULT 0 NOT NULL,
	"errors" jsonb DEFAULT '[]'::jsonb NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "journal_pages" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"journal_id" text NOT NULL,
	"parent_id" text,
	"title" text NOT NULL,
	"is_folder" boolean DEFAULT false NOT NULL,
	"content" text DEFAULT '' NOT NULL,
	"tags" text[] DEFAULT '{}'::text[] NOT NULL,
	"properties" jsonb DEFAULT '{}'::jsonb NOT NULL,
	"favorite" boolean DEFAULT false NOT NULL,
	"page_date" date,
	"archived_at" timestamp with time zone,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "journals" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"name" text NOT NULL,
	"kind" "journal_kind" NOT NULL,
	"description" text,
	"archived_at" timestamp with time zone,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "password_reset_tokens" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"token_hash" text NOT NULL,
	"expires_at" timestamp with time zone NOT NULL,
	"used_at" timestamp with time zone,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "password_reset_tokens_token_hash_unique" UNIQUE("token_hash")
);
--> statement-breakpoint
CREATE TABLE "prop_rules" (
	"id" text PRIMARY KEY NOT NULL,
	"account_id" text NOT NULL,
	"firm_name" text NOT NULL,
	"account_size" numeric(20, 8) NOT NULL,
	"phase" text,
	"profit_target" numeric(20, 8),
	"daily_loss_limit" numeric(20, 8),
	"max_drawdown" numeric(20, 8),
	"drawdown_method" "drawdown_method" DEFAULT 'STATIC' NOT NULL,
	"min_trading_days" integer,
	"deadline" timestamp with time zone,
	"challenge_fee" numeric(20, 8),
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "prop_rules_account_id_unique" UNIQUE("account_id")
);
--> statement-breakpoint
CREATE TABLE "raw_import_rows" (
	"id" text PRIMARY KEY NOT NULL,
	"batch_id" text NOT NULL,
	"row_index" integer NOT NULL,
	"raw" jsonb NOT NULL,
	"source_id" text
);
--> statement-breakpoint
CREATE TABLE "risk_settings" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"max_daily_loss" numeric(20, 8),
	"max_trades_per_day" integer,
	"max_risk_per_trade" numeric(20, 8),
	"max_consecutive_loss" integer,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "risk_settings_user_id_unique" UNIQUE("user_id")
);
--> statement-breakpoint
CREATE TABLE "sessions" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"token_hash" text NOT NULL,
	"user_agent" text,
	"expires_at" timestamp with time zone NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "sessions_token_hash_unique" UNIQUE("token_hash")
);
--> statement-breakpoint
CREATE TABLE "strategies" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"name" text NOT NULL,
	"description" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "strategy_versions" (
	"id" text PRIMARY KEY NOT NULL,
	"strategy_id" text NOT NULL,
	"version" integer NOT NULL,
	"rules" text DEFAULT '' NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "sync_logs" (
	"id" text PRIMARY KEY NOT NULL,
	"account_id" text NOT NULL,
	"kind" text NOT NULL,
	"received" integer DEFAULT 0 NOT NULL,
	"imported" integer DEFAULT 0 NOT NULL,
	"updated" integer DEFAULT 0 NOT NULL,
	"duplicates" integer DEFAULT 0 NOT NULL,
	"error" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "trades" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"account_id" text NOT NULL,
	"journal_id" text NOT NULL,
	"strategy_id" text,
	"symbol" text NOT NULL,
	"direction" "trade_direction" NOT NULL,
	"status" "trade_status" DEFAULT 'CLOSED' NOT NULL,
	"lots" numeric(20, 8) NOT NULL,
	"entry_price" numeric(20, 8) NOT NULL,
	"exit_price" numeric(20, 8),
	"stop_loss" numeric(20, 8),
	"take_profit" numeric(20, 8),
	"risk_amount" numeric(20, 8),
	"profit" numeric(20, 8) DEFAULT '0' NOT NULL,
	"commission" numeric(20, 8) DEFAULT '0' NOT NULL,
	"swap" numeric(20, 8) DEFAULT '0' NOT NULL,
	"opened_at" timestamp with time zone NOT NULL,
	"closed_at" timestamp with time zone,
	"timeframe" text,
	"session" text,
	"setup" text,
	"emotion" text,
	"mistakes" text[] DEFAULT '{}'::text[] NOT NULL,
	"tags" text[] DEFAULT '{}'::text[] NOT NULL,
	"notes" text DEFAULT '' NOT NULL,
	"source" text DEFAULT 'MANUAL' NOT NULL,
	"source_id" text,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "trading_accounts" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"name" text NOT NULL,
	"kind" "account_kind" NOT NULL,
	"broker" text,
	"platform" text,
	"external_login" text,
	"currency" text DEFAULT 'USD' NOT NULL,
	"starting_balance" numeric(20, 8) DEFAULT '0' NOT NULL,
	"connection_state" "connection_state" DEFAULT 'MANUAL_TRACKING' NOT NULL,
	"last_sync_at" timestamp with time zone,
	"last_sync_error" text,
	"archived_at" timestamp with time zone,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "users" (
	"id" text PRIMARY KEY NOT NULL,
	"email" text NOT NULL,
	"password_hash" text NOT NULL,
	"name" text,
	"timezone" text DEFAULT 'UTC' NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	"updated_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "users_email_unique" UNIQUE("email")
);
--> statement-breakpoint
ALTER TABLE "audit_events" ADD CONSTRAINT "audit_events_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE set null ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "balance_snapshots" ADD CONSTRAINT "balance_snapshots_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "cash_flows" ADD CONSTRAINT "cash_flows_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "cash_flows" ADD CONSTRAINT "cash_flows_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_attachments" ADD CONSTRAINT "chart_attachments_image_id_chart_images_id_fk" FOREIGN KEY ("image_id") REFERENCES "public"."chart_images"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_attachments" ADD CONSTRAINT "chart_attachments_link_id_chart_links_id_fk" FOREIGN KEY ("link_id") REFERENCES "public"."chart_links"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_attachments" ADD CONSTRAINT "chart_attachments_trade_id_trades_id_fk" FOREIGN KEY ("trade_id") REFERENCES "public"."trades"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_attachments" ADD CONSTRAINT "chart_attachments_page_id_journal_pages_id_fk" FOREIGN KEY ("page_id") REFERENCES "public"."journal_pages"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_images" ADD CONSTRAINT "chart_images_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "chart_links" ADD CONSTRAINT "chart_links_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "connector_keys" ADD CONSTRAINT "connector_keys_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "connector_keys" ADD CONSTRAINT "connector_keys_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "import_batches" ADD CONSTRAINT "import_batches_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "import_batches" ADD CONSTRAINT "import_batches_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "import_batches" ADD CONSTRAINT "import_batches_journal_id_journals_id_fk" FOREIGN KEY ("journal_id") REFERENCES "public"."journals"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "journal_pages" ADD CONSTRAINT "journal_pages_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "journal_pages" ADD CONSTRAINT "journal_pages_journal_id_journals_id_fk" FOREIGN KEY ("journal_id") REFERENCES "public"."journals"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "journal_pages" ADD CONSTRAINT "journal_pages_parent_id_journal_pages_id_fk" FOREIGN KEY ("parent_id") REFERENCES "public"."journal_pages"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "journals" ADD CONSTRAINT "journals_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "password_reset_tokens" ADD CONSTRAINT "password_reset_tokens_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "prop_rules" ADD CONSTRAINT "prop_rules_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "raw_import_rows" ADD CONSTRAINT "raw_import_rows_batch_id_import_batches_id_fk" FOREIGN KEY ("batch_id") REFERENCES "public"."import_batches"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "risk_settings" ADD CONSTRAINT "risk_settings_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "sessions" ADD CONSTRAINT "sessions_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "strategies" ADD CONSTRAINT "strategies_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "strategy_versions" ADD CONSTRAINT "strategy_versions_strategy_id_strategies_id_fk" FOREIGN KEY ("strategy_id") REFERENCES "public"."strategies"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "sync_logs" ADD CONSTRAINT "sync_logs_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "trades" ADD CONSTRAINT "trades_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "trades" ADD CONSTRAINT "trades_account_id_trading_accounts_id_fk" FOREIGN KEY ("account_id") REFERENCES "public"."trading_accounts"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "trades" ADD CONSTRAINT "trades_journal_id_journals_id_fk" FOREIGN KEY ("journal_id") REFERENCES "public"."journals"("id") ON DELETE restrict ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "trades" ADD CONSTRAINT "trades_strategy_id_strategies_id_fk" FOREIGN KEY ("strategy_id") REFERENCES "public"."strategies"("id") ON DELETE set null ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "trading_accounts" ADD CONSTRAINT "trading_accounts_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
CREATE INDEX "audit_user_created_idx" ON "audit_events" USING btree ("user_id","created_at");--> statement-breakpoint
CREATE INDEX "audit_action_idx" ON "audit_events" USING btree ("action");--> statement-breakpoint
CREATE INDEX "snapshot_account_idx" ON "balance_snapshots" USING btree ("account_id","taken_at");--> statement-breakpoint
CREATE INDEX "cashflow_account_idx" ON "cash_flows" USING btree ("account_id","occurred_at");--> statement-breakpoint
CREATE UNIQUE INDEX "cashflow_account_source_uq" ON "cash_flows" USING btree ("account_id","source_id");--> statement-breakpoint
CREATE INDEX "att_trade_idx" ON "chart_attachments" USING btree ("trade_id");--> statement-breakpoint
CREATE INDEX "att_page_idx" ON "chart_attachments" USING btree ("page_id");--> statement-breakpoint
CREATE INDEX "att_image_idx" ON "chart_attachments" USING btree ("image_id");--> statement-breakpoint
CREATE UNIQUE INDEX "chart_user_sha_uq" ON "chart_images" USING btree ("user_id","sha256");--> statement-breakpoint
CREATE INDEX "chart_user_created_idx" ON "chart_images" USING btree ("user_id","created_at");--> statement-breakpoint
CREATE INDEX "chartlink_user_idx" ON "chart_links" USING btree ("user_id");--> statement-breakpoint
CREATE INDEX "connector_account_idx" ON "connector_keys" USING btree ("account_id");--> statement-breakpoint
CREATE INDEX "import_user_created_idx" ON "import_batches" USING btree ("user_id","created_at");--> statement-breakpoint
CREATE INDEX "page_user_journal_idx" ON "journal_pages" USING btree ("user_id","journal_id");--> statement-breakpoint
CREATE INDEX "page_parent_idx" ON "journal_pages" USING btree ("parent_id");--> statement-breakpoint
CREATE INDEX "page_user_date_idx" ON "journal_pages" USING btree ("user_id","page_date");--> statement-breakpoint
CREATE UNIQUE INDEX "journal_user_name_uq" ON "journals" USING btree ("user_id","name");--> statement-breakpoint
CREATE INDEX "journal_user_kind_idx" ON "journals" USING btree ("user_id","kind");--> statement-breakpoint
CREATE INDEX "prt_user_idx" ON "password_reset_tokens" USING btree ("user_id");--> statement-breakpoint
CREATE INDEX "raw_batch_idx" ON "raw_import_rows" USING btree ("batch_id");--> statement-breakpoint
CREATE INDEX "sessions_user_idx" ON "sessions" USING btree ("user_id");--> statement-breakpoint
CREATE INDEX "sessions_expires_idx" ON "sessions" USING btree ("expires_at");--> statement-breakpoint
CREATE UNIQUE INDEX "strategy_user_name_uq" ON "strategies" USING btree ("user_id","name");--> statement-breakpoint
CREATE UNIQUE INDEX "strategy_version_uq" ON "strategy_versions" USING btree ("strategy_id","version");--> statement-breakpoint
CREATE INDEX "synclog_account_idx" ON "sync_logs" USING btree ("account_id","created_at");--> statement-breakpoint
CREATE UNIQUE INDEX "trade_account_source_uq" ON "trades" USING btree ("account_id","source_id");--> statement-breakpoint
CREATE INDEX "trade_user_journal_open_idx" ON "trades" USING btree ("user_id","journal_id","opened_at");--> statement-breakpoint
CREATE INDEX "trade_user_account_open_idx" ON "trades" USING btree ("user_id","account_id","opened_at");--> statement-breakpoint
CREATE INDEX "trade_user_symbol_idx" ON "trades" USING btree ("user_id","symbol");--> statement-breakpoint
CREATE UNIQUE INDEX "acct_user_name_uq" ON "trading_accounts" USING btree ("user_id","name");--> statement-breakpoint
CREATE INDEX "acct_user_kind_idx" ON "trading_accounts" USING btree ("user_id","kind");
===== FILE: drizzle/0001_signals_ai.sql =====
CREATE TABLE "ai_analyses" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"trade_id" text,
	"kind" text NOT NULL,
	"model" text NOT NULL,
	"question" text,
	"content" jsonb NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
CREATE TABLE "signal_secrets" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"token_hash" text NOT NULL,
	"prefix" text NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL,
	CONSTRAINT "signal_secrets_user_id_unique" UNIQUE("user_id"),
	CONSTRAINT "signal_secrets_token_hash_unique" UNIQUE("token_hash")
);
--> statement-breakpoint
CREATE TABLE "signals" (
	"id" text PRIMARY KEY NOT NULL,
	"user_id" text NOT NULL,
	"signal_id" text NOT NULL,
	"symbol" text NOT NULL,
	"direction" "trade_direction" NOT NULL,
	"timeframe" text,
	"strategy" text,
	"signal_time" timestamp with time zone NOT NULL,
	"entry_low" numeric(20, 8),
	"entry_high" numeric(20, 8),
	"stop_loss" numeric(20, 8),
	"take_profit" numeric(20, 8),
	"raw" jsonb NOT NULL,
	"created_at" timestamp with time zone DEFAULT now() NOT NULL
);
--> statement-breakpoint
ALTER TABLE "ai_analyses" ADD CONSTRAINT "ai_analyses_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "ai_analyses" ADD CONSTRAINT "ai_analyses_trade_id_trades_id_fk" FOREIGN KEY ("trade_id") REFERENCES "public"."trades"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "signal_secrets" ADD CONSTRAINT "signal_secrets_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
ALTER TABLE "signals" ADD CONSTRAINT "signals_user_id_users_id_fk" FOREIGN KEY ("user_id") REFERENCES "public"."users"("id") ON DELETE cascade ON UPDATE no action;--> statement-breakpoint
CREATE INDEX "ai_user_trade_idx" ON "ai_analyses" USING btree ("user_id","trade_id");--> statement-breakpoint
CREATE UNIQUE INDEX "signal_user_sid_uq" ON "signals" USING btree ("user_id","signal_id");--> statement-breakpoint
CREATE INDEX "signal_user_time_idx" ON "signals" USING btree ("user_id","signal_time");
===== FILE: drizzle/meta/0000_snapshot.json =====
{
  "id": "0a6d8347-be2e-4859-adc5-c357d5a87df0",
  "prevId": "00000000-0000-0000-0000-000000000000",
  "version": "7",
  "dialect": "postgresql",
  "tables": {
    "public.audit_events": {
      "name": "audit_events",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "action": {
          "name": "action",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "entity": {
          "name": "entity",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "entity_id": {
          "name": "entity_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "ip": {
          "name": "ip",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "meta": {
          "name": "meta",
          "type": "jsonb",
          "primaryKey": false,
          "notNull": true,
          "default": "'{}'::jsonb"
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "audit_user_created_idx": {
          "name": "audit_user_created_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "created_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "audit_action_idx": {
          "name": "audit_action_idx",
          "columns": [
            {
              "expression": "action",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "audit_events_user_id_users_id_fk": {
          "name": "audit_events_user_id_users_id_fk",
          "tableFrom": "audit_events",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "set null",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.balance_snapshots": {
      "name": "balance_snapshots",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "balance": {
          "name": "balance",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "equity": {
          "name": "equity",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "taken_at": {
          "name": "taken_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "snapshot_account_idx": {
          "name": "snapshot_account_idx",
          "columns": [
            {
              "expression": "account_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "taken_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "balance_snapshots_account_id_trading_accounts_id_fk": {
          "name": "balance_snapshots_account_id_trading_accounts_id_fk",
          "tableFrom": "balance_snapshots",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.cash_flows": {
      "name": "cash_flows",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "type": {
          "name": "type",
          "type": "cash_flow_type",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true
        },
        "amount": {
          "name": "amount",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "occurred_at": {
          "name": "occurred_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true
        },
        "note": {
          "name": "note",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "source_id": {
          "name": "source_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "cashflow_account_idx": {
          "name": "cashflow_account_idx",
          "columns": [
            {
              "expression": "account_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "occurred_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "cashflow_account_source_uq": {
          "name": "cashflow_account_source_uq",
          "columns": [
            {
              "expression": "account_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "source_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": true,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "cash_flows_user_id_users_id_fk": {
          "name": "cash_flows_user_id_users_id_fk",
          "tableFrom": "cash_flows",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "cash_flows_account_id_trading_accounts_id_fk": {
          "name": "cash_flows_account_id_trading_accounts_id_fk",
          "tableFrom": "cash_flows",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.chart_attachments": {
      "name": "chart_attachments",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "image_id": {
          "name": "image_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "link_id": {
          "name": "link_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "trade_id": {
          "name": "trade_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "page_id": {
          "name": "page_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "att_trade_idx": {
          "name": "att_trade_idx",
          "columns": [
            {
              "expression": "trade_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "att_page_idx": {
          "name": "att_page_idx",
          "columns": [
            {
              "expression": "page_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "att_image_idx": {
          "name": "att_image_idx",
          "columns": [
            {
              "expression": "image_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "chart_attachments_image_id_chart_images_id_fk": {
          "name": "chart_attachments_image_id_chart_images_id_fk",
          "tableFrom": "chart_attachments",
          "tableTo": "chart_images",
          "columnsFrom": [
            "image_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "chart_attachments_link_id_chart_links_id_fk": {
          "name": "chart_attachments_link_id_chart_links_id_fk",
          "tableFrom": "chart_attachments",
          "tableTo": "chart_links",
          "columnsFrom": [
            "link_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "chart_attachments_trade_id_trades_id_fk": {
          "name": "chart_attachments_trade_id_trades_id_fk",
          "tableFrom": "chart_attachments",
          "tableTo": "trades",
          "columnsFrom": [
            "trade_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "chart_attachments_page_id_journal_pages_id_fk": {
          "name": "chart_attachments_page_id_journal_pages_id_fk",
          "tableFrom": "chart_attachments",
          "tableTo": "journal_pages",
          "columnsFrom": [
            "page_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.chart_images": {
      "name": "chart_images",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "storage_key": {
          "name": "storage_key",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "content_type": {
          "name": "content_type",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "size_bytes": {
          "name": "size_bytes",
          "type": "integer",
          "primaryKey": false,
          "notNull": true
        },
        "sha256": {
          "name": "sha256",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "original_name": {
          "name": "original_name",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "caption": {
          "name": "caption",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "tags": {
          "name": "tags",
          "type": "text[]",
          "primaryKey": false,
          "notNull": true,
          "default": "'{}'::text[]"
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "chart_user_sha_uq": {
          "name": "chart_user_sha_uq",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "sha256",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": true,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "chart_user_created_idx": {
          "name": "chart_user_created_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "created_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "chart_images_user_id_users_id_fk": {
          "name": "chart_images_user_id_users_id_fk",
          "tableFrom": "chart_images",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "chart_images_storage_key_unique": {
          "name": "chart_images_storage_key_unique",
          "nullsNotDistinct": false,
          "columns": [
            "storage_key"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.chart_links": {
      "name": "chart_links",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "url": {
          "name": "url",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "caption": {
          "name": "caption",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "chartlink_user_idx": {
          "name": "chartlink_user_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "chart_links_user_id_users_id_fk": {
          "name": "chart_links_user_id_users_id_fk",
          "tableFrom": "chart_links",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.connector_keys": {
      "name": "connector_keys",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "token_hash": {
          "name": "token_hash",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "prefix": {
          "name": "prefix",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "last_used_at": {
          "name": "last_used_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "revoked_at": {
          "name": "revoked_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "connector_account_idx": {
          "name": "connector_account_idx",
          "columns": [
            {
              "expression": "account_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "connector_keys_user_id_users_id_fk": {
          "name": "connector_keys_user_id_users_id_fk",
          "tableFrom": "connector_keys",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "connector_keys_account_id_trading_accounts_id_fk": {
          "name": "connector_keys_account_id_trading_accounts_id_fk",
          "tableFrom": "connector_keys",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "connector_keys_token_hash_unique": {
          "name": "connector_keys_token_hash_unique",
          "nullsNotDistinct": false,
          "columns": [
            "token_hash"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.import_batches": {
      "name": "import_batches",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "journal_id": {
          "name": "journal_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "file_name": {
          "name": "file_name",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "file_sha256": {
          "name": "file_sha256",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "format": {
          "name": "format",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "status": {
          "name": "status",
          "type": "import_status",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true,
          "default": "'COMPLETED'"
        },
        "total_rows": {
          "name": "total_rows",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "imported": {
          "name": "imported",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "duplicates": {
          "name": "duplicates",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "failed": {
          "name": "failed",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "errors": {
          "name": "errors",
          "type": "jsonb",
          "primaryKey": false,
          "notNull": true,
          "default": "'[]'::jsonb"
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "import_user_created_idx": {
          "name": "import_user_created_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "created_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "import_batches_user_id_users_id_fk": {
          "name": "import_batches_user_id_users_id_fk",
          "tableFrom": "import_batches",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "import_batches_account_id_trading_accounts_id_fk": {
          "name": "import_batches_account_id_trading_accounts_id_fk",
          "tableFrom": "import_batches",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "import_batches_journal_id_journals_id_fk": {
          "name": "import_batches_journal_id_journals_id_fk",
          "tableFrom": "import_batches",
          "tableTo": "journals",
          "columnsFrom": [
            "journal_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.journal_pages": {
      "name": "journal_pages",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "journal_id": {
          "name": "journal_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "parent_id": {
          "name": "parent_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "title": {
          "name": "title",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "is_folder": {
          "name": "is_folder",
          "type": "boolean",
          "primaryKey": false,
          "notNull": true,
          "default": false
        },
        "content": {
          "name": "content",
          "type": "text",
          "primaryKey": false,
          "notNull": true,
          "default": "''"
        },
        "tags": {
          "name": "tags",
          "type": "text[]",
          "primaryKey": false,
          "notNull": true,
          "default": "'{}'::text[]"
        },
        "properties": {
          "name": "properties",
          "type": "jsonb",
          "primaryKey": false,
          "notNull": true,
          "default": "'{}'::jsonb"
        },
        "favorite": {
          "name": "favorite",
          "type": "boolean",
          "primaryKey": false,
          "notNull": true,
          "default": false
        },
        "page_date": {
          "name": "page_date",
          "type": "date",
          "primaryKey": false,
          "notNull": false
        },
        "archived_at": {
          "name": "archived_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        },
        "updated_at": {
          "name": "updated_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "page_user_journal_idx": {
          "name": "page_user_journal_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "journal_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "page_parent_idx": {
          "name": "page_parent_idx",
          "columns": [
            {
              "expression": "parent_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "page_user_date_idx": {
          "name": "page_user_date_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "page_date",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "journal_pages_user_id_users_id_fk": {
          "name": "journal_pages_user_id_users_id_fk",
          "tableFrom": "journal_pages",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "journal_pages_journal_id_journals_id_fk": {
          "name": "journal_pages_journal_id_journals_id_fk",
          "tableFrom": "journal_pages",
          "tableTo": "journals",
          "columnsFrom": [
            "journal_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        },
        "journal_pages_parent_id_journal_pages_id_fk": {
          "name": "journal_pages_parent_id_journal_pages_id_fk",
          "tableFrom": "journal_pages",
          "tableTo": "journal_pages",
          "columnsFrom": [
            "parent_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.journals": {
      "name": "journals",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "name": {
          "name": "name",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "kind": {
          "name": "kind",
          "type": "journal_kind",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true
        },
        "description": {
          "name": "description",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "archived_at": {
          "name": "archived_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        },
        "updated_at": {
          "name": "updated_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "journal_user_name_uq": {
          "name": "journal_user_name_uq",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "name",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": true,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "journal_user_kind_idx": {
          "name": "journal_user_kind_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "kind",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "journals_user_id_users_id_fk": {
          "name": "journals_user_id_users_id_fk",
          "tableFrom": "journals",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.password_reset_tokens": {
      "name": "password_reset_tokens",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "token_hash": {
          "name": "token_hash",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "expires_at": {
          "name": "expires_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true
        },
        "used_at": {
          "name": "used_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "prt_user_idx": {
          "name": "prt_user_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "password_reset_tokens_user_id_users_id_fk": {
          "name": "password_reset_tokens_user_id_users_id_fk",
          "tableFrom": "password_reset_tokens",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "password_reset_tokens_token_hash_unique": {
          "name": "password_reset_tokens_token_hash_unique",
          "nullsNotDistinct": false,
          "columns": [
            "token_hash"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.prop_rules": {
      "name": "prop_rules",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "firm_name": {
          "name": "firm_name",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "account_size": {
          "name": "account_size",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "phase": {
          "name": "phase",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "profit_target": {
          "name": "profit_target",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "daily_loss_limit": {
          "name": "daily_loss_limit",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "max_drawdown": {
          "name": "max_drawdown",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "drawdown_method": {
          "name": "drawdown_method",
          "type": "drawdown_method",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true,
          "default": "'STATIC'"
        },
        "min_trading_days": {
          "name": "min_trading_days",
          "type": "integer",
          "primaryKey": false,
          "notNull": false
        },
        "deadline": {
          "name": "deadline",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": false
        },
        "challenge_fee": {
          "name": "challenge_fee",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        },
        "updated_at": {
          "name": "updated_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {},
      "foreignKeys": {
        "prop_rules_account_id_trading_accounts_id_fk": {
          "name": "prop_rules_account_id_trading_accounts_id_fk",
          "tableFrom": "prop_rules",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "prop_rules_account_id_unique": {
          "name": "prop_rules_account_id_unique",
          "nullsNotDistinct": false,
          "columns": [
            "account_id"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.raw_import_rows": {
      "name": "raw_import_rows",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "batch_id": {
          "name": "batch_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "row_index": {
          "name": "row_index",
          "type": "integer",
          "primaryKey": false,
          "notNull": true
        },
        "raw": {
          "name": "raw",
          "type": "jsonb",
          "primaryKey": false,
          "notNull": true
        },
        "source_id": {
          "name": "source_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        }
      },
      "indexes": {
        "raw_batch_idx": {
          "name": "raw_batch_idx",
          "columns": [
            {
              "expression": "batch_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "raw_import_rows_batch_id_import_batches_id_fk": {
          "name": "raw_import_rows_batch_id_import_batches_id_fk",
          "tableFrom": "raw_import_rows",
          "tableTo": "import_batches",
          "columnsFrom": [
            "batch_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.risk_settings": {
      "name": "risk_settings",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "max_daily_loss": {
          "name": "max_daily_loss",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "max_trades_per_day": {
          "name": "max_trades_per_day",
          "type": "integer",
          "primaryKey": false,
          "notNull": false
        },
        "max_risk_per_trade": {
          "name": "max_risk_per_trade",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "max_consecutive_loss": {
          "name": "max_consecutive_loss",
          "type": "integer",
          "primaryKey": false,
          "notNull": false
        },
        "updated_at": {
          "name": "updated_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {},
      "foreignKeys": {
        "risk_settings_user_id_users_id_fk": {
          "name": "risk_settings_user_id_users_id_fk",
          "tableFrom": "risk_settings",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "risk_settings_user_id_unique": {
          "name": "risk_settings_user_id_unique",
          "nullsNotDistinct": false,
          "columns": [
            "user_id"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.sessions": {
      "name": "sessions",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "token_hash": {
          "name": "token_hash",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "user_agent": {
          "name": "user_agent",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "expires_at": {
          "name": "expires_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "sessions_user_idx": {
          "name": "sessions_user_idx",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        },
        "sessions_expires_idx": {
          "name": "sessions_expires_idx",
          "columns": [
            {
              "expression": "expires_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "sessions_user_id_users_id_fk": {
          "name": "sessions_user_id_users_id_fk",
          "tableFrom": "sessions",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {
        "sessions_token_hash_unique": {
          "name": "sessions_token_hash_unique",
          "nullsNotDistinct": false,
          "columns": [
            "token_hash"
          ]
        }
      },
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.strategies": {
      "name": "strategies",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "name": {
          "name": "name",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "description": {
          "name": "description",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        },
        "updated_at": {
          "name": "updated_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "strategy_user_name_uq": {
          "name": "strategy_user_name_uq",
          "columns": [
            {
              "expression": "user_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "name",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": true,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "strategies_user_id_users_id_fk": {
          "name": "strategies_user_id_users_id_fk",
          "tableFrom": "strategies",
          "tableTo": "users",
          "columnsFrom": [
            "user_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.strategy_versions": {
      "name": "strategy_versions",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "strategy_id": {
          "name": "strategy_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "version": {
          "name": "version",
          "type": "integer",
          "primaryKey": false,
          "notNull": true
        },
        "rules": {
          "name": "rules",
          "type": "text",
          "primaryKey": false,
          "notNull": true,
          "default": "''"
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "strategy_version_uq": {
          "name": "strategy_version_uq",
          "columns": [
            {
              "expression": "strategy_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "version",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": true,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "strategy_versions_strategy_id_strategies_id_fk": {
          "name": "strategy_versions_strategy_id_strategies_id_fk",
          "tableFrom": "strategy_versions",
          "tableTo": "strategies",
          "columnsFrom": [
            "strategy_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.sync_logs": {
      "name": "sync_logs",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "kind": {
          "name": "kind",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "received": {
          "name": "received",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "imported": {
          "name": "imported",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "updated": {
          "name": "updated",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "duplicates": {
          "name": "duplicates",
          "type": "integer",
          "primaryKey": false,
          "notNull": true,
          "default": 0
        },
        "error": {
          "name": "error",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "created_at": {
          "name": "created_at",
          "type": "timestamp with time zone",
          "primaryKey": false,
          "notNull": true,
          "default": "now()"
        }
      },
      "indexes": {
        "synclog_account_idx": {
          "name": "synclog_account_idx",
          "columns": [
            {
              "expression": "account_id",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            },
            {
              "expression": "created_at",
              "isExpression": false,
              "asc": true,
              "nulls": "last"
            }
          ],
          "isUnique": false,
          "concurrently": false,
          "method": "btree",
          "with": {}
        }
      },
      "foreignKeys": {
        "sync_logs_account_id_trading_accounts_id_fk": {
          "name": "sync_logs_account_id_trading_accounts_id_fk",
          "tableFrom": "sync_logs",
          "tableTo": "trading_accounts",
          "columnsFrom": [
            "account_id"
          ],
          "columnsTo": [
            "id"
          ],
          "onDelete": "cascade",
          "onUpdate": "no action"
        }
      },
      "compositePrimaryKeys": {},
      "uniqueConstraints": {},
      "policies": {},
      "checkConstraints": {},
      "isRLSEnabled": false
    },
    "public.trades": {
      "name": "trades",
      "schema": "",
      "columns": {
        "id": {
          "name": "id",
          "type": "text",
          "primaryKey": true,
          "notNull": true
        },
        "user_id": {
          "name": "user_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "account_id": {
          "name": "account_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "journal_id": {
          "name": "journal_id",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "strategy_id": {
          "name": "strategy_id",
          "type": "text",
          "primaryKey": false,
          "notNull": false
        },
        "symbol": {
          "name": "symbol",
          "type": "text",
          "primaryKey": false,
          "notNull": true
        },
        "direction": {
          "name": "direction",
          "type": "trade_direction",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true
        },
        "status": {
          "name": "status",
          "type": "trade_status",
          "typeSchema": "public",
          "primaryKey": false,
          "notNull": true,
          "default": "'CLOSED'"
        },
        "lots": {
          "name": "lots",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "entry_price": {
          "name": "entry_price",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": true
        },
        "exit_price": {
          "name": "exit_price",
          "type": "numeric(20, 8)",
          "primaryKey": false,
          "notNull": false
        },
        "stop_loss": {
          "name": "stop_loss",
  
