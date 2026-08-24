# Meeting Summarizer

Upload a meeting recording and get back a concise summary, the key decisions,
and action items with owners and deadlines — plus the full transcript.

The pipeline is deliberately honest: the model is instructed **never to invent
information**. Any field the meeting didn't state (an owner, a deadline, even
the summary of an empty recording) comes back as the literal string
`"Not specified"`.

```
Audio file  →  Whisper (speech-to-text)  →  Structured LLM prompt  →  Notes
```

---

## Tech stack

| Layer     | Choice                          | Why                                   |
| --------- | ------------------------------- | ------------------------------------- |
| Frontend  | React + Vite                    | Fast dev server, tiny footprint       |
| Backend   | FastAPI + Uvicorn               | Minimal, typed, async file uploads    |
| Speech    | OpenAI Whisper (`whisper-1`)    | Reliable transcription                |
| Summary   | OpenAI Chat (`gpt-4o-mini`)     | Cheap, strong JSON-mode output        |

No database, no auth, no build tooling beyond Vite — just the pipeline.

---

## Project layout

```
meeting-summarizer/
├── backend/
│   ├── main.py            # FastAPI app: /api/health, /api/summarize
│   ├── requirements.txt
│   └── .env.example       # copy to .env and add your key
├── frontend/
│   ├── src/
│   │   ├── App.jsx        # upload → loading → results flow
│   │   ├── components/    # Uploader, Results, WaveLoader
│   │   └── icons.jsx
│   ├── package.json
│   └── vite.config.js     # proxies /api → localhost:8000
└── README.md
```

---

## Use it (no key, nothing to set up)

Open **`easy2meet.html`** — double-click it, or host it anywhere (drag it onto
[app.netlify.com/drop](https://app.netlify.com/drop) for an instant public URL,
or use GitHub Pages). It runs **entirely in your browser**: no account, no API
key, no server — so there's nothing that can fail.

- **Paste a transcript** → it generates the summary, decisions, and action items
  from your own text.
- **Upload audio** → it shows a sample result (real speech-to-text needs a
  server; see below).

It's a full multi-page app — landing → workspace → History → Settings — and your
summaries are saved on your device.

### Optional: real Whisper transcription
If you want real speech-to-text from audio, the FastAPI backend is included. It
runs the OpenAI calls server-side using `OPENAI_API_KEY` (never in the browser),
and can be deployed with the bundled `Dockerfile` / `render.yaml`. See
`backend/` and the deploy notes further down.

---

## Getting started

You'll need **Python 3.10+**, **Node 18+**, and an
[OpenAI API key](https://platform.openai.com/api-keys).

### 1. Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env               # then edit .env and paste your key
uvicorn main:app --reload --port 8000
```

The API is now at `http://localhost:8000`. Check `http://localhost:8000/api/health`.

### 2. Frontend

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL Vite prints (default `http://localhost:5173`). The dev server
proxies `/api/*` to the backend, so no CORS setup is needed while developing.

---

## Environment variables

Configured in `backend/.env` (see `backend/.env.example`):

| Variable           | Required | Default                                       |
| ------------------ | -------- | --------------------------------------------- |
| `OPENAI_API_KEY`   | yes      | —                                             |
| `TRANSCRIBE_MODEL` | no       | `whisper-1`                                    |
| `SUMMARY_MODEL`    | no       | `gpt-4o-mini`                                  |
| `ALLOWED_ORIGINS`  | no       | `http://localhost:5173,http://127.0.0.1:5173` |

`.env` is gitignored and must never be committed.

---

## API

### `GET /api/health`

```json
{ "status": "ok", "configured": true }
```

### `POST /api/summarize`

Multipart form with a single `file` field (audio, ≤ 25 MB).

**Response**

```json
{
  "summary": "The team reviewed Q3 launch readiness and agreed to ship on the 14th.",
  "key_decisions": [
    "Launch date set to October 14.",
    "Marketing to lead the announcement."
  ],
  "action_items": [
    { "task": "Finalize the release notes", "owner": "Priya", "deadline": "Oct 10" },
    { "task": "Set up the status page", "owner": "Not specified", "deadline": "Not specified" }
  ],
  "transcript": "Full verbatim transcript…"
}
```

Supported audio: `mp3, mp4, mpeg, mpga, m4a, wav, webm, ogg, flac`.

---

## How honesty is enforced

- The system prompt forbids inventing owners, dates, decisions, or outcomes.
- Missing fields are returned as `"Not specified"`, and the UI renders that in a
  muted, italic style so it's obvious at a glance what the meeting didn't cover.
- The model runs at `temperature=0` in JSON mode, and the backend re-validates
  and normalizes the shape before returning it.

---

## Notes & limits

- Whisper's hard limit is 25 MB per file; longer meetings should be compressed
  or split first.
- Uploaded audio is written to a temp file, sent to the API, and deleted
  immediately after processing — nothing is persisted.
- This is a single-purpose tool by design: no auth, payments, storage, or
  dashboards.

## License

MIT — do what you like.
