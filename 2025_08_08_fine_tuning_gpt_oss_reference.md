
# Fine-Tuning GPT-OSS and GPU Memory Planning — Comprehensive Reference

## 1. Fine-Tuning Hardware Requirements

**Question:** Is a single A100 enough for fine-tuning as per the OpenAI GPT-OSS fine-tune transformers tutorial?

**Answer Summary:**
- **Yes, one A100 can be enough**, depending on model size and method (LoRA vs full fine-tuning).
- For LoRA on ≤13B params, an A100-80 GB is comfortable; an A100-40 GB works with reduced batch size, gradient accumulation, and gradient checkpointing.
- **Full FT**:  
  - ≤7–8B possible on 1×80 GB (tight) with checkpointing.  
  - >8–13B or high context: multi-GPU strongly recommended.

---

## 2. GPU Memory Estimation

**Question:** Any good visual documentation to explain memory estimation, covering:
1) Parameter precision (4, 8, 16, 32-bit) and optimizer
2) Memory needs for GPT embeddings, attention, fully-connected, MoE?

**Answer Summary:**
- **Parameter Memory by Precision:**
  - FP32: 4 bytes/param
  - FP16/BF16: 2 bytes/param
  - INT8: 1 byte/param
  - INT4: 0.5 byte/param
- **Training Overhead:**
  - Optimizer states (Adam): +2× param size
  - Gradients: +1× param size
  - Activations: +0.5–1× param size
  - **Rule of Thumb (Adam):** 3–4× param size total VRAM
- **Component Breakdown:**
  - **Embeddings:** vocab_size × embedding_dim
  - **Attention:** grows with sequence length² × dim; FlashAttention can reduce
  - **Feed-forward:** ~4× embedding dim hidden size
  - **MoE:** extra experts increase parameters unless sparsely activated

---

## 3. Why "3–4×" VRAM for Adam Optimizer

**Question:** Adam is 8B per param (2×), so why mention 3–4×?

**Answer Summary:**
- Base params: 1×
- Gradients: +1×
- Adam states (m, v): +2×
- Activations/buffers: +0.5–1×
- **Total:** 3.5–4× parameter size for full fine-tuning with Adam.

---

## 4. AWS & Azure A100/H100 Options for Fine-Tuning

**AWS:**
- **P4d (A100-40 GB)**: 8× GPUs, 320 GB total
- **P4de (A100-80 GB)**: 8× GPUs, 640 GB total
- **P5 (H100-80 GB)**: 8× GPUs, 640 GB total

**Azure:**
- **NDm_A100_v4**: 1/2/4/8× A100-80 GB
- **ND H100 v5**: 1/2/8× H100-80 GB
- **NCads H100 v5**: up to 2× H100 NVL 94 GB

**Decision Tree for Fine-Tuning:**
1. **LoRA/QLoRA:**
   - ≤13B, ≤2k context: 1×80 GB
   - >30B or high context/batch: 2–4×80 GB
2. **Full FT:**
   - ≤8B: 1×80 GB (tight) or 2–4×80 GB
   - >13B: 8×80 GB
3. **Choose GPU Type:**
   - Need speed/high context: H100
   - Price/perf: A100-80 GB

---

## 5. LoRA Target Layers in GPT-OSS

**Question:** Which layers are usually fine-tuned in LoRA on GPT-OSS?

**Answer Summary:**
- **Common Targets:**
  - Q & V projection matrices in attention (most common)
  - Sometimes K projection
  - MLP up/down projection layers
  - LM head (optional)
- **Rare Targets:**
  - Embeddings (only if new vocab/domain)
  - MoE experts (if domain-specific adaptation needed)
- **Why Q+V:** Changes *where* tokens look (Q) and *what* gets passed (V), covering much of domain/style adaptation needs.

**Typical Hugging Face PEFT Config:**
```python
target_modules = ["q_proj", "v_proj", "o_proj", "up_proj", "down_proj"]
lora_r = 8
lora_alpha = 16
lora_dropout = 0.05
bias = "none"
```

---

## 6. Quick Reference Table: VRAM Multipliers

| Component              | Multiplier (vs. param size) |
|------------------------|-----------------------------|
| Parameters             | 1×                          |
| Gradients              | +1×                         |
| Adam states (m, v)     | +2×                         |
| Activations/buffers    | +0.5–1×                      |
| **Total (Adam)**       | **3.5–4×**                   |

---

## 7. Practical Takeaways

- For LoRA, you can fit larger models on fewer GPUs; for full fine-tuning, plan for 3–4× param size in VRAM.
- On AWS, **P4de** (A100-80 GB) and **P5** (H100-80 GB) are go-to choices; on Azure, **NDm_A100_v4** and **ND H100 v5** are top picks.
- For GPT-OSS LoRA: Q+V + MLP up/down projections give strong performance gains with minimal parameters.
