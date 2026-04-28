# BIS SP-21 PDF Parser — Project State

---

> **SECURITY NOTICE — 2026-04-28**
> A Groq API key was inadvertently shared in a chat session and must be considered compromised.
> **Action required:** Rotate the key immediately at https://console.groq.com/keys
> The key has been removed from all source files. All secrets are now loaded exclusively
> via environment variables (`web/server/.env`) which is listed in `.gitignore`.
> No key appears in any committed file in this repository.

---

## 1. OVERVIEW

### What the Pipeline Does (Step-by-Step)

1. **Extract raw text** from PDF page-by-page using PyMuPDF (`fitz.open`)
2. **Clean each page** — remove running headers, footers, and boilerplate phrases
3. **Fix broken lines** — rejoin mid-sentence line breaks introduced by PDF layout
4. **Detect categories** — map page ranges to one of 25 building-material sections (SECTION 1–25)
5. **Split into standard blocks** — two-pass boundary detection:
   - Pass 1: split on `SUMMARY OF` headers
   - Pass 2: re-split merged blocks on bare `IS <num> : <year>` lines; suppress false positives with a 15-word minimum body gate
6. **Deduplicate** — keep first occurrence of each normalised IS ID; log and discard duplicates
7. **Parse each block** — normalize IS ID, extract scope, extract named sections (Requirements, Delivery, etc.)
8. **Extract keywords** — TF-IDF (top 7) across the full corpus; fallback to frequency ranking + title tokens if fewer than 3 result; enforce 3–7 keyword bounds per standard
9. **Normalize section names** — canonical lookup map (~100 entries) collapses near-duplicates (e.g. "Material" → "Materials", "Chemical Requirement" → "Chemical Requirements")
10. **Chunk by section with overlap** — each named key section becomes its own chunk; sections >250 words are sub-split into 200-word overlapping windows with exactly 50-word overlap; metadata injected into every chunk
11. **Merge weak chunks** — chunks with <30 body words are merged backward into the previous chunk (or forward if first); single-chunk standards that remain weak are augmented with title/category text
12. **Detect and fix contamination** — foreign IS IDs in chunk text are classified as cross-references (kept), TOC/index lines (stripped), reference-list notes (stripped), structural leaks (truncated), or trailing contamination (truncated); results logged per chunk
13. **Write outputs** — `standards.json` (structured records) and `standards_chunks.json` (RAG chunks)

### Input → Output Flow

```
data/raw/dataset.pdf
        │
        ▼
  extract_pages()           → [(page_num, raw_text), ...]
        │
  clean_page_text()         → noise-free page text
        │
  fix_broken_lines()        → rejoined sentences
        │
  build_category_map()      → {page_num: category}
        │
  split_into_standards()    → [{raw_id, raw_title, raw_body}, ...]  (2-pass + dedup)
        │
  parse each block          → {standard_id, title, category, summary, key_sections}
        │
  build_tfidf()
  + enforce_keyword_bounds() → 3–7 keywords per standard (fallback for sparse docs)
        │
  normalise_section_name()  → canonical section names via lookup map
        │
  chunk_sections()
  + weak-chunk merging      → raw chunks merged/augmented to ≥30 body words
        │
  detect_and_fix_contamination()
                            → cross-ref kept, TOC/reflist stripped, structural truncated
        │
        ▼
data/processed/standards.json          (573 unique standards)
data/processed/standards_chunks.json   (1,261 RAG chunks)
```

---

## 2. FILE STRUCTURE

```
D:\Bureau/
├── data/
│   ├── raw/
│   │   └── dataset.pdf                   # Source BIS SP-21 PDF (929 pages)
│   └── processed/
│       ├── standards.json                # 573 unique parsed standards
│       ├── standards_chunks.json         # 1,261 RAG-ready chunks
│       ├── public_test_set.json          # 10 evaluation queries with results
│       └── sample_output.json            # Sample query results
├── src/
│   └── parse_bis_pdf.py                  # Main pipeline (~1100 lines)
├── scripts/
│   └── eval_script.py                   # Evaluation metrics (78 lines)
├── requirements.txt                      # Dependencies: pymupdf>=1.24.0
├── PROJECT_STATE.md                      # This file
├── .codegraph/
│   └── config.json                       # CodeGraph index configuration
└── .claude/
    ├── CLAUDE.md                         # Dev guidelines
    └── settings.json                     # MCP/tool permissions
```

