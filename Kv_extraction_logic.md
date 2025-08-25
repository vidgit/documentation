# Adaptive Key–Value Extraction for Technical Logs

This document summarizes approaches for parsing semi-structured logs containing arbitrary technical tokens. It covers regex-based parsing, sequence-labeling with Flair, async I/O, GPU inference, and live learning loops.

***

## Table of Contents

1. [Regex-Based Parsing (Fast Parser)](#regex-based-parsing)  
2. [SpaCy Matcher Approach](#spacy-matcher-approach)  
3. [Sequence-Labeling with Flair (BiLSTM-CRF)](#sequence-labeling-with-flair)  
4. [Dependency-Parse + Heuristics](#dependency-parse--heuristics)  
5. [Training the Flair Tagger](#training-the-flair-tagger)  
6. [Async I/O + Parallel CPU Inference](#async-io--parallel-cpu-inference)  
7. [GPU-Accelerated Inference](#gpu-accelerated-inference)  
8. [Live Learning Loop for Unknown Tokens](#live-learning-loop)  
9. [Handling Multi-Word vs Single-Word Keys](#handling-multi-word-vs-single-word-keys)  

***

## Regex-Based Parsing

```python
import re

class RegexKeyValueParser:
    def __init__(self):
        # Matches ISO timestamps: 2024-08-25T10:30:15
        self.timestamp_re = re.compile(r'(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})')
        # Simple key:value or key=value
        self.simple_kv_re = re.compile(r'(\w+)[:=]([^\s,]+)')
        # Complex multi-word keys + values up to next key
        self.complex_kv_re = re.compile(
            r'(\w+(?:\s+\w+)*)\s*[:=]\s*([^,\n]+?)(?=\s+\w+\s*[:=]|$)'
        )

    def parse(self, line: str) -> dict:
        result = {}
        # Extract timestamp
        m = self.timestamp_re.match(line)
        if m:
            result['timestamp'] = m.group(1)
            line = line[m.end():].strip()
        # Try simple
        simple = self.simple_kv_re.findall(line)
        if simple:
            for k, v in simple:
                result[k] = self._convert(v)
        else:
            # Fallback to complex
            complex_matches = self.complex_kv_re.findall(line)
            for k, v in complex_matches:
                result[k.strip()] = self._convert(v.strip())
        return result

    @staticmethod
    def _convert(val: str):
        val = val.strip().rstrip(',;')
        if val.lower() in ('true','false'):
            return val.lower() == 'true'
        if val.lower() in ('null','none','nil',''):
            return None
        try:
            return float(val) if '.' in val else int(val)
        except ValueError:
            return val.strip("'\"")
```

***

## SpaCy Matcher Approach

```python
import re
import spacy
from spacy.matcher import Matcher
from spacy.tokens import Doc

nlp = spacy.blank("en")
nlp.tokenizer = lambda text: Doc(nlp.vocab, words=text.split())

matcher = Matcher(nlp.vocab)
token_pattern = r"[A-Za-z0-9_]+"
matcher.add("KEY_VALUE", [[
    {"TEXT": {"REGEX": token_pattern}, "OP": "+"},
    {"TEXT": {"REGEX": r"\s*[:=]\s*"}},
    {"TEXT": {"REGEX": token_pattern}, "OP": "+"}
]])

def split_event_and_remainder(line: str):
    # Locate first complex key:value
    m = re.search(r'(\w+(?:\s+\w+)*)\s*[:=]', line)
    if not m:
        return line.strip(), ""
    pos = m.start()
    return line[:pos].strip(), line[pos:].strip()

def extract_kv_spacy(line: str) -> dict:
    event, data = split_event_and_remainder(line)
    out = {}
    if event:
        out["event"] = event
    doc = nlp(data)
    for _, start, end in matcher(doc):
        span = doc[start:end].text
        if ":" in span:
            k, v = span.split(":",1)
        else:
            k, v = span.split("=",1)
        out[k.strip()] = v.strip()
    return out

# Example
print(extract_kv_spacy("system update failed error id:0xabb14c error message:could not connect"))
```

***

## Sequence-Labeling with Flair

### Training Data Format (CoNLL)

```
system O
id B-KEY
: O
12334 B-VALUE

l2 O
rx B-KEY
: O
100 B-VALUE
```

### Training Script

```python
from flair.data import Corpus
from flair.datasets import ColumnCorpus
from flair.embeddings import WordEmbeddings, CharacterEmbeddings, StackedEmbeddings
from flair.models import SequenceTagger
from flair.trainers import ModelTrainer

# 1) Prepare corpus
columns = {0:'text',1:'ner'}
data_folder = 'resources/taggers/log-kv-ner'
corpus = ColumnCorpus(data_folder, columns,
                      train_file='train.txt',
                      dev_file='dev.txt',
                      test_file='test.txt')

tag_dict = corpus.make_tag_dictionary(tag_type='ner')
embeddings = StackedEmbeddings([
    WordEmbeddings('glove'),
    CharacterEmbeddings()
])

# 2) Initialize tagger
tagger = SequenceTagger(hidden_size=128,
                        embeddings=embeddings,
                        tag_dictionary=tag_dict,
                        tag_type='ner',
                        use_crf=True)

# 3) Train
trainer = ModelTrainer(tagger, corpus)
trainer.train(data_folder,
              learning_rate=0.1,
              mini_batch_size=32,
              max_epochs=10)
```

### Inference & Learning Loop

```python
from flair.data import Sentence, Token
from flair.models import SequenceTagger
import re

UNKNOWN_LOG = 'unknown_tokens.txt'
tok_re = re.compile(r'[A-Za-z0-9_]+')

def extract_kv(line, model_folder):
    # Load once
    if not hasattr(extract_kv, 'tagger'):
        extract_kv.tagger = SequenceTagger.load(f"{model_folder}/final-model.pt")
    tagger = extract_kv.tagger

    sent = Sentence()
    for w in line.split():
        sent.add_token(Token(w))
    tagger.predict(sent)

    tokens = [t.text for t in sent]
    labels = [t.get_tag('ner').value for t in sent]
    out = {}
    i=0
    while i<len(tokens):
        if labels[i]=='B-KEY':
            key=[tokens[i]]; i+=1
            while i<len(tokens) and labels[i]=='I-KEY':
                key.append(tokens[i]); i+=1
            if i<len(tokens) and tokens[i] in {':','='}: i+=1
            val=[]
            if i<len(tokens) and labels[i]=='B-VALUE':
                val=[tokens[i]]; i+=1
                while i<len(tokens) and labels[i]=='I-VALUE':
                    val.append(tokens[i]); i+=1
            out[' '.join(key)]=' '.join(val)
        else:
            if labels[i]=='O' and tok_re.fullmatch(tokens[i]):
                with open(UNKNOWN_LOG,'a') as f: f.write(tokens[i]+'\n')
            i+=1
    return out
```

***

## Dependency-Parse + Heuristics

```python
import spacy
nlp = spacy.blank("en")

def extract_by_dep(line):
    doc = nlp(line)
    out={}
    heads={'status','error','timeout','time','rx'}
    for token in doc:
        if token.text.lower() in heads:
            for child in token.children:
                if child.dep_ in ('nummod','amod','dobj','attr'):
                    out[token.text] = child.text
    return out

print(extract_by_dep("l2 rx: 100"))
print(extract_by_dep("computeFFT size 2048 windowType Hamming"))
```

***

## Async I/O + Parallel CPU Inference

```python
import asyncio, aiofiles, json
from concurrent.futures import ThreadPoolExecutor

MODEL_FOLDER = 'resources/taggers/log-kv-ner'
async def process_chunk(lines, executor):
    loop=asyncio.get_event_loop()
    tasks=[loop.run_in_executor(executor, extract_kv, l, MODEL_FOLDER) for l in lines]
    return await asyncio.gather(*tasks)

async def stream_parse(path, chunk_size=1000):
    exec=ThreadPoolExecutor(max_workers=4)
    async with aiofiles.open(path,'r') as f:
        buf=[]
        async for r in f:
            l=r.strip()
            if l:
                buf.append(l)
            if len(buf)>=chunk_size:
                for rec in await process_chunk(buf,exec):
                    yield rec
                buf=[]
        if buf:
            for rec in await process_chunk(buf,exec):
                yield rec

async def main():
    async with aiofiles.open('out.jsonl','w') as out:
        async for rec in stream_parse('large.log'):
            await out.write(json.dumps(rec)+'\n')

asyncio.run(main())
```

***

## GPU-Accelerated Inference

```python
import asyncio, aiofiles, json
from concurrent.futures import ProcessPoolExecutor

from train_and_use_tagger import extract_kv

MODEL_FOLDER='resources/taggers/log-kv-ner'
# Warm GPU in worker init
def init_gpu():
    extract_kv("warmup: true", MODEL_FOLDER)

def gpu_infer(lines):
    return [extract_kv(l, MODEL_FOLDER) for l in lines]

async def stream_gpu(path, chunk_size=1000):
    exec=ProcessPoolExecutor(max_workers=1, initializer=init_gpu)
    async with aiofiles.open(path,'r') as f:
        buf=[]
        loop=asyncio.get_event_loop()
        async for r in f:
            l=r.strip()
            if l: buf.append(l)
            if len(buf)>=chunk_size:
                fut=loop.run_in_executor(exec,gpu_infer,buf.copy())
                buf=[]
                for rec in await fut: yield rec
        if buf:
            fut=loop.run_in_executor(exec,gpu_infer,buf.copy())
            for rec in await fut: yield rec

async def main():
    async with aiofiles.open('out_gpu.jsonl','w') as out:
        async for rec in stream_gpu('large.log'):
            await out.write(json.dumps(rec)+'\n')

asyncio.run(main())
```

***

## Live Learning Loop

```python
# part of extract_kv: logs O-tagged tokens for later annotation
if labels[i]=='O' and tok_re.fullmatch(tokens[i]):
    with open('unknown_tokens.txt','a') as f:
        f.write(tokens[i]+'\n')
```

***

## Handling Multi-Word vs Single-Word Keys

- **Tokenizer + Sequence-Labeler** learns context: same token can be B-KEY in one context, I-KEY or O in another.  
- Regex alone cannot decide; sequence-labeler uses BiLSTM + character embeddings for open vocabulary.  
- Dependency approach grabs head nouns and modifiers based on parse relations.

***

**Copy this raw MD into your own `.md` file for reference.**

