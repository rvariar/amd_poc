# S.A.F.E.-T.

## Smart Authentication & Fraud Elimination via Telecom-LLM

### Hackathon Themes

* Telecom-Specific LLM
* Fraud Detection & Financial Crime Intelligence
Large generated artifacts are not committed due to GitHub size limits.

Recreate with:
Build 2 -> generates data + Transformer checkpoint
Build 3 -> generates SFT data + LoRA adapter
Build 4 -> runs demo using generated artifacts
---

# Problem Statement

Telecom operators, banks, and digital service providers rely heavily on OTP-based authentication to protect users from fraud.

However:

* Excessive OTP challenges create user friction.
* Legitimate users experience authentication fatigue.
* Telecom networks process billions of authentication events daily.
* Fraud patterns evolve rapidly through SIM swaps, device changes, behavioral anomalies, and account takeover attempts.
* Security teams often lack explainable AI-driven risk decisions.

The challenge is to reduce unnecessary OTP usage while maintaining strong fraud protection and providing explainable, auditable authentication decisions.

---

# Solution Overview

S.A.F.E.-T. combines:

1. Telecom behavioral sequence modeling
2. Transformer-based fraud prediction
3. Rule-based telecom guardrails
4. Telecom-specialized LLM explanations

to deliver adaptive authentication decisions:

* NO_OTP
* STEP_UP_AUTH
* OTP_REQUIRED

with transparent AI-generated audit explanations.

---

# Architecture

User Login Attempt
|
v
Collect Telecom + Device + Behavioral Signals
|
v
Build User Authentication Sequence
(Last 30 Events)
|
v
Build 2:
Transformer Risk Engine
Predict Fraud Probability
Within Next 5 Events
|
v
Telecom Guardrails Engine
SIM Swap Detection
Device Change Detection
OTP Retry Burst Detection
Failed Authentication Burst Detection
|
v
Adaptive Authentication Decision
NO_OTP
STEP_UP_AUTH
OTP_REQUIRED
|
+-------------------+
|                   |
v                   v
Build 4A            Build 4B
Qwen3-30B-A3B       Telecom LoRA Qwen
General LLM         Fine-Tuned LLM
Explanation         Explanation
|                   |
+---------+---------+
|
v
Audit Report
Recommended Action
Decision Transparency

---

# Project Structure

src/
├── telco_auth/
│   ├── generators/
│   ├── models/
│   ├── build3/
│   ├── serving/
│   ├── rules/
│   ├── llm/
│   ├── schemas/
│   └── utils/

scripts/
reports/
artifacts/
data/

---

# Build 1 – Concept Validation

Purpose:

* Validate adaptive authentication concept.
* Verify fraud signals can drive authentication decisions.
* Demonstrate end-to-end workflow.

Output:

* Rule-based authentication decisions.

---

# Build 2 – Transformer Fraud Prediction Engine

Purpose:

Train a Transformer Encoder model on telecom authentication sequences.

Goal:

Predict whether fraud will occur within the next 5 authentication events.

Technology:

* PyTorch
* ROCm
* AMD GPU Acceleration

Input:

Generated telecom authentication events.

Example Run:

```bash
PYTHONPATH=src python scripts/run_build2_pipeline.py \
  --num-users 100000 \
  --events-per-user 80 \
  --epochs 5
```

Generated Dataset:

```text
100,000 users
80 events/user
8,000,000 authentication events
```

Model:

Custom Transformer Encoder

Features:

* Device consistency
* SIM change
* IMEI change
* Behavior score
* Location jump
* Roaming
* 5G status
* OTP usage
* Authentication failures
* Velocity spikes
* Additional telecom signals

Training Output:

```text
AUC      : 0.7441
Recall   : 0.9337
Precision: 0.5284
F1 Score : 0.6749
```

Artifacts:

```text
artifacts/checkpoints/transformer_risk_model.pt
artifacts/checkpoints/metrics.json
```

Reports:

```text
reports/build2_metrics.json
```

---