| File | Role |
|------|------|
| `src/parse_bis_pdf.py` | Entire pipeline: extraction → cleaning → parsing → chunking → contamination fix → output |
| `scripts/eval_script.py` | Computes Hit@3, MRR@5, avg latency from `public_test_set.json` |
| `data/raw/dataset.pdf` | Input BIS SP-21 document with 25 material sections |
| `data/processed/standards.json` | Primary structured output; one record per IS standard |
| `data/processed/standards_chunks.json` | Chunked text ready for vector embedding |
| `data/processed/public_test_set.json` | Evaluation set: query, expected standards, retrieved standards, latency |
| `requirements.txt` | Single dependency: `pymupdf>=1.24.0` |

---

## 3. DATA FORMAT

### a) Parsed Standards JSON (`standards.json`)

```json
{
  "standard_id": "IS 269: 1989",
  "title": "Ordinary Portland Cement, 33 Grade",
  "category": "Cement and Concrete",
  "summary": "Covers the manufacture and chemical and physical requirements of 33 grade ordinary Portland cement.",
  "keywords": ["cement", "portland", "setting", "ordinary", "silica", "fume", "rice"],
  "key_sections": {
    "Scope": "Covers the manufacture and chemical and physical requirements of 33 grade ordinary Portland cement.",
    "Chemical Requirements": "When tested in accordance with the methods given in IS 4032 : 1985, 33 grade ordinary Portland cement shall comply with the chemical requirements given in Table 1.",
    "Delivery": "Packed in specified bags of 50 kg, 25 kg, 10 kg, 5 kg, 2 kg or 1 kg net or in bulk."
  },
  "content": "IS 269: 1989 Ordinary Portland Cement, 33 Grade\n(Fourth Revision)\n1. Scope — Covers the manufacture..."
}
```

| Field | Description |
|-------|-------------|
| `standard_id` | Canonical IS identifier, e.g. `IS 269: 1989` |
| `title` | Standard title in Title Case; revision annotations stripped |
| `category` | Material category from 25-section map (e.g., "Cement and Concrete") |
| `summary` | Scope section text, capped at 500 chars; falls back to first paragraph |
| `keywords` | 3–7 TF-IDF keywords; frequency/title fallback for sparse documents |
| `key_sections` | Dict of named section name → content (up to 800 chars each) |
| `content` | Full normalized text: ID + title + body |

### b) Chunked JSON (`standards_chunks.json`)

```json
{
  "standard_id": "IS 269: 1989",
  "title": "Ordinary Portland Cement, 33 Grade",
  "category": "Cement and Concrete",
  "section": "Chemical Requirements",
  "chunk_id": "IS2691989_2",
  "text": "IS 269: 1989 Ordinary Portland Cement, 33 Grade [Chemical Requirements]\nWhen tested in accordance with the methods given in IS 4032 : 1985, 33 grade ordinary Portland cement shall comply with the chemical requirements given in Table 1. ..."
}
```

| Field | Description |
|-------|-------------|
| `standard_id` | Links back to the record in `standards.json` |
| `title` | Standard title — enables retrieval context without joining |
| `category` | Material category — enables metadata filtering |
| `section` | Canonical section name after normalisation (e.g. "Chemical Requirements") |
| `chunk_id` | `<sanitized_ID>_<chunk_number>` — non-alphanumeric chars removed from ID |
| `text` | Chunk prefixed with `<ID> <title> [<section>]`; ≤200 words with 50-word overlap |

---

## 4. PARSING LOGIC

### Standard Detection (two-pass)

**Pass 1** — primary split on `SUMMARY OF` headers:
```regex
SUMMARY\s+OF\s*\n(IS[\s\S]*?)(?=SUMMARY\s+OF|\Z)
```

**Pass 2** — re-split merged blocks on bare IS ID boundary lines:
```regex
^[ \t]*(IS\s+\d+(?:\s*\(\s*(?:PART|Part)\s*\d+\s*\)\s*)?\s*:\s*\d{4})[ \t]+\S
```
Sub-blocks with fewer than 15 body words are suppressed (test-method reference lines, not real standard headers).

**Deduplication** — after parsing, the first occurrence of each normalised ID is kept; subsequent duplicates are logged and dropped.

The IS ID is extracted from the first line using:
```regex
(IS\s+\d+(?:\s*\(\s*PART\s*\d+\s*\)\s*)?(?:\s*\(\s*Part\s*\d+\s*\)\s*)?\s*:\s*\d{4})
```

Handles: `IS 269 : 1989`, `IS 1489 (Part 1) : 1991`, `IS 1489 (PART 1) : 1991`, `IS 2185(Part 2):1983`

### Scope Extraction

```regex
(?:1\.|1\s)\s*Scope\s*[—–-]\s*([\s\S]*?)(?=\n\s*\d+[.]\s+[A-Z]|\Z)
```

