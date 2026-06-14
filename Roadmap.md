# Project Roadmap — PDF Prompt Injection Detector

> **Legend**
> - 🔴 **BLOCKING** — must be 100% complete before the next phase starts
> - 🟡 **PARALLEL** — both partners work simultaneously on different tasks
> - 🟢 **SHARED** — both partners work on this together

---

## Phase 0 — Kickoff
> **Duration: ~2 days | Both partners together**

### 🟢 Step 0.1 — Sahar Approval Meeting
- Send the email requesting a Zoom meeting.
- During the meeting: walk through the problem, injection types, dataset plan, and app pipeline.
- Get explicit approval to proceed.

> 🔴 **BLOCKING** — do not write a single line of code until this is approved.

### 🟢 Step 0.2 — Finalize Dataset Schema
- Agree on the exact column names and data types for the dataset.
- Agree on injection type labels (exact strings, no variation later).
- Agree on the document types to generate (invoice, contract, report, email, resume, form).
- Write down all agreed values in a shared reference doc or notebook cell — this prevents inconsistency during parallel generation.

> 🔴 **BLOCKING** — generation cannot start until both partners are using the exact same schema.

### 🟢 Step 0.3 — Set Up Infrastructure
- Create a shared HF account or organization.
- Create 3 HF repos:
  - Dataset repo (e.g., `pdf-injection-dataset`)
  - Model repo (for winning embedding model, if needed)
  - Space repo (e.g., `pdf-injection-detector`)
- Set up a shared GitHub or Google Drive folder for notebooks and intermediate files.
- Make sure both partners can access all repos.

---

## Phase 1 — Synthetic Data Generation
> **Duration: ~4 days | Parallel**

Both partners generate different injection types using the same schema and the same base prompt templates. Merge at the end.

### 🟡 Partner 1 — Generate: `clean` + `zero_font` + `white_text`
- **Clean documents (~2,000 rows)**
  - Prompt: *"Generate a realistic [document_type] body of 3–6 sentences."*
  - No injection_payload, is_injected = False
- **Zero-font (~1,600 rows)**
  - Generate visible_text (document body) + injection_payload separately
  - extracted_text = visible_text + " " + injection_payload
  - injection_type = "zero_font"
- **White-text (~1,600 rows)**
  - Same structure as zero_font
  - injection_type = "white_text"
- Save as `part1_P1.parquet`

### 🟡 Partner 2 — Generate: `hidden_layer` + `metadata` + `annotation`
- **Hidden-layer (~1,600 rows)**
  - Generate visible_text + injection_payload
  - injection_type = "hidden_layer"
- **Metadata (~1,600 rows)**
  - metadata_fields column = JSON with injected Title/Author/Subject/Keywords
  - injection_type = "metadata"
- **Annotation (~1,600 rows)**
  - injection_type = "annotation"
- Save as `part1_P2.parquet`

### 🟢 Step 1.3 — Merge & Validate Dataset
- Concatenate both parquets → `dataset_raw.parquet`
- Validate:
  - Row count ≥ 10,000
  - No null doc_id values, no duplicate doc_ids
  - injection_payload is null for all `clean` rows only
  - is_injected matches injection_type (clean → False, all others → True)
  - All columns match the agreed schema exactly
- Fix any issues found.
- Save final `dataset.parquet`

> 🔴 **BLOCKING** — EDA cannot start until the merged dataset is validated.

---

## Phase 2 — EDA
> **Duration: ~3 days | Parallel**

Both partners work on different EDA aspects in separate notebook sections, then combine into one final EDA notebook.

### 🟡 Partner 1 — Statistical Analysis
- Class distribution bar chart (injection_type counts)
- is_injected ratio (pie chart)
- Document type breakdown per injection_type (stacked bar)
- Target action distribution
- Text length analysis:
  - visible_text length vs extracted_text length
  - Length delta histogram (how much hidden text each injection type adds)
- Word clouds per injection_type (on injection_payload column)
- Flag hallucinations: rows where injection_type = "metadata" but metadata_fields is empty, etc.

### 🟡 Partner 2 — Embedding Visualization
- Embed a sample of 1,000 rows using `sentence-transformers/all-MiniLM-L6-v2` (quick, just for visualization)
- Reduce to 2D with UMAP (preferred) or t-SNE
- Plot colored by injection_type — clusters should be visible
- Plot colored by document_type
- Plot colored by target_action
- Discuss what the clusters reveal about the data

