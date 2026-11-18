# RAG Workshop in VS Code: setup and run order

This folder runs the whole Vizuara workshop locally in VS Code: the Part 1 and Part 2 notebooks first, then the Supabase + Next.js app from the workshop's own files.

## Where things come from

| Folder | What | Source |
| --- | --- | --- |
| `notebooks/` | 01 to 04: ingestion, chunking, embed/retrieve/generate, evaluation | Written to follow the videos. The instructor's Colab notebooks are not in the shared Drive folder. |
| `deploy/` | `ingest.py`, `ingest_vision.py`, `test_embeddings.py`, SQL | The workshop's files, unchanged. The SQL is split into two files (see step 7). |
| `deploy/nextjs-files/` | `route.ts`, `page.tsx` | The workshop's files, unchanged. Copied into `rag-chat` in step 9. |

## Final layout

```
rag-workshop/
├── .env                         <- you create this (step 3)
├── .env.example
├── .gitignore
├── requirements.txt
├── README.md
├── human-nutrition-text.pdf     <- copy your PDF here (step 2)
├── notebooks/
│   ├── 01_ingestion.ipynb
│   ├── 02_chunking_strategies.ipynb
│   ├── 03_embed_retrieve_generate.ipynb
│   ├── 04_evaluation_ragas.ipynb
│   └── outputs/                 <- created automatically
├── deploy/
│   ├── supabase_01_setup.sql
│   ├── supabase_02_index.sql
│   ├── ingest.py
│   ├── ingest_vision.py
│   ├── test_embeddings.py
│   └── nextjs-files/
│       ├── route.ts
│       └── page.tsx
└── rag-chat/                    <- created by create-next-app (step 8)
```

---

## Part A: one-time setup

### 1. Install the tools

- **Python 3.12** from python.org. On Windows, tick "Add python.exe to PATH" in the installer.
- **VS Code** with the **Python** and **Jupyter** extensions (both from Microsoft).
- **Node.js LTS** from nodejs.org. Only needed for the web app (Part C).

### 2. Open the folder and add the PDF

1. Unzip this folder somewhere simple, e.g. `C:\dev\rag-workshop`.
2. VS Code: File > Open Folder > `rag-workshop`.
3. Copy `human-nutrition-text.pdf` into the `rag-workshop` folder itself (next to this README, not inside `notebooks` or `deploy`).

### 3. Create `.env`

Copy `.env.example` to a new file named `.env` in the same folder and fill in:

| Key | Where to find it | Needed for |
| --- | --- | --- |
| `OPENAI_API_KEY` | platform.openai.com > Settings > API keys | LLM chunking (02), generation (03), RAGAS (04), all deploy scripts |
| `SUPABASE_URL` | Supabase > Project Settings > Data API > Project URL | Part C |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase > Project Settings > API Keys > `service_role` | Part C |
| `HF_TOKEN` | huggingface.co > Settings > Access Tokens | Only for the Gemma option in notebook 03 |

You can leave the Supabase keys empty until Part C.

### 4. Create the virtual environment and install packages

Open the VS Code terminal (Terminal > New Terminal) in the `rag-workshop` folder.

