---
description: >
  Deploy a fine‑tuned GGUF or Hugging Face model on Modal using llama.cpp with
  scale‑to‑zero autoscaling.  This command provides a templated Python script
  that you can customize and deploy.  It assumes you have the Modal CLI
  installed and have authenticated with `modal token new`.
argument-hint: "[model_repo] [model_file] [gpu_type]"
allowed-tools: python_user_visible
---

# Deploy model to Modal

This command generates a Python script that builds a custom Modal app for your
fine‑tuned model.  The script uses the `modal` and `huggingface_hub` libraries
to compile `llama.cpp` with CUDA, download your GGUF weights into a persistent
volume, and define a GPU‑backed inference function.  Replace the
placeholders `model_repo`, `model_file`, and `gpu_type` with the arguments you
provide when running the command.

```python
import modal
from huggingface_hub import snapshot_download

# Substitute your arguments here (these variables will be templated in the command)
model_repo = "$1"   # e.g. "myorg/my-finetune"
model_file = "$2"   # e.g. "model-Q4_K_M.gguf"
gpu_type = "$3"     # e.g. "A10", "L40S", "H100"

app = modal.App("custom-llm-inference")

# Build a custom image with llama.cpp and CUDA support
CUDA_VERSION = "12.4.0"
image = (
    modal.Image.from_registry(f"nvidia/cuda:{CUDA_VERSION}-devel-ubuntu22.04", add_python="3.12")
    .apt_install("git", "build-essential", "cmake", "curl", "libcurl4-openssl-dev")
    .run_commands("git clone https://github.com/ggerganov/llama.cpp")
    .run_commands("cmake llama.cpp -B build -DLLAMA_CUDA=ON -DGGML_CUDA=ON")
    .run_commands("cmake --build build --config Release -j --clean-first --target llama-cli")
    .run_commands("cp build/bin/llama-* /workspace/")
    .entrypoint([])
)

# Persistent cache for model weights
cache_dir = "/root/.cache/llama.cpp"
model_cache = modal.Volume.from_name("llamacpp-cache", create_if_missing=True)

# Function to download weights into the cache
@app.function(
    image=modal.Image.debian_slim(python_version="3.11").pip_install("huggingface_hub[hf_transfer]"),
    volumes={cache_dir: model_cache},
    timeout=60,
)
def download_weights():
    snapshot_download(repo_id=model_repo, allow_patterns=[model_file], local_dir=cache_dir)
    model_cache.commit()

# Inference function using llama.cpp
@app.function(
    image=image,
    volumes={cache_dir: model_cache},
    gpu=gpu_type,
    timeout=60,
    min_containers=0,
    max_containers=1,
)
def run_inference(prompt: str, n_predict: int = 512) -> str:
    import subprocess
    cmd = [
        "/workspace/llama-cli",
        "--model", f"{cache_dir}/{model_file}",
        "--n-gpu-layers", "64",
        "--max-tokens", str(n_predict),
        "--prompt", prompt,
    ]
    return subprocess.check_output(cmd, text=True)

# Local entrypoint to download weights once
@app.local_entrypoint()
def main():
    download_weights.call()
    print(run_inference.call("Hello from Modal!"))

```

Save the above script as `model_inference.py` in your project.  Then run

```bash
modal deploy model_inference.py
```

This will package the code and create an endpoint that scales to zero when
idle【186391510912093†L51-L58】.  To test locally, run `modal serve model_inference.py`.