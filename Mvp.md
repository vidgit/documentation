# Proof of Concept (PoC) Requirements and Design Document

## 1. Objectives  
Validate the end-to-end viability of the Hybrid High-Throughput Log Parsing System, demonstrating:  
- ≥50 000 lines/sec sustained throughput on an 8-core + GPU node  
- ≥98% extraction accuracy  
- Ordered output guarantee  
- Seamless fallback across Regex → SpaCy CNN → LLM tiers  

## 2. Scope  
-  Implement core components in Python with minimal infrastructure:  
  1. Multi-file async reader with Rust/C++ preprocessing helper  
  2. Ray Serve deployments for ingestion, parsers, and merger  
  3. Sample `patterns.json` with 20 composite regex entries  
  4. ONNX-exported SpaCy CNN NER model and a small LLM (e.g., local GPT-2)  
  5. Simple file-based output sink  

-  Exclude: production-grade observability dashboards, dynamic pattern migration, large-scale cluster deployment, commercial LLM API integration.

## 3. Functional Requirements  

1. File Ingestion & Preprocessing  
   - Watch a folder of log files (150 MB–2 GB each).  
   - Read and chunk lines (10 000 per chunk).  
   - Extract timestamps, normalize to `yyyy-MM-dd-HH-mm-ss.SS`.  
   - Clean lines of noise, perform simple key–value sniffing.  

2. Ray Serve Pipeline  
   - **Ingestion Endpoint**: Accepts preprocessed chunks.  
   - **RegexParser Endpoint**: Applies composite/fallback regex.  
   - **SpaCyParser Endpoint**: Loads ONNX model on GPU, dynamic batching.  
   - **LLMFallback Endpoint**: Local small-scale LLM for JSON extraction.  
   - **Merger Endpoint**: Orders by chunk ID and writes to JSONL file.  

3. Configuration  
   - `config.yaml` for chunk size, batch sizes, resource allocations.  
   - `patterns.json` sample for PoC.  

## 4. Non-Functional Requirements  

- **Performance**:  
  - Regex: ≥100 000 lines/sec per CPU replica  
  - SpaCy ONNX GPU: ≥40 000 lines/sec at 1 000-line batches  
  - End-to-end: ≥50 000 lines/sec  
- **Accuracy**: ≥98% via test suite of 10 000 labeled lines  
- **Reliability**: Fallback rate ≤5% LLM  
- **Maintainability**: Modular code, containerized via Docker  
- **Observability**: Basic logs and counters emitted to console  

## 5. High-Level Architecture  

```text
┌──────────────────┐   mmap + Rust   ┌──────────────────┐     ┌─────────────┐
│ File Watcher     │───helper──chunks►│ Ingestion (Ray)  │──►  │ Router (Ray)│
│ & Async Readers  │                │                  │     └────┬────────┘
└──────────────────┘                             │               │
                                                  ▼               │
                           ┌─────────┐  ┌─────────┐ ┌───────────┐ │
                           │ Regex   │►│ SpaCy   │►│ LLM        │─┘
                           │ Parser  │ │ Parser  │ │ Fallback   │
                           └─────────┘ └─────────┘ └───────────┘
                                 ▲           ▲           ▲
                                 │           │           │
                                 └───Merger (Ray)───────┘
                                          │
                                          ▼
                                     JSONL Output
```

## 6. PoC Timeline (4 Weeks)

| Week | Activities                                         |
|------|----------------------------------------------------|
| 1    | Environment setup, Rust/C++ helper prototype, Docker containers |
| 2    | Ray Serve ingestion, RegexParser endpoint, sample patterns |
| 3    | ONNX export of SpaCy CNN, SpaCyParser endpoint with GPU integration |
| 4    | LLMFallback endpoint, Merger, end-to-end testing, performance tuning |

## 7. Success Metrics  

- **Throughput Test**: ≥50 000 lines/sec sustained over 5 min  
- **Accuracy Test**: ≥98% correct extractions on labeled dataset  
- **Output Order**: No out-of-order records in JSONL  
- **Resource Utilization**: CPU <80% per core, GPU utilization >70%  

## 8. Risks & Mitigations  

- **Serialization Overhead**  
  - Mitigate via Ray object store and shared memory.  
- **Model Load Latency**  
  - Warm-up endpoints; batch initial requests.  
- **File-Watcher Misses**  
  - Use robust inotify/backoff logic; fallback polling.  

***

This PoC design focuses on rapid assembly of the core pipeline, measurable performance tests, and clear criteria for evaluating feasibility before full-scale production development.

