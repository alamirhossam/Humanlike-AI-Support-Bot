# 🤖 Humanlike AI Support Bot

> An n8n automation workflow that simulates a real human sales agent on Telegram — with a persistent identity, long-term memory, message debouncing, typing indicators, and sentence-by-sentence replies.

---

## 📌 Overview

This workflow powers **Sara**, an AI-driven sales assistant for a car dealership in Egypt selling **Audi**, **Volkswagen**, and **Skoda**. Sara communicates entirely in Egyptian colloquial Arabic, handles customer inquiries naturally, and behaves indistinguishably from a real human agent on Telegram.

The bot is built on **n8n** and uses **Groq (LLaMA 3.3 70B)** as the language model, **Upstash Redis** for memory and message queuing, and the **Telegram Bot API** for delivery.

---

## ✨ Features

### 🧠 Persistent Identity
Sara has a fixed name, role, and personality defined in a dedicated `Prompt & Rules` node. She uses feminine Egyptian Arabic, never breaks character, and follows strict conversational rules — no filler words, no echoing the customer, no internal reasoning leaking into replies.

### 💾 Long-Term Memory
After every reply, the bot saves a memory snapshot to **Upstash Redis** (TTL: 1 hour), including:
- Last user message
- Last bot reply
- Customer name
- Timestamp

On the next message, memory is fetched and injected into the prompt context — so Sara remembers what was discussed without the user repeating themselves.

### ⏳ Message Debouncing (Human-Like Batching)
The workflow uses a **Redis-based queuing system** to wait for the user to finish typing before responding. If a user sends multiple messages in quick succession, they are all collected and processed together as one input — just like a real person would wait before replying.

This is achieved through a pipeline of:
- `Pipeline_A` — pushes each incoming message to a Redis list and increments a counter
- `Pipeline_B` — sets a deadline (7 seconds) and acquires a distributed lock
- `IF_lock` + `Wait` + `GET_deadline` + `IF_deadline` — loops until the deadline passes, then drains the queue
- `Pipeline_C` — atomically reads all queued messages and clears the queue + lock

### ⌨️ Typing Indicator
Before every response part is sent, the bot fires a **Telegram `sendChatAction` (typing)** event, so the customer sees the "typing..." status — exactly like a real agent.

### 🗂️ Split Replies
The bot's reply is split by sentence (`.` delimiter) and delivered **one sentence at a time** with a **0.5-second delay** between each part, using a loop + wait node. This mimics the natural rhythm of a human typing and sending short bursts.

### 🛡️ Output Sanitization
A dedicated `Parse Output` node strips any AI reasoning leakage (`<think>` tags, code blocks, internal patterns like "the user said" or "according to the rules"), and falls back to pre-written Arabic responses if the output quality check fails.


---

## 🏗️ Architecture

```
Telegram Message
    └── Queue & Debounce (Upstash Redis)
        └── Fetch Long-Term Memory
            └── Build Prompt (identity + memory + input)
                └── AI Agent (Groq LLaMA 3.3 70B)
                    └── Sanitize Output
                        └── Save Memory
                            └── Split by Sentence
                                └── Send Typing → Wait → Send Part (loop)
```

---

## 🛠️ Tech Stack

| Component | Tool |
|---|---|
| Automation engine | [n8n](https://n8n.io) |
| Language model | [Groq — LLaMA 3.3 70B](https://groq.com) |
| Memory & queue | [Upstash Redis](https://upstash.com) |
| Messaging platform | [Telegram Bot API](https://core.telegram.org/bots) |

---

## ⚙️ Setup

### Prerequisites
- A running n8n instance (self-hosted or cloud)
- A Telegram Bot token (via [@BotFather](https://t.me/BotFather))
- A Groq API key
- An Upstash Redis database (REST API)

### Credentials to Configure

| Credential | Used In |
|---|---|
| `telegramApi` | Telegram Trigger, Send Chat Action, Reply to Customer |
| `groqApi` | Groq Chat Model |
| `httpBearerAuth` | All Upstash Redis HTTP calls |

### Steps

1. Import the workflow JSON into your n8n instance.
2. Set up the three credentials listed above.
3. Update the Upstash Redis base URL in all HTTP Request nodes to point to your own database.
4. Activate the workflow — the Telegram webhook registers automatically.
5. Customize Sara's identity, rules, and car catalog in the **`Prompt & Rules - Edit Here`** node.

---

## 🧩 Customization

Everything about Sara's behavior lives in a single node: **`Prompt & Rules - Edit Here`**.

You can modify:
- **Persona** — name, role, tone, language style
- **Car catalog** — models, prices, monthly installments
- **Conversation rules** — greeting logic, recommendation logic, fallback behavior
- **Intent detection** — buying, test drive, service/maintenance

The rest of the workflow is persona-agnostic and can be reused for any industry or language.

---

## 📁 Project Structure

```
workflow.json          ← The full n8n workflow export
README.md              ← This file
```

---

## 📄 License

MIT — free to use, modify, and deploy.
