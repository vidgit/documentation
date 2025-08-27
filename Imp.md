# Implementation Document: Regex-Based Log Parser

## 1. Overview  
This component provides high-speed extraction of key–value pairs (and optional event spans) from log lines using compiled regular expressions. Patterns automatically migrate from the ML parser to regex when stable.

## 2. Prerequisites  
- Python 3.8+  
- No external ML dependencies  
- Standard library (`re`, `json`, `time`, `collections`)

## 3. File Structure  
```
regex_parser/
├── parser.py
├── patterns.json       # Registered pattern signatures, regex strings, expected_keys
├── migration_stats.json# Learned pattern statistics
└── README.md
```

## 4. patterns.json Format  
```jsonc
{
  "pat_ab": {
    "regex": "^(?P<a>\\w+)[:=](?P<a_val>\\S+)\\s+(?P<b>\\w+)[:=](?P<b_val>\\S+)$",
    "expected_keys": ["a","b"]
  },
  "pat_abc": { ... }
}
```

## 5. Core Classes & Functions

### 5.1 RegexPatternParser  
File: `parser.py`

```python
import re, json
from typing import Dict, List

class RegexPatternParser:
    def __init__(self, patterns_path: str):
        self.patterns_path = patterns_path
        self._load_patterns()
        self._compile_combined_regex()

    def _load_patterns(self):
        with open(self.patterns_path) as f:
            raw = json.load(f)
        self.patterns = {}
        self.expected_keys = {}
        for sig, data in raw.items():
            self.patterns[sig] = data["regex"]
            self.expected_keys[sig] = data["expected_keys"]

    def _compile_combined_regex(self):
        # Build alternation with named groups
        parts = [f"(?P<{sig}>{pat})" for sig, pat in self.patterns.items()]
        self.combined = re.compile(r"^\s*(?:" + "|".join(parts) + r")\s*$")

    def parse(self, line: str) -> Dict[str,str]:
        # 1) Try composite patterns
        m = self.combined.match(line)
        if m:
            sig = m.lastgroup
            keys = self.expected_keys[sig]
            result = {}
            for key in keys:
                val = m.group(key) or m.group(f"{key}_val")
                if val is None:
                    return {}  # missing expected
                result[key] = val
            return result

        # 2) Generic single-key fallback
        return self._generic_parse(line)

    def _generic_parse(self, line: str) -> Dict[str,str]:
        # find all key:value or key=value
        result = {}
        for m in re.finditer(r"(\w+)\s*[:=]\s*([^\s()]+)", line):
            result[m.group(1)] = m.group(2)
        return result
```

### 5.2 Pattern Migration Hooks  
Append to `parser.py`:

```python
from collections import defaultdict

class AdaptiveRegexParser(RegexPatternParser):
    def __init__(self, patterns_path, stats_path, threshold=100):
        super().__init__(patterns_path)
        self.stats_path = stats_path
        self.threshold = threshold
        self.stats = defaultdict(lambda: {"count":0, "examples":[]})
        self._load_stats()

    def _load_stats(self):
        try:
            with open(self.stats_path) as f:
                self.stats.update(json.load(f))
        except FileNotFoundError:
            pass

    def record_example(self, signature: str, line: str):
        st = self.stats[signature]
        st["count"] += 1
        if len(st["examples"]) < 10:
            st["examples"].append(line)
        if st["count"] == self.threshold:
            self._generate_and_register(signature, st["examples"])

    def _generate_and_register(self, sig, examples: List[str]):
        # call out to ML migration component …
        regex_pattern, expected_keys = generate_regex_from_examples(examples)
        # update patterns.json
        self.patterns[sig] = regex_pattern
        self.expected_keys[sig] = expected_keys
        self._save_patterns()
        self._compile_combined_regex()

    def _save_patterns(self):
        data = {
            sig: {"regex": pat, "expected_keys": self.expected_keys[sig]}
            for sig, pat in self.patterns.items()
        }
        with open(self.patterns_path, "w") as f:
            json.dump(data, f, indent=2)
        with open(self.stats_path, "w") as f:
            json.dump(self.stats, f, indent=2)
```

## 6. Usage  

```python
from regex_parser.parser import AdaptiveRegexParser

parser = AdaptiveRegexParser("patterns.json", "migration_stats.json")
line = "a:5 b:6 c:7"
result = parser.parse(line)
if not result:
    # fallback to ML parser
    pass
```

***

# Implementation Document: SpaCy CNN Log Parser

## 1. Overview  
This module uses SpaCy v3’s CNN-based token-classification to extract `EVENT`, `KEY`, and `VALUE` spans from log lines. It employs SentencePiece subword tokenization and number normalization.

## 2. Prerequisites  
- Python 3.8+  
- pip install `spacy sentencepiece`  
- spaCy v3  
- Trained SentencePiece model (`.model`, `.vocab`)  
- CoNLL-formatted training files

## 3. File Structure  
```
spacy_cnn_parser/
├── train.py           # training script
├── parser.py          # inference API
├── data/
│   ├── train.txt      # CoNLL token-label
│   ├── dev.txt
│   └── test.txt
├── raw_logs.txt       # for SentencePiece training
├── spiece.model
├── spiece.vocab
└── models/
    └── spacy_kv_cnn/
```