# Build 3 – Telecom LLM Fine-Tuning

Purpose:

Fine-tune a telecom-domain language model capable of explaining fraud decisions.

Base Model:

Qwen 0.6B

Fine-Tuning Method:

LoRA (Low Rank Adaptation)

---

## Step 1: Generate Fine-Tuning Dataset

```bash
PYTHONPATH=src python scripts/create_build3_finetune_data.py \
  --events data/raw/auth_events.parquet \
  --num-samples 50000 \
  --out artifacts/llm_finetune/telco_security_sft.jsonl
```

Output:

```text
artifacts/llm_finetune/telco_security_sft.jsonl
```

Dataset Size:

```text
50,000 telecom security examples
~53 MB JSONL
```

---

## Step 2: Train Telecom LoRA Adapter
```text
    date ; PYTHONPATH=src python scripts/train_qwen_lora_skeleton.py \
  --model Qwen/Qwen3-0.6B \
  --train-jsonl artifacts/llm_finetune/telco_security_sft.jsonl \
  --out-dir artifacts/lora_adapters/qwen_telco_security_0_6b \
  --max-samples 20000 \
  --epochs 1 \
  --batch-size 1 \
  --grad-accum 8  ; date
```
Training Runtime Observed:

```text
~57 minutes
```

Training Output:

```text
Train Loss ≈ 0.2915
Epochs = 1
Steps = 2500
```

Artifacts:

```text
artifacts/lora_adapters/qwen_telco_security_0_6b/
```

Key File:

```text
adapter_model.safetensors
```

Purpose:

Teach the LLM:

* Telecom fraud terminology
* Authentication decision explanation
* Security audit language
* Telecom-specific reasoning

---

# Build 4 – Real-Time Adaptive Authentication

Purpose:

Combine:

* Transformer Risk Engine
* Telecom Guardrails
* vLLM Inference
* LLM Audit Generation

into a real-time authentication pipeline.

---

## Start vLLM

General Model:

```bash
vllm serve Qwen/Qwen3-30B-A3B
```

Telecom Fine-Tuned Model:

```bash
vllm serve Qwen/Qwen3-0.6B \
  --lora-modules qwen-telco-security=artifacts/lora_adapters/qwen_telco_security_0_6b
```

---

## Run Build 4

General LLM:

```bash
PYTHONPATH=src python scripts/run_build4_demo.py \
  --base-url http://localhost:8001/v1 \
  --api-key abc-123 \
  --model Qwen3-30B-A3B
```

Telecom Fine-Tuned LLM:

```bash
PYTHONPATH=src python scripts/run_build4_demo.py \
  --base-url http://localhost:8000/v1 \
  --api-key abc-123 \
  --model qwen-telco-security
```

Output:

```json
{
  "decision": "NO_OTP",
  "risk_score": 0.2793,
  "explanation": "...",
  "audit_notes": "...",
  "recommended_action": "..."
}
```

---

# Decision Distribution Analysis

Run:

```bash
PYTHONPATH=src python scripts/build4_decision_distribution.py \
  --sample-users 1000
```

Observed Results:

```json
{
  "NO_OTP": 42.2,
  "STEP_UP_AUTH": 52.5,
  "OTP_REQUIRED": 5.3
}
```

Interpretation:

For 1,000 sampled users:

* 42.2% avoided OTP entirely.
* 52.5% required additional authentication.
* Only 5.3% required full OTP challenge.

This demonstrates potential OTP reduction while preserving security.

Report:

```text
reports/auth_decision_distribution.json
```

---

# GPU Utilization

Observed During Training

Build 2:

```text
GPU Utilization: up to 96%
VRAM Usage: up to 93%
```

Build 3:

```text
GPU Utilization: up to 96%
VRAM Usage: up to 96%
```

Platform:

```text
AMD ROCm
192 GB VRAM Environment
```

---

# Reports

Generated Reports:

```text
reports/build2_metrics.json
reports/auth_decision_distribution.json
reports/final_run_summary.txt
```

---

# Key Innovation

