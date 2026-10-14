<div align="center"> 

# 🛡️ CyberSec-BugBounty-AI-Assistant

**Defensive Code LLM Advisor** — Domain-adapted LLM for bug bounty research, vulnerability root-cause analysis, and secure code patch generation.

[![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)  
[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](#)  
[![Gradio](https://img.shields.io/badge/Gradio-4.x-orange)](#)  

</div>

## 1. Executive summary & purpose

Project Name: CyberSec-BugBounty-AI-Assistant (Defensive Code LLM Advisor)

Domain: Applied AI Engineering / Cybersecurity (Bug Bounty Research & Vulnerability Remediation)

Core function: a domain-adapted decoder-only LLM application engineered to perform root-cause analysis on web and system vulnerabilities (OWASP Top 10, XSS, SQLi, OS Command Injection) and synthesize secure, production-ready defensive code patches (Python, JavaScript, C/C++). The notebook demonstrates dataset ingestion, instruction-tuning style prompting, safety- and sanitization-oriented persona steering, and a Gradio-based interactive UI for exploration and patch generation.

---

## 2. Architectural properties & mathematical specifications

This project uses a Qwen2.5-Coder (decoder-only transformer) design and incorporates several modern LLM engineering choices. The following summarizes the model building blocks and the mathematical intuition used in the notebook.

A. Root Mean Square Normalization (RMSNorm)

RMSNorm replaces standard Layer Normalization by omitting mean-centering while preserving scale invariance and reducing compute:

$$\text{RMS}(x) = \sqrt{\tfrac{1}{d} \sum_{i=1}^d x_i^2 + \epsilon}, \qquad \bar{x}_i = \frac{x_i}{\text{RMS}(x)} \cdot \gamma_i$$

where d is the hidden dimension, $\epsilon=10^{-6}$, and $\gamma$ is a learnable scale vector.

B. Rotary Positional Embeddings (RoPE)

RoPE encodes absolute token positions by applying complex-plane rotations to paired components of query and key vectors so that attention depends on relative distances:

$$e^{im\theta_i} = \cos(m\theta_i) + i\sin(m\theta_i), \qquad \theta_i = 10000^{-2(i-1)/d_{\text{head}}}$$

This yields position-dependent rotations that preserve relative positioning in dot-products.

C. Grouped Query Attention (GQA) & KV Caching

- GQA groups multiple query heads ($N_Q$) to share a smaller number of key/value heads ($N_{KV}$), lowering memory bandwidth for QK^T operations.  
- KV cache: Keys and Values computed during autoregressive decoding are appended into a static cache (at start_pos) to avoid recomputing O(N^2) attention for the prompt history.

D. SwiGLU Activation in Feed-Forward Blocks

The MLP uses a gated SiLU (SwiGLU) non-linearity:

$$\text{FFN}(x) = \left(\text{SiLU}(xW_1) \otimes (xW_3)\right) W_2, \quad \text{SiLU}(z)=z\cdot\sigma(z)$$

Hidden dimensions are padded to multiples of 256 for CUDA memory alignment in the notebook implementations.

---

## 3. Data pipeline & domain alignment

- Dataset source: Trendyol/Trendyol-Cybersecurity-Instruction-Tuning-Dataset (referenced in the notebook).  
- Corpus size: ~53,201 structured security instruction-response pairs.  

Schema (instruction-style examples):
- system: persona and global steering (defensive security analyst).  
- user: bug bounty queries, vulnerable snippets, attacker vectors.  
- assistant: root-cause explanations and remediated, production-oriented patches.

System persona steering ensures outputs emphasize root cause analysis, parameterized secure code, input sanitization, least privilege patterns, and guidance suitable for professional remediation.

---

## 4. Hardware constraints & memory optimization (designed for Google Colab T4)

This notebook is engineered to run on free NVIDIA T4 GPUs (16 GB VRAM). Key strategies used to fit model inference and sampling into 16 GB:

- FP16 weights: model parameters are loaded into torch.float16 to halve VRAM usage and leverage Tensor Cores.  
- Inference-only context: model runs inside @torch.no_grad() and avoids storing gradients.  
- Periodic cleanup: calls to gc.collect() and torch.cuda.empty_cache() are used after large data transfers or dataset downloads.  
- KV caching & grouped attention: minimizes repeated compute and bandwidth during autoregressive generation.  

Recommended generation defaults in the notebook:
- Temperature T = 0.6  
- Top-p (nucleus) = 0.9  
- max_new_tokens = 512

Note: these settings trade determinism for creative but focused repairs; adjust for your use-case.

---

## 5. Full software & system stack

- Language: Python 3.10+  
- Deep learning engine: PyTorch 2.0+ (CUDA backend)  
- Model & dataset tooling: transformers (or Qwen-equivalent wrappers), datasets, accelerate  
- Base model: Qwen2.5-Coder-1.5B-Instruct (decoder-only)  
- UI: Gradio 4.x (notebook exposes a Gradio interface and shows public-tunnel deployment examples)  
- Versioning & docs: Git + GitHub (this README and embedded Mermaid pipeline diagrams)

---

## Quickstart — run the notebook (Colab or local)

Easiest: open in Google Colab and run cells (recommended for free GPU):

[Open in Colab](https://colab.research.google.com/github/muhammadhurres/CyberSec-BugBounty-AI-Assistant/blob/main/CyberSec_BugBounty_AI_Assistant.ipynb)

Local (conda / venv):

```bash
# create environment
python -m venv .venv && source .venv/bin/activate
pip install --upgrade pip
pip install torch torchvision --extra-index-url https://download.pytorch.org/whl/cu117  # pick the CUDA wheel matching your system
pip install transformers datasets accelerate gradio sentencepiece

# open the notebook
jupyter notebook CyberSec_BugBounty_AI_Assistant.ipynb
```

Notes & required resources:
- GPU strongly recommended (T4-class or better).  
- Sufficient disk space for the referenced dataset (the notebook downloads a ~195MB dataset artifact).  

---

## Notebook structure (high-level)

- Data ingestion & preprocessing — dataset download, JSONL parsing, instruction format checks.  
- Model loading — FP16, grouped attention & kv-cache-aware loading of Qwen2.5-Coder-1.5B-Instruct.  
- Prompt templates & persona steering — system/user/assistant structured prompts used for root-cause + patch synthesis.  
- Inference & generation — sampling with the tuned generation parameters.  
- Gradio UI — small web interface for pasting vulnerable snippets, reviewing the assistant's analysis, and copying remediated patches.

```mermaid
flowchart TD
  A[Dataset: Trendyol Cybersecurity Instructions] --> B[Preprocessing & Schema Validation]
  B --> C[Prompt Templates (system/user/assistant)]
  C --> D[Model (Qwen2.5-Coder-1.5B-Instruct) — FP16, GQA, KV-Cache]
  D --> E[Autoregressive Generation (T=0.6, top-p=0.9)]
  E --> F[Gradio UI: Analysis + Defensive Patch Output]
```

---

## Security & Responsible use

This tool is built for defensive purposes: analyzing vulnerabilities and generating secure patches. Do not use it to create or optimize exploit code. Outputs are generated by a statistical model and must be reviewed by a qualified security engineer before deployment. The project includes instruction-steering to reduce harmful generations, but it is NOT a replacement for human review.

---

## License

This project is licensed under the MIT License — see LICENSE.

---

## Try asking

- "Where in the notebook are the vulnerability-analysis prompt templates defined?"  
- "How is the Trendyol cybersecurity dataset validated and converted to instruction examples?"  
- "What changes are needed to swap Qwen2.5-Coder for another decoder-only model (weights + tokenizer differences)?"
