# VedaAI Exam Mapping

Teacher-facing **Next.js** app: upload a question paper + handwritten answer sheet, extract questions/answers, map side-by-side, highlight answer regions, and optionally grade with Groq.

**Full pipeline documentation:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — methods, flow, models, and implementation details.

## Stack

| Layer | Choice |
|-------|--------|
| Framework | Next.js 16 (App Router) |
| Extraction (default / deploy) | HF Inference VL — `meta-llama/Llama-4-Scout-17B-16E-Instruct:novita` |
| Chemistry specialist (optional local) | **`AI4Chem/ChemVLM-8B`** via `ml/serve_chem.py` — refines chem diagram/formula blocks; `USE_CHEM_VLM=1` |
| Extraction (legacy offline dev) | **Qwen2.5-VL-3B** via `ml/serve_extract.py` — requires `USE_LEGACY_LOCAL_EXTRACT=1` |
| Matching | Label normalize + cosine (lexical embeddings) |
| Bbox repair | Same HF vision model (when HF mode) |
| Grading | Groq (`openai/gpt-oss-20b`) |
| PDF → images | `pdfjs-dist` (client-side) |

## Setup

```bash
pnpm install   # or npm install
cp .env.example .env.local
# HF_TOKEN: fine-grained token with "Make calls to Inference Providers"
# HF_QWEN_MODEL defaults to meta-llama/Llama-4-Scout-17B-16E-Instruct:novita
# fill GROQ_API_KEY (required for grading)
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000).

### Legacy local extract (optional — offline dev when HF quota exhausted)

```bash
cd ml && pip install -r requirements.txt
npm run extract:local   # from repo root
```

In `.env.local`:

```env
USE_LEGACY_LOCAL_EXTRACT=1
LOCAL_EXTRACT_URL=http://127.0.0.1:8001
```

Do **not** set these on Vercel. Production and normal dev use HF Scout + Groq.

## Pipeline

1. **Upload** — PDF/images rasterized to per-page PNGs  
2. **`POST /api/extract`** — HF Scout (default) or legacy local Qwen when opted in  
3. **`POST /api/validate-bbox`** — validate boxes; optional HF localize  
4. **`POST /api/map-answers`** — exact label match, then lexical similarity  
5. **`POST /api/grade`** — Groq batch score/feedback + summary  
6. **UI** — click a question → highlight answer bbox  

Post-extract: [`lib/enrichAnswers.ts`](lib/enrichAnswers.ts) expands parent labels (`9` → `9(a)`/`9(b)`) and corrects common mislabels.

## Per-stage evaluation

```bash
pnpm recheck   # needs `pnpm dev`; writes .recheck-out/live-report.json + stage-*.json
pnpm score     # stage accuracies + 9 assignment conditions
pnpm eval      # score existing live-report
```

Each stage reports accuracy separately:

| Stage | Metrics |
|-------|---------|
| Extract | question/answer label P/R/F1, bbox coverage |
| Mapping | match P/R/F1, highlight bbox rate, edge cases |
| Grading | row coverage, score bounds, unanswered=0, feedback |

Gold fixtures: [`ml/fixtures/`](ml/fixtures/). Model CER/WER/IoU: `python ml/evaluate.py --demo`.

## Fine-tuning (Colab)

See [`ml/README.md`](ml/README.md). Pipeline: preprocess → load Qwen2.5-VL-3B → LoRA → evaluate → export adapter → local FastAPI.

Scout is used for the **live URL** and default dev. Legacy local Qwen is opt-in for offline training demos when HF credits are exhausted.

## Deploy (Vercel)

1. `pnpm build` locally to verify.  
2. Import the repo in the Vercel dashboard.  
3. Env: `HF_TOKEN`, `GROQ_API_KEY` only (do not set `USE_LEGACY_LOCAL_EXTRACT` or `LOCAL_EXTRACT_URL`).  
4. `vercel.json` sets API `maxDuration: 300`.

## Project layout

- `app/page.tsx` — UI orchestration  
- `app/api/*/route.ts` — pipeline endpoints  
- `lib/extract.ts`, `lib/hf-qwen.ts`, `lib/local-extract.ts`, `lib/groq.ts`, `lib/matching.ts`  
- `lib/eval/*` — per-stage evaluation  
- `ml/` — Colab train + local serve  
- `components/*` — Upload, ProgressStepper, QuestionList, AnswerSheetViewer, GradingSummary  

## Sample Input and Output

### Sample input

Use the bundled fixture PDFs to try the full pipeline locally:

| File | Path | Description |
|------|------|-------------|
| Question paper | [`test/question.pdf`](test/question.pdf) | 15 questions with sub-parts (`1(a)`, `3(b)`, `10(a)`, …) |
| Answer sheet | [`test/answer.pdf`](test/answer.pdf) | Handwritten student responses (11 answered, 4 unanswered) |

**How to run**

1. Start the app (`pnpm dev`) and open [http://localhost:3000](http://localhost:3000).
2. Upload `test/question.pdf` as **Question Paper** and `test/answer.pdf` as **Answer Sheet**.
3. Click **Process** — the app rasterizes each PDF, then runs extract → validate → map → grade.

Accepted formats: `.pdf`, `.png`, `.jpg`, `.jpeg`.

### Sample output (UI)

After processing, the results screen shows:

- **Left** — extracted question list with match status (matched / unanswered).
- **Right** — answer-sheet viewer; clicking a question highlights its bbox on the sheet.
- **Bottom** — grading summary with per-question score and feedback.

<img width="1527" height="905" alt="Mapping screen: question list, answer-sheet highlight, and grading summary" src="https://github.com/user-attachments/assets/df2e13fa-2cce-4368-b105-07038214fdee" />

### Sample output (API)

Abbreviated responses from the fixture run (values vary slightly by model run):

**1. Extract** — `POST /api/extract` (one page, `role: "answer"`)

```json
{
  "via": "hf",
  "blocks": [
    {
      "id": "a-0-1",
      "pageIndex": 0,
      "labelNumber": "1(a)",
      "text": "f(x) = 2x - 5, g(x) = x² + 1, find g(f(3)) …",
      "contentKind": "numerical",
      "bbox": { "x": 0.08, "y": 0.12, "w": 0.84, "h": 0.18 },
      "bboxSource": "qwen"
    }
  ]
}
```

**2. Map** — `POST /api/map-answers`

```json
{
  "pairs": [
    {
      "id": "pair-1a",
      "status": "matched",
      "question": { "labelNumber": "1(a)", "text": "Find g(f(3)) …" },
      "answer": { "labelNumber": "1(a)", "text": "g(f(3)) = 2", "bbox": { "x": 0.08, "y": 0.12, "w": 0.84, "h": 0.18 } },
      "similarity": 1
    },
    {
      "id": "pair-2",
      "status": "unanswered",
      "question": { "labelNumber": "2", "text": "Draw and label a plant cell …" },
      "answer": null
    }
  ]
}
```

Expected gold pairs for this fixture: [`ml/fixtures/expected-pairs.json`](ml/fixtures/expected-pairs.json) (11 matched, 4 unanswered).

**3. Grade** — `POST /api/grade`

```json
{
  "summary": {
    "totalScore": 19,
    "maxScore": 30,
    "answered": 11,
    "unanswered": 4,
    "unmatched": 0,
    "overallFeedback": "Strong work on algebra and GK; review unanswered diagram questions.",
    "grades": [
      { "pairId": "pair-1a", "score": 2, "maxScore": 2, "isCorrect": true, "feedback": "Correct substitution." },
      { "pairId": "pair-2", "score": 0, "maxScore": 2, "isCorrect": false, "feedback": "Unanswered." }
    ]
  }
}
```

### What to expect with the fixture

| Metric | Expected |
|--------|----------|
| Questions extracted | 15 (with sub-parts) |
| Answers extracted | ~11 labeled blocks |
| Matched pairs | 11 (`1(a)`–`10(b)` subset — see fixtures) |
| Unanswered | `2`, `5`, `6(a)`, `6(b)` |
| Highlight on click | Matched answers show a bbox overlay on the answer sheet |

Reproduce metrics locally: `pnpm recheck` → `pnpm score` (requires `pnpm dev` running).

