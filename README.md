# Synthesis Wall

**Workshop sticky notes → segments, use case links, themes and stats.**

A Padlet-style board where participants post notes from their phones. An engine then:

- **allocates every note to your segments**, with a confidence score and a reason;
- **links notes to your use case directory**, flagging possible duplicates, ideas that were tried and failed (with the original failure reason) and new ideas;
- **pulls supporting passages** from reference documents you upload;
- **groups notes into themes** and produces **stats and a summary**;
- **learns from your corrections**: drag a note to another segment and similar notes go there next time.

![Board arranged by segment](docs/board.png)

| Analysis | Note details | Participant (phone) |
|---|---|---|
| ![Analysis](docs/analysis.png) | ![Note panel](docs/note-panel.png) | ![Participant view](docs/participant.png) |

Everything runs in the browser. AI writing (Claude, OpenAI, any OpenAI-compatible API, or a free local model via Ollama) is **optional and can be switched on or off** at any time.

---

## Quick start (2 minutes, no accounts)

```bash
git clone <your-repo-url> synthesis-wall
cd synthesis-wall
python3 -m http.server 8000      # or: npx serve .
```

Open <http://localhost:8000>, then **Open facilitator dashboard → Load demo workshop → Analyse → Run analysis**.

> The app uses ES modules, so it must be served over http(s). Double-clicking `index.html` will not work.

To try the participant view, click **Invite participants → Open participant view**. In local mode it works between tabs on the same computer, and notes appear on the board live.

## Live participants on their own phones (Supabase, free tier)

