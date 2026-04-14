---
title: "Fine-Tuning โมเดล AI"
type: concept
tags: [fine-tuning, lora, qlora, rlhf, sft, training, llm]
sources: [raw/notes/ai-engineering/05_fine_tuning.md]
related: [wiki/concepts/llm-large-language-model, wiki/concepts/rag-retrieval-augmented-generation, wiki/concepts/prompt-engineering]
created: 2026-04-14
updated: 2026-04-14
---

## สรุปสั้น

Fine-Tuning คือการนำ Pre-trained Model มา Train เพิ่มเติมด้วยข้อมูลเฉพาะทาง เพื่อให้โมเดลเก่งงานนั้นเป็นพิเศษ — เหมาะเมื่อต้องการ style/tone เฉพาะ หรือ task ซ้ำๆ ที่ Prompt Engineering ไม่พอ

## อธิบาย

### Fine-Tuning คืออะไร

```
Pre-trained Model (รู้ทุกเรื่องแต่ไม่เชี่ยวชาญอะไร)
          +
ข้อมูลเฉพาะทาง (เช่น เอกสารกฎหมาย, บทสนทนาการแพทย์)
          ↓
Fine-tuned Model (เชี่ยวชาญเรื่องนั้นเป็นพิเศษ)
```

### เมื่อไรควรใช้ Fine-tuning

**ใช้เมื่อ:**
- ต้องการ Style/Tone เฉพาะ เช่น "ตอบแบบทนายความกฎหมายไทยเสมอ"
- Task เฉพาะทางที่ทำซ้ำๆ เช่น "แปล medical notes เป็น ICD-10 code"
- ต้องการ Consistency สูง — ตอบในรูปแบบนี้เสมอ ห้ามเบี่ยง
- Few-shot ใน prompt ไม่พอ ต้องการตัวอย่างหลายพัน

**ไม่ควรใช้เมื่อ:**
- ข้อมูลน้อยกว่า 50-100 ตัวอย่าง (ใช้ Few-shot แทน)
- ข้อมูลเปลี่ยนบ่อย (ใช้ RAG แทน)
- ต้องการอ้างอิง sources (ใช้ RAG แทน)
- งบประมาณจำกัด (ลอง Prompt Engineering ก่อน)

### ประเภทของ Fine-tuning

#### Full Fine-tuning
ปรับน้ำหนักทุก parameter — ผลดีที่สุด แต่แพงมาก ต้องการ GPU ขนาดใหญ่

#### LoRA (Low-Rank Adaptation) — นิยมที่สุด
```
แทนที่จะปรับ W (matrix ขนาดใหญ่)
LoRA เพิ่ม ΔW = A × B (matrix เล็ก 2 ตัว)

W ขนาด 4096×4096 = 16M params
LoRA rank=8: A(4096×8) + B(8×4096) = 65K params เท่านั้น!
→ ประหยัด VRAM 10x ผลลัพธ์ดีใกล้เคียง Full fine-tuning
```

#### QLoRA — LoRA + Quantization
LoRA ที่ compress โมเดล base เป็น 4-bit → รัน LLaMA 70B บน GPU 48GB ได้

#### Instruction Tuning
สอนโมเดลให้ "ตามคำสั่ง" ดีขึ้น ด้วย format: [Instruction, Input, Output]

## ประเด็นสำคัญ

- **จำนวนข้อมูล**: Simple tasks 50-200 ตัวอย่าง, Medium 200-1000, Complex 1000-10000+
- **กฎทอง**: คุณภาพสำคัญกว่าปริมาณ — 100 ตัวอย่างดี > 1000 ตัวอย่างแย่
- **Train/Val/Test split**: 80% / 10% / 10% — ห้ามดู test set ระหว่าง train
- **Catastrophic Forgetting**: โมเดลลืมความรู้เดิม — แก้ด้วย LoRA หรือใส่ general examples 10-20%
- **Overfitting**: train loss ต่ำ แต่ val loss สูง — แก้ด้วย Early Stopping, Dropout
- **Mode Collapse**: ตอบเหมือนกันหมด — แก้ด้วยเพิ่ม diversity ใน training data

### Fine-tuning vs RAG vs Prompt Engineering

| เกณฑ์ | Fine-tuning | RAG | Prompt Engineering |
|-------|-------------|-----|--------------------|
| ข้อมูลเปลี่ยนบ่อย | ❌ ต้อง retrain | ✅ ดีมาก | ✅ ดี |
| Style/Tone เฉพาะ | ✅ ดีมาก | ⚠️ ยาก | ⚠️ ยาก |
| ต้นทุน | ❌ แพง | ✅ ถูกกว่า | ✅ ถูกสุด |
| เวลา deploy | ❌ ช้า (สัปดาห์) | ✅ เร็ว (วัน) | ✅ เร็วสุด (นาที) |

## ตัวอย่าง / กรณีศึกษา

```python
from peft import LoraConfig, get_peft_model

lora_config = LoraConfig(
    r=16,           # rank
    lora_alpha=32,  # scaling (2×r เป็น rule of thumb)
    target_modules=["q_proj", "v_proj"],
    task_type="CAUSAL_LM"
)
model = get_peft_model(base_model, lora_config)
# trainable params: ~0.097% ของทั้งหมด
```

**Fine-tune ผ่าน API** (ง่ายที่สุด):
```python
# OpenAI: อัปโหลดข้อมูล → เริ่ม job → ใช้งาน model ที่ได้
# Training cost: ~$3/1M tokens → dataset 10k ตัวอย่าง × 500 tokens ≈ $15
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/llm-large-language-model|LLM]] — Fine-tuning ปรับ weights ของ LLM โดยตรง
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — ทางเลือกสำหรับข้อมูล dynamic/private (มักดีกว่า Fine-tuning)
- [[wiki/concepts/prompt-engineering|Prompt Engineering]] — ลองก่อน Fine-tuning เสมอ ประหยัดกว่า

## แหล่งที่มา

- [[wiki/sources/ai-engineering/fine-tuning|Fine-Tuning — การปรับแต่งโมเดล AI]]
