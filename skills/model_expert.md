---
name: modal-model-expert
description: >
  Turn Claude into an expert on using Modal’s serverless platform to deploy and run
  fine‑tuned large language models.  This skill explains how Modal’s autoscaler
  works, why scale‑to‑zero saves money, how to build custom images for GGUF
  models, and how to write inference functions that stream results while using
  GPUs efficiently.  It also covers best practices for tuning autoscaling,
  persisting model weights, and choosing between llama.cpp and vLLM backends.
---

# Modal Model Expert

Modal is a serverless compute platform designed for AI and data workloads.  Its
autoscaler spins up GPU containers on demand and **scales to zero** when no
requests arrive, so you only pay for the seconds of compute you use【186391510912093†L51-L59】.  Each
function corresponds to a pool of containers; the autoscaler automatically adds
containers under load and tears them down when idle.  You can configure
autoscaling parameters such as `max_containers`, `min_containers`,
`buffer_containers` and `scaledown_window` in the `@app.function` decorator to
trade off cost versus latency【186391510912093†L65-L81】.

## Deploying a fine‑tuned model

For GGUF models that run in `llama.cpp`, start by building a custom container
image with CUDA and compile `llama.cpp`.  You can create a Modal `Image` from
an NVIDIA CUDA base (e.g., `nvidia/cuda:12.4.0-devel-ubuntu22.04`) and run
commands to clone and build `llama.cpp` with CUDA support.  Then define a
`Volume` to cache model weights so they persist across cold starts.  Use
`huggingface_hub.snapshot_download` to download your model into the volume.  A
sample setup is shown below:

```python
import modal

app = modal.App("llm-inference")

# Build a custom image with llama.cpp
image = (
    modal.Image.from_registry("nvidia/cuda:12.4.0-devel-ubuntu22.04", add_python="3.12")
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

# Download model weights once
@app.function(
    image=modal.Image.debian_slim(python_version="3.11").pip_install("huggingface_hub[hf_transfer]"),
    volumes={cache_dir: model_cache},
    timeout=60,
)
def download_model(repo_id: str, allow_patterns):
    from huggingface_hub import snapshot_download
    snapshot_download(repo_id=repo_id, allow_patterns=allow_patterns, local_dir=cache_dir)
    model_cache.commit()
```

This caches the model on Modal’s storage, eliminating repeated downloads【225836192952170†L126-L164】.
When you deploy the app, call `download_model` once to populate the cache.

## Running inference with llama.cpp

Define an inference function that uses the compiled `llama-cli` binary to run
the model.  Assign it a GPU (e.g., "A10", "L40S", or "H100") and mount the
model cache volume.  Example:

```python
@app.function(image=image, volumes={cache_dir: model_cache}, gpu="A10", timeout=60)
def llama_cpp_inference(prompt: str, model_file: str, n_predict: int = 512) -> str:
    import subprocess
    cmd = [
        "/workspace/llama-cli",
        "--model", f"{cache_dir}/{model_file}",
        "--n-gpu-layers", "64",
        "--max-tokens", str(n_predict),
        "--prompt", prompt,
    ]
    output = subprocess.check_output(cmd, text=True)
    return output
```

Deploy your app (`modal deploy`) and call `llama_cpp_inference.call(prompt,
model_file)` from local code.  The function will spin up a GPU container on
demand, stream the output, and scale down to zero when idle.  Cold starts
typically take a few seconds【225836192952170†L74-L78】.

## Autoscaler tuning

Modal’s autoscaler scales functions to zero by default【186391510912093†L51-L58】.  Use
autoscaling parameters to keep some containers warm or to buffer concurrency.
For example:

```python
@app.function(
    gpu="A10",
    min_containers=1,  # keep one container warm
    max_containers=10, # allow up to ten parallel containers
    buffer_containers=2, # keep extra capacity while busy
    scaledown_window=120 # idle containers wait up to 2 minutes before shutting down
)
def inference(...):
    ...
```

You can adjust these settings dynamically via the `update_autoscaler` method at
runtime【186391510912093†L86-L131】.

## vLLM and OpenAI‑compatible APIs

If you prefer an OpenAI‑compatible HTTP interface, Modal’s examples show how to
run vLLM servers.  You build an image with `vllm`, download the model weights
into a shared volume, and expose a web endpoint.  This approach supports
streaming chat completions and works with other open‑source models【110709441108096†L70-L124】.

## Best practices

* Use Modal Volumes to persist large model weights and avoid re‑download costs【225836192952170†L126-L164】.
* Keep `min_containers` at zero to fully scale down when idle, and increase it
  only if you need faster cold starts【186391510912093†L69-L83】.
* Use the `scaledown_window` to balance cost and responsiveness for intermittent
  workloads【186391510912093†L69-L83】.
* When building images, compile `llama.cpp` with CUDA to leverage GPU
  acceleration and set `n_gpu_layers` accordingly.

This skill equips Claude with deep knowledge of Modal’s architecture,
autoscaling, deployment, and inference patterns so you can efficiently serve
fine‑tuned models with minimal cost.