1. Create a project at [supabase.com](https://supabase.com).
2. Open **SQL Editor → New query**, paste [`supabase/schema.sql`](supabase/schema.sql) and press **Run**.
3. Copy **Project URL** and the **anon public key** from *Project Settings → API*.
4. Either:
   - put them in [`config.js`](config.js) and redeploy, so **everyone** using your deployment shares the database (recommended); or
   - paste them in **Settings → Data** in the facilitator dashboard. The join link and QR code then carry the connection details for participants.

The top bar shows **Live · Supabase** when connected. If Realtime is unavailable, the board falls back to refreshing every 4 seconds.

> The schema ships with open **demo** policies (anyone with the anon key can read and write). That is fine for a demo or an internal tool. For production, add Supabase Auth and restrict facilitator tables. See the comments at the bottom of `schema.sql`.

## Deploy to GitHub Pages

There is no build step.

1. Push this folder to a GitHub repository.
2. Go to **Settings → Pages → Build and deployment**, choose **Deploy from a branch**, branch `main`, folder `/ (root)`.
3. Your demo is live at `https://<user>.github.io/<repo>/` within a minute.

Netlify, Vercel or any static host works the same way.

---

## How it works

```
 Participants (phones)           Facilitator (browser)
 join.html ──notes──▶ Supabase / local ◀── app.html: Setup · Board · Analysis · Use cases · Settings
                                              │
                         ┌────────────────────┴──────────────────────┐
                         │ 1. Engine (free, in the browser)          │
                         │    embeddings → segments, use cases, docs │
                         │    clustering → themes · stats            │
                         │ 2. AI layer (optional, switchable)        │
                         │    hard notes only, batched               │
                         │    theme names · executive summary        │
                         └───────────────────────────────────────────┘
                                              │
                                Excel / JSON / PDF export
```

### 1. The engine (no API key needed)

| Step | What happens |
|---|---|
| Embeddings | **Smart** (default): a small sentence-transformer (`all-MiniLM-L6-v2`, or `multilingual-e5-small` for Arabic) runs in the browser via transformers.js. It downloads about 25 MB once, then the browser caches it. **Fast**: TF-IDF over words, phrases and character n-grams, fully offline. If Smart can't load, Fast is used automatically. |
| Segments | Each note is compared with each segment's **description**, with the **use cases tagged with that segment**, and with **notes you confirmed before**. The top match wins and gets a confidence score. A close second is shown as "also fits". Notes below the threshold stay **Unassigned**. |
| Use cases | The top 3 most similar use cases are kept. Each is labelled **possible duplicate**, **similar to a failed attempt** (with *why it failed*) or **related**. A note with no close match is flagged as a **new idea**. |
| Documents | Uploaded PDF/DOCX/TXT/MD/CSV/XLSX files are split into passages. The best passages are attached to each note, with page numbers. |
| Themes | Average-linkage clustering groups notes. Problems and ideas on the same topic end up together. Answers to *standard* questions (polls, one-word answers) are clustered separately. Themes are named from their most distinctive words. |
| Stats | Notes per segment, confidence, tone (pain point / idea / positive / neutral), use case coverage, never-mentioned use cases, past-failure echoes, and an automatic summary. |

### 2. The AI layer (optional)

Turn it on with the **AI** switch in the top bar. Configure the provider in **Settings → AI writing**. It adds judgment where similarity alone struggles:

- negation ("we *don't* need more parking");
- two ideas in one note (suggests a split);
- same idea in different words, or same words with a different meaning;
- whether a note **repeats** a past failure or **fixes** the reason it failed;
- human-readable theme names, suggested segments, and an executive summary with next steps.

**Built for limited API keys.** By default only the *hard* notes are sent: low confidence, two possible segments, or a past-failure match. They go in batches of 25, plus one call for theme names and one for the summary. A 300-note workshop typically uses **around 10 calls**. The *Summaries only* setting uses exactly **2 calls** per run.

| Provider | Notes |
|---|---|
| Claude (Anthropic) | Default model `claude-haiku-5-5`. Called directly from the browser. |
| OpenAI | Default `gpt-4o-mini`. You can change the model name. |
| OpenAI-compatible | OpenRouter, Groq, Together, LM Studio… Set the base URL. |
| Ollama (local, free) | `ollama pull qwen2.5:7b`, then start Ollama with `OLLAMA_ORIGINS=*`. Nothing leaves your laptop. |

API keys are stored only in the facilitator's browser (localStorage). They are never written to Supabase or to the repository.

### 3. Learning

Every correction (moving a note on the board in *By segment* view, changing its segment in the analysis, or confirming it in the note panel) is saved to **learning memory**. Future runs blend those examples into each segment's profile, so allocation improves across workshops. Memory can be exported and imported as JSON (**Settings → Learning memory**).

## Segments don't have to be fixed

Each workshop chooses one of three modes in **Setup**:

- **Predefined**: you list the segments, e.g. Wellness, Building Materials, Retail.
- **Hybrid**: your segments, plus *emerging themes* for notes that fit none. Promote an emerging theme to a segment with one click.
- **Discovered**: no segments up front. Notes are grouped and segments are suggested. Accept them and re-run.

You can copy segments from a previous workshop.

## Your use case directory

Go to **Use case directory → Import Excel / CSV**. Columns are detected automatically:

| Field | Recognised column names (examples) |
|---|---|
| Code | Code, ID, Ref |
| Title | Title, Name, Use case |
| Description | Description, Summary, Details |
| Segment | Segment, Unit, Category, Area |
| Type | Type, Capability |
| Status | Status, Stage, Outcome (e.g. Idea, Pilot, Live, **Failed**) |
| What was tried | What was tried, Approach, Attempt |
| Why it failed | Why it failed, Failure reason, Lessons |
| Tags | Tags, Keywords |

See [`samples/sample-use-cases.xlsx`](samples/sample-use-cases.xlsx), or use **Download template**. Ideas from a workshop can be added back to the directory from the note panel.

## Exports

**Analysis → Download Excel** gives a workbook with these sheets:

- **Notes**: segment, confidence, reason, theme, tone, linked use cases, document reference, and who checked it (Engine / AI / Facilitator)
- **Segments**
- **Themes**
- **Use case links**, including never-mentioned use cases
- **Stats**
- **Summary**

JSON export and print-to-PDF are also available.

Notes typed up after an in-room session can be imported on the **Board** (paste one per line, or upload Excel/CSV). See [`samples/sample-notes.xlsx`](samples/sample-notes.xlsx).

## Project structure

```
index.html            landing page
app.html              facilitator dashboard
join.html             participant view (mobile friendly)
config.js             optional Supabase URL + anon key for the deployment
css/style.css
js/engine.js          embeddings, segment allocation, use case matching, clustering, stats
js/llm.js             AI providers (Claude, OpenAI, compatible, Ollama) + prompts
js/synth.js           orchestrates a run: engine → optional AI → summary
js/store.js           data layer: LocalStore (browser) and SupabaseStore (live)
js/files.js           Excel/CSV import & export, PDF/DOCX text extraction
js/app.js             facilitator UI
js/join.js            participant UI
js/demo-data.js       demo workshop
lib/                  vendored libraries (SheetJS, supabase-js, pdf.js, mammoth, qrcode)
supabase/schema.sql   database tables, voting function, realtime, demo policies
samples/              sample use cases, notes and a reference document
```

## Roadmap ideas

- Supabase Auth for facilitators, with stricter row-level security
- Google Sheets sync for the use case directory (service account)
- Cross-workshop trends dashboard
- Presenter mode with timer and anonymous reveal
