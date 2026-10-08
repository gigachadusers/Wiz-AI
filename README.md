# WIZ AI (ReadMe Created By wiz ai as part of a test lol)

**Local AI created by Wizz3ard.**

Wiz AI is a complete, local-first AI platform that runs on your own computer using [Ollama](https://ollama.com).
It chats with streaming responses, learns from your Markdown documents through retrieval (RAG), keeps a
transparent long-term memory, learns lessons from your corrections and feedback, writes an honest daily learning
journal, and tracks its own learning goals — all stored locally.

> Wiz AI's learning system improves its future responses through local memory, document retrieval, feedback,
> corrections and lessons. The underlying Ollama model's neural weights are not automatically changed.

---

## Contents

1. [Features](#features)
2. [Requirements](#requirements)
3. [Windows setup](#windows-setup)
4. [Linux and macOS setup](#linux-and-macos-setup)
5. [Running Wiz AI](#running-wiz-ai)
6. [Models](#models)
7. [Knowledge system](#knowledge-system)
8. [Memory system](#memory-system)
9. [Learning system](#learning-system)
10. [Backups](#backups)
11. [Configuration](#configuration)
12. [Architecture](#architecture)
13. [Testing](#testing)
14. [Troubleshooting](#troubleshooting)

---

## Features

- **Chat** — ChatGPT-style interface, streaming answers, Stop button, regenerate, delete messages, export
  (Markdown / JSON), automatic titles, search, rename, delete, chats grouped by Today / Yesterday / Previous 7 days /
  Older. Chats persist across restarts.
- **Code blocks** — every code block has a language label, syntax highlighting, horizontal scrolling, an optional
  wrap toggle and a real **Copy** button that copies only the code and briefly shows **Copied**.
- **Knowledge (RAG)** — upload whole folders or single files: code in ~60 languages, Markdown, text, notebooks,
  PDF, Word and config files. Code is split by functions and classes, documents by sections. Includes local
  embeddings, versioning, unchanged/duplicate detection, folder groups, knowledge search, and a "Knowledge used"
  panel showing the exact file, function and lines behind each answer.
- **Long-term memory** — automatic, suggested or manual memory; importance / confidence / stability scores; secret
  filtering; duplicate and conflict detection; full history; "Why was this memory created?" inspection; edit, forget
  and delete.
- **Learning** — correction detection (verified against your documents), lessons from 👎 feedback, a transparent
  learning timeline, daily learning reports generated from real data, and learning goals with measured progress.
- **Models & hardware** — Ollama status, installed models, install (with progress) / select / delete, hardware
  detection (CPU, RAM, NVIDIA / AMD / Intel / Apple Silicon GPU, storage) and hardware-aware recommendations
  (Best quality, Best balanced, Fastest, Lowest memory) plus a recommended embedding model.
- **Prompt file** — `prompt/prompt.md` is read before every response and followed as instructions.
- **Settings, health dashboard, backups, first-run welcome check**, responsive layout (mobile drawer sidebar), a
  strict black / white / grey interface.
- **100% local** — no cloud AI, no telemetry. Chats, documents, embeddings, memories and reports stay on your computer.

---

## Requirements

| Software | Version | Download |
|----------|---------|----------|
| Ollama | latest | https://ollama.com/download |
| Python | 3.10 – 3.13 | https://www.python.org/downloads/ |
| Node.js | 18 or newer (LTS recommended) | https://nodejs.org/ |

Hardware: any 64-bit computer with at least 8 GB RAM. A GPU (NVIDIA, AMD, Apple Silicon) makes answers much faster.
Wiz AI recommends a model that suits your computer.

---

## Windows setup

### 1. Install Ollama

1. Download and run the Windows installer from https://ollama.com/download.
2. Ollama starts automatically in the system tray. Check it in **Command Prompt**:

```bat
ollama --version
```

### 2. Install Python

1. Download Python 3.12 (or 3.11 / 3.13) from https://www.python.org/downloads/windows/.
2. In the installer **tick "Add python.exe to PATH"**, then click *Install Now*.
3. Check it:

```bat
python --version
```

### 3. Install Node.js

1. Download the **LTS** installer from https://nodejs.org/ and install it with the default options.
2. Check it:

```bat
node --version
npm --version
```

### 4. Start Wiz AI

Double-click **`start.bat`** in the `wiz-ai` folder (or run it from Command Prompt):

```bat
cd %USERPROFILE%\Desktop\wiz-ai
start.bat
```

The first start creates a Python virtual environment and installs dependencies (a few minutes). Then it opens two
windows — **Wiz AI Backend** and **Wiz AI Frontend** — and your browser at **http://127.0.0.1:5173**.

The welcome screen checks your system and offers **Install Recommended Model**, which downloads the recommended chat
model and the `nomic-embed-text` embedding model. To stop Wiz AI, close the two windows.

---

## Linux and macOS setup

```bash
# Ollama
curl -fsSL https://ollama.com/install.sh | sh      # Linux  (macOS: install the app from ollama.com)

# Python 3.10+ and Node.js 18+ from your package manager, for example:
sudo apt install python3 python3-venv nodejs npm   # Debian / Ubuntu
brew install python node                           # macOS (Homebrew)

# Start Wiz AI
cd ~/Desktop/wiz-ai
chmod +x start.sh
./start.sh
```

Press **Ctrl+C** in the terminal to stop Wiz AI.

---

## Running Wiz AI

| URL | What |
|-----|------|
| http://127.0.0.1:5173 | The Wiz AI interface (development server) |
| http://127.0.0.1:8000/api/health | Backend health |
| http://127.0.0.1:8000/docs | Interactive API documentation |

**Single-port mode (optional).** Build the interface once with `scripts\build-frontend.bat` (or
`scripts/build-frontend.sh`). The backend then serves the full app at **http://127.0.0.1:8000**, so you only need
to run the backend:

```bat
cd backend
.venv\Scripts\python.exe -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

**Manual start (any OS).**

```bash
# Terminal 1 — backend
cd backend
python -m venv .venv
.venv/bin/pip install -r requirements.txt          # Windows: .venv\Scripts\pip install -r requirements.txt
.venv/bin/python -m uvicorn app.main:app --host 127.0.0.1 --port 8000

# Terminal 2 — frontend
cd frontend
npm install
npm run dev
```

---

## Models

Open **Models** in the sidebar.

- **Chat model** — Wiz AI uses this model to generate answers.
- **Embedding model** — Wiz AI uses this model to search your knowledge and memories (default
  `nomic-embed-text`). Changing it re-indexes your documents automatically.

Recommendations are calculated from your RAM, GPU memory, GPU type, CPU cores, model size, quantisation and context
size, with an estimated speed for each model. The biggest model is not automatically recommended: "Best quality"
is the most capable model that still answers at a usable speed.

**Model browser.** The Models page lists about 50 hand-picked Ollama models in four groups:

| Group | What it's for | Examples |
|-------|---------------|----------|
| General | Everyday chat, writing and questions | Qwen3, Llama 3.x, Gemma 3, Mistral, Phi-4, Granite, OLMo 2 |
| Coding | Writing, explaining and reviewing code | Qwen2.5-Coder, Qwen3-Coder, DeepSeek-Coder-V2, Codestral, Devstral, StarCoder2 |
| Reasoning | Step-by-step maths, logic and hard problems | DeepSeek-R1, QwQ, Phi-4 reasoning, Magistral |
| Security | Defensive work: secure code review, explaining vulnerabilities and fixes, reading logs | Qwen2.5-Coder, Qwen3, DeepSeek-R1, Granite, Llama 3.1 |

Each model shows whether it runs well, slowly or not at all on your PC, its download size, the memory it needs,
and an estimated speed. You can search, filter by size and sort. Every group has a "Best pick" for your hardware.

**Import your own model.** Under **Import your own model**, upload a `.gguf` file, or give the path to one already
on your PC (faster for big files). Wiz AI checks it's a real GGUF model, copies it into Ollama, and it appears in
Installed models. You can also install anything by name, including Hugging Face GGUF repos such as
`hf.co/user/repo:Q4_K_M`.

You can also install models from a terminal:

```bat
ollama pull qwen3:4b
ollama pull nomic-embed-text
```

Typical choices: 8 GB RAM without a GPU → `qwen3:1.7b` or `qwen3:4b`; 8 GB GPU → `qwen3:8b`; 12–16 GB GPU →
`qwen3:14b`; 24 GB GPU → `gemma3:27b`, `qwen3:30b` or `qwen3:32b`.

---

## Knowledge system

Open **Knowledge** and click **Upload Folder** (or drag a folder onto the drop zone). Wiz AI walks the whole folder,
including subfolders, and keeps each file's path, for example `coding-course/python/basics/loops.py`. You can also
use **Upload Files** for single files. Try `samples/wiz-ai-project.md`, then ask *"What database does Wiz AI use?"*.

**Supported files**

| Kind | Examples | How it is split |
|------|----------|-----------------|
| Code | `.py .js .ts .tsx .jsx .java .cs .c .cpp .h .go .rs .php .rb .kt .swift .sql .sh .ps1 .bat .html .css .scss .vue` and ~40 more | By functions, classes and methods. Comments and docstrings stay attached; methods stay grouped under their class |
| Markdown / text | `.md .markdown .txt .rst` and README / LICENSE files | By headings, paragraphs, lists and tables |
| Notebooks | `.ipynb` | Markdown cells as text, code cells as code (with short outputs) |
| Documents | `.pdf .docx` | Text and headings are extracted |
| Data / config | `.json .yaml .toml .xml .csv .ini`, `Dockerfile`, `Makefile`, … | By lines |
| Anything else | any file containing readable text | As text |

**Skipped automatically, with a reason shown:**

- images, archives, executables, media and compiled files;
- `node_modules`, `.git`, `__pycache__`, `.venv`, `dist` and similar folders (Wiz AI doesn't even open them);
- lock files;
- files that usually hold secrets (`.env`, `.pem`, `.key`, private SSH keys, …);
- empty files and files over 25 MB.

What happens to each file:

```text
file → detect type → extract text → split into meaningful pieces (functions/classes or sections)
     → local embeddings → stored in SQLite with path, language and line numbers → retrieved during chat
```

- **Folders.** The top folder becomes a group in the library. You can filter by path or type and delete a whole
  folder at once. Files with the same name in different folders are kept apart.
- **Re-uploading.** Unchanged files are reported as *unchanged* and skipped. Changed files become a new version,
  and their old pieces are removed from search. Identical content at another path is reported as a duplicate.
- **Answers.** When knowledge is used, the **Knowledge used** panel shows the file path, function or section and
  line numbers. Click it to read the exact code or text. Wiz AI is told to follow the conventions your examples use.
- Large folders are uploaded in batches with a progress bar. You can cancel between batches.

**About "training".** Uploading files does **not** change the model's weights. Wiz AI searches your files and puts
the most relevant pieces into the prompt each time you ask something. In practice, that's what makes it answer
from your own examples and explanations. It works immediately and can be undone by deleting a file. True
fine-tuning would be a separate, much heavier process.

---

## Prompt file

Wiz AI reads **`prompt\prompt.md`** in the Wiz AI folder before **every** response and follows it as instructions.
Use it for anything you want applied all the time, for example:

```markdown
# My instructions for Wiz AI

- Explain code step by step.
- Always include a complete, runnable example.
- Use Python unless I ask for another language.
```

- You can edit it in any text editor, type it in **Settings → Prompt file**, or click **Upload prompt file** there
  to use a `.md` or `.txt` file from your computer. The file is read fresh for each message,
  so changes apply from the next message with no restart.
- An empty or missing file adds nothing; Wiz AI creates an empty `prompt.md` on start if it doesn't exist and never
  overwrites yours.
- Each answer's **Context used** panel confirms the file was read and how many characters were used. The **Prompt
  File** line on the Hardware/Settings health panel shows its status.
- Up to 40,000 characters are used. If your context window is small, the prompt file is limited to half of it so
  the conversation still fits.
- Wiz AI's built-in identity and safety rules come first; the prompt file comes right after them.

---

## Memory system

Open **Memory**. Wiz AI keeps memories in these categories: user preference, project information, technical fact,
important decision, correction, workflow, goal and personalisation.

- Memory candidates are extracted after conversations and scored for **importance** and **confidence** (and
  stability). Only candidates above your thresholds are kept.
- Greetings, temporary questions, one-off statements and secrets (passwords, API keys, tokens…) are not remembered.
- Conflicts are understood instead of blindly stored. *"I prefer Python for backend projects"* followed by *"I've
  started using TypeScript for my new backend projects"* becomes a general preference plus a context-specific one.
- Every change is kept in the memory's history. Click **Why?** to see the source conversation, reason, confidence,
  importance and full history.
- Say *"Remember that …"* to save something explicitly.

**Memory modes** (Settings or Learning page): **Automatic** (default), **Suggest** (memories wait for your
approval in the *Suggested* tab) and **Manual** (only what you explicitly save).

---

## Learning system

- **Corrections** — *"That's wrong. The project uses PostgreSQL, not MySQL."* is detected, checked against your
  documents, stored as a correction memory (replacing outdated memories, with history) and turned into a lesson.
- **Feedback** — 👍 / 👎 on every answer. On 👎 Wiz AI asks *"What was wrong?"*, analyses the response and may
  create a reusable lesson.
- **Learning timeline** — every learning event (memories, documents, corrections, lessons, feedback, reports,
  goals) with time and inspectable details.
- **Daily Reports** — *What I learned*, *What I'm learning*, *Mistakes & corrections*, *Improvements* and *My goals*,
  written by your local model from real statistics (chats, messages, documents, chunks, memories, lessons,
  corrections, feedback, searches, retrievals, goals). Quality metrics say **"Not enough data yet."** instead of
  inventing numbers. Past days are kept in the history list.
- **Learning Goals** — generated from real activity, measured automatically, completed (✓) or archived when no
  longer relevant. You can add, complete, archive or delete your own goals.
- You stay in control: disable automatic learning, feedback learning, correction learning, daily reports or goal
  generation in **Settings**, and use **Delete learning history** on the Learning page.

Wiz AI never modifies its own code, runs commands, installs packages or changes system settings.

---

## Starting over

**Settings → Danger zone → Reset Wiz AI** deletes everything Wiz AI has stored or learned: chats, memories (and
their history), lessons, feedback, learning events, daily reports, goals and uploaded knowledge. You can create a
backup first, and choose to keep your settings, keep your prompt file, or also delete saved backups. You must type
`RESET` to confirm. Your installed Ollama models are never removed; delete them on the Models page if you want.

---

## Backups

Open **Backups** (System section).

- **Create backup** saves a ZIP in the `backups` folder; **Download backup** downloads one directly.
- Backups contain conversations, messages, memories and memory history, lessons, feedback, document metadata and
  Markdown content, learning events, daily reports, goals and settings.
- Embeddings are not included — after **Import backup** Wiz AI re-indexes documents and re-embeds memories locally.
- Importing replaces all current data, so create a backup first.

---

## Configuration

Most options are in **Settings**: appearance (sidebar, font size, Enter to send), AI (chat model, temperature,
context window, maximum output, custom instructions), knowledge (embedding model, chunk size, overlap, retrieval
count, minimum relevance), memory (mode, thresholds), learning toggles and system information.

Optional environment variables are documented in `.env.example`:

| Variable | Default | Purpose |
|----------|---------|---------|
| `WIZ_OLLAMA_URL` (or `OLLAMA_HOST`) | `http://127.0.0.1:11434` | Ollama address |
| `WIZ_PORT` | `8000` | Backend port |
| `WIZ_FRONTEND_PORT` | `5173` | Interface port |
| `WIZ_DATA_DIR`, `WIZ_DB_PATH` | `data/`, `data/wiz.db` | Database location |
| `WIZ_UPLOADS_DIR`, `WIZ_BACKUPS_DIR` | `uploads/`, `backups/` | File storage |
| `WIZ_MAX_UPLOAD_MB` | `25` | Maximum size per uploaded file |
| `WIZ_REPORT_INTERVAL` | `3600` | Background daily-report refresh (seconds) |
| `WIZ_PROMPT_FILE` | `prompt/prompt.md` | Prompt file read before every response |

---

## Architecture

```text
React + TypeScript (Vite, Tailwind)  ──/api (HTTP + SSE)──▶  FastAPI
                                                              ├── SQLite (chats, knowledge, vectors, memory, learning)
                                                              ├── RAG (Markdown parser, chunker, NumPy vector search)
                                                              ├── Learning engine (memory, lessons, corrections,
                                                              │                    reports, goals, events)
                                                              └── Ollama (chat, streaming, embeddings, models)
```

See **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** for the full architecture, design decisions (including why
SQLite + NumPy was chosen as the vector store), the project tree and the API reference.

---

## Testing

```bat
scripts\run-tests.bat
```

```bash
./scripts/run-tests.sh
```

- **Backend (pytest)** — database, settings, security, Markdown parsing and chunking, Ollama client (streaming,
  errors, `<think>` filtering), chat API and streaming, uploads, RAG end-to-end (the sample document is uploaded,
  indexed and used to answer *"What database does Wiz AI use?"*), memory extraction, conflicts and modes,
  corrections, feedback lessons, daily reports from real data, goals, backups and model recommendations for
  simulated 8 GB CPU-only and 64 GB RAM / 24 GB VRAM machines. The tests use a small fake Ollama server, so no model
  download is needed.
- **Frontend (Vitest)** — code block rendering and Copy for Python, JavaScript, TypeScript, HTML, CSS, JSON, SQL,
  Bash, PowerShell, C#, Java, C++, Rust and Go; Markdown rendering; sidebar; streaming chat with Stop; feedback;
  upload UI; memory UI; daily learning dashboard.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Wiz AI couldn't connect to Ollama." | Start the Ollama app (or run `ollama serve`), then press **Retry**. Check http://127.0.0.1:11434/api/version in a browser. |
| "No chat model is installed" | Open **Models** and install a recommended model, or run `ollama pull qwen3:4b`. |
| Documents show **Failed** — "embedding model is not installed" | Install `nomic-embed-text` on the Models page, then click **Re-index**. |
| Answers ignore my documents | Check the document is **Indexed** and searchable, try **Search knowledge** on the Knowledge page, or lower *Minimum relevance* in Settings. |
| Answers are slow | Choose the *Best balanced* or *Fastest* model, or lower the context window in Settings. |
| `python` not found on Windows | Reinstall Python with *Add python.exe to PATH* ticked, or disable the Microsoft Store "App execution alias" for Python. |
| Port 8000 or 5173 already in use | Set `WIZ_PORT` / `WIZ_FRONTEND_PORT` before running the start script. |
| Interface shows "couldn't reach its backend" | Look at the **Wiz AI Backend** window for errors and make sure it is running. |
| Reset everything | Stop Wiz AI and move the `data` folder somewhere else (it contains the database). Create a backup first. |

---

**Wiz AI** — local AI created by **Wizz3ard**.
