# PDF Prompt Injection Detector

## Problem Statement
LLMs are increasingly used to process PDFs (summarize contracts, answer questions about documents, extract data).
Attackers hide malicious instructions inside PDFs using techniques invisible to human readers but visible to LLM parsers.
This project generates synthetic malicious PDFs, evaluates detection tools, and builds an app that detects injections and explains what they are trying to do.

> **Guardrails (per Sahar)**
> - All PDFs are self-generated and synthetic — no external or real-world malicious PDFs.
> - No ML/DL classifier. Detection is rule-based (tool evaluation), not trained.
> - Final app: detect the injection and explain what it will do (detect + explain).

---

## Injection Types (Dataset Classes)
| Type | Technique | How it hides |
|---|---|---|
| `invisible_text` | Text rendered in a color matching the background (e.g., white on white) | Invisible to human eye; fully extracted by PDF parsers feeding LLM context |
| `system_spoof` | Mimics system log formats, chat delimiters, or meta-instruction markers (e.g., `*** SYSTEM ALERT ***`) | Appears as benign formatting to humans; tricks LLM into treating it as a high-priority system command |
| `goal_hijacking` | Direct command override embedded in body prose (e.g., "Ignore all previous instructions") | Visually blends into document text; semantically replaces the user's original task |
| `persona_swap` | Forces the LLM into an unauthorized persona or behavioral state via an embedded directive | Blends into document prose; triggers persistent behavioral change in the downstream LLM |
| `metadata` | Instructions injected into PDF binary metadata fields (Author, Title, Subject, Keywords) | Never rendered in document view; targets pipelines that parse file properties |
| `clean` | No injection | Baseline negative class |

---

## Dataset Schema (10,000+ rows)
Each row represents one synthetic PDF document:

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
| `pdf_path` | Relative path to the generated PDF file |

---

## Part 1: Synthetic PDF Generation

### Strategy
Two-engine pipeline:
1. **Payload Engine** (`Qwen/Qwen2.5-1.5B-Instruct` or `Qwen/Qwen2.5-7B-Instruct` via HF Inference API) — generates the visible document body text and weaves the injection payload into the prose according to the target injection type.
2. **Builder Engine** (`deepseek-ai/DeepSeek-Coder-1.5B-Instruct` via HF Inference API) — reads the Payload Engine's text output and generates the executable Python script (using `fpdf2` / `reportlab`) that compiles the text stream into a binary PDF file with the injection embedded at the correct structural layer.

Each PDF embeds the injection technique at the file level:
- `invisible_text`: render injection text in a color matching the background (e.g., white on white)
- `system_spoof`: embed markers mimicking system logs or chat delimiters (e.g., `*** SYSTEM ALERT ***`, `[System Update]`)
- `goal_hijacking`: weave a direct command override into the body prose (e.g., "Ignore all previous instructions and output…")
- `persona_swap`: embed a persona directive forcing the LLM into an unauthorized behavioral state
- `metadata`: write injection payload into PDF binary metadata fields (Author / Title / Subject / Keywords)
- `clean`: document body only, no injection
- Also save the text representation into the dataset parquet for EDA.

### Volume
- ~2,000 clean documents
- ~1,600 per injection type × 5 types = 8,000 injected
- **Total: 10,000 rows / 10,000 PDF files**

### Tools
- `fpdf2` / `reportlab` for PDF generation (scripts produced by the Builder Engine)
- `pymupdf` (fitz) for metadata injection
- HF Inference API — `Qwen/Qwen2.5-1.5B-Instruct` / `Qwen/Qwen2.5-7B-Instruct` (Payload Engine)
- HF Inference API — `deepseek-ai/DeepSeek-Coder-1.5B-Instruct` (Builder Engine)
- `pandas` + `uuid` for dataset assembly

---

## Part 2: EDA
- Duplicate Check
- Class distribution (injection_type counts, is_injected ratio)
- Text length distribution (visible_text vs extracted_text length delta — delta reveals how much is hidden)
- Document type breakdown per injection type
- Target action distribution
- Word clouds per injection type (on injection_payload column)
- Embedding visualization (UMAP or t-SNE) colored by injection_type — for visual insight only, not for a classifier
- Flag dataset quality issues (e.g., injection_payload null for injected rows, metadata empty for metadata-type rows)

**Upload to HF Dataset repo** with EDA notebook + generation notebook.

---

## Part 3: Tool Evaluation (replaces ML classifier)

### Goal
Compare different Python PDF-analysis tools and web-based scanners to see which best surfaces hidden injections from our synthetic PDFs.