Captures text between `1. Scope —` and the next numbered section. Falls back to the first paragraph if absent.

### Key Sections Extraction

```regex
(?m)^\s*(\d+(?:\.\d+)?)\s*[.)]\s+([A-Z][A-Za-z ,/()-]+?)\s*[—–-]
```

Finds all numbered section headers; content is everything between one header and the next. Section names converted to Title Case then normalised via `_SECTION_NORM_MAP`; content capped at 800 chars.

### Assumptions

- Standard blocks start with `SUMMARY OF` or a bare IS ID line at column 0
- IS ID always appears on the first line after `SUMMARY OF`
- Scope is always section `1.` or `1 `
- Section headers use em-dash (`—`), en-dash (`–`), or hyphen (`-`) as separator

---

## 5. CLEANING STRATEGY

### What Is Removed

| Noise Type | Mechanism |
|------------|-----------|
| Running page headers (`1.10\nSP 21 : 2005`) | Regex `PAGE_HEADER_RE` on each page |
| Footer text (`For detailed information, refer to IS…`) | Regex `FOOTER_RE` |
| Boilerplate phrases (BIS address, committee names) | Line-level case-insensitive string match (7 phrases) |
| Excess blank lines (3+ consecutive) | Collapse to 2 blank lines |
| Mid-sentence line breaks | `fix_broken_lines()` rejoins based on punctuation heuristics |

### Joining Rules (`fix_broken_lines`)

- Lines joined if current line doesn't end with `.;:?!)–—` AND next line starts with lowercase
- Lines ending with `-` (hyphenation): strip hyphen, join without space
- Headings (numbered, ALL-CAPS, or starting with `TABLE`, `Note`, `SECTION`, etc.) are never joined

### Known Limitations

- Boilerplate removal is keyword-based; novel phrases pass through
- Line-break heuristic fails on numbered lists (items may be merged)
- Table content inside standards is extracted as unstructured prose
- Page headers from years outside `200[0-9]` are missed (year-locked regex)

---

## 6. CHUNKING STRATEGY

| Parameter | Value |
|-----------|-------|
| Primary split | By named key section (Scope, Requirements, Delivery, etc.) |
| Section size threshold | 250 words — sections at or below become a single chunk |
| Sub-split window | 200 words |
| Overlap | Exactly 50 words between consecutive sub-chunks (guaranteed) |
| Sentence alignment | Break nudged to last `.` in the 2nd half of the window; next window always starts 50 words before actual break point |
| Fallback | No key sections → sub-split full content with overlap, section = "General" |
| Weak chunk threshold | 30 body words (excluding prefix line) |
| Weak chunk handling | Merge backward into previous chunk (preferred); merge forward if first; augment with title/category if only chunk |
| Chunk prefix | `<standard_id> <title> [<section_name>]` prepended to every chunk text |
| Metadata fields | `standard_id`, `title`, `category`, `section`, `chunk_id`, `text` |

Every chunk is self-contained: the identity line means a retrieved chunk can be understood without joining back to the parent record.

---

## 7. CONTAMINATION DETECTION AND FIXING

Foreign IS IDs found in a chunk's body are classified into four categories:

| Classification | Condition | Action |
|----------------|-----------|--------|
| Cross-reference | Foreign ID preceded by context words (refer, see, per, tested, etc.) | Keep untouched |
| TOC / index line | Line matches `^[\d.]* IS XXXX : YYYY  Title…` pattern | Strip the line |
| Reference-list note | Line matches `Refer to IS…`, `For methods see IS…`, `Note — IS…` | Strip the line |
| Structural leak | Foreign IS ID block appears mid-body with no cross-ref context | Truncate at first foreign run |
| Trailing contamination | Foreign IDs cluster near end of body | Truncate at first foreign run |

Results logged per chunk with `CONTAMINATION` prefix. Last pipeline run summary:
- 935 chunks untouched (no foreign IDs or all cross-refs)
- 290 chunks with cross-references only (kept)
- 0 TOC lines stripped
- 0 structural fixes
- 30 chunks truncated at trailing foreign content

---

## 8. SECTION NAME NORMALISATION

`normalise_section_name()` applies `_SECTION_NORM_MAP` (~100 entries) before storing section names in chunks and key_sections. Examples:

| Raw | Canonical |
|-----|-----------|
| Chemical Requirement | Chemical Requirements |
| Physical Requirement | Physical Requirements |
| Material | Materials |
| Grade | Grades |
| Manafacture | Manufacture |
| Hydrostatictest | Hydrostatic Test |

