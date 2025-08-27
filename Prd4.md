# Product Requirements Document (PRD)

## Project: Hybrid High-Throughput Log Parsing System

### 1. Overview  
Build a scalable, self-optimizing log parsing pipeline combining:
1. **Regex Parser Layer**: Ultra-fast extraction of known multi-word and simple key–value patterns.  
2. **SpaCy CNN Parser Layer**: Machine-learning fallback for novel or complex spans.  
3. **LLM Parser Layer**: Final fallback using a large language model for any remaining cases.  
4. **Parallel I/O & Processing Infrastructure**: Asynchronous, memory-mapped reading of large log files in 10 000-line chunks, multi-process parsing, and ordered merging via a priority queue.

***

### 2. Requirements

#### 2.1 Functional Requirements

1. **Regex Parser Component**
   - Load `patterns.json` on startup containing:
     - `signature`: unique name  
     - `regex`: multi-word composite pattern  
     - `expected_keys`: list of keys  
   - Compile a single alternation regex for all composite patterns.  
   - Generic fallback regex for any `key[:=]value` pairs.  
   - Validation:
     - Ensure all `expected_keys` extracted.  
     - No empty values.  
     - No leftover key–value substrings remain.  
   - API: `parse(line: str) -> Dict[str,str]`.

2. **SpaCy CNN Parser Component**
   - SentencePiece subword tokenizer with numeric normalization.  
   - SpaCy v3 blank English pipeline with `ner` CNN token-classification:
     - Labels: `B-EVENT`, `I-EVENT`, `B-KEY`, `I-KEY`, `B-VALUE`, `I-VALUE`.  
   - Training script consumes CoNLL `train.txt`, `dev.txt`, `test.txt`.  
   - Inference:
     - `parse(line: str) -> Dict[str,str]`.  
     - `needs_fallback(line, extracted) -> bool` to detect leftover pairs.

3. **LLM Fallback Parser Component**
   - OpenAI (or similar) chat-completion API integration.  
   - Few-shot prompt template with examples.  
   - Retry logic for JSON extraction.  
   - API: `parse(line: str) -> Dict[str,str]`.

4. **Parallel Processing Infrastructure**
   - **Async File Reader**:
     - Use `aiofiles` to read lines asynchronously in chunks of 10 000.  
     - Strip timestamps and unnecessary words pre-enqueue.  
     - Enqueue `(chunk_index, [lines])` into `asyncio.Queue`.
   - **Worker Pool**:
     - `ProcessPoolExecutor` with worker per CPU core.  
     - Each worker calls HybridParser on its chunk:
       - Strip stopwords/unnecessary tokens.  
       - Call regex → spaCy → LLM fallback.  
     - Return `(chunk_index, [parsed_records])`.
   - **Priority Queue Merger**:
     - Thread-safe `heapq` ordered by `chunk_index`.  
     - Emit chunks in increasing index order to output sink (file or database).

#### 2.2 Non-Functional Requirements

- **Performance**:
  - Regex layer: ≥100 000 lines/sec per process, <0.01 ms latency.  
  - SpaCy layer: ≥10 000 lines/sec on CPU.  
  - LLM layer: <5% of lines, 100–500 ms per call.  
  - End-to-end pipeline: sustain ≥50 000 lines/sec on 8-core machine.
- **Scalability**: Adjustable chunk size and worker count.  
- **Reliability**: Fallbacks at each layer; ordered output guaranteed.  
- **Configurability**: Thresholds for pattern migration, ML confidence, chunk size via config file.  
- **Observability**: Metrics for parser hits (regex/spacy/LLM), migration events, throughput, latency; logging for validation failures and API errors.

***

### 3. Architecture & Data Flow

1. **Reader**  
   - Async task reads file in 10 000-line batches, normalizes timestamp, strips unnecessary words, enqueues `(idx, raw_lines)`.
2. **Parser Pool**  
   - Each process dequeues a batch, processes each line:
     1. `RegexPatternParser.parse` → if valid, emit.  
     2. Else `SpaCyKVParser.parse` + `needs_fallback` → if valid, emit.  
     3. Else `LLMFallbackParser.parse` → emit.
   - Return `(idx, parsed_records)`.
3. **Merger**  
   - Push results into a min-heap keyed by `idx`.  
   - Pop and write output when `idx == next_expected`, increment pointer.

***

### 4. Data Models & Files

- **patterns.json**: Persistent JSON of composite regex patterns.
- **migration_stats.json**: ML-driven pattern discovery stats.
- **CoNLL Data**: `data/train.txt`, `dev.txt`, `test.txt` for SpaCy training.
- **Output Sink**: JSON-L file or database table of parsed records.

***

### 5. Success Metrics

- **Accuracy**: ≥98% correct key–value extraction on test logs.
- **Performance**: ≥50 000 lines/sec sustained.
- **Migration Efficiency**: ≥90% of stable patterns migrated within 24 hours of observation.
- **Fallback Rate**: Regex + SpaCy handle ≥95% of lines; LLM <5%.

***

### 6. Roadmap

1. **Week 1–2**: Implement RegexPatternParser, generic pipeline, initial patterns.json.  
2. **Week 3–4**: Train SpaCyCNNParser, integrate fallback logic, needs_fallback detection.  
3. **Week 5**: Build LLMFallbackParser and integrate as final tier.  
4. **Week 6**: Develop async reader, processing pool, and priority queue merger.  
5. **Week 7**: End-to-end testing, performance tuning, observability dashboards.  
6. **Week 8**: Deployment, monitoring setup, pattern migration automation.

***

This PRD defines all components, data flows, performance goals, and development phases needed to deliver a robust, high-yield log parsing system leveraging regex, ML, and LLM technologies.

