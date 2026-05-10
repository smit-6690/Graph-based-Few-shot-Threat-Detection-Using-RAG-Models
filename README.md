<div align="center">

# 🛡️ Graph-Based Few-Shot Threat Detection

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org/)
[![GraphSAGE](https://img.shields.io/badge/GNN-GraphSAGE-orange?style=for-the-badge)](https://arxiv.org/abs/1706.02216)
[![RAG](https://img.shields.io/badge/RAG-MITRE%20%2F%20CVE-red?style=for-the-badge)](https://attack.mitre.org/)
[![QLoRA](https://img.shields.io/badge/Fine--Tuning-LoRA%204--Model-purple?style=for-the-badge)](https://arxiv.org/abs/2106.09685)
[![HumanEval](https://img.shields.io/badge/Hard--Case%20Accuracy-100%25-brightgreen?style=for-the-badge)](https://github.com/)
[![Streamlit](https://img.shields.io/badge/UI-Streamlit-ff4b4b?style=for-the-badge&logo=streamlit&logoColor=white)](https://streamlit.io/)

**An end-to-end AI cybersecurity system for few-shot network threat detection, explanation, hard-case correction, and analyst-facing reporting — combining GNN, RAG, LLM reasoning, and a fine-tuned adjudicator layer.**

[Overview](#-overview) • [Architecture](#-architecture) • [Results](#-results) • [Installation](#-installation) • [Usage](#-usage) • [Datasets](#-datasets) • [Notebooks](#-notebooks--phase-summary) • [Future Work](#-future-work)

</div>

---

## 📌 Overview

Modern intrusion detection systems struggle with rare attack classes, imbalanced network traffic, and ambiguous flows that resemble normal enterprise activity. This project addresses those challenges by building a complete threat detection pipeline that can classify network flows, ground decisions in cybersecurity intelligence, explain predictions, and correct difficult edge cases.

The system was developed across 8 phases — from data processing and GNN training, through RAG construction and LLM evaluation, to a four-model LoRA adjudicator and a Streamlit SOC dashboard.

The system integrates:
- **GraphSAGE GNN** over 481K nodes and 3.28M edges for graph-based flow classification
- **FAISS RAG** over 1,151 MITRE ATT&CK and CVE documents for grounded reasoning
- **LLaMA 4 Scout 17B** as the base LLM reasoning layer
- **Four LoRA fine-tuned adjudicators** for hard-case prediction correction
- **Streamlit web UI** for SOC-style review, CSV upload, and natural language input

> **Key Results:** GraphSAGE F1 of **0.989** (vs XGBoost 0.533) on 5-shot episodes · RAG improved accuracy from **17% → 84%** · Adjudicator achieved **100% accuracy on all hard cases**

---

## 🏗 Architecture

The full pipeline routes each network flow through six sequential layers, with the adjudicator activating on low-confidence or hard-case predictions:

```
┌──────────────────────────────────────────────────────────────────────────┐
│               GRAPH-BASED FEW-SHOT THREAT DETECTION PIPELINE             │
│                                                                          │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────┐               │
│  │ 1. NETWORK  │──▶│ 2. GRAPHSAGE │──▶│ 3. RAG         │               │
│  │    FLOW     │   │    GNN       │   │   RETRIEVAL    │               │
│  │             │   │              │   │                │               │
│  │ Raw traffic │   │ Node embed.  │   │ MITRE ATT&CK   │               │
│  │ features    │   │ Edge classif.│   │ + CVE / NVD    │               │
│  └─────────────┘   └──────────────┘   └───────┬────────┘               │
│                                               │                         │
│                                               ▼                         │
│  ┌─────────────┐   ┌──────────────┐   ┌────────────────┐               │
│  │ 6. STREAMLIT│◀──│ 5. FINE-TUNED│◀──│ 4. BASE LLM    │               │
│  │    UI       │   │  ADJUDICATOR │   │   REASONING    │               │
│  │             │   │              │   │                │               │
│  │ SOC review  │   │ LoRA 4-model │   │ LLaMA 4 Scout  │               │
│  │ + export    │   │ hard-case fix│   │ 17B · JSON out │               │
│  └─────────────┘   └──────────────┘   └────────────────┘               │
│                                                                          │
│         Adjudicator activates on low-confidence or hard-case patterns   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

| Layer | Role | Key Technology |
|---|---|---|
| **Network Flow** | Raw feature ingestion (bytes, packets, ports, IPs, duration) | Parquet / Bronze→Gold ETL |
| **GraphSAGE GNN** | Node embedding via neighborhood aggregation; edge classification | PyTorch Geometric, 481K nodes / 3.28M edges |
| **RAG Retrieval** | FAISS vector search over 1,151 MITRE ATT&CK + CVE documents | FAISS IndexFlatL2, all-MiniLM-L6-v2 (384-dim) |
| **Base LLM Reasoning** | Structured JSON threat analysis with MITRE mapping, CVE refs, remediation | LLaMA 4 Scout 17B via Groq |
| **Fine-Tuned Adjudicator** | Hard-case correction via 4 LoRA-tuned models; auto winner selection | PEFT / LoRA, LLaMA 3.1 8B selected |
| **Streamlit UI** | Natural language, CSV, and manual flow input; downloadable SOC results | Streamlit |

---

## 📊 Results

### GraphSAGE vs XGBoost — Few-Shot Evaluation

| Few-Shot Setting | XGBoost F1 | GraphSAGE F1 | Improvement |
|---|---:|---:|---:|
| 1-shot | 0.533 | 0.983 | **+0.450** |
| 5-shot | 0.533 | 0.989 | **+0.456** |
| 10-shot | 0.533 | 0.977 | **+0.444** |

### RAG Impact

| Setting | Accuracy |
|---|---:|
| Without RAG | 17% |
| With RAG | 84% |

> RAG grounded LLM reasoning in MITRE/CVE intelligence, improving classification accuracy by **67 percentage points**.

### Base LLM Comparison

| Model | Quality Score | MITRE Accuracy | JSON Success | Avg Time / Flow |
|---|---:|---:|---:|---:|
| **LLaMA 4 Scout 17B** ✅ | **87.5%** | **68.8%** | **100%** | **0.73s** |
| Qwen 3 32B | 84.8% | N/A | 100% | 4.12s |
| Kimi K2 | 82.6% | N/A | 100% | 1.34s |
| LLaMA 3.1 8B | 76.3% | 38.8% | 100% | 0.53s |
| LLaMA 3.3 70B | Disqualified | N/A | 2.5% | N/A |

### Integrated Pipeline Accuracy — Version Progression

| Version | Main Change | Accuracy |
|---|---|---:|
| V1 | Raw flow features only | 0% |
| V2 | Added behavior hints | 70% |
| V3 | Added src/dst port context | 52% |
| V4 | Fixed packets-per-second artifact logic | 69% |
| V5 | Added C2, DNS, and business traffic rules | 84% |
| **Phase 7** | Added fine-tuned adjudicator | **100% on hard cases** |

### Integrated Pipeline — Evaluation Scenarios

| Evaluation Scenario | Accuracy |
|---|---:|
| Real dataset test | 69.2% |
| 10-flow challenge | 80% |
| 50-flow challenge | 84% |
| Natural language input test | 90% |
| Universal mixed-input test | 76% |
| Hard-case adjudicator test | **100%** |

### Four-Model Adjudicator Comparison

| Model | Size | Base Accuracy | Adjudicator Accuracy | Improvement | JSON Rate | Avg Latency |
|---|---:|---:|---:|---:|---:|---:|
| **LLaMA 3.1 8B** ✅ | 8B | 60% | **100%** | **+40%** | 5/5 | **13.1s** |
| Mistral 7B | 7B | 60% | 100% | +40% | 5/5 | 16.4s |
| Qwen 2.5 3B | 3B | 60% | 80% | +20% | 5/5 | 14.7s |
| Gemma 2 2B | 2B | 60% | 80% | +20% | 5/5 | 15.4s |

> LLaMA 3.1 8B selected as final adjudicator — matched highest hard-case accuracy with lowest latency.

### Hard-Case Correction Results

| Hard Case | True Label | Base Prediction | Final Decision | Result |
|---|---|---|---|---|
| Slow C2 on port 443 | Malware/C2 | Malware/C2 | Malware/C2 | ✅ Confirmed |
| DNS tunneling | Malware/C2 | Recon/DoS | Malware/C2 | 🔧 Corrected |
| Exploit vs Recon | Exploitation | Exploitation | Exploitation | ✅ Confirmed |
| No-GNN C2 | Malware/C2 | Exploitation | Malware/C2 | 🔧 Corrected |
| PPS artifact | Benign | Benign | Benign | ✅ Confirmed |

---

## 🗂 Datasets

| Dataset | Flow Count | Primary Use |
|---|---:|---|
| CIC-IDS2018 | 12.28M | Benign, Recon/DoS, exploitation-style attacks |
| CTU-13 | 13.01M | Botnet and Malware/C2 traffic |
| UNSW-NB15 | 257K | Exploitation, Recon/DoS, benign traffic |

Datasets are processed through Bronze → Silver → Gold → graph-ready layers into a unified schema with Parquet output.

### RAG Knowledge Base

| Component | Value |
|---|---:|
| Total embedded documents | 1,151 |
| MITRE ATT&CK records | 691 |
| CVE records | 460 |
| Embedding model | all-MiniLM-L6-v2 |
| Embedding size | 384 dimensions |
| Vector index | FAISS IndexFlatL2 |
| Retrieved context | Top 3 matches |

---

## ⚙️ Installation

### Prerequisites
- Python 3.10+
- CUDA 11.8+ (recommended: RTX 3090 / 4090 with 24GB VRAM)
- Git

### Setup

```bash
# Clone the repository
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>

# Create virtual environment
python -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Environment Variables

```bash
export GROQ_API_KEY="your_groq_api_key"
export HF_TOKEN="your_huggingface_token"
export BASE="/path/to/DATA_298A"
```

For Colab, mount Google Drive and set:

```python
BASE = "/content/drive/MyDrive/DATA_298A"
```

### Suggested `requirements.txt`

```text
pandas numpy scikit-learn pyarrow matplotlib plotly
faiss-cpu sentence-transformers
torch torch-geometric
transformers accelerate peft trl bitsandbytes datasets
groq huggingface_hub
streamlit networkx python-dotenv
```

---

## 🚀 Usage

### Build the RAG Knowledge Base

```bash
# Run rag_builder.ipynb to generate:
# data_processed/rag_kb/faiss_index.bin
# data_processed/rag_kb/rag_documents.pkl
```

### Train or Load the GNN

```bash
# Run gnn_training.ipynb to generate models/real_graph_v2/
```

### Run the Final v7 Pipeline

```python
from pipeline import analyze_flow_v7

result = analyze_flow_v7(flow)

# Key output fields:
# result["final_attack_family"]   ← use this as the final decision
# result["final_confidence"]
# result["adj_activated"]
# result["adj_changed"]
# result["adj_correction_reason"]
# result["adj_model_used"]
```

### Example Output

```json
{
  "base_attack_family": "Recon/DoS",
  "base_confidence": 0.72,
  "adj_activated": true,
  "adj_attack_family": "Malware/C2",
  "adj_confidence": 0.91,
  "adj_correction_reason": "DNS traffic has unusually high bytes per packet consistent with tunneling rather than normal DNS lookup.",
  "adj_key_indicator": "High bytes-per-packet DNS flow",
  "adj_changed": true,
  "adj_model_used": "LLaMA-3.1-8B-LoRA",
  "final_attack_family": "Malware/C2",
  "final_confidence": 0.91
}
```

### Launch the Streamlit App

```bash
streamlit run models/app.py --server.port 8501
```

In Colab:

```bash
streamlit run /content/app.py --server.port 8501
```

---

## 📥 CSV Input Format

The Streamlit app supports CSV upload with these columns:

**Required:**

```text
bytes · packets · duration_s · dst_port · proto
```

**Optional** (computed from required fields if missing):

```text
src_ip · dst_ip · src_port · pkts_per_s · bytes_per_s · bytes_per_pkt · is_internal_src · is_internal_dst
```

---

## 📓 Notebooks & Phase Summary

| Phase | Notebook / Artifact | Purpose |
|---|---|---|
| Phase 1 | Workbook 1 | Problem setup, datasets, early modeling direction |
| Phase 2 | `episode_generator.ipynb` | Few-shot episode generation (900 episodes) |
| Phase 3 | `rag_builder.ipynb` | MITRE/CVE RAG knowledge base + FAISS index |
| Phase 4 | `llm_comparison.ipynb` | LLM evaluation and model selection |
| Phase 5 | `gnn_training.ipynb` | GraphSAGE training and few-shot evaluation |
| Phase 6 | `pipeline_v5.ipynb` | Integrated GNN + RAG + LLM pipeline |
| Phase 7 | `Phase7_4Model_Adjudicator.ipynb` | Four LoRA adjudicators + winner selection |
| Phase 8 | `models/app.py` | Streamlit UI integration and SOC dashboard |

---

## 🗃 Repository Structure

```text
.
├── data_processed/
│   ├── silver/                       # Cleaned and standardized datasets
│   ├── gold/                         # ML-ready flow features
│   ├── graph/                        # Graph-ready data
│   ├── rag_kb/                       # FAISS index and RAG documents
│   └── adjudicator_train.jsonl       # Phase 7 adjudicator training data
├── models/
│   ├── real_graph_v2/
│   │   ├── real_graph.pt
│   │   ├── gnn_real_graph_best.pt
│   │   ├── ip_to_idx.pkl
│   │   ├── feature_scaler.pkl
│   │   └── column_info.json
│   ├── adjudicator/
│   │   ├── llama_lora_adapter/
│   │   ├── qwen_lora_adapter/
│   │   ├── gemma_lora_adapter/
│   │   └── mistral_lora_adapter/
│   └── app.py                        # Streamlit application
├── notebooks/
│   ├── episode_generator.ipynb
│   ├── rag_builder.ipynb
│   ├── llm_comparison.ipynb
│   ├── gnn_training.ipynb
│   ├── pipeline_v5.ipynb
│   ├── Phase7_4Model_Adjudicator.ipynb
│   └── Final.ipynb
├── reports/
│   └── llm_eval/
│       ├── phase7_model_comparison.csv
│       └── phase7_adjudicator_eval.csv
├── requirements.txt
└── README.md
```

> Large datasets, model checkpoints, FAISS indexes, and LoRA adapters should be stored via Google Drive or Git LFS rather than committed directly.

---

## 🔭 Future Work

- Add temporal multi-flow analysis for repeated beaconing and lateral movement detection.
- Integrate live network stream processing in place of batch-only CSV workflows.
- Enrich telemetry with TCP flags, JA3 fingerprints, and payload signatures.
- Add automated MITRE/CVE knowledge base refresh pipelines.
- Deploy the Streamlit / FastAPI backend on a cloud platform with role-based authentication.
- Integrate SOC ticketing systems for alert-to-ticket automation.

---

## 📎 Citation

```text
Team 3. Graph-Based Few-Shot Threat Detection Using GNN, RAG, LLM, and Fine-Tuned Adjudicator.
San Jose State University, DATA 298B, 2026.
```

---

<div align="center">

**Built by [Smit Ardeshana](https://www.linkedin.com/in/smit-ardeshana-956512220/) · [GitHub](https://github.com/smit-6690)**

*If this project helped you, please consider giving it a ⭐*

</div>
