# PDF Prompt Injection Detector

## Problem Statement
LLMs are increasingly used to process PDFs (summarize contracts, answer questions about documents, extract data).
Attackers hide malicious instructions inside PDFs using techniques invisible to human readers but visible to LLM parsers.
This app detects those injections, classifies them, and explains what they are trying to do.

---

## Injection Types (Dataset Classes)
| Type | Technique | How it hides |
|---|---|---|
| `zero_font` | Text with font-size 0pt | Invisible to eye, extracted by PDF parsers |
| `white_text` | White-colored text on white background | Invisible to eye, extracted by PDF parsers |
| `hidden_layer` | Text placed in a PDF layer with visibility off | Hidden in layer stack, extracted by parsers |
| `metadata` | Instructions in PDF document metadata fields (Title, Author, Subject, Keywords) | Never shown in document view |
| `annotation` | Instructions hidden in PDF comments/annotations | Hidden by default in most viewers |
| `clean` | No injection | Baseline negative class |

---

## Dataset Schema (Text Modality → 10,000+ rows)
Each row represents one PDF document (real or synthetic):

| Column | Description |
|---|---|
| `doc_id` | Unique ID |
| `document_type` | invoice, contract, report, email, resume, form |
| `visible_text` | Text a human reader would see |
| `extracted_text` | Full text a PDF parser extracts (includes hidden text) |
| `metadata_fields` | JSON of PDF metadata (title, author, subject, keywords) |
| `injection_type` | zero_font / white_text / hidden_layer / metadata / annotation / clean |
| `injection_payload` | The actual injected instruction string (null if clean) |
| `target_action` | What the injection tries to do: ignore_prior_instructions / exfiltrate_data / change_persona / bypass_filter / other |
| `is_injected` | Boolean |

---

## Part 1: Synthetic Data Generation

### Strategy
- Do NOT generate actual PDF files (unnecessary complexity, no grading value).
- Generate the **text content** as it would appear to a PDF parser.
- Use a HF model (e.g., `mistralai/Mistral-7B-Instruct` via inference API, or `google/flan-t5-large`) to generate:
  - Realistic document bodies (invoices, contracts, reports) as `visible_text`
  - Diverse injection payloads per type as `injection_payload`
  - Combine them into `extracted_text` (visible_text + hidden injection string)

### Volume
- ~2,000 clean documents
- ~1,600 per injection type × 5 types = 8,000 injected
- **Total: 10,000 rows**

### Prompting Approach
Batch-generate with structured prompts:
- "Generate a realistic invoice document body (3-5 sentences)."
- "Generate a prompt injection instruction that attempts to make an LLM ignore its previous instructions. Be creative and varied."
- Combine outputs into dataset rows programmatically.

### Tools
- HF Inference API or `transformers` pipeline
- `pandas` for assembly
- `uuid` for doc_id

---

## Part 2: EDA

- Class distribution (injection_type counts, is_injected ratio)
- Text length distribution (visible_text vs extracted_text length delta — the delta reveals how much is hidden)
- Document type breakdown per injection type
- Target action distribution
- Word clouds per injection type
- Embedding visualization (UMAP or t-SNE) to show clusters by injection type
- Flag model hallucinations / inconsistent outputs (e.g., injection_payload that doesn't match target_action)

**Upload to HF Dataset repo** with EDA notebook + generation notebook.

---

## Part 3: Recommendation with Embeddings

### Goal
Given extracted text from a user-uploaded PDF, find the **top-3 most similar known injection patterns** in the dataset.

### 3 HF Embedding Models to Compare
1. `sentence-transformers/all-MiniLM-L6-v2` — fast, lightweight
2. `BAAI/bge-small-en-v1.5` — strong retrieval performance
3. `sentence-transformers/all-mpnet-base-v2` — higher quality, slower

### Evaluation
- Embed a held-out test set
- Use cosine similarity to retrieve top-k matches
- Evaluate: precision@3 (do returned matches share the same injection_type as the query?)
- Pick the model with the best precision@3 / speed tradeoff

### Storage
- Save chosen model's embeddings as `.parquet` (upload to HF Space repo)
- Use `FAISS` for fast nearest-neighbor lookup at inference time

---

## Part 4: Generation

Given a detected injection, use a small HF language model to generate a natural-language explanation:

> *"This PDF contains a zero-font injection. The hidden text instructs the AI to ignore its prior instructions and reveal confidential system prompts. Severity: High."*

### Model Candidates (HF)
- `google/flan-t5-large` (small, fast, instruction-following)
- `HuggingFaceH4/zephyr-7b-beta` (if compute allows)
- `microsoft/phi-2` (efficient)

If no HF model produces adequate quality → fall back to MaaS (e.g., Claude/OpenAI API) with documented justification.

### Prompt Template
```
You are a security analyst. The following text was extracted from a PDF and contains a hidden prompt injection.

Injection type: {injection_type}
Hidden payload: {injection_payload}
Target action: {target_action}

Explain in 2-3 sentences what this injection is attempting to do and why it is dangerous.
```

---

## Part 5: Gradio Application (HF Space)

### User Flow
1. User uploads a PDF file
2. Backend extracts text using `pdfplumber` or `pymupdf`:
   - Visible text
   - Hidden text (zero-font detected via font-size attribute, white text via color attribute)
   - Metadata fields (Title, Author, Subject, Keywords)
   - Annotations
3. Concatenate all extracted content → embed with winning model
4. FAISS lookup → top-3 most similar injection patterns from dataset
5. HF LLM generates explanation
6. Display:
   - Is this PDF injected? (Yes/No + confidence)
   - If yes: injection type + target action
   - Top-3 similar known attacks (with similarity scores)
   - Generated explanation of what the injection is doing

### 3 Quick Starters (required by assignment)
Pre-loaded example PDFs:
1. Clean invoice — no injection found
2. Contract with zero-font injection — role override detected
3. Report with metadata injection — data exfiltration attempt

### Constraints
- Read dataset from HF Dataset repo
- Read embedding model from HF Model repo
- Embeddings parquet loaded from HF Space repo files

---

## Tech Stack
| Task | Library |
|---|---|
| PDF parsing | `pdfplumber`, `pymupdf` (fitz) |
| Data generation | HF Inference API + `transformers` |
| Embeddings | `sentence-transformers` |
| Vector search | `faiss-cpu` |
| Generation | HF `transformers` pipeline |
| App UI | `gradio` |
| Dataset hosting | Hugging Face Dataset repo |
| App hosting | Hugging Face Space |

---

## Grading Map
| Criterion | How we hit it |
|---|---|
| Problem (10%) | Real attack vector: LLMs processing PDFs with hidden injections |
| Data Generation (10%) | 10,000 text rows via HF model, 6 classes, structured schema |
| EDA (10%) | Class distribution, length delta analysis, UMAP clusters, hallucination flagging |
| Recommendation (10%) | Top-3 similar injections via FAISS + best of 3 HF embedding models |
| Generation (10%) | HF LLM explains injection type, intent, and danger |
| Application (10%) | Gradio app: PDF upload → full pipeline → results |
| Walkthrough (20%) | Live HF Space demo with 3 quick starters |
| Q&A (20%) | Understand every model, metric, and design decision |

---

## Open Questions for Sahar
- Can we use the HF Inference API (hosted inference endpoints) for data generation, or must we run models locally?
- For the actual PDF parsing in the app — do hidden-text detectors (checking font size / color attributes) count as our detection model, or do we need an ML classifier on top?
- Does the generation component need to output something *new* (novel text), or is a structured explanation sufficient?
