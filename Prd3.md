# Project Product Requirements Document (PRD)

## Project Overview  
Build a **Hybrid Log Parsing System** that combines high-speed, rule-based regex parsing with a SpaCy CNN token-classification model. The system initially handles common, stable multi-word key–value patterns via compiled regex; falls back to a generic regex catch-all; and uses the SpaCy CNN model for novel or complex cases. Over time, patterns discovered by the ML component migrate into the regex layer for continuous performance gains.

***

## 1. Regex-Based Log Parser

### 1.1 Purpose  
Provide ultra-fast extraction of known multi-word key–value combinations and single key–value pairs, with validation to ensure completeness. Serve as the first two tiers of parsing.

### 1.2 Key Features  
- **Persistent Regex Database**:  
  - `patterns.json` stores pattern signatures, regex strings, and expected key lists.  
  - Admins may seed complex patterns manually.  
- **Combined Pattern Compilation**:  
  - Compile all complex patterns into one alternation regex for O(1) dispatch.  
- **Generic Fallback Regex**:  
  - Single-pass `finditer` extracts any `key[:=]value` for simple lines.  
- **Validation Logic**:  
  - Ensures all expected keys are captured, no empty values, and no leftover pairs remain in the line.  
- **Pattern Migration Hooks**:  
  - After high-confidence SpaCy parses, auto-generate new composite regex via consensus patterns and add to `patterns.json`.  
- **Performance Targets**:  
  - ≥100 000 lines/sec on 8-core CPU, <0.01 ms latency per line for composite patterns.

### 1.3 Inputs & Outputs  
- **Input**: Single log line string.  
- **Output**: Dictionary of extracted key–value pairs (and optional `event`).

### 1.4 Success Criteria  
- 90%+ of log lines handled solely by regex layer.  
- Validation failure rate <1% for composite patterns.  
- New pattern migration reduces ML fallback rate by 5–10% per week.

***

## 2. SpaCy CNN Log Parser

### 2.1 Purpose  
Serve as the robust ML fallback and pattern discovery component, accurately extracting multi-token events and key–value spans from logs using subword tokenization.

### 2.2 Key Features  
- **Subword Tokenization**:  
  - SentencePiece model with numeric normalization (`<NUM>`, `<FLOAT>`, etc.).  
  - Parentheses and separators split into individual tokens.  
- **Token Classification**:  
  - CNN-based `ner` pipe with BIO labels: `B-EVENT`, `I-EVENT`, `B-KEY`, `I-KEY`, `B-VALUE`, `I-VALUE`.  
- **Training Pipeline**:  
  - Consume CoNLL-formatted `train.txt`, `dev.txt`, `test.txt` with token–label pairs.  
  - Align subword tokens: first subtoken retains label, others labeled `O`.  
  - Train for configurable epochs with dev-set evaluation.  
- **Inference API**:  
  - `parse(line)` returns `{event:…, key: value, …}`.  
  - `needs_fallback(line, extracted)` signals if leftover pairs require additional parsing.  
- **Confidence Scoring**:  
  - Compute per-line confidence to guide pattern migration.

### 2.3 Inputs & Outputs  
- **Input**: Single log line string.  
- **Output**: Dictionary with optional `"event"` plus key–value pairs.

### 2.4 Success Criteria  
- ≥95% extraction accuracy on held-out test set.  
- Inference throughput ≥10 000 lines/sec on CPU, ≥50 000 lines/sec on GPU batch.  
- New regex patterns generated with ≥90% precision from ML parses.

***

## 3. Hybrid Integration & Workflow

### 3.1 Parsing Sequence  
1. **Composite Regex**: Attempt combined multi-word patterns.  
2. **Generic Regex**: Fallback to catch-all key–value extraction.  
3. **ML Parser**: SpaCy CNN for remaining complex cases.

### 3.2 Validation & Routing  
- Validate regex outputs before acceptance; fall back on incomplete extractions.  
- ML parser flags leftover pairs via `needs_fallback`; triggers pattern generation.

### 3.3 Pattern Migration  
- Track co-occurrence stats and confidence for ML-parsed lines.  
- After threshold hits with stable structure, auto-generate composite regex and register it.

### 3.4 Monitoring & Metrics  
- Track parsing distribution: regex vs. generic vs. ML fallback rates.  
- Log migration events and evaluate performance improvements.  
- Maintain dashboard showing throughput, accuracy, and pattern growth.

***

## 4. Implementation Roadmap

1. **Phase 1: Core Regex Parser**  
   - Develop `RegexPatternParser` with persistent DB, combined compilation, generic fallback, and validation.  
   - Seed initial `patterns.json` with common patterns.

2. **Phase 2: SpaCy CNN Parser**  
   - Train SentencePiece model and SpaCy CNN on annotated CoNLL data.  
   - Build `SpaCyKVParser` with tokenization, parsing, and confidence scoring.

3. **Phase 3: Integration & Fallback Logic**  
   - Implement hybrid `parse_line` workflow combining both parsers.  
   - Add validation and fallback routing.

4. **Phase 4: Pattern Migration Engine**  
   - Develop analytics to collect ML parse stats, generate regex via consensus, and update DB.  
   - Automate pattern registration and combined regex recompilation.

5. **Phase 5: Testing & Deployment**  
   - End-to-end tests on production log samples.  
   - Performance benchmarking, monitoring setup.  
   - CI/CD integration for model retraining and pattern updates.

***

## 5. Dependencies & Environment

- Python ≥3.8  
- Libraries: `spacy`, `sentencepiece`, standard `re`, `json`, `time`  
- Hardware: CPU with 8+ cores, optional GPU for faster ML inference

***

## 6. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Composite regex DB grows too large | Archive rarely used patterns; limit composite patterns to top N frequent. |
| Regex brittle on format drift | Ensure robust validation; fallback to ML. |
| ML model accuracy degrades over time | Schedule periodic retraining with new annotated data. |
| Performance bottleneck on combined regex | Benchmark regex size; shard patterns by log source. |

***

This PRD outlines the complete design, features, workflows, and success metrics for building a robust, self-optimizing hybrid log parsing system using both **regex** and **SpaCy CNN** components.