### 🟢 Step 2.3 — Combine & Upload to HF
- Merge both partners' notebook sections into one clean EDA notebook.
- Write a README for the HF Dataset repo summarizing:
  - What the dataset is
  - Schema description
  - Key EDA findings (2–3 bullet points with the most interesting insights)
- Upload to HF Dataset repo:
  - `dataset.parquet`
  - `generation_notebook.ipynb`
  - `eda_notebook.ipynb`
  - `README.md`

> 🔴 **BLOCKING** — embedding comparison cannot start until the dataset is uploaded to HF (the app must read from HF, so the repo must exist and be public).

---

## Phase 3 — Embedding Model Comparison
> **Duration: ~3 days | Parallel**

Each partner tests different HF embedding models on the same held-out test split. Goal: pick the one with the best precision@3 for injection_type retrieval.

### Setup (shared, 30 minutes)
- Create a test split: randomly sample 300 rows (50 per injection_type).
- Save as `test_split.parquet` — both partners use the exact same file.
- Define the evaluation function:
  ```python
  # For each test row, embed it, find top-3 nearest neighbors from the training set,
  # check if their injection_type matches the query's injection_type.
  # precision@3 = (matching neighbors) / 3
  ```

### 🟡 Partner 1 — Test Model 1 & Model 2
- `sentence-transformers/all-MiniLM-L6-v2`
- `BAAI/bge-small-en-v1.5`
- For each model: embed full dataset, embed test split, run evaluation, record precision@3 and embedding time.

### 🟡 Partner 2 — Test Model 3 & Write Comparison Table
- `sentence-transformers/all-mpnet-base-v2`
- Same evaluation process.
- Combine all 3 results into a comparison table:

  | Model | Precision@3 | Embed Time (10k rows) | Size |
  |---|---|---|---|
  | MiniLM-L6-v2 | ? | ? | 80MB |
  | BGE-small-en-v1.5 | ? | ? | 130MB |
  | MPNet-base-v2 | ? | ? | 420MB |

- Recommend the winner based on precision@3 / speed tradeoff.

### 🟢 Step 3.3 — Save Winning Embeddings
- Embed the full dataset with the winning model.
- Save as `embeddings.parquet` (columns: doc_id, embedding vector).
- Upload `embeddings.parquet` to the HF Space repo files.

> 🔴 **BLOCKING** — the app backend cannot be built until the winning model is chosen and embeddings are saved.

---

## Phase 4 — Generation Component
> **Duration: ~4 days | Partner 2, runs in parallel with Phase 3**

Partner 2 can start this during Phase 3 (after the dataset is uploaded), because it does not depend on the embedding model choice.

### 🟡 Partner 2 — HF LLM Pipeline
- Test candidate HF models for the explanation generation task:
  - `google/flan-t5-large` (fast, small)
  - `HuggingFaceH4/zephyr-7b-beta` (better quality)
  - `microsoft/phi-2` (efficient)
- For each model, run the prompt template on 10 sample injections and evaluate output quality manually.
- Choose the best model.
- Wrap in a clean function:
  ```python
  def generate_explanation(injection_type, injection_payload, target_action) -> str:
      # returns a 2-3 sentence analyst explanation
  ```
- If no HF model is adequate, document why and fall back to MaaS API.

> This phase runs **in parallel with Phase 3**. It must be done before app development starts.

---

## Phase 5 — App Development
> **Duration: ~4 days | Parallel**

> 🔴 Phases 3 and 4 must both be complete before this phase starts.

### 🟡 Partner 1 — PDF Parsing Backend
Build the function that takes a raw PDF file and returns all extractable content:
```
extract_pdf(file) → {
    visible_text: str,
    hidden_text: list[{text, reason}],   # zero-font, white-text findings
    metadata: dict,                       # title, author, subject, keywords
    annotations: list[str]
}
```
- Use `pdfplumber` for text + font-size/color inspection
- Use `pymupdf` (fitz) for metadata and annotations
- Detection rules:
  - font size == 0 → zero_font finding
  - text color == white (RGB 1,1,1 or similar) → white_text finding
  - metadata fields contain imperative language → metadata finding
  - annotations exist → annotation finding
- Build the FAISS index from `embeddings.parquet`
- Build the retrieval function: embed query → cosine search → return top-3 rows with metadata

### 🟡 Partner 2 — Gradio UI
- Build the Gradio interface layout:
  - File upload component (PDF only)
  - Result panel: verdict (Injected / Clean), injection type, target action
  - Top-3 similar injections panel (show doc_type, injection_type, similarity score, snippet)
  - Generated explanation text box
  - Severity badge (High / Medium / Low based on target_action)
