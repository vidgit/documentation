
# Implementation Document: Hybrid High-Throughput Log Parsing System

This document provides detailed implementation guidance for all components:  
1. Regex Parser  
2. SpaCy CNN Parser  
3. LLM Fallback Parser  
4. Asynchronous File Reader  
5. Processing Pool Worker  
6. Priority-Queue Merger  
7. Orchestration Script  

---  

## 1. Regex Parser Component  

### 1.1 File Structure  
```
regex_parser/
├── parser.py
├── patterns.json
├── migration_stats.json
└── utils.py
```

### 1.2 utils.py  
```python
import re

def load_json(path):
    try:
        import json
        return json.load(open(path))
    except:
        return {}

def save_json(obj, path):
    import json
    json.dump(obj, open(path, "w"), indent=2)
```

### 1.3 parser.py  
```python
import re
from .utils import load_json, save_json
from typing import Dict, List

class RegexPatternParser:
    def __init__(self, db_path="patterns.json"):
        self.db_path = db_path
        self.regex_db = load_json(db_path)  # {sig:{regex,expected_keys}}
        self._compile_combined()
        self.generic_re = re.compile(r"(\w+)\s*[:=]\s*([^\s()]+)")

    def _compile_combined(self):
        parts = []
        for sig,data in self.regex_db.items():
            parts.append(f"(?P<{sig}>{data['regex']})")
        pattern = r"^\s*(?:" + "|".join(parts) + r")\s*$"
        self.combined_re = re.compile(pattern) if parts else None

    def add_pattern(self, sig: str, regex: str, expected_keys: List[str]):
        self.regex_db[sig] = {"regex": regex, "expected_keys": expected_keys}
        save_json(self.regex_db, self.db_path)
        self._compile_combined()

    def parse(self, line: str) -> Dict[str,str]:
        # 1) Composite patterns
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
        extracted = {k:v for k,v in self.generic_re.findall(line)}
        if extracted and self._validate(extracted, list(extracted), line):
            return extracted
        return {}

    def _validate(self, extracted: Dict[str,str], keys: List[str], line: str) -> bool:
        if not all(k in extracted for k in keys): return False
        if any(not v for v in extracted.values()): return False
        rem = line
        for k,v in extracted.items():
            rem = re.sub(rf"{re.escape(k)}\s*[:=]\s*{re.escape(v)}", "", rem)
        return not self.generic_re.search(rem)
```

***

## 2. SpaCy CNN Parser Component  

### 2.1 File Structure  
```
spacy_cnn_parser/
├── tokenizer.py
├── loader.py
├── train.py
└── parser.py
```

### 2.2 tokenizer.py  
```python
import re, sentencepiece as spm
from spacy.tokens import Doc

class NumericalAwareTokenizer:
    def __init__(self, model_path):
        self.sp = spm.SentencePieceProcessor(); self.sp.load(model_path)
        self.norms = [(re.compile(r"\b\d+\.\d+\b"),"<FLOAT>"),
                      (re.compile(r"\b\d+%\b"),"<PERCENT>"),
                      (re.compile(r"\b0x[0-9A-Fa-f]+\b"),"<HEX>"),
                      (re.compile(r"\b\d+\b"),"<NUM>")]

    def normalize(self, text: str) -> str:
        for p,t in self.norms: text = p.sub(t, text)
        return text

    def __call__(self, nlp, text: str) -> Doc:
        norm = self.normalize(text)
        pieces = self.sp.encode_as_pieces(norm)
        tokens=[] 
        for p in pieces:
            while p.startswith("("): tokens.append("("); p=p[1:]
            trail="" 
            while p.endswith(")"): trail=")"+trail; p=p[:-1]
            if p: tokens.append(p)
            tokens.extend(list(trail))
        return Doc(nlp.vocab, words=tokens)
```

### 2.3 loader.py  
```python
from spacy.training import Example

def load_conll(path):
    data,toks,labels=[],[],[]
    for line in open(path,encoding="utf8"):
        line=line.strip()
        if not line:
            ents,i=[],0
            while i<len(labels):
                if labels[i]!="O":
                    tag=labels[i].split("-",1)[1];start=i;i+=1
                    while i<len(labels) and labels[i].startswith("I-"):i+=1
                    ents.append((start,i,tag))
                else:i+=1
            data.append((" ".join(toks),{"entities":ents}))
            toks,labels=[],[]
        else:
            tok,lbl=line.split(); toks.append(tok); labels.append(lbl)
    return data
```

### 2.4 train.py  
```python
import spacy, random, os
from spacy.training import Example
from tokenizer import NumericalAwareTokenizer
from loader import load_conll

SP_MODEL="spiece.model";DATA="data";OUT="models/spacy_kv_cnn"
nlp=spacy.blank("en")
nlp.tokenizer = NumericalAwareTokenizer(SP_MODEL)(nlp, "")
ner=nlp.add_pipe("ner")
for lbl in ["EVENT","KEY","VALUE"]:
    ner.add_label(f"B-{lbl}"); ner.add_label(f"I-{lbl}")
opt = nlp.begin_training()
train=load_conll(f"{DATA}/train.txt"); dev=load_conll(f"{DATA}/dev.txt")
for epoch in range(10):
    random.shuffle(train); losses={}
    for text,ann in train:
        doc=nlp.make_doc(text)
        example=Example.from_dict(doc,ann)
        nlp.update([example],sgd=opt,losses=losses)
    print(f"Epoch{epoch+1} Loss{losses['ner']:.3f}")
nlp.to_disk(OUT)
```

