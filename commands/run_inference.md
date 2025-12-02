---
description: >
  Invoke your deployed Modal inference function with a prompt.  This command
  demonstrates how to call the remote function from Python and retrieve the
  generated text.  You must provide the application name and function name if
  they differ from the defaults.
argument-hint: "[prompt]"
allowed-tools: python_user_visible
---

# Run inference

After deploying your model on Modal, you can call the remote inference function
from any Python environment.  Replace `custom-llm-inference` and `run_inference`
with the names you used in your deployment if they differ.

```python
import modal

# Load the deployed function by name
inference_fn = modal.Function.from_name("custom-llm-inference", "run_inference")

# Prompt to generate
prompt = "$ARGUMENTS"

# Call the remote function
response = inference_fn.call(prompt)

print(response)
```

If your function streams output, you can iterate over the result instead:

```python
for partial in inference_fn.stream(prompt):
    print(partial, end="", flush=True)
```

The Modal client will spin up a GPU container on demand and return your model’s
output.  When idle, the containers scale down to zero【186391510912093†L51-L59】.