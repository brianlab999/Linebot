# Linebot — 3C Shopping Assistant

A multi-feature LINE chatbot that helps users explore 3C (Computer, Communication, Consumer electronics) products, query prices across multiple retailers, read the latest tech news, get AI-powered consultation, and submit customer feedback — all through a single conversational LINE interface.

The project demonstrates a full end-to-end workflow: web scraping price data from multiple e-commerce sites, persisting the data in SQLite, exposing it through a Flask backend, integrating with the LINE Messaging API for the user-facing chat layer, and connecting to both cloud LLM (OpenAI GPT) and a local LLM (Ollama / Taide) for natural-language interaction.

---

## Overview

The bot is organized into five independent feature modules, each implemented as a Jupyter notebook, plus a unified entry-point notebook that combines all features behind a single menu. Users interact with the bot through structured button menus (LINE template messages) and free-text input, with state management handled via in-memory global variables and SQLite persistence.

Core feature pillars:

- **AI consultation** — choose between OpenAI GPT-3.5 and a self-hosted Taide LLM through a button menu, then chat back and forth on any 3C-related topic.
- **Phone price lookup** — multi-step menu (store -> system -> brand -> model) querying scraped phone data, with a fallback free-text search.
- **Laptop price lookup** — parallel flow for laptops, with separate Mac / Windows branching and store-specific brand lists.
- **3C news headlines** — on-demand scraping of the latest article from a tech news site.
- **Customer feedback** — accepts structured `title、content` submissions and persists them to a SQLite reports table.
- **Budget-based search** — user picks category, store, and price range; the bot returns matching products.

---

## Architecture

```
LINE App (user)
      |
      v
LINE Messaging API webhook
      |
      v
Flask server (notebook)  <----- ngrok tunnel (for local dev)
      |
      +--> SQLite databases
      |       products.db   (phone data)
      |       products2.db  (laptop data)
      |       reports.db    (customer feedback)
      |
      +--> Web scrapers (Selenium / requests + BeautifulSoup)
      |       traditional 3C retailers + tech news sites
      |
      +--> LLM clients
              OpenAI API   (GPT-3.5)
              Ollama       (Taide local model)
```

The Flask server exposes a single `POST /` callback that the LINE webhook hits. Inside, the LINE SDK dispatches text and postback events to the appropriate handler functions, which in turn query the SQLite databases or call out to the scrapers / LLMs.

---

## Tech Stack

