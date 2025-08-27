
# Implementation Document: Regex-Based Log Parser

## 1. Overview  
The **Regex-Based Log Parser** extracts key–value pairs (and optional event spans) at high throughput using a hierarchy of compiled regex patterns. Complex, multi-word key–value patterns are stored in a persistent regex database; generic catch-all regex handles simple pairs; failures route to ML parsers.

***

## 2. Prerequisites  
- Python 3.8+  
- Standard library modules: `re`, `json`, `time`, `collections`  

***

## 3. File Structure  
```
regex_parser/
├── parser.py
├── patterns.json       # Persistent regex DB
├── migration_stats.json
└── README.md
```

***

## 4. patterns.json Format  
```json
{
  "pat_2": {
    "regex": "(?P<k1>[\\w\\s]+?)[:=]\\s*(?P<v1>[^:=]+?)\\s+(?P<k2>[\\w\\s]+?)[:=]\\s*(?P<v2>[^:=]+)",
    "expected_keys": ["k1","k2"]
  },
  "pat_3": {
    "regex": "(?P<k1>[\\w\\s]+?)[:=]\\s*(?P<v1>[^:=]+?)\\s+(?P<k2>[\\w\\s]+?)[:=]\\s*(?P<v2>[^:=]+?)\\s+(?P<k3>[\\w\\s]+?)[:=]\\s*(?P<v3>[^:=]+)",
    "expected_keys": ["k1","k2","k3"]
  }
}
```

***

## 5. Core Implementation (`parser.py`)

```python
import re, json
from typing import Dict, List

# Load and save regex DB
def load_regex_db(path="patterns.json") -> Dict[str, Dict]:
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        return {}

def save_regex_db(db: Dict[str, Dict], path="patterns.json"):
    with open(path, "w") as f:
        json.dump(db, f, indent=2)

class RegexPatternParser:
    def __init__(self, db_path="patterns.json"):
        self.db_path = db_path
        self.regex_db = load_regex_db(db_path)
        self._compile_combined()
        # Generic fallback regex for single key:value
        self.generic_re = re.compile(r"(\w+)\s*[:=]\s*([^\s()]+)")

    def _compile_combined(self):
        parts = [
            f"(?P<{sig}>{data['regex']})"
            for sig, data in self.regex_db.items()
        ]
        pattern = r"^\s*(?:" + "|".join(parts) + r")\s*$"
        self.combined_re = re.compile(pattern) if parts else None

    def add_pattern(self, signature: str, regex: str, expected_keys: List[str]):
        """Add new composite pattern to DB and recompile."""
        self.regex_db[signature] = {
            "regex": regex,
            "expected_keys": expected_keys
        }
        save_regex_db(self.regex_db, self.db_path)
        self._compile_combined()

    def parse(self, line: str) -> Dict[str, str]:
        """Extract using composite patterns, then generic, else empty."""
        # 1) Composite multi-word patterns
        if self.combined_re:
            m = self.combined_re.match(line)
            if m:
                sig = m.lastgroup
                keys = self.regex_db[sig]["expected_keys"]
                extracted = {
                    m.group(f"k{i+1}").strip(): m.group(f"v{i+1}").strip()
                    for i in range(len(keys))
                }
                if self._validate(extracted, keys, line):
                    return extracted
        # 2) Generic fallback
        extracted = {k: v for k, v in self.generic_re.findall(line)}
        if extracted and self._validate(extracted, list(extracted), line):
            return extracted
        # 3) Fallback to ML tomorrow
        return {}

    def _validate(self, extracted: Dict[str, str], expected_keys: List[str], line: str) -> bool:
        # Must have all expected keys
        if not all(k in extracted for k in expected_keys):
            return False
        # No empty values
        if any(not val for val in extracted.values()):
            return False
        # Remove extracted substrings and check for leftovers
        rem = line
        for k, v in extracted.items():
            rem = re.sub(rf"{re.escape(k)}\s*[:=]\s*{re.escape(v)}", "", rem)
        if self.generic_re.search(rem):
            return False
        return True
```

***

## 6. Usage Example

```python
from regex_parser.parser import RegexPatternParser

parser = RegexPatternParser()
# Manual pattern preloaded from patterns.json
result = parser.parse("cpu temp: 65 degrees disk usage: 75 percent")
# If new stable pattern detected via ML, call:
# parser.add_pattern("pat_new", generated_regex, ["cpu temp","disk usage"])
```

***

# Implementation Document: SpaCy CNN Log Parser

## 1. Overview  
The **SpaCy CNN Log Parser** uses SpaCy v3’s CNN-based token-classification with subword tokenization (SentencePiece) to extract `EVENT`, `KEY`, and `VALUE` spans. Serves as ML fallback and pattern generator.

***

## 2. Prerequisites  
- Python 3.8+  
- Install `spacy`, `sentencepiece`  
- Trained SentencePiece model (`.model`, `.vocab`)  
- CoNLL training files: `train.txt`, `dev.txt`, `test.txt`

***

## 3. File Structure  
```
spacy_cnn_parser/
├── train.py
├── parser.py
├── tokenizer.py
├── loader.py
├── data/
│   ├── train.txt
│   ├── dev.txt
│   └── test.txt
└── spiece.model
```

***

## 4. Custom Tokenizer (`tokenizer.py`)

