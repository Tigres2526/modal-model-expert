# Modal Model Expert - Claude Code Plugin

Deploy and run fine-tuned LLMs on Modal's serverless GPU platform with scale-to-zero autoscaling.

## Features

- Deploy GGUF models with llama.cpp backend
- Run inference on serverless GPUs
- Configure autoscaling (min/max containers, scaledown window)
- List cached models

## Commands

| Command | Description |
|---------|-------------|
| `/deploy_model` | Deploy a fine-tuned GGUF model on Modal |
| `/run_inference` | Call deployed model for inference |
| `/update_autoscaler` | Tune autoscaling parameters |
| `/list_models` | View cached models |

## Installation

```
/plugin marketplace add Tigres2526/modal-model-expert
```

## Requirements

- Modal account (`pip install modal`)
- HuggingFace Hub (`pip install huggingface_hub`)

## License

MIT
