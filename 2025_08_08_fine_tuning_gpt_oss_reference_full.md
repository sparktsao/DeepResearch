
# Fine-Tuning GPT-OSS and GPU Memory Planning — Full Technical Reference

## 1. Fine-Tuning Hardware Requirements

**Q:** Is a single A100 enough for fine-tuning as per the OpenAI GPT-OSS tutorial?

**A:**
- Yes — one A100 (preferably **80 GB**) is enough for LoRA or small full FT.
- The OpenAI tutorial was tested on a **single 80 GB GPU** (H100-80 GB). An A100-80 GB works similarly, just slower.
- A100-40 GB can still work but requires:
  - LoRA or QLoRA
  - Gradient checkpointing
  - Smaller per_device_train_batch_size (1–2) and grad accumulation
  - Lower max_length (e.g., 1024–1536 instead of 2048)
  - ZeRO-3/offload if needed

**Guidelines:**
- Full FT of ~20B params not practical on single 40–80 GB without sharding/offload.
- Full FT of 7–8B possible on 80 GB with bfloat16 + checkpointing.
- LoRA on up to 70B feasible on 80 GB (with 4-bit and small batch).

---

## 2. GPU Memory Estimation — Theory and Practice

**Parameter memory vs. precision:**
- FP32: 4 bytes/param
- FP16/BF16: 2 bytes/param
- INT8: 1 byte/param
- INT4: 0.5 byte/param

**Example — 13B params:**
- FP32 → 52 GB
- FP16 → 26 GB
- INT8 → 13 GB
- INT4 → 6.5 GB

**Training overhead:**
- Gradients (FP16): +1× param size
- Optimizer states (Adam m,v): +2× param size
- Activations/buffers: +0.5–1× param size
- **Rule of thumb:** Full FT VRAM = ~3.5–4× param size with Adam.

**Component-level costs:**
- **Embeddings:** vocab_size × embed_dim
- **Attention:** O(L²) scaling with seq length L; FlashAttention reduces cost
- **FFN layers:** ~4× embed_dim hidden size
- **MoE:** multiple experts; increase params unless sparse activation

---

## 3. Why “3–4×” VRAM for Adam

**Calculation:**
- Params: 1×
- Gradients: +1×
- Adam states (m,v): +2×
- Activations/buffers: +0.5–1×
- Total: ~3.5–4× param size.

**Optimizer variations:**
- SGD w/ momentum: +1× states → ~2.5× total
- Adafactor: 0.5–1× → large savings
- 8-bit optimizers: ~0.25–0.5× states

---

## 4. AWS and Azure A100/H100 Instance Options

### AWS
- **P4d (A100-40 GB)** — 8× A100-40 GB, 320 GB HBM total
- **P4de (A100-80 GB)** — 8× A100-80 GB, 640 GB HBM total
- **P5 (H100-80 GB)** — 8× H100-80 GB, 640 GB HBM3

### Azure
- **NDm_A100_v4** — 1/2/4/8× A100-80 GB
- **ND H100 v5** — 1/2/8× H100-80 GB
- **NCads H100 v5** — up to 2× H100 NVL 94 GB

---

## 5. Fine-Tuning Decision Tree

**Step 1 — Method**
- LoRA/QLoRA → Step 2
- Full FT → Step 3

**Step 2 — LoRA/QLoRA**
- ≤13B, ≤2k ctx: 1×80 GB
- 34–70B or ≥4k ctx/large batch: 2–4×80 GB
- If speed matters → H100

**Step 3 — Full FT**
- ≤8B: 1×80 GB (tight) or 2–4×80 GB
- >13B: 8×80 GB
- Always plan for 3–4× param size VRAM

---

## 6. LoRA Target Layers in GPT-OSS

**Typical LoRA injection points:**
- **Q and V projection matrices** in self-attention (most common)
- Sometimes K projection
- MLP up/down projection layers
- LM head if task-specific vocab

**Why Q and V:**
- Q: Controls *where* tokens attend
- V: Controls *what* info passes through
- This pairing is highly expressive for style/domain adaptation

**Not usually tuned:**
- Embeddings (except for new vocab)
- MoE experts (unless targeted specialization)

**Example PEFT config:**
```python
target_modules = ["q_proj", "v_proj", "o_proj", "up_proj", "down_proj"]
lora_r = 8
lora_alpha = 16
lora_dropout = 0.05
bias = "none"
```

---

## 7. VRAM Multiplier Table

| Component              | Multiplier |
|------------------------|------------|
| Parameters             | 1×         |
| Gradients              | +1×        |
| Adam states (m,v)      | +2×        |
| Activations/buffers    | +0.5–1×    |
| **Total (Adam)**       | 3.5–4×     |

---

## 8. Practical Takeaways

- LoRA fits bigger models into fewer GPUs; full FT needs 3–4× param size VRAM.
- AWS: **P4de** and **P5** are go-to; Azure: **NDm_A100_v4**, **ND H100 v5**.
- For GPT-OSS LoRA, Q+V+MLP up/down offers strong gains with minimal params.