## 4. train.py: Model Training

```python
import spacy, sentencepiece as spm, random
from spacy.tokens import Doc
from spacy.training import Example

# 1. Train SentencePiece
spm.SentencePieceTrainer.train(
    input="raw_logs.txt", model_prefix="spiece", vocab_size=30000,
    character_coverage=0.995, model_type="unigram"
)

# 2. Custom tokenizer factory
def create_tokenizer(nlp):
    sp = spm.SentencePieceProcessor(); sp.load("spiece.model")
    def tokenizer(text):
        pieces = sp.encode_as_pieces(text)
        tokens=[]
        for p in pieces:
            # split parens
            while p.startswith("("):
                tokens.append("("); p=p[1:]
            trailing=""; 
            while p.endswith(")"):
                trailing=")"+trailing; p=p[:-1]
            if p: tokens.append(p)
            tokens.extend(list(trailing))
        return Doc(nlp.vocab, words=tokens)
    return tokenizer

# 3. Load CoNLL data
def load_conll(path):
    data=[]; tokens=[]; labels=[]
    for line in open(path, encoding="utf8"):
        line=line.strip()
        if not line:
            # build entities
            ents=[]; i=0
            while i<len(labels):
                lbl=labels[i]
                if lbl!="O":
                    tag=lbl.split("-")[1]; start=i; i+=1
                    while i<len(labels) and labels[i].endswith(tag):
                        i+=1
                    ents.append((start,i,tag))
                else: i+=1
            data.append((" ".join(tokens), {"entities":ents}))
            tokens=[]; labels=[]
        else:
            tok,lbl=line.split(); tokens.append(tok); labels.append(lbl)
    return data

train_data = load_conll("data/train.txt")
dev_data   = load_conll("data/dev.txt")

# 4. Create and configure pipeline
nlp=spacy.blank("en")
nlp.tokenizer = create_tokenizer(nlp)
ner=nlp.add_pipe("ner")
for lbl in ["EVENT","KEY","VALUE"]:
    ner.add_label(f"B-{lbl}"); ner.add_label(f"I-{lbl}")

# 5. Training loop
optimizer = nlp.begin_training()
for epoch in range(10):
    random.shuffle(train_data)
    losses={}
    for text, ann in train_data:
        doc=nlp.make_doc(text)
        example = Example.from_dict(doc, ann)
        nlp.update([example], sgd=optimizer, losses=losses)
    print(f"Epoch {epoch}: Loss {losses['ner']:.3f}")

    # Optional dev evaluation
    correct=total=0
    for text, ann in dev_data:
        doc=nlp(text)
        preds=[(e.start,e.end,e.label_) for e in doc.ents]
        for span in ann["entities"]:
            total+=1
            if span in preds: correct+=1
    print(f" Dev Precision: {correct/total:.3f}")

# 6. Save model
nlp.to_disk("models/spacy_kv_cnn")
```

## 5. parser.py: Inference

```python
import spacy, re, sentencepiece as spm
from spacy.tokens import Doc

class SpaCyKVParser:
    def __init__(self, model_path, sp_model="spiece.model"):
        self.nlp = spacy.load(model_path)
        self.sp = spm.SentencePieceProcessor(); self.sp.load(sp_model)
        self._override_tokenizer()

    def _override_tokenizer(self):
        def tokenize(text):
            # number normalization
            text = re.sub(r"\b\d+\b","<NUM>",text)
            text = re.sub(r"\b\d+\.\d+\b","<FLOAT>",text)
            # subword & parens
            pieces = self.sp.encode_as_pieces(text)
            tokens=[] 
            for p in pieces:
                while p.startswith("("):
                    tokens.append("("); p=p[1:]
                trailing=""
                while p.endswith(")"):
                    trailing=")"+trailing; p=p[:-1]
                if p: tokens.append(p)
                tokens.extend(list(trailing))
            return Doc(self.nlp.vocab, words=tokens)
        self.nlp.tokenizer = tokenize

    def parse(self, line: str) -> dict:
        doc = self.nlp(line)
        result = {}
        # extract event
        events = [e for e in doc.ents if e.label_=="EVENT"]
        if events:
            result["event"] = " ".join([t.text for t in events[0]])
        # extract kv
        key=None
        for ent in doc.ents:
            if ent.label_=="KEY":
                key = " ".join([t.text for t in ent])
            elif ent.label_=="VALUE" and key:
                result[key] = " ".join([t.text for t in ent])
                key=None
        return result

    def parse_with_confidence(self, line: str):
        # Estimate confidence as average tag probability
        doc = self.nlp(line)
        confidences = []
        for token in doc:
            tag = token.ent_iob_ + "-" + token.ent_type_ if token.ent_iob_!="O" else "O"
            prob = token.ent_dist.prob(token.ent_type_) if token.ent_iob_!="O" else 1.0
            confidences.append(prob)
        return self.parse(line), sum(confidences)/len(confidences)
```

***

These implementation documents provide all necessary details—file layouts, code structure, and configuration—to build and deploy both the **Regex-Based** and **SpaCy CNN** log parsers as described.

