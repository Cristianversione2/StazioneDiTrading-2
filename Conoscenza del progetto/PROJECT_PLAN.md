# PROJECT PLAN — Aware Trading Workspace

> Documento di riferimento architetturale. Consultare prima di scrivere nuovo codice.

---

## 1. Architettura generale

```
Browser (React)
    │
    ├── Dashboard Page
    ├── Workspace Page (analisi)
    │       ├── Chat Panel
    │       ├── Upload Panel
    │       └── Session Memory Panel (sidebar)
    └── Journal Page
    
    │ HTTP / fetch
    
Node.js + Express (API Server)
    │
    ├── /api/sessions       → CRUD sessioni
    ├── /api/messages       → messaggi chat + upload screenshot
    ├── /api/journal        → righe CSV journal
    └── /api/agent          → orchestrazione AI (OpenRouter)
    
    │
    ├── SQLite (better-sqlite3)   → persistenza locale
    ├── /uploads/                 → screenshot su disco
    └── Agent Orchestrator
            │
            ├── Skill Loader      → carica i file del kit (01,04,06,07,08,09)
            ├── Prompt Builder    → costruisce il system prompt + history
            └── Provider Client   → chiamata OpenRouter (vision + text)
```

---

## 2. Data Model (SQLite)

### Tabella `sessions`

| Campo | Tipo | Note |
|---|---|---|
| id | TEXT (UUID) | PK |
| created_at | TEXT (ISO8601) | timestamp creazione |
| updated_at | TEXT (ISO8601) | timestamp ultimo aggiornamento |
| asset | TEXT | es. "XAUUSD", nullable fino a rilevamento |
| status | TEXT | "active" \| "closed" |
| title | TEXT | generato automaticamente o inserito |

### Tabella `messages`

| Campo | Tipo | Note |
|---|---|---|
| id | TEXT (UUID) | PK |
| session_id | TEXT | FK → sessions.id |
| created_at | TEXT (ISO8601) | timestamp |
| role | TEXT | "user" \| "assistant" |
| content | TEXT | testo del messaggio |
| screenshots | TEXT (JSON) | array di path file, es. `["uploads/abc.png"]` |

### Tabella `session_memory`

| Campo | Tipo | Note |
|---|---|---|
| id | TEXT (UUID) | PK |
| session_id | TEXT | FK → sessions.id, UNIQUE |
| asset | TEXT | asset rilevato |
| timeframes | TEXT (JSON) | es. `["15m","4H"]` |
| structure | TEXT | descrizione struttura corrente |
| levels | TEXT | livelli chiave osservati |
| notes | TEXT | note libere |
| updated_at | TEXT (ISO8601) | |

### Tabella `journal_entries`

| Campo | Tipo | Note |
|---|---|---|
| id | TEXT (UUID) | PK |
| session_id | TEXT | FK → sessions.id |
| created_at | TEXT (ISO8601) | |
| data | TEXT | data trade YYYY-MM-DD |
| ora | TEXT | HH:MM |
| asset | TEXT | |
| timeframe | TEXT | |
| bias | TEXT | Long \| Short \| Neutrale |
| setup | TEXT | Solido \| Discutibile \| Debole |
| modalita | TEXT | |
| entry | TEXT | prezzo o "—" |
| stop_loss | TEXT | prezzo o "—" |
| take_profit_1 | TEXT | |
| take_profit_2 | TEXT | |
| risk_reward | TEXT | "1:X.X" o "—" |
| size | TEXT | |
| decisione_agente | TEXT | |
| decisione_trader | TEXT | |
| esito | TEXT | |
| durata_trade | TEXT | |
| nota | TEXT | |
| screenshot_link | TEXT | |
| csv_row | TEXT | riga CSV completa generata |

---

## 3. Agent Architecture

### 3.1 Skill System

I file del kit (`01`, `04`, `06`, `07`, `08`, `09`) sono caricati come **system prompt** all'avvio della sessione. Sono file `.md` statici letti da disco (`/kit/`).

```
AgentOrchestrator
  .buildSystemPrompt()
    → legge /kit/01_METODO_OPERATIVO.md
    → legge /kit/04_TEMPLATE_OUTPUT.md
    → legge /kit/06_PROFILI_ASSET.md
    → legge /kit/07_CAUTELE_TECNICHE.md
    → legge /kit/08_STILE_RISPOSTA.md
    → legge /kit/09_PROFILO_AWARE_TRADER.md
    → concatena con separatori
    → antepone il PROMPT_MASTER (02)
    → restituisce system prompt completo
```

### 3.2 Costruzione del messaggio all'AI

Per ogni turno della chat:

```
buildMessages(session_id, new_user_message, screenshots[])
  1. carica history messaggi dalla DB (session_id)
  2. formatta history come array [{role, content}]
  3. aggiunge screenshot come content blocks (base64 o URL)
  4. aggiunge nuovo messaggio utente
  5. invia a OpenRouter con system prompt
  6. salva risposta in DB
  7. aggiorna session_memory se l'agente ha rilevato nuove info
```