- **Language** Python 3
- **Web framework** Flask
- **Messaging platform** LINE Messaging API (linebot SDK v2 and v3)
- **Tunneling** ngrok / pyngrok (for exposing the local Flask server to LINE's webhook)
- **Scraping** requests, BeautifulSoup, Selenium
- **Database** SQLite (`products.db`, `products2.db`, `reports.db`)
- **LLM clients** OpenAI Python SDK (GPT-3.5-turbo), Ollama-compatible client (Taide llama3-taide-lx-8b-chat-alpha1)

---

## Project Structure

```
linebot_project/
  3c_assistant.ipynb         Main hub — AI consultation (GPT / Taide selection + chat)
  unified_assistant.ipynb    All-in-one entry point combining every feature behind one menu
  phone_scraper.ipynb        Scraper for phone data from 傑昇通信 and 地標網通 -> products.db
  phone_module.ipynb         LINE bot flow for the phone price-lookup feature
  laptop_module.ipynb        LINE bot flow for the laptop price-lookup feature
  news_module.ipynb          Scrapes the latest 3C news article and replies on demand
  customer_service.ipynb     Customer feedback collection -> reports.db
  budget_planner.ipynb       Budget-based product search across phones and laptops
  products.db                Scraped phone data (name, price, store)
  products2.db               Scraped laptop data
  reports.db                 Customer feedback (title, content)
```

---

## Feature Walkthrough

### 1. AI Consultation (`3c_assistant.ipynb`)

User sends the keyword `3C助手` to trigger a button menu that lets them pick **GPT** or **Taide**. After selection, the bot enters an interactive chat mode where the user's questions are appended to a running conversation history and sent to the chosen model. Typing `結束` exits chat mode.

Both clients share the same OpenAI-compatible API surface — the Taide client points to a local Ollama endpoint, making it possible to switch between cloud and local inference with a single button.

### 2. Phone & Laptop Price Lookup (`phone_module.ipynb` / `laptop_module.ipynb`)

A guided multi-step flow:

1. User types `手機` or `筆電` -> bot shows store options (傑昇通信 / 地標網通 / 皆可 for phones; 創捷國際 / 台灣大哥大 / 皆可 for laptops).
2. Store selection -> system / OS menu (Android / iOS for phones; Mac OS / Windows for laptops).
3. System selection -> brand menu (dynamically adjusted; Taiwan Mobile shows a reduced Windows brand list, for example).
4. Brand selection -> top three models pulled from the SQLite database matching the chosen brand and store, plus an `其他` (Other) option.
5. Model selection -> bot replies with the matched product record (name, price, store, optional URL).

If the user picks `其他` at any level, the bot enters free-text search mode, allowing them to type a product name (e.g. `iPhone 15`, `Samsung A55`) until they type `結束` to exit.

### 3. 3C News (`news_module.ipynb`)

Typing `3C新聞` triggers a one-shot scrape of the latest article on a Taiwanese tech news site. The bot replies with the article title and URL.

### 4. Customer Feedback (`customer_service.ipynb`)

Accepts messages in the format `標題、內文`. The bot splits on the full-width comma `、`, persists `(title, content)` into the `reports` table of `reports.db`, and acknowledges receipt with a confirmation message.

### 5. Budget-Based Search (`budget_planner.ipynb`)

Users pick a category (phone / laptop), then a store, then a preset price range (e.g. `10000以下`, `10000-20000`, `20000以上`, `其他`). The bot runs a parameterized SQL query over the matching SQLite table — using `CAST(REPLACE(REPLACE(price, '$', ''), ',', '') AS INTEGER)` to handle raw scraped price strings — and returns up to 20 matching items.

`其他` puts the bot into custom-range mode, accepting input in the form `min-max` (e.g. `15000-25000`).

### 6. Unified Entry Point (`unified_assistant.ipynb`)

Combines all five features behind a single LINE bot instance. A `current_function` global tracks which sub-flow the user is in so that shared menus (e.g. store selection) route correctly. This is the version intended to run in production.

---

## Database Schemas

### `products.db` — Phone data (table: `products`)

| Column | Type | Description |
| --- | --- | --- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Row ID |
| `name` | TEXT | Product name (model) |
| `price` | REAL / TEXT | Listed price |
| `store` | TEXT | Source retailer (`傑昇通信` / `地標網通`) |

### `products2.db` — Laptop data (table: `products2`)

Same schema as above; sources are `創捷國際` and `台灣大哥大`.

### `reports.db` — Customer feedback (table: `reports`)

| Column | Type | Description |
| --- | --- | --- |
| `id` | INTEGER PRIMARY KEY AUTOINCREMENT | Report ID |
| `title` | TEXT NOT NULL | Feedback title |
| `content` | TEXT NOT NULL | Feedback body |

---

## How to Run

### Prerequisites

- Python 3.10+
- LINE Messaging API channel (channel access token + channel secret)
- OpenAI API key (for GPT mode)
- Ollama running locally on the configured host/port (for Taide mode; optional)
- ngrok account + auth token (for local development webhook exposure)

### Environment variables

Set these before launching any notebook:

```bash
LINE_ACCESS_TOKEN=<your LINE channel access token>
LINE_SECRET=<your LINE channel secret>
OPENAI_API_KEY=<your OpenAI API key>
NGROK_KEY=<your ngrok auth token>
```

### Install dependencies

```bash
pip install flask line-bot-sdk requests beautifulsoup4 openai pyngrok selenium
```

### Steps

1. **Run the scraper** (`phone_scraper.ipynb`) to populate `products.db` with phone data. Repeat the equivalent for laptops to populate `products2.db`.
2. **Open the unified entry point** (`unified_assistant.ipynb`) and run all cells. Flask listens on `http://127.0.0.1:5000`.
3. **Start an ngrok tunnel** pointing at port 5000 and copy the generated `https://...ngrok-free.app` URL into the LINE Developers console as the webhook URL (append `/` since the callback route is the root).
4. **Add your LINE bot as a friend** in the LINE app, then send the trigger keywords (`3C助手` / `手機` / `筆電` / `3C新聞` / `預算`) to start each flow.

---

## Highlights

- **Multi-feature LINE bot** in a single, modular codebase — every feature can be run standalone or as part of the unified flow.
- **Cloud + local LLM switching** at runtime via a button menu, demonstrating a flexible LLM abstraction.
- **End-to-end data pipeline**: web scraping -> SQLite persistence -> parameterized SQL queries -> LINE Messaging API replies.
- **Multi-step button-driven UX** using LINE's `TemplateSendMessage` + `ButtonsTemplate` + `PostbackTemplateAction`, including dynamic option lists pulled directly from the database.
- **Stateful conversation handling** — global flags (`awaiting_custom_query`, `current_function`, `selected_store`) coordinate multi-turn user flows without an external session store.
- **Safe SQL** via parameterized queries throughout, including for price-range filtering.

---

## Notes and Limitations

- The linebot SDK v2 classes (`LineBotApi`, `WebhookHandler` from `linebot`) trigger deprecation warnings in newer SDK versions. The customer-service and news modules already use the v3 API (`linebot.v3.*`); the other modules can be migrated by following the same pattern.
- Scraper selectors target specific HTML class names of the source sites at the time of development. Site redesigns will require updating selectors.
- The Taide endpoint URL points to a local Ollama instance on a private IP — adjust `taide_client.base_url` to your own host before running.
- Free-tier ngrok limits a single agent session at a time; the same token cannot run two tunnels in parallel.
- The implementation prioritizes clarity over production-readiness — global state is held in module-level variables (fine for a single-user demo, but would need to be moved into per-user session storage for real multi-user deployment).
