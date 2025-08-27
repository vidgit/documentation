

# Product Requirements Document (PRD)

## 1. Regex-Based Log Parser

### 1.1 Overview  
A high-performance rule-based component that extracts key–value pairs (and optional event spans) from log lines using compiled regular expressions. Frequently observed key groups are migrated into specialized composite patterns for maximum speed.

### 1.2 Goals  
- Extract single and multi-key–value patterns in one pass.  
- Validate completeness of extracted keys before accepting.  
- Dynamically migrate stable patterns from ML layer to regex layer.  
- Achieve ≥100 000 lines/sec throughput on typical server CPU.

### 1.3 Inputs  
- New log line string.  
- Compiled combined regex of all registered patterns.  
- Map of `pattern_signature → expected_keys` for validation.

### 1.4 Outputs  
- Python dict of extracted fields, e.g.  
  ```json
  {"event":"startup","a":"5","b":"true"}
  ```

### 1.5 Functional Requirements  

1. **Pattern Registration**  
   - **Generic pattern**: a single-key catch-all using `finditer`.  
   - **Composite patterns**: anchored multi-key sequences (named groups for each key).  
   - Store `expected_keys` list per composite pattern.

2. **Combined Regex Compilation**  
   - Build one alternation regex:  
     ```regex
     (?P<pat_ab>…)|(?P<pat_abc>…)|…
     ```
   - Compile once on startup for O(1) match dispatch.

3. **Line Parsing Logic**  
   1. Attempt `match = combined_regex.match(line)`  
   2. If `match` and `expected_keys ⊆ match.groupdict().keys()`:  
      - Extract values by group names.  
      - Return only those keys.  
   3. Else, run generic `finditer` to extract all `key[:=]value` pairs.  
      - Return dict of all matches.  
   4. If still empty or incomplete, delegate to ML parser.  

4. **Validation**  
   - For composite patterns, require all `expected_keys` present.  
   - On partial match, use partial output for those keys and fallback ML for rest.

5. **Pattern Migration (Analytics)**  
   - Collect stats on ML-parsed lines: signature, key co-occurrence, confidence.  
   - When a key group appears ≥N times with high confidence and low variance, auto-generate composite regex via structural consensus.  
   - Register new pattern and recompile combined regex.

### 1.6 Non-Functional Requirements  
- **Latency**: <0.01 ms per line for composite patterns.  
- **Throughput**: ≥100 000 lines/sec on 8-core CPU.  
- **Memory**: <200 MB for compiled patterns.  
- **Configurability**: thresholds (N occurrences, confidence) adjustable.  
- **Logging**: migration events, validation failures.

***

## 2. SpaCy CNN Log Parser

### 2.1 Overview  
A machine-learning component using SpaCy’s CNN token-classification to extract `EVENT`, `KEY`, and `VALUE` spans from semi-structured logs, powered by subword tokenization (SentencePiece) and trained on CoNLL data.

### 2.2 Goals  
- Accurately tag multi-word events and key–value spans, including implicit patterns (`key(value)`).  
- Handle unlimited unique tokens via subwords.  
- Serve as fallback and initial parser for novel log formats.

### 2.3 Inputs  
- Trained model directory containing `meta.json`, weights, config.  
- SentencePiece model (`.model` & `.vocab`).  
- Raw log line string.

### 2.4 Outputs  
- Dict with optional `"event"` and key–value pairs:  
  ```python
  {"event":"disk error","usage":"75%","temperature":"75°C"}
  ```

### 2.5 Functional Requirements  

1. **Initialization**  
   - Load SentencePiece processor and SpaCy model.  
   - Override `nlp.tokenizer` to:  
     - Normalize numbers (`<NUM>`, `<FLOAT>`, etc.).  
     - Subword tokenize via SentencePiece.  
     - Split parentheses into separate tokens.

2. **Parsing Pipeline**  
   1. Tokenize line → `Doc`.  
   2. `doc.ents` yields spans labeled `EVENT`, `KEY`, `VALUE`.  
   3. Reconstruct:  
      - **Event**: concatenate tokens in first `EVENT` span.  
      - **Key–Value pairs**: iterate spans; each `KEY` matched to next `VALUE`.  

3. **Edge-Case Handling**  
   - Implicit patterns: `key(value)` tokenized as `key`, `(`, `value`, `)`.  
   - Parentheses and separators (`:`, `=`, `-`) treated as `O`.  
   - Numeric placeholders mapped back to original values via stored list.

4. **Confidence Scoring**  
   - Provide confidence score per line (e.g., average token-tag probability) for migration analytics.

5. **Integration with Regex Layer**  
   - Expose `parse_with_confidence()` for adaptive migration.  
   - Only invoked when regex validation fails or no composite pattern matches.

### 2.6 Training Data & Process  

- **CoNLL Format**: one `token LABEL` per line, blank line between sentences.  
- **Labels**: `B-EVENT`, `I-EVENT`, `B-KEY`, `I-KEY`, `B-VALUE`, `I-VALUE`, `O`.  
- **Subword Alignment**: first subword keeps label; subsequent subtokens labeled `O`.  
- **Training Script**:  
  - SentencePiece training on normalized raw logs.  
  - Load CoNLL files (`train.txt`, `dev.txt`, `test.txt`) via spaCy loader.  
  - Train for N epochs with dev evaluation.  
  - Save model to disk.

### 2.7 Non-Functional Requirements  
- **Inference Speed**: ≥10 000 lines/sec on CPU; ≥50 000 lines/sec on GPU batching.  
- **Memory**: <2 GB including embeddings.  
- **Scalability**: support incremental retraining with new annotated data.  
- **Dependencies**: SpaCy v3+, sentencepiece, Python 3.8+.  
- **Logging**: report token-classification losses and dev precision.

***

**Together**, these two parsers form a **self-optimizing hybrid log-parsing system**: SpaCy CNN provides robust initial extraction and pattern discovery, while the regex parser absorbs stable patterns for blazing-fast processing over time.