**Windows (PowerShell)**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If PowerShell refuses to run the activate script, run this once, then activate again:
`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

The install takes a few minutes (it includes PyTorch for sentence-transformers).

### 5. Point VS Code at the environment

- `Ctrl+Shift+P` > **Python: Select Interpreter** > choose the one in `.venv`.
- Open any notebook, click **Select Kernel** (top right) > **Python Environments** > `.venv`.

---

## Part B: the notebooks (follow along with the videos)

Run them in order; each saves what the next one needs into `notebooks/outputs/`. Run cells top to bottom with `Shift+Enter`.

| Notebook | Video | What you do | Costs money? |
| --- | --- | --- | --- |
| `01_ingestion` | Part 1, 1:06:43 to 1:27:00 | Read the PDF, page statistics | No |
| `02_chunking_strategies` | Part 1, 2:38:51 to end | All five chunkers, comparison table and charts | Only the LLM-chunking cell (10 pages, a few cents). Set `RUN_LLM_CHUNKING = False` to skip. |
| `03_embed_retrieve_generate` | Part 2, 11:03 to 2:14:20 | Embed chunks, retrieve, build the prompt, generate, `ask()` | Only if you choose `LLM = "openai"` |
| `04_evaluation_ragas` | Part 2, 2:14:20 to 2:35:24 | Hit-rate check, RAGAS scores | Yes, small (OpenAI is the judge) |

**You are at Part 1, 2:35, so start with `01_ingestion` (quick), then `02_chunking_strategies` as he codes.**

What I checked: notebooks 01 and 02 ran end to end on the nutrition PDF and gave 1,208 pages, 3,318 fixed chunks, 1,843 ten-sentence chunks (1,684 after dropping short ones) and 135 chapter chunks. The cells that download a model or call OpenAI could not be run where I built this, so expect possible small fixes there.

Notes:
- The first run of notebook 02 downloads `all-MiniLM-L6-v2` (about 90 MB); notebook 03 downloads `all-mpnet-base-v2` (about 420 MB). After that they are cached.
- Notebook 03 embeds on CPU in a few minutes and saves the vectors, so later runs are instant.
- Notebook 03's LLM choice: `"openai"` (default), `"ollama"` (local and free; install Ollama and run `ollama pull llama3.2`), or `"gemma"` (what he runs; needs an NVIDIA GPU).

---

## Part C: the deployed app (Part 2 from 2:35:24)

All commands below run from the `rag-workshop` folder with `.venv` active, unless stated.

### 6. Create the Supabase project

supabase.com > New project. Pick the region nearest you. Then copy the Project URL and `service_role` key into `.env` (step 3).

### 7. Run the SQL, in two steps

1. Supabase > **SQL Editor** > paste `deploy/supabase_01_setup.sql` > **Run**. Expect "Success. No rows returned". **Table Editor** should now show an empty `chunks` table.
2. Run step 8's `ingest.py` first, **then** come back and run `deploy/supabase_02_index.sql`.

Why the split: the workshop's single script builds the IVFFlat index on an empty table. IVFFlat picks its clusters from the rows present at build time, so built on nothing, searches can miss good matches. That may be part of why he got "I couldn't find this" answers in the video. Building it after loading avoids this.

### 8. Load the book into Supabase

```powershell
python deploy/ingest.py
```

- Run it from the `rag-workshop` folder: the script looks for `human-nutrition-text.pdf` and `.env` relative to where you run it.
- It deletes any earlier rows for `doc_id = nutrition-v1`, so re-running is safe.
- Expect "Built N chunks", two progress bars, then "Done! Inserted N chunks". Check **Table Editor > chunks**.
- Now run `deploy/supabase_02_index.sql` (step 7.2).

Check retrieval before building any web page:

```powershell
python deploy/test_embeddings.py
```

You should see 3 matches per question with page numbers and similarity scores.

Optional, `deploy/ingest_vision.py`: the same as `ingest.py`, plus it sends **every image in the book** to `gpt-4o` for a description and stores the image itself in the metadata. This is slow and noticeably more expensive, so try it only after the text version works. It replaces the text-only rows (same `doc_id`).

### 9. Create the Next.js app and add the two files

From the `rag-workshop` folder:

```powershell
npx create-next-app@latest rag-chat --ts
```

Answer the prompts so the paths match the workshop files:

| Prompt | Answer |
| --- | --- |
| ESLint | Yes |
| Tailwind CSS | **Yes** (`page.tsx` uses Tailwind classes) |
| Code inside a `src/` directory | **Yes** |
| App Router | **Yes** |
| Turbopack / import alias | Defaults |

If it first asks whether to use the recommended defaults, choose to customise, so that you can answer Yes to `src/`.

Then:

```powershell
cd rag-chat
npm i @supabase/supabase-js openai
```

Place the files:

| Copy | To |
| --- | --- |
| `deploy/nextjs-files/route.ts` | `rag-chat/src/app/api/chat/route.ts` (create the `api` and `chat` folders) |
| `deploy/nextjs-files/page.tsx` | `rag-chat/src/app/page.tsx` (replace the existing file) |

Create `rag-chat/.env.local` with the same three lines from your `.env`:

```
SUPABASE_URL=...
SUPABASE_SERVICE_ROLE_KEY=...
OPENAI_API_KEY=...
```

### 10. Run it

```powershell
npm run dev
```

Open http://localhost:3000 and ask "How does saliva help with digestion?". You should get an answer with `[1]`-style citations and a Sources list with page numbers and similarity scores.

### 11. Optional: Lovable front end

The prompt is in the workshop's "Next.js + Lovable deployment instructions" file. Before pasting it, remove any keys from it, and let Lovable ask for the OpenAI key itself.

---

## Things in the workshop code to know about

| Where | What | Effect |
| --- | --- | --- |
| `ingest.py` | Prints "10 sentences per chunk" but `SENTS_PER_CHUNK = 20` | Chunks are 20 sentences with 2 overlapping. Change the number to experiment. |
| `ingest.py` | Page numbers are PDF pages (1-based), not book pages | The app's page numbers are about 41 higher than the book's printed numbers. The notebooks use book page numbers. |
| `route.ts` system prompt | "Answer ONLY using the CONTEXT ... say I couldn't find this" | Strict; in the video it refused answerable questions. Loosen the wording if that happens. |
| `route.ts` | Retrieves `match_count: 8` chunks; `test_embeddings.py` uses 3 | Normal: the app passes more context to the model than the test shows. |
| Everywhere | Uses the `service_role` key | Fine for local scripts and the server route. Never put it in browser code or share it. |

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `KeyError: 'SUPABASE_URL'` | `.env` is missing, misnamed (e.g. `.env.txt`) or you ran the script from another folder. |
| `FileNotFoundError: human-nutrition-text.pdf` | Run from the `rag-workshop` folder, and the PDF must be directly in it. |
| Notebook can't import a package | The kernel is not `.venv`; re-select it (step 5). |
| `function match_documents ... does not exist` | `supabase_01_setup.sql` was not run, or was run in a different project than the URL in `.env`. |
| "expected 1536 dimensions" | The embedding model changed; the table is built for `text-embedding-3-small`. |
| App answers "I couldn't find this" for questions that are in the book | Run `supabase_02_index.sql` after ingest; check with `test_embeddings.py`; loosen the prompt in `route.ts`. |
| `npx` or `npm` not found | Install Node.js LTS and reopen VS Code. |
| API route returns 500 | `rag-chat/.env.local` missing; restart `npm run dev` after creating it. |
