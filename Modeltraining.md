# Product Requirements Document (PRD)

## Overview  
We need two components for key–value extraction from semi-structured log lines using a SpaCy CNN model with subword tokenization and CoNLL-formatted training data:

1. **Training Pipeline**: Consumes CoNLL `train.txt`, `dev.txt`, `test.txt`, trains a SpaCy CNN token‐classification model for `EVENT`, `KEY`, `VALUE` spans.  
2. **Parser Module**: Loads the trained model, applies it to new log lines, and returns structured key–value pairs (plus an optional event field).

***

## 1. Training Pipeline PRD

### 1.1 Goals  
- Train a SpaCy CNN sequence tagger on CoNLL data.  
- Support subword tokenization via SentencePiece.  
- Produce a serialized model ready for inference.

### 1.2 Inputs  
- Directory `data/spacy_kv/` containing:  
  - `train.txt`, `dev.txt`, `test.txt` in CoNLL format:  
    ```  
    token1 LABEL1  
    token2 LABEL2  
    ...  
      
    tokenA LABELA  
    ...  
    ```
- Raw log corpus `raw_logs.txt` for SentencePiece training.

### 1.3 Outputs  
- SentencePiece model files: `log_tokenizer.model` & `log_tokenizer.vocab`.  
- Trained SpaCy model directory: `models/spacy_kv_cnn/` containing model weights and config.

### 1.4 Functional Requirements  
1. **SentencePiece Training**  
   - Train a subword model (`vocab_size` configurable, default 30 000) on `raw_logs.txt`.  
   - Save `log_tokenizer.model` & `log_tokenizer.vocab`.  
2. **Custom Tokenizer**  
   - Load SentencePiece model.  
   - Define `nlp.tokenizer` to:  
     - Encode text to subword pieces.  
     - Split parentheses `(`, `)` into separate tokens.  
3. **SpaCy Pipeline Setup**  
   - Initialize blank `nlp = spacy.blank("en")`.  
   - Add `ner` pipe with CNN architecture.  
   - Register labels: `B-EVENT`, `I-EVENT`, `B-KEY`, `I-KEY`, `B-VALUE`, `I-VALUE`.  
4. **Data Loader**  
   - Read CoNLL files into memory, preserving token order and labels.  
   - Convert each CoNLL block into a SpaCy `Example` with token‐index spans for non-O labels.  
   - Support splitting data into training/dev/test via files.  
5. **Training Loop**  
   - Use `nlp.begin_training()` to obtain optimizer.  
   - Iterate for configurable epochs (default 10).  
   - Shuffle training data each epoch.  
   - Update model with minibatches of Examples.  
   - Optionally evaluate on dev set each epoch, reporting precision of entity spans.  
6. **Model Serialization**  
   - Save trained model to `models/spacy_kv_cnn/` using `nlp.to_disk()`.  

### 1.5 Non-Functional Requirements  
- **Performance**: Training on 1 M lines with ~30 K subwords should complete within 1–2 hours on GPU.  
- **Modularity**: Components (tokenizer, data loader, training loop) implemented in separate functions or classes.  
- **Configurability**: Paths and hyperparameters (vocab size, epochs, batch size) via CLI flags or config file.  
- **Logging**: Print epoch losses and dev precision metrics.

***

## 2. Parser Module PRD

### 2.1 Goals  
- Load the SpaCy CNN model and subword tokenizer.  
- Process new log lines to extract an optional `event` field and multiple key–value pairs.  
- Handle implicit separators like `key(value)` and explicit separators `:`, `=`, `-` interchangeably.

### 2.2 Inputs  
- Trained model directory: `models/spacy_kv_cnn/`.  
- SentencePiece model: `log_tokenizer.model`.  
- Raw log lines, one per invocation or streamed from file.

### 2.3 Outputs  
- For each input line, a Python dict:  
  ```python
  {
    "event": "<event text>"  # optional, empty if none
    "key1": "value1",
    "key2": "value2",
    ...
  }
  ```

### 2.4 Functional Requirements  
1. **Initialization**  
   - Load SentencePiece processor with `log_tokenizer.model`.  
   - Load SpaCy model: `nlp = spacy.load("models/spacy_kv_cnn/")`.  
   - Override `nlp.tokenizer` to use the same custom subword tokenizer logic.  
2. **Parsing Logic**  
   - Tokenize input line to subword tokens, splitting parentheses.  
   - Call `nlp(text)` to predict `doc.ents`, each `ent` with `start`, `end`, and label (`EVENT`, `KEY`, `VALUE`).  
   - Reconstruct `event` by concatenating tokens in the first `EVENT` span (if any).  
   - Iterate matched `KEY` and `VALUE` spans in document order to build key–value pairs:  
     - For each `KEY` span, find the immediately following `VALUE` span; map `key_text`→`value_text`.  
3. **Edge-Case Handling**  
   - If `key(value)` appears, parentheses are tokenized separately; `KEY` span excludes `(` and `)`, and `VALUE` span includes inner token.  
   - If `VALUE` span missing or out of order, assign `""`.  
4. **API**  
   - Provide a function `parse_line(line: str) -> dict`.  
   - Support bulk parsing: `parse_lines(lines: List[str]) -> List[dict]`.  
   - Optionally, accept async I/O or batching for high throughput.  

### 2.5 Non-Functional Requirements  
- **Throughput**: ≥10 000 lines/sec on CPU, ≥50 000 lines/sec on GPU with batching.  
- **Robustness**: Gracefully handle empty lines or lines with no key–value pairs by returning `{}` or `{"event": <text>}`.  
- **Dependencies**: Only SpaCy v3+, SentencePiece, Python≥3.8.  
- **Logging**: Warnings for lines with ambiguous or missing spans.

***

## Implementation Checklist

- [ ] Install dependencies: `pip install spacy sentencepiece`  
- [ ] Prepare `raw_logs.txt`, `data/spacy_kv/{train,dev,test}.txt`  
- [ ] Execute `train_spacy_kv_cnn.py` with appropriate CLI args  
- [ ] Verify model saved in `models/spacy_kv_cnn/`  
- [ ] Develop `parser.py` that imports and initializes model/tokenizer  
- [ ] Write unit tests for edge cases: `key(value)`, missing values, multi-word events  
- [ ] Benchmark throughput on sample log files  

This PRD ensures a clear, end-to-end design for training and deployment of a SpaCy CNN key-value extraction model using CoNLL training data and subword tokenization.

