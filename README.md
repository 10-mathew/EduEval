# EduEval — Automated Exam Marking System

AI-powered exam marking: upload question papers + handwritten answer sheets → get marks and feedback automatically.

---

## Features

- **Teacher auth** — register / login / session management
- **Exam management** — create exams, upload question papers
- **Question parsing** — manual entry or auto-parse from pasted text
- **Gemini model answers** — AI-generated marking guides per question
- **Google Vision OCR** — handwriting recognition with confidence scoring
- **Gemini evaluation** — per-question marks + reasons vs model answer
- **Results dashboard** — totals, pass/fail, per-question breakdown, AI feedback
- **SQLite persistence** — full database of exams, questions, students, evaluations

---

## Quick Start

### 1. Install dependencies

```bash
cd edueval
pip install -r requirements.txt
```

### 2. Set environment variables

```bash
# Required for Gemini AI
export GEMINI_API_KEY="your-gemini-api-key"

# Required for Google Cloud Vision OCR
# Download your service account JSON from GCP Console
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"

# Optional — change for production
export SECRET_KEY="your-flask-secret-key"
```

**Getting API keys:**
- **Gemini**: https://aistudio.google.com/app/apikey
- **Google Cloud Vision**: Enable the Vision API in GCP Console, create a service account, download the JSON key.

> **No API keys?** The app runs in **demo/mock mode** — OCR returns sample text and Gemini returns canned responses, so you can explore the full flow without any credentials.

### 3. Run

```bash
python app.py
```

Open http://localhost:5000/login in your browser.

---

## Deploy (Render-only)

### Render

- This repo includes a Render Blueprint at `render.yaml` (Python + Gunicorn).
- In Render: **New → Blueprint**, connect the GitHub repo, then set required secrets:
  - `GEMINI_API_KEY`
  - **Either** `GOOGLE_APPLICATION_CREDENTIALS` (file path) **or** `GOOGLE_CREDENTIALS_JSON` (JSON string payload)
- Note: SQLite + `uploads/` are stored on the service filesystem; for long-term persistence you’ll need a paid Persistent Disk or move to a managed database.

### Optional split (Vercel + Render)

- If you still want a separate static frontend, deploy `frontend/` to Vercel and point it at your Render URL.

---

## User Flow

```
1. Login
2. Create Exam  →  upload question paper
3. Configure Questions  →  parse or enter manually with max marks
4. Generate Model Answers  →  Gemini creates marking guides
5. Add Students  →  upload scanned answer sheets per student
6. Run OCR  →  Google Cloud Vision extracts handwritten text
7. Evaluate  →  Gemini compares student answers vs model answers
8. View Results  →  marks, reasons, feedback, pass/fail
```

---

## Project Structure

```
EduEval/
├── app.py                    # Flask application
├── requirements.txt          # App dependencies
├── requirements-eval.txt     # Optional evaluation dependencies
├── datasets/                 # Ground truth + student answer CSVs
├── ocr/                      # Standalone OCR runner (Vision)
├── models/                   # LLM evaluators (Gemini + optional FLAN-T5)
├── evaluation/               # Metric scripts + evaluation runner
├── instance/
│   └── edueval.db            # SQLite database (auto-created)
├── uploads/
│   ├── questions/            # Uploaded question papers
│   └── answers/              # Uploaded answer sheets
└── templates/
    ├── login.html
    └── index.html
```

---

## Offline Evaluation (Metrics + Model Comparison)

This repo includes a standalone evaluation pipeline that:

- exports **ground truth** (reference answers) + **student answers** from the SQLite DB
- computes metrics (Accuracy, Precision, BLEU, ROUGE-L, WUPS, etc.)
- compares Gemini against an optional second model

### 1) Export datasets from SQLite

```bash
python3 datasets/export_from_db.py --db instance/edueval.db --out datasets
```

### 2) Run metrics and generate tables

Text-only metrics:

```bash
python3 evaluation/run_eval.py --no-llm
```

Gemini scoring + metrics:

```bash
export GEMINI_API_KEY="..."
python3 evaluation/run_eval.py --llm gemini
```

Compare Gemini vs FLAN‑T5 (optional):

```bash
pip install -r requirements-eval.txt
pip install transformers torch
python3 evaluation/run_eval.py --llm gemini --llm flan-t5
```

Outputs are written to `outputs/`:

- `outputs/evaluation_table_<model>.csv`
- `outputs/summary_<model>.csv`
- `outputs/model_comparison.csv` (when comparing multiple models)


## Database Schema

| Table           | Purpose                                         |
|-----------------|-------------------------------------------------|
| `teachers`      | Auth: username + hashed password                |
| `exams`         | Exam metadata + optional QP file path/text      |
| `questions`     | Questions per exam: number, text, max marks, model answer |
| `students`      | Student name linked to exam                     |
| `answer_sheets` | Uploaded files + OCR text + confidence          |
| `evaluations`   | Per-question marks + reason per student         |
| `feedback_logs` | Overall feedback + total marks per student      |

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/login` | Login page |
| GET | `/app` | Main app page (auth required) |
| POST | `/api/login` | Authenticate |
| POST | `/api/logout` | End session |
| GET | `/api/me` | Session status |
| POST | `/api/register` | Create teacher account |
| GET | `/api/exams` | List all exams |
| POST | `/api/exams` | Create exam (multipart: title + question_paper) |
| GET | `/api/exams/:id` | Exam details with questions + students |
| POST | `/api/exams/:id/questions` | Save questions list |
| POST | `/api/exams/:id/generate-model-answers` | Gemini model answers |
| POST | `/api/exams/:id/students` | Add student + upload sheets |
| POST | `/api/students/:id/ocr` | Run Vision OCR on student sheets |
| POST | `/api/exams/:id/evaluate` | Evaluate all students with Gemini |
| GET | `/api/exams/:id/results` | Fetch stored results |

---

## Deploy Backend (Render + GitHub Pages Frontend)

`docs/index.html` is configured for GitHub Pages and can call a deployed backend.

### Backend on Render

This repo includes `render.yaml` and `Procfile`.

1. Create a new Render Web Service from this repo.
2. Render should detect:
   - Build command: `pip install -r requirements.txt`
   - Start command: `gunicorn app:app`
3. Set required environment variables in Render:
   - `GEMINI_API_KEY`
   - One of:
     - `GOOGLE_APPLICATION_CREDENTIALS` (file path in runtime), or
     - `GOOGLE_CREDENTIALS_JSON` (raw service account JSON string)
4. Confirm/adjust:
   - `FRONTEND_ORIGINS=https://10-mathew.github.io`
   - `SESSION_COOKIE_SECURE=true`
   - `SESSION_COOKIE_SAMESITE=None`
   - `SECRET_KEY` (strong random value)
5. Deploy and verify health endpoint:
   - `GET /health` returns `{ "status": "ok" }`

### GitHub Pages frontend

1. In GitHub Pages settings, use:
   - Branch: `main`
   - Folder: `/docs`
2. Open the Pages site and set backend URL in the top bar:
   - e.g. `https://your-render-service.onrender.com`
3. Login from the in-page Connect form and run the workflow.

---

## Production Checklist

- [ ] Set `SECRET_KEY` to a long random string
- [ ] Use PostgreSQL/MySQL instead of SQLite for concurrent access
- [ ] Store uploads on S3/GCS instead of local filesystem
- [ ] Add HTTPS (nginx + certbot)
- [ ] Rate-limit API endpoints
- [ ] Add password reset / email verification
- [ ] Add manual review UI for low-confidence OCR regions