Unlike traditional OTP systems, S.A.F.E.-T. combines:

* Telecom behavioral intelligence
* Transformer fraud prediction
* Telecom guardrails
* Domain-specialized LLM reasoning
* Explainable adaptive authentication

to reduce authentication friction while maintaining fraud protection.

---

# Demo Highlights

1. Generate telecom authentication sequences.
2. Train Transformer fraud prediction model.
3. Fine-tune Telecom Security LLM.
4. Run adaptive authentication.
5. Compare:

   * General Qwen3-30B
   * Telecom LoRA Qwen
6. Generate auditable security explanations.
7. Demonstrate OTP reduction statistics across users.

---

# Author
Ratheesh G Variar

S.A.F.E.-T.
Smart Authentication & Fraud Elimination via Telecom-LLM



root@jupyter-hack-team-2094-260616163225-54f006d4:/workspace/shared/rgv1/telco_auth_full# ls -lrt */*/*/*/*
-rw-r--r-- 1 root root      173 Jun 12 13:08 src/telco_auth/generators/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root      170 Jun 12 13:08 src/telco_auth/schemas/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root     1075 Jun 12 13:08 src/telco_auth/schemas/__pycache__/event_schema.cpython-312.pyc
-rw-r--r-- 1 root root      168 Jun 12 13:08 src/telco_auth/utils/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root      831 Jun 12 13:08 src/telco_auth/utils/__pycache__/seed.cpython-312.pyc
-rw-r--r-- 1 root root      169 Jun 12 13:09 src/telco_auth/models/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root    13105 Jun 12 13:09 src/telco_auth/models/__pycache__/train_transformer.cpython-312.pyc
-rw-r--r-- 1 root root     2857 Jun 12 13:09 src/telco_auth/models/__pycache__/sequence_dataset.cpython-312.pyc
-rw-r--r-- 1 root root     1178 Jun 12 13:09 src/telco_auth/utils/__pycache__/device.cpython-312.pyc
-rw-r--r-- 1 root root    14926 Jun 13 04:12 src/telco_auth/generators/__pycache__/sequence_generator.cpython-312.pyc
-rw-r--r-- 1 root root      169 Jun 13 04:50 src/telco_auth/build3/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root     5187 Jun 13 04:50 src/telco_auth/build3/__pycache__/finetune_data.cpython-312.pyc
-rw-r--r-- 1 root root      168 Jun 13 04:50 src/telco_auth/rules/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root     1737 Jun 13 04:50 src/telco_auth/rules/__pycache__/guardrails.cpython-312.pyc
-rw-r--r-- 1 root root     6147 Jun 13 05:13 src/telco_auth/build3/__pycache__/train_lora_skeleton.cpython-312.pyc
-rw-r--r-- 1 root root     5184 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/README.md
-rw------- 1 root root 20236472 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/scheduler.pt
-rw-r--r-- 1 root root      988 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/scaler.pt
-rw-r--r-- 1 root root    14244 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/rng_state.pth
-rw-r--r-- 1 root root     1090 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25/trainer_state.json
-rw-r--r-- 1 root root      170 Jun 13 05:34 src/telco_auth/serving/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root     3708 Jun 13 05:34 src/telco_auth/serving/__pycache__/risk_pipeline.cpython-312.pyc
-rw-r--r-- 1 root root      166 Jun 13 05:34 src/telco_auth/llm/__pycache__/__init__.cpython-312.pyc
-rw-r--r-- 1 root root     2388 Jun 13 06:16 src/telco_auth/llm/__pycache__/qwen_client.cpython-312.pyc
-rw-r--r-- 1 root root     2432 Jun 13 08:09 src/telco_auth/models/__pycache__/transformer_risk_model.cpython-312.pyc
-rw-r--r-- 1 root root     5184 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/README.md
-rw------- 1 root root 20236472 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/scheduler.pt
-rw-r--r-- 1 root root      988 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/scaler.pt
-rw-r--r-- 1 root root    14244 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/rng_state.pth
-rw-r--r-- 1 root root    40743 Jun 16 12:01 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400/trainer_state.json
-rw-r--r-- 1 root root     5184 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/README.md
-rw------- 1 root root 20236472 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/scheduler.pt
-rw-r--r-- 1 root root      988 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/scaler.pt
-rw-r--r-- 1 root root    14244 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/rng_state.pth
-rw-r--r-- 1 root root    42415 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500/trainer_state.json


