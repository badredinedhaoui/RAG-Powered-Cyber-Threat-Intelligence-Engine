# RAG-Powered Cyber Threat Intelligence Engine

A retrieval-augmented generation (RAG) pipeline that takes raw CTI reports and threat descriptions as input and outputs production-ready **Sigma rules**, **YARA rules**, or structured **CTI reports** — automatically.

Built on top of a 52k-document threat intelligence knowledge base, domain-specific cybersecurity embeddings, and a local LLM running through Ollama.

---

## What it does

You paste in a threat report — a phishing campaign, a malware write-up, an incident summary — and the pipeline:

1. Embeds your query using **SecureBERT 2.0**, a BERT model pre-trained specifically on cybersecurity corpora by Cisco AI Research
2. Retrieves the most semantically relevant threat intelligence from a local **ChromaDB** vector store (52k+ CTI examples)
3. Constructs a structured, output-type-specific prompt
4. Sends it to **sylink:32b** running locally via **Ollama**
5. Validates the output and saves it to disk

No API calls to external LLM providers. Everything runs locally.

---

## Output modes

Switch between three output types by changing a single variable:

| Mode | Output | File |
|------|--------|------|
| `OutputType.SIGMA` | Sigma detection rule (YAML) | `sigma_rules/` |
| `OutputType.YARA` | YARA rule with meta, strings, condition | `yara_rules/` |
| `OutputType.CTI_REPORT` | Full structured CTI report in Markdown | `cti_reports/` |

Each mode has its own prompt template, validation logic, and output directory.

---

## Stack

| Component | Technology |
|-----------|-----------|
| Embedding model | `cisco-ai/SecureBERT2.0-base` (768-dim, mean pooling) |
| Vector store | ChromaDB (persistent, cosine similarity) |
| LLM backend | Ollama — `sylink:32b` / `sylink:8b` |
| Dataset | `reloading0101/threat-intelligence-dataset` (~52k CTI examples, 38 categories) |
| Framework | Python, HuggingFace Transformers, PyTorch |

---

## How it works — pipeline breakdown

```
CTI Input Text
      │
      ▼
[SecureBERT2.0 Embedder]  ←  mean pooling over 512 tokens
      │
      ▼
[ChromaDB Query]  ←  top-K cosine similarity search
      │
      ▼
[Prompt Builder]  ←  role + output-specific instructions + context
      │
      ▼
[Ollama / sylink:32b]  ←  temperature=0.1 for deterministic output
      │
      ▼
[Validator + File Save]  →  .yml / .yar / .md
```

The documents in the vector store are structured as `[category] + Instruction + Input + Response` — the category tag helps SecureBERT anchor the embedding to the right cybersecurity subdomain, and the response field (which carries the densest technical content: IOCs, TTPs, CVE details, MITRE mappings) is placed last to benefit from mean pooling weighting.

---

## Dataset

**`reloading0101/threat-intelligence-dataset`** — 52,279 instruction-response pairs covering 38 cybersecurity categories including:
- Malware analysis & reverse engineering
- MITRE ATT&CK techniques and sub-techniques
- CVE exploitation details
- Phishing & social engineering
- Network intrusion & lateral movement
- Incident response procedures

---

## Getting started

### Prerequisites

- Python 3.9+
- [Ollama](https://ollama.com) installed and running
- A HuggingFace account (for dataset access)

### Install dependencies

```bash
pip install datasets pandas tqdm numpy
pip install torch torchvision
pip install transformers accelerate tokenizers
pip install chromadb
pip install requests pyyaml
```

### Pull the model

```bash
ollama pull sylink:32b
ollama serve
```

### Run

Open `Rag_Cybersec_v2.ipynb` and run cells top to bottom.

On first run, the pipeline will:
- Download SecureBERT2.0 (~440MB)
- Load and embed the full 52k dataset into ChromaDB

This takes ~5–10 minutes on GPU or ~30–60 minutes on CPU. The ChromaDB collection persists locally, so subsequent runs skip this step entirely.

To generate a rule, drop your CTI text into `example_cti` and set your output type:

```python
OUTPUT_TYPE = OutputType.SIGMA   # or OutputType.YARA / OutputType.CTI_REPORT

result = cti_to_rule(
    cti_report=example_cti,
    output_type=OUTPUT_TYPE,
    top_k=5,
    save=True
)
```

---

## Project structure

```
├── Rag_Cybersec_v2.ipynb   # Main notebook
├── db_v2/                  # Persistent ChromaDB vector store (generated on first run)
├── sigma_rules/            # Generated Sigma rules (.yml)
├── yara_rules/             # Generated YARA rules (.yar)
└── cti_reports/            # Generated CTI reports (.md)
```

---

## Roadmap

- [ ] Web UI for interactive querying
- [ ] Support for additional local models (Mistral, Llama 3, etc.)
- [ ] Sigma rule syntax validation via `sigma-cli`
- [ ] Batch processing mode for multiple reports
- [ ] MITRE ATT&CK coverage heatmap output

---

## Notes

- The HuggingFace access token in the notebook is for dataset loading — replace it with your own from [hf.co/settings/tokens](https://huggingface.co/settings/tokens)
- ChromaDB is configured with cosine similarity (`hnsw:space: cosine`) which works well for semantic retrieval over mixed-length security documents
- Temperature is intentionally set to `0.1` to keep Sigma/YARA output deterministic and structurally valid

---

## License

MIT