### 2.5 parser.py  
```python
import spacy, re
from tokenizer import NumericalAwareTokenizer

class SpaCyKVParser:
    def __init__(self, model_dir, sp_model):
        self.nlp=spacy.load(model_dir)
        self.nlp.tokenizer = NumericalAwareTokenizer(sp_model)(self.nlp,"")
        self.generic_re = re.compile(r"(\w+)\s*[:=]\s*([^\s()]+)")

    def parse(self, line: str) -> Dict[str,str]:
        doc=self.nlp(line);res={}
        evs=[e for e in doc.ents if e.label_=="EVENT"]
        if evs: res["event"]=" ".join([t.text for t in evs[0]])
        keys=[e for e in doc.ents if e.label_=="KEY"]
        vals=[e for e in doc.ents if e.label_=="VALUE"]
        for k,v in zip(keys,vals): res[k.text]=v.text
        return res

    def needs_fallback(self, line, extracted):
        rem=line
        for k,v in extracted.items():
            rem=re.sub(rf"{re.escape(k)}\s*[:=]\s*{re.escape(v)}","",rem)
        return bool(self.generic_re.search(rem))
```

***

## 3. LLM Fallback Parser Component  

### 3.1 File: llm_parser.py  
```python
import re, json, openai
from typing import Dict

class LLMFallbackParser:
    def __init__(self, api_key, model="gpt-4o-mini", retries=2):
        openai.api_key=api_key; self.model=model; self.retries=retries
        self.prompt="""
You extract key-value pairs from log lines. Output ONLY JSON.
Examples:
Input:"system start cpu:65 disk:75"
Output:{{"event":"system start","cpu":"65","disk":"75"}}
Now parse:"{line}"
"""

    def parse(self, line:str)->Dict[str,str]:
        p=self.prompt.format(line=line)
        for _ in range(self.retries+1):
            r=openai.ChatCompletion.create(
                model=self.model,
                messages=[{"role":"user","content":p}],
                temperature=0.0
            )
            txt=r.choices[0].message.content.strip()
            m=re.search(r"\{.*\}",txt,re.DOTALL)
            if m:
                try:
                    obj=json.loads(m.group())
                    return {k:str(v) for k,v in obj.items() if k and v}
                except:
                    continue
        return {}
```

***

## 4. Asynchronous File Reader  

### 4.1 File: reader.py  
```python
import aiofiles, asyncio, json

async def read_chunks(path, queue, size=10000):
    idx=0; chunk=[]
    async with aiofiles.open(path,'r') as f:
        async for line in f:
            obj=json.loads(line)
            raw=obj["line"]
            chunk.append(raw)
            if len(chunk)>=size:
                await queue.put((idx,chunk)); idx+=1; chunk=[]
        if chunk: await queue.put((idx,chunk))
    await queue.put((None,None))
```

***

## 5. Processing Pool Worker  

### 5.1 File: worker.py  
```python
from hybrid_parser import HybridParser  # combines regex, spacy, llm

def process_chunk(args):
    idx, lines = args
    parser = HybridParser()  # init per process
    out=[]
    for ln in lines:
        clean = strip_stopwords(ln)  # user-defined
        out.append(parser.parse_line(clean))
    return idx, out
```

***

## 6. Priority-Queue Merger  

### 6.1 File: merger.py  
```python
import heapq, threading, json

class ChunkMerger:
    def __init__(self, writer):
        self.heap=[]; self.lock=threading.Lock()
        self.next=0; self.writer=writer

    def push(self, idx, data):
        with self.lock:
            heapq.heappush(self.heap,(idx,data))
            self._flush()

    def _flush(self):
        while self.heap and self.heap[0][0]==self.next:
            idx,data=heapq.heappop(self.heap)
            for record in data:
                self.writer.write(json.dumps(record)+"\n")
            self.next+=1
```

***

## 7. Orchestration Script  

### 7.1 File: main.py  
```python
import asyncio, multiprocessing
from reader import read_chunks
from worker import process_chunk
from merger import ChunkMerger
from concurrent.futures import ProcessPoolExecutor

async def main(path,out_path):
    queue=asyncio.Queue(maxsize=10)
    merger=ChunkMerger(open(out_path,'w'))
    # start reader
    reader_task=asyncio.create_task(read_chunks(path,queue))
    # start processing
    loop=asyncio.get_event_loop()
    with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as pool:
        while True:
            idx,chunk=await queue.get()
            if idx is None: break
            fut=loop.run_in_executor(pool, process_chunk,(idx,chunk))
            fut.add_done_callback(lambda f: merger.push(*f.result()))
    await reader_task

if __name__=="__main__":
    import sys
    asyncio.run(main(sys.argv[1],sys.argv[2]))
```

---  

**Notes:**  
- `strip_stopwords` is a user-defined function to remove unwanted tokens.  
- `HybridParser` integrates `RegexPatternParser`, `SpaCyKVParser`, and `LLMFallbackParser`.  
- Adjust chunk size, worker count, and queue size per workload and hardware.

