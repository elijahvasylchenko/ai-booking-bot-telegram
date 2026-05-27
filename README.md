# AI Booking Bot for Beauty Studios

> 24/7 Telegram bot that handles client bookings, cancellations, 
> rescheduling, and automated reminders — built with n8n + Claude/GPT + Cal.com.

## Demo

**Try the bot:** [@democalendar_bot](https://t.me/democalendar_bot)  
*(Ukrainian-language demo for a beauty studio)*

![Bot Menu](docs/bot-preview.png)

---

## What It Does

A client writes in Telegram — the bot handles everything automatically:

- **Books appointments** — shows available slots, collects name + phone, confirms booking via Cal.com
- **Cancels and reschedules** — finds existing booking, cancels, books new slot
- **Auto-reminders** — sends reminder 24h and 2h before the appointment
- **Shows prices** — pulls live service catalog from PostgreSQL
- **Handles voice messages** — transcribes via Whisper API, processes like text
- **Escalates to human** — routes to master when needed

---

## Architecture

```
Telegram message / voice
        ↓
   Voice check (If node)
   ├── Voice → Whisper transcription
   └── Text → Intent router (JS)
              ↓
         Switch (3 routes)
         ├── /start → Welcome message + inline buttons
         ├── price/services → PostgreSQL → formatted list
         └── everything else → AI Agent (Claude/GPT)
                                    ↓
                              Cal.com Tools:
                              GET_Free_Slots
                              Book_Appointment
                              Find_Booking
                              Cancel_Appointment
                              Escalate_To_Master
                                    ↓
                         Telegram reply to client
                         PostgreSQL log
```

**Reminder pipeline** (runs hourly via Schedule Trigger):
```
Cal.com API → fetch upcoming bookings → filter 24h/2h window → Telegram reminder
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Automation | n8n |
| AI / LLM | OpenAI GPT-4o-mini |
| Voice transcription | OpenAI Whisper API |
| Calendar / booking | Cal.com v2 API |
| Database | PostgreSQL |
| Messaging | Telegram Bot API |
| Infrastructure | Docker, self-hosted |

---

## Key Features

**Smart intent routing** — JS router classifies every message before hitting the AI agent. Price/catalog queries bypass the LLM entirely (saves tokens, 3x faster response).

**State via Cal.com** — no session state stored in n8n or Redis. Client identity is derived from Telegram chat ID → email pattern `tg_{chat_id}@salon.local`.

**Production-quality system prompt** — covers booking flow, date math, slot search fallback (up to 14 days), phone validation, guardrails against prompt injection.

**Reusable template** — all client-specific config (services, prices, bot name, master chat ID) lives in PostgreSQL. New client onboarding = 15 minutes.

---

## PostgreSQL Schema

```sql
-- Client registry
CREATE TABLE clients_cal (
  id SERIAL PRIMARY KEY,
  telegram_id BIGINT UNIQUE NOT NULL,
  first_name TEXT,
  username TEXT,
  last_interaction TIMESTAMP,
  total_bookings INT DEFAULT 0
);

-- Message log (optional, disabled by default)
CREATE TABLE messages_log_cal (
  id SERIAL PRIMARY KEY,
  telegram_id BIGINT,
  direction TEXT,  -- 'in' | 'out'
  content TEXT,
  intent TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Service catalog
CREATE TABLE services_cal (
  id INT PRIMARY KEY,  -- Cal.com eventTypeId
  title TEXT,
  price TEXT,
  duration INT
);
```

---

## Setup

### Requirements

- n8n instance (self-hosted or cloud)
- Telegram Bot Token ([@BotFather](https://t.me/BotFather))
- OpenAI API key
- Cal.com account + API key (v2)
- PostgreSQL database

### Installation

1. Import `workflow/booking-bot-workflow.json` into n8n
2. Set up credentials:
   - `Telegram API` → your bot token
   - `OpenAI API` → your key
   - `Cal.com API` → your v2 key
   - `PostgreSQL` → your connection string
3. Replace placeholders in workflow:
   - `{YOUR_BOT_TOKEN}` in HTTP Request nodes
   - `YOUR_MASTER_TELEGRAM_ID` in Escalate_To_Master node
4. Run SQL schema (see above) on your PostgreSQL
5. Populate `services_cal` with your Cal.com event type IDs and prices
6. Activate workflow
7. Enable Schedule Trigger for reminders

### Configuration (per client)

| What to change | Where |
|---|---|
| Services + prices + IDs | `services_cal` table |
| Bot name and welcome text | `Вітання` node |
| Master notification chat ID | `Escalate_To_Master` node |
| Reminder message text | `Send a text message1/2` nodes |

---

## Project Status

Production-ready template. Currently in go-to-market as a **$50/month SaaS service** for beauty studios (nail, lash, brow masters).

First month free. ~15-minute setup per new client using this template.

---

## Author

**Illia Vasylchenko** — AI Automation Engineer  
Founder @ [SystemFlow](https://elijahvasylchenko.github.io/services/)  
[elijah.vasylchenko@gmail.com](mailto:elijah.vasylchenko@gmail.com) · [t.me/ilya_vasylchenko](https://t.me/ilya_vasylchenko)