- Build the 3 Quick Starters:
  - Generate 3 example PDFs programmatically using `reportlab`:
    1. Clean invoice
    2. Contract with zero-font injection
    3. Report with metadata injection
  - Add one-click buttons that load each example

### 🟢 Step 5.3 — Integration
- Wire Partner 1's backend functions into Partner 2's Gradio app.
- Full end-to-end test:
  - Upload each of the 3 quick starter PDFs manually
  - Verify correct injection type is detected
  - Verify top-3 results make sense
  - Verify generated explanation is coherent
- Fix bugs.

> 🔴 **BLOCKING** — do not deploy until the end-to-end test passes for all 3 quick starters.

---

## Phase 6 — HF Space Deployment
> **Duration: ~2 days | Both partners**

- Create `app.py` (main Gradio app file)
- Create `requirements.txt`:
  ```
  gradio
  pdfplumber
  pymupdf
  sentence-transformers
  faiss-cpu
  transformers
  pandas
  pyarrow
  datasets
  ```
- Upload to HF Space:
  - `app.py`
  - `requirements.txt`
  - `embeddings.parquet`
  - Quick starter example PDFs
- Verify the Space builds successfully (check build logs).
- Test the live Space URL — run all 3 quick starters on the deployed version.
- Share the Space URL publicly.

---

## Phase 7 — Presentation Prep
> **Duration: ~3 days | Both partners**

### Script the Walkthrough (~10–12 min)
Structure:
1. Problem statement (1 min) — why PDF injections are dangerous
2. Dataset overview (2 min) — how it was generated, EDA highlights
3. Embedding comparison (2 min) — which model won and why
4. Live demo (4 min) — run all 3 quick starters in the HF Space
5. Generation output (1 min) — show the LLM explanation
6. Architecture summary (1 min) — diagram of the full pipeline

### Prepare for Q&A (~8–10 min)
Expect questions on:
- Why these specific embedding models?
- How does precision@3 work?
- What is FAISS and why is it used?
- How does pdfplumber detect zero-font vs white-text?
- Why generate text instead of actual PDFs?
- What are the limitations of this approach?
- How would a real attacker evade this system?

---

## Dependency Summary

```
Phase 0 (Approval + Schema)
    └── Phase 1 (Data Generation) — PARALLEL
            └── Merge & Validate
                    └── Phase 2 (EDA) — PARALLEL
                            └── Upload to HF
                                    ├── Phase 3 (Embeddings) — PARALLEL
                                    │       └── Save winning embeddings
                                    │               └── Phase 5 (App Dev) — PARALLEL
                                    │                       └── Integration
                                    │                               └── Phase 6 (Deploy)
                                    │                                       └── Phase 7 (Presentation)
                                    └── Phase 4 (Generation) — PARALLEL with Phase 3
                                            └── feeds into Phase 5
```

---

## Gantt Chart

```mermaid
gantt
    title PDF Injection Detector — Work Split
    dateFormat  YYYY-MM-DD
    axisFormat  %b %d

    section Shared (Blocking)
    Approval Meeting & Schema Finalization  :crit, 2026-06-16, 2d
    Merge & Validate Dataset                :crit, 2026-06-22, 1d
    Upload Dataset + Notebooks to HF        :crit, 2026-06-26, 1d
    Choose Best Embedding Model             :crit, 2026-06-30, 1d
    App Integration & End-to-End Testing    :crit, 2026-07-05, 2d
    HF Space Deployment                     :crit, 2026-07-07, 2d
    Presentation Prep & Rehearsal           :crit, 2026-07-09, 3d

    section Partner 1
    Generate clean + zero_font + white_text     :2026-06-18, 4d
    EDA – Statistical Analysis & Word Clouds    :2026-06-23, 3d
    Embed Model 1 (MiniLM) & Model 2 (BGE)     :2026-06-27, 3d
    PDF Parsing Backend (pdfplumber + pymupdf)  :2026-07-01, 4d

    section Partner 2
    Generate hidden_layer + metadata + annotation   :2026-06-18, 4d
    EDA – UMAP / t-SNE Visualization                :2026-06-23, 3d
    Embed Model 3 (MPNet) + Evaluation Table        :2026-06-27, 3d
    Generation Component (HF LLM pipeline)          :2026-06-27, 4d
    Gradio UI + Quick Starters                      :2026-07-01, 4d
```

---

> **Total estimated duration: ~26 days (June 16 – July 11)**
> Adjust dates based on your actual deadline and Sahar's approval date.