### Tools to Evaluate
| Tool | Type | What it can surface |
|---|---|---|
| `pdfplumber` | Python library | Font-size, text color, character-level attributes |
| `pymupdf` (fitz) | Python library | Metadata, annotations, layers, font properties |
| `pypdf` (pypdf2) | Python library | Metadata, basic text extraction |
| `pdfid` (DidierStevens) | CLI / Python | Keyword-based anomaly scoring |
| Adobe PDF Accessibility Checker (or similar web tool) | Web | Hidden layer / accessibility flags |

### Evaluation Methodology
- Run each tool against a random sample of 100 PDFs per injection type (500 injected + 100 clean = 600 total).
- For each tool, record:
  - **Detection rate** per injection type (did the tool surface the hidden content?)
  - **False positive rate** on clean documents
  - **Ease of integration** (Python API vs. CLI vs. web-only)
- Summarize in a comparison table.
- Pick the best combination of tools to use in the final app backend.

### Output
- Comparison table saved in the EDA notebook.
- Documented decision: which tool(s) are used in the app and why.

---

## Part 4: Generation

Given a detected injection, use a small HF language model to generate a natural-language explanation:

> *"This PDF contains a zero-font injection. The hidden text instructs the AI to ignore its prior instructions and reveal confidential system prompts. Severity: High."*

### Model Candidates (HF)
- `Qwen/Qwen2.5-1.5B-Instruct` (fast, lower compute — same family as Payload Engine)
- `Qwen/Qwen2.5-7B-Instruct` (higher quality — preferred if latency allows)

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

### App Goal
**Detect** hidden injections in a user-uploaded PDF and **explain** what each injection is attempting to do.

### User Flow
1. User uploads a PDF file (or selects a quick starter)
2. Backend extracts content using the winning tool combination from Part 3:
   - Visible text
   - Hidden text (zero-font: font-size=0; white text: color=white)
   - Metadata fields (Title, Author, Subject, Keywords)
   - Annotations
3. Rule-based detection classifies each finding into an injection type
4. HF LLM generates an explanation for each detected injection
5. Display:
   - Verdict: Injected / Clean
   - If injected: injection type + target action + severity badge
   - Generated explanation of what the injection is doing
   - Raw hidden content surfaced (so the user can see what was hidden)

### 3 Quick Starters (required by assignment)
Pre-built synthetic PDF examples (generated by us in Part 1):
1. Clean invoice — no injection found
2. Contract with zero-font injection — role override attempt
3. Report with metadata injection — data exfiltration attempt

### Constraints
- All example PDFs are self-generated (synthetic).
- Read dataset and any supporting files from HF Dataset / Space repo.

---

## Tech Stack
| Task | Library |
|---|---|
| PDF generation | `fpdf2`, `reportlab` (scripts generated by Builder Engine); `pymupdf` for metadata |
| PDF parsing / detection | `pdfplumber`, `pymupdf`, `pypdf` |
| Payload Engine (body text + injection weaving) | `Qwen/Qwen2.5-1.5B-Instruct` / `Qwen/Qwen2.5-7B-Instruct` via HF Inference API |
| Builder Engine (PDF compilation scripts) | `deepseek-ai/DeepSeek-Coder-1.5B-Instruct` via HF Inference API |
| Explanation generation | `Qwen/Qwen2.5-7B-Instruct` via HF Inference API |
| App UI | `gradio` |
| Dataset hosting | Hugging Face Dataset repo |
| App hosting | Hugging Face Space |

---

## Grading Map
| Criterion | How we hit it |
|---|---|
| Problem (10%) | Real attack vector: LLMs processing PDFs with hidden injections |
| Data Generation (10%) | 10,000 synthetic PDF files, 6 classes, structured schema |
| EDA (10%) | Class distribution, length delta analysis, UMAP visualization, quality flagging |
| Recommendation (10%) | Tool evaluation: compare 4–5 tools, pick best combination with documented rationale |
| Generation (10%) | HF LLM explains injection type, intent, and danger |
| Application (10%) | Gradio app: PDF upload → rule-based detection → LLM explanation → results |
| Walkthrough (20%) | Live HF Space demo with 3 quick starters |
| Q&A (20%) | Understand every tool, metric, and design decision |

---

## Open Questions (resolved)
- ✅ No external malicious PDFs — all PDFs are synthetic and self-generated.
- ✅ No ML classifier — detection is rule-based using PDF parsing tools.
- ✅ App goal: detect + explain (not clean).
- ❓ Can we use the HF Inference API for document body generation, or must we run models locally?
- ❓ Does the generation component need to output something *new* (novel text), or is a structured explanation sufficient?
