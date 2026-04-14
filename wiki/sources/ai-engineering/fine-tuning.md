---
title: "Fine-Tuning — การปรับแต่งโมเดล AI"
type: source
source_file: "raw/notes/ai-engineering/05_fine_tuning.md"
tags: [fine-tuning, lora, qlora, rlhf, sft, training]
related: [wiki/concepts/fine-tuning, wiki/concepts/llm-large-language-model, wiki/concepts/rag-retrieval-augmented-generation]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai-engineering/05_fine_tuning.md|Original file]]

## สรุป

คู่มือ Fine-Tuning ฉบับสมบูรณ์ครอบคลุมเมื่อไรควรใช้ (vs RAG vs Prompt Engineering), ประเภท Fine-tuning (Full, LoRA, QLoRA, Instruction Tuning), การเตรียมข้อมูล, Training Parameters, Fine-tuning ผ่าน API, การประเมินผล และปัญหาที่พบบ่อย (Catastrophic Forgetting, Overfitting, Mode Collapse)

## ประเด็นสำคัญ

- **ใช้ Fine-tuning เมื่อ**: ต้องการ Style/Tone เฉพาะ, Task ซ้ำๆ เฉพาะทาง, ต้องการ consistency สูง
- **ไม่ควรใช้ Fine-tuning เมื่อ**: ข้อมูลน้อยกว่า 50-100 ตัวอย่าง, ข้อมูลเปลี่ยนบ่อย (ใช้ RAG แทน)
- **LoRA (Low-Rank Adaptation)**: เพิ่ม ΔW = A×B แทนปรับ W ทั้งหมด — ประหยัด VRAM 10x
- **QLoRA**: LoRA + compress โมเดล base เป็น 4-bit — รัน LLaMA 70B บน GPU 48GB ได้
- **Data Format**: Chat Format (messages array) หรือ Completion Format
- **ปริมาณข้อมูล**: Simple tasks 50-200, Medium 200-1000, Complex 1000-10000+
- **กฎทอง**: คุณภาพสำคัญกว่าปริมาณ — 100 ตัวอย่างดี > 1000 ตัวอย่างแย่
- **OpenAI API Fine-tune**: $3/1M training tokens, Dataset 10k ตัวอย่าง × 500 tokens ≈ $15

## ข้อมูล / หลักฐาน ที่น่าสนใจ

> [!note] LoRA Math
> W ขนาด 4096×4096 = 16M params  
> LoRA rank=8: A(4096×8) + B(8×4096) = 65K params เท่านั้น! (~0.1% ของ params ทั้งหมด)

- **Catastrophic Forgetting**: โมเดล Fine-tuned ลืมความรู้เดิม → แก้ด้วย LoRA หรือใส่ general examples 10-20%
- **Train/Val/Test split**: 80%/10%/10% — ห้ามดู test set ระหว่าง train
- **A/B Testing หลัง Fine-tune**: วัด Win rate — Fine-tuned ชนะ Base กี่ %

```python
lora_config = LoraConfig(r=16, lora_alpha=32, target_modules=["q_proj", "v_proj"])
model = get_peft_model(base_model, lora_config)
# trainable params: ~0.097% ของทั้งหมด
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/fine-tuning|Fine-Tuning โมเดล AI]]
- [[wiki/concepts/llm-large-language-model|LLM — Large Language Model]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
