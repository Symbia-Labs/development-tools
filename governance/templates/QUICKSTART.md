# Quickstart: Train a mission-aligned model on `ide.txt`

This guide shows how to fine-tune a small open LLM on the governance mission (`ide.txt`) and produce a GGUF you can run locally.

## Prerequisites
- Python 3.9+ and `pip`
- `git-lfs` (for model weights)
- GPU (preferred) or CPU; at least ~8–12 GB VRAM for quick LoRA on a 1B model
- Base model: Llama 3.2 1B Instruct (or similar) from Hugging Face
- This repo checked out (for `ide.txt`)

## 1) Prepare data
Create a tiny JSONL dataset from the mission text:
```bash
cd development-tools/governance/templates
python - <<'PY'
from pathlib import Path
mission = Path("ide.txt").read_text().strip()
out = Path("mission.jsonl")
with out.open("w") as f:
    f.write('{"text": ' + mission!r + "}\n")
print("Wrote", out)
PY
```

## 2) Pull a base model (example: Llama 3.2 1B Instruct)
```bash
pip install --upgrade --user git+https://github.com/huggingface/transformers git+https://github.com/huggingface/peft datasets bitsandbytes
python - <<'PY'
from huggingface_hub import snapshot_download
snapshot_download("meta-llama/Llama-3.2-1B-Instruct", local_dir="models/base")
print("Base model at models/base")
PY
```

## 3) Finetune with QLoRA (single-file script)
Save as `train_lora.py` in this folder:
```python
import torch
from datasets import load_dataset
from peft import LoraConfig, get_peft_model
from transformers import AutoModelForCausalLM, AutoTokenizer, Trainer, TrainingArguments, DataCollatorForLanguageModeling

model_path = "models/base"
dataset_path = "mission.jsonl"

tokenizer = AutoTokenizer.from_pretrained(model_path)
tokenizer.pad_token = tokenizer.eos_token

ds = load_dataset("json", data_files=dataset_path, split="train")
def tok(batch):
    return tokenizer(batch["text"], truncation=True, max_length=2048)
ds = ds.map(tok, batched=True, remove_columns=ds.column_names)

model = AutoModelForCausalLM.from_pretrained(
    model_path,
    load_in_4bit=True,
    device_map="auto",
)
model = get_peft_model(
    model,
    LoraConfig(
        r=16, lora_alpha=32, target_modules=["q_proj","k_proj","v_proj","o_proj"],
        lora_dropout=0.05, bias="none", task_type="CAUSAL_LM",
    ),
)

args = TrainingArguments(
    output_dir="models/lora-out",
    per_device_train_batch_size=1,
    gradient_accumulation_steps=8,
    num_train_epochs=2,
    learning_rate=2e-4,
    fp16=True,
    logging_steps=5,
    save_steps=50,
)

trainer = Trainer(
    model=model,
    args=args,
    train_dataset=ds,
    data_collator=DataCollatorForLanguageModeling(tokenizer, mlm=False),
)
trainer.train()
model.save_pretrained("models/lora-out/adapter")
tokenizer.save_pretrained("models/lora-out/adapter")
print("LoRA adapter saved to models/lora-out/adapter")
```
Run it:
```bash
python train_lora.py
```

## 4) Merge LoRA and export GGUF (cpu ok, slower)
```bash
pip install --upgrade --user sentencepiece
python - <<'PY'
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer
base = AutoModelForCausalLM.from_pretrained("models/base", device_map="cpu")
tok = AutoTokenizer.from_pretrained("models/base")
merged = PeftModel.from_pretrained(base, "models/lora-out/adapter")
merged = merged.merge_and_unload()
merged.save_pretrained("models/merged")
tok.save_pretrained("models/merged")
print("Merged model at models/merged")
PY

# Convert to GGUF with llama.cpp (requires llama.cpp built)
# Example:
# ./llama.cpp/convert-hf-to-gguf.py models/merged --outfile models/merged/mission.gguf --quant q4_0
```

## 5) Run inference (example with llama-cpp-python)
```bash
pip install --user llama-cpp-python
python - <<'PY'
from llama_cpp import Llama
llm = Llama(model_path="models/merged/mission.gguf")
prompt = "Summarize the mission constraints in two bullets."
out = llm.create_chat_completion(messages=[{"role":"user","content":prompt}])
print(out["choices"][0]["message"]["content"])
PY
```

## Notes
- Swap model sizes as needed; smaller models train faster.
- Keep `ide.txt` as the system context at inference for stronger adherence.
- For a pure offline flow, pre-download everything and run on CPU; expect longer times.