root@jupyter-hack-team-2094-260616163225-54f006d4:/workspace/shared/rgv1/telco_auth_full# ls -lrt data/*
-rw-r--r-- 1 root root 104 Jun 16 09:06 data/view_data.py

data/processed:
total 0

data/raw:
total 262292
-rw-r--r-- 1 root root 268583351 Jun 16 11:18 auth_events.parquet
root@jupyter-hack-team-2094-260616163225-54f006d4:/workspace/shared/rgv1/telco_auth_full# ls -lrt artifacts/*/*/*
-rw-r--r-- 1 root root     5184 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/README.md
-rw------- 1 root root 20236472 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 13 05:15 artifacts/lora_adapters/qwen_telco_security_smoke/tokenizer.json
-rw-r--r-- 1 root root     5184 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/README.md
-rw------- 1 root root 20236472 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 16 12:03 artifacts/lora_adapters/qwen_telco_security_0_6b/tokenizer.json

artifacts/lora_adapters/qwen_telco_security_smoke/checkpoint-25:
total 70728
-rw-r--r-- 1 root root     5184 Jun 13 05:15 README.md
-rw------- 1 root root 20236472 Jun 13 05:15 adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 13 05:15 adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 13 05:15 chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 13 05:15 tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 13 05:15 tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 13 05:15 training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 13 05:15 optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 13 05:15 scheduler.pt
-rw-r--r-- 1 root root      988 Jun 13 05:15 scaler.pt
-rw-r--r-- 1 root root    14244 Jun 13 05:15 rng_state.pth
-rw-r--r-- 1 root root     1090 Jun 13 05:15 trainer_state.json

artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2400:
total 70764
-rw-r--r-- 1 root root     5184 Jun 16 12:01 README.md
-rw------- 1 root root 20236472 Jun 16 12:01 adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 16 12:01 adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 16 12:01 chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 16 12:01 tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 16 12:01 tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 16 12:01 training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 16 12:01 optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 16 12:01 scheduler.pt
-rw-r--r-- 1 root root      988 Jun 16 12:01 scaler.pt
-rw-r--r-- 1 root root    14244 Jun 16 12:01 rng_state.pth
-rw-r--r-- 1 root root    40743 Jun 16 12:01 trainer_state.json

artifacts/lora_adapters/qwen_telco_security_0_6b/checkpoint-2500:
total 70768
-rw-r--r-- 1 root root     5184 Jun 16 12:03 README.md
-rw------- 1 root root 20236472 Jun 16 12:03 adapter_model.safetensors
-rw-r--r-- 1 root root     1093 Jun 16 12:03 adapter_config.json
-rw-r--r-- 1 root root     4168 Jun 16 12:03 chat_template.jinja
-rw-r--r-- 1 root root      694 Jun 16 12:03 tokenizer_config.json
-rw-r--r-- 1 root root 11422749 Jun 16 12:03 tokenizer.json
-rw-r--r-- 1 root root     4792 Jun 16 12:03 training_args.bin
-rw-r--r-- 1 root root 40699434 Jun 16 12:03 optimizer.pt
-rw-r--r-- 1 root root     1064 Jun 16 12:03 scheduler.pt
-rw-r--r-- 1 root root      988 Jun 16 12:03 scaler.pt
-rw-r--r-- 1 root root    14244 Jun 16 12:03 rng_state.pth
-rw-r--r-- 1 root root    42415 Jun 16 12:03 trainer_state.json
root@jupyter-hack-team-2094-260616163225-54f006d4:/workspace/shared/rgv1/telco_auth_full# 