### 3.3 Gestione screenshot

- Upload via `multipart/form-data` a `/api/messages`
- Multer salva i file in `/uploads/{session_id}/`
- I file vengono inviati all'AI come base64 inline (OpenRouter vision)
- Il path relativo viene salvato in `messages.screenshots`

### 3.4 Session Memory (pannello laterale)

La session memory NON è gestita automaticamente dall'AI. Dopo ogni risposta dell'agente, il backend fa una seconda chiamata AI leggera per estrarre in JSON strutturato:
- asset rilevato
- timeframe menzionati
- struttura descritta
- livelli citati

Questo viene salvato in `session_memory` e mostrato nel pannello laterale.  
*(Ottimizzazione MVP: si può fare manualmente con un pulsante "aggiorna memoria" invece di automaticamente)*

### 3.5 Provider AI (OpenRouter)

Endpoint: `https://openrouter.ai/api/v1/chat/completions`  
Modello MVP: `anthropic/claude-3.5-sonnet` (vision support)  
Headers richiesti:
```
Authorization: Bearer {OPENROUTER_API_KEY}
HTTP-Referer: http://localhost:3000
X-Title: Aware Trading Workspace
Content-Type: application/json
```

---

## 4. Struttura cartelle del progetto

```
aware-trading-workspace/
├── client/                    ← React + Vite
│   ├── src/
│   │   ├── pages/
│   │   │   ├── Dashboard.jsx
│   │   │   ├── Workspace.jsx
│   │   │   └── Journal.jsx
│   │   ├── components/
│   │   │   ├── chat/
│   │   │   │   ├── ChatPanel.jsx
│   │   │   │   ├── MessageBubble.jsx
│   │   │   │   └── UploadArea.jsx
│   │   │   ├── session/
│   │   │   │   ├── SessionMemory.jsx
│   │   │   │   └── SessionList.jsx
│   │   │   └── journal/
│   │   │       └── JournalTable.jsx
│   │   ├── store/
│   │   │   ├── sessionStore.js    ← Zustand
│   │   │   └── uiStore.js
│   │   ├── api/
│   │   │   └── client.js          ← fetch wrapper
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── index.html
│   └── vite.config.js
│
├── server/                    ← Node.js + Express
│   ├── src/
│   │   ├── routes/
│   │   │   ├── sessions.js
│   │   │   ├── messages.js
│   │   │   ├── journal.js
│   │   │   └── agent.js
│   │   ├── agent/
│   │   │   ├── orchestrator.js    ← logica principale
│   │   │   ├── skillLoader.js     ← carica i file kit
│   │   │   ├── promptBuilder.js   ← costruisce i messaggi
│   │   │   └── providerClient.js  ← chiamata OpenRouter
│   │   ├── db/
│   │   │   ├── database.js        ← init SQLite
│   │   │   └── migrations/
│   │   │       └── 001_init.sql
│   │   └── index.js               ← entry point Express
│   └── uploads/                   ← screenshot (gitignored)
│
├── kit/                       ← file skill agent (read-only)
│   ├── 01_METODO_OPERATIVO.md
│   ├── 02_PROMPT_MASTER_AGENT.md
│   ├── 04_TEMPLATE_OUTPUT.md
│   ├── 06_PROFILI_ASSET.md
│   ├── 07_CAUTELE_TECNICHE.md
│   ├── 08_STILE_RISPOSTA.md
│   └── 09_PROFILO_AWARE_TRADER.md
│
├── .env.example
├── .gitignore
└── package.json               ← root (o workspace separati client/server)
```

---

## 5. Configurazione ambiente

### `.env.example`
```
OPENROUTER_API_KEY=sk-or-...
PORT=3001
DB_PATH=./server/data/aware_trading.db
UPLOADS_PATH=./server/uploads
```

### `.gitignore` (elementi chiave)
```
.env
server/data/
server/uploads/
node_modules/
dist/
```

---

## 6. Decisioni tecniche e motivazioni

| Decisione | Alternativa scartata | Motivazione |
|---|---|---|
| SQLite | PostgreSQL / MongoDB | Zero infrastruttura, uso singolo, file locale portabile |
| OpenRouter | Anthropic diretto | Multi-modello, un'unica chiave API, fallback facile |
| React + Vite | Next.js | Niente SSR necessario, setup più veloce con Copilot |
| Session memory separata | Solo history messaggi | Il pannello laterale richiede dati strutturati, non testo |
| File kit statici | DB skills | I file cambiano raramente, lettura da disco è sufficiente |
| Express puro | Fastify / Hono | Massima documentazione disponibile per Copilot |

---

## 7. Considerazioni sicurezza (uso locale)

- La webapp gira solo in localhost, nessun accesso esterno
- La API key OpenRouter va SOLO in `.env`, mai nel codice
- Gli upload sono limitati a immagini (png, jpg, webp) con validazione MIME
- Nessuna autenticazione necessaria (uso singolo locale)
- Aggiungere un semplice PIN locale solo se si decide di esporre su rete locale

---
*Ultima modifica: — | Versione: 0.1*