```python
import re, sentencepiece as spm
from spacy.tokens import Doc

class NumericalAwareTokenizer:
    def __init__(self, model_path: str):
        self.sp = spm.SentencePieceProcessor()
        self.sp.load(model_path)
        self.norms = [
            (re.compile(r"\b\d+\.\d+\b"), "<FLOAT>"),
            (re.compile(r"\b\d+%\b"), "<PERCENT>"),
            (re.compile(r"\b0x[0-9A-Fa-f]+\b"), "<HEX>"),
            (re.compile(r"\b\d+\b"), "<NUM>")
        ]

    def normalize(self, text: str) -> str:
        for p, tok in self.norms:
            text = p.sub(tok, text)
        return text

    def __call__(self, nlp, text: str) -> Doc:
        norm = self.normalize(text)
        pieces = self.sp.encode_as_pieces(norm)
        tokens = []
        for p in pieces:
            while p.startswith("("):
                tokens.append("("); p = p[1:]
            trail = ""
            while p.endswith(")"):
                trail = ")" + trail; p = p[:-1]
            if p: tokens.append(p)
            tokens.extend(list(trail))
        return Doc(nlp.vocab, words=tokens)

def create_tokenizer(model_path: str):
    return lambda nlp: NumericalAwareTokenizer(model_path)(nlp, "")
```

***

## 5. Data Loader (`loader.py`)

```python
from spacy.training import Example

def load_conll(path: str):
    data, tokens, labels = [], [], []
    with open(path, encoding="utf8") as f:
        for line in f:
            line = line.strip()
            if not line:
                ents, i = [], 0
                while i < len(labels):
                    if labels[i] != "O":
                        tag = labels[i].split("-",1)[1]
                        start = i; i+=1
                        while i < len(labels) and labels[i].startswith("I-"):
                            i+=1
                        ents.append((start, i, tag))
                    else:
                        i+=1
                data.append((" ".join(tokens), {"entities": ents}))
                tokens, labels = [], []
            else:
                tok, lbl = line.split()
                tokens.append(tok); labels.append(lbl)
    return data
```

***

## 6. Training Script (`train.py`)

```python
import spacy, random, os
from spacy.training import Example
from tokenizer import create_tokenizer
from loader import load_conll

SP_MODEL = "spiece.model"
DATA_DIR = "data"
OUTPUT_DIR = "models/spacy_kv_cnn"
EPOCHS = 10

def main():
    train_data = load_conll(f"{DATA_DIR}/train.txt")
    dev_data   = load_conll(f"{DATA_DIR}/dev.txt")

    nlp = spacy.blank("en")
    nlp.tokenizer = create_tokenizer(SP_MODEL)(nlp)
    ner = nlp.add_pipe("ner")
    for lbl in ["EVENT","KEY","VALUE"]:
        ner.add_label(f"B-{lbl}"); ner.add_label(f"I-{lbl}")

    optimizer = nlp.begin_training()
    for epoch in range(EPOCHS):
        random.shuffle(train_data)
        losses = {}
        for text, ann in train_data:
            doc = nlp.make_doc(text)
            example = Example.from_dict(doc, ann)
            nlp.update([example], sgd=optimizer, losses=losses)
        print(f"Epoch{epoch+1} Loss:{losses['ner']:.3f}")

        # Optional dev evaluation here...

    os.makedirs(OUTPUT_DIR, exist_ok=True)
    nlp.to_disk(OUTPUT_DIR)
    print("Saved model to", OUTPUT_DIR)

if __name__=="__main__":
    main()
```

***

## 7. Inference Parser (`parser.py`)

```python
import spacy, re
from tokenizer import create_tokenizer

class SpaCyKVParser:
    def __init__(self, model_dir: str, sp_model: str):
        self.nlp = spacy.load(model_dir)
        self.nlp.tokenizer = create_tokenizer(sp_model)(self.nlp)
        # Generic kv regex for leftover detection
        self.generic_re = re.compile(r"(\w+)\s*[:=]\s*([^\s()]+)")

    def parse(self, line: str) -> Dict[str,str]:
        doc = self.nlp(line)
        result = {}
        # Event
        ev = [e for e in doc.ents if e.label_=="EVENT"]
        if ev:
            result["event"] = " ".join([t.text for t in ev[0]])
        # KV pairs
        keys = [e for e in doc.ents if e.label_=="KEY"]
        vals = [e for e in doc.ents if e.label_=="VALUE"]
        for k, v in zip(keys, vals):
            result[k.text] = v.text
        return result

    def needs_fallback(self, line: str, extracted: Dict[str,str]) -> bool:
        rem = line
        for k, v in extracted.items():
            rem = re.sub(rf"{re.escape(k)}\s*[:=]\s*{re.escape(v)}", "", rem)
        return bool(self.generic_re.search(rem))
```

***

## 8. Integration & Migration

- **Regex parser** calls `SpaCyKVParser.parse` and checks `needs_fallback` to decide adding new patterns.  
- New patterns generated from stable ML parses are added via `RegexPatternParser.add_pattern(...)`.

***

These documents provide full implementation details for both the **Regex-Based** and **SpaCy CNN** log parsers, including persistent pattern storage, combined regex compilation, validation logic, training workflows, and ML integration.