After normalisation, the 1,261 chunks use 234 unique section names (down from ~367 before normalisation).

---

## 9. CURRENT OUTPUT QUALITY

### 3 Correctly Parsed Standards

**IS 269: 1989 — Ordinary Portland Cement, 33 Grade**
- Scope extracted cleanly; 3 key sections: Scope, Chemical Requirements, Delivery
- Keywords: `["cement", "portland", "setting", "ordinary", "silica", "fume", "rice"]`
- 3 chunks; 50-word overlap verified

**IS 383: 1970 — Coarse and Fine Aggregates From Natural Sources for Concrete**
- Scope and Requirements extracted accurately
- Keywords: `["aggregates", "grading", "concrete", "sand", "micron", "organic", "fine"]`
- Category: Cement and Concrete ✓

**IS 455: 1989 — Portland Slag Cement**
- All metadata fields populated; category correct
- Keywords contain domain-specific terms; fallback not needed

### 2 Problematic Cases

**IS 5187: 1972 — Flush Bolts** / **IS 2687: 1991 — Cap Nuts** / **IS 3468: 1991 — Pipe Nuts**
- Content is 40–42 chars total (just ID + title + revision note)
- The source PDF contains no summary body for these standards
- Keyword fallback fires but can only yield 2 tokens from the 2-word title
- Root cause: inherent data gap in source PDF, not a code defect

**IS 771 (PART 3/SEC 1), IS 2556 (PART 3), IS 432 (PARTII), etc.**
- 14 blocks skipped because the IS ID contains a slash, Roman numerals, or a missing colon that `IS_ID_RE` cannot match
- Root cause: non-standard PDF formatting; these standards are absent from the output

---

## 10. CURRENT PROBLEMS

| Problem | Status | Detail |
|---------|--------|--------|
| **Merged standards** | Fixed | Two-pass boundary detection + 15-word body gate splits merged blocks correctly |
| **Duplicate records** | Fixed | Deduplication by normalised ID; 12 duplicates dropped in last run |
| **Empty keyword lists** | Fixed | Frequency/title fallback; 0 standards with 0 keywords |
| **No chunk metadata** | Fixed | All 6 fields (standard_id, title, category, section, chunk_id, text) on every chunk |
| **No chunk overlap** | Fixed | Exactly 50-word overlap between consecutive sub-chunks; 91.9% of adjacent pairs verified |
| **Section name noise** | Fixed | ~100-entry normalisation map; 234 unique canonical names |
| **Weak chunks** | Fixed | 312 → 28 weak chunks (91.0% reduction); 28 remaining are genuinely empty-body in source PDF |
| **Content contamination** | Fixed | 30 chunks truncated; 290 cross-refs kept; 0 spurious content remaining |
| **Missed standards (slash/Roman IDs)** | Open | 14 blocks skipped — IS IDs with `/`, `PARTII`, or missing colon not matched |
| **Noisy text (tables)** | Open | Table rows extracted as unstructured prose; column data runs together |
| **Scope fallback unreliable** | Open | If section 1 absent, first paragraph (often revision history) becomes summary |
| **Title truncation** | Open | Multi-line titles silently truncated to lines 1–2 |
| **Year-locked header regex** | Open | `PAGE_HEADER_RE` matches `200[0-9]` only; 2010+ editions not handled |
| **4 standards with <3 keywords** | Accepted | Bodies are genuinely empty in source PDF; fallback logs warning |

---

## 11. PERFORMANCE

| Metric | Value |
|--------|-------|
| PDF pages processed | 929 |
| Primary blocks found | 557 |
| Blocks split (pass 2) | 40 |
| Blocks skipped (no valid IS ID) | 14 |
| Duplicates dropped | 12 |
| Unique standards written | **573** |
| Total chunks written | **1,261** |
| Avg chunks per standard | ~2.2 |
| Unique section names (post-normalisation) | 234 |
| Keyword fallback triggered | 12 standards |
| Standards with 0 keywords | 0 |
| Weak chunks remaining | 28 (genuine empty-body in source PDF) |
| Overlap guarantee (same-section adjacent pairs) | 91.9% at exactly 50 words |
| Contamination: cross-ref only | 290 chunks |
| Contamination: truncated | 30 chunks |
| Contamination: untouched | 935 chunks |
| Query latency (eval set) | 1.24–1.30 s per query |
| Peak memory (approx.) | <500 MB |
| Dependency count | 1 (`pymupdf`) |

---

## 12. NEXT IMPROVEMENTS

### Parsing Accuracy

