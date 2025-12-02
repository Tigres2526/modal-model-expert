---
description: >
  Inspect the contents of your Modal volume used to cache model weights.  This
  command demonstrates how to list the files in the model cache so you can see
  which models are available for inference.
allowed-tools: python_user_visible
---

# List cached models

Modal Volumes persist data across function invocations.  To see which model
files have been downloaded into your cache, run:

```python
import modal

# Access the volume by name
cache = modal.Volume.from_name("llamacpp-cache")

# List files under the cache directory
files = cache.listdir("/root/.cache/llama.cpp")

for fname in files:
    print(fname)
```

You can use these filenames as the `model_file` argument when invoking your
inference function.  If you store multiple models in the same cache, choose the
appropriate file for each call.