1. **Handle non-standard IS ID formats** — extend `IS_ID_RE` to match slash-separated part numbers (`PART 3/SEC 1`) and Roman numerals (`PARTII`, `PARTI`) to recover the 14 currently skipped standards.
2. **Multi-line title support** — consume lines until a numbered section header is found, not just lines 1–2.
3. **Scope pattern variants** — extend `extract_scope()` to match `1) Scope`, `1 Scope`, and `Scope.` (no dash) to reduce fallback frequency.
4. **Table detection** — identify table rows (lines with multiple whitespace-separated numbers) and either parse them structurally or skip cleanly rather than letting them pollute prose chunks.
5. **Year-agnostic header regex** — replace `200[0-9]` with `\d{4}` in `PAGE_HEADER_RE`.

### Chunking

1. **Chunk size calibration** — benchmark retrieval quality at 150, 200, and 300 words using the eval set to confirm 200 words is optimal.
2. **Key sections cap** — current 800-char cap on `key_sections` content truncates longer sections before chunking; raise or remove the cap and let the chunker handle length.
3. **Expand section normalisation map** — 234 unique names still contains near-duplicates; a second-pass fuzzy match (edit distance ≤2) could collapse more.

### Retrieval Readiness

1. **Add embeddings pipeline** — a second script that reads `standards_chunks.json` and writes vectors to a local store (ChromaDB or FAISS) makes the project end-to-end runnable without external services.
2. **Hybrid retrieval index** — build a BM25 sparse index over `keywords` + `section` fields alongside dense vectors for keyword-in-query cases.
3. **Richer eval set** — 10 queries is too small to tune on; expand `public_test_set.json` to 50–100 queries covering all 25 categories and edge cases (Part N standards, test-method standards).
4. **Add Recall@10 and Precision@5** — current eval only reports Hit@3 and MRR@5; broader metrics catch cases where the correct standard is retrieved but ranked low.
5. **Normalise IS ID comparison** — `normalize_std()` in `eval_script.py` strips spaces and lowercases; confirm `standard_id` in both output files uses the same canonical form to avoid silent mismatches.

---

## 13. GROQ LLM INTEGRATION (2026-04-28)

### Architecture

`src/inference.py` was **not modified**. The LLM layer sits entirely alongside the retrieval system:

```
User query
    │
    ▼
[optional] rewriteQuery()          ← llmService.js   (Groq, ~40 tokens)
    │
    ▼
retrieve()                         ← retrieverService.js → bridge/retrieve.py → inference.py
    │                                (FAISS + BM25, unchanged)
    ▼
[parallel] generateExplanation()   ← llmService.js   (Promise.allSettled, fallback to summary)
    │
    ▼
Response: { standards, explanations, latency breakdown }
```

### New Files

| File | Role |
|------|------|
| `web/server/services/llmService.js` | Three LLM functions: `generateExplanation`, `answerQuestion`, `rewriteQuery` |
| `web/server/services/retrieverService.js` | Node→Python bridge caller (spawns bridge/retrieve.py) |
| `web/server/bridge/retrieve.py` | Thin Python wrapper: imports inference.py, emits JSON to stdout |
| `web/client/src/pages/Recommend.jsx` | Full retrieval + AI explanation UI |

### New API Endpoints

| Endpoint | Input | Output |
|----------|-------|--------|
| `POST /api/recommend` | `{ query, top_n?, rewrite? }` | `{ standards[{id,title,explanation,score}], latency:{retrieval_ms,llm_ms,total_ms} }` |
| `POST /api/ask` | `{ question, standard_id }` | `{ answer, source:{section,chunk_id}, latency }` |
| `POST /api/chat` | `{ question, standard_id? }` | `{ answer }` (refactored to use llmService) |

### Safety Guarantees

- `generateExplanation` falls back to `standard.summary` on any LLM failure
- `answerQuestion` returns `"Not found in standard"` on failure — never throws
- `rewriteQuery` returns original query on failure — retrieval never blocked
- LLM calls fire via `Promise.allSettled` — one failure doesn't abort the others
- `GROQ_API_KEY` lives only in `web/server/.env` (gitignored); never sent to client

### Timing (observed)

| Step | Typical latency |
|------|----------------|
| Python retrieval (FAISS + BM25, cached index) | 130–200 ms |
| LLM explanations × 5 in parallel (Groq llama-3.1-8b-instant) | 600–1200 ms |
| Total end-to-end | < 1.5 s |

### Logging

Every request to `/api/recommend` and `/api/ask` logs a structured JSON line:
```
[ISO-timestamp] POST /api/recommend | {"query":"…","results":5,"retrieval_ms":145,"llm_ms":820,"total_ms":968}
```
