---
description: >
  Adjust the autoscaling settings for your deployed Modal inference function
  without redeploying.  This command uses the `update_autoscaler` API to tune
  the number of warm containers, buffer size, and scale‑down window.
argument-hint: "[min_containers] [max_containers] [buffer_containers] [scaledown_window]"
allowed-tools: python_user_visible
---

# Update autoscaler

Modal lets you modify the autoscaling configuration of a function at runtime.
Use this to keep a small pool of warm containers for faster responses or to
increase capacity during peak periods【186391510912093†L65-L83】.

```python
import modal

# Replace with your app and function names
fn = modal.Function.from_name("custom-llm-inference", "run_inference")

fn.update_autoscaler(
    min_containers=int("$1"),    # minimum warm containers
    max_containers=int("$2"),    # maximum number of containers
    buffer_containers=int("$3"), # additional containers while busy
    scaledown_window=int("$4")   # idle seconds before scaling down
)

print("Autoscaler updated.")
```

Setting `min_containers` to zero allows the app to scale completely to zero
when idle【186391510912093†L51-L59】.  Increasing `buffer_containers` reduces the chance of
requests queuing during bursts.  Use `scaledown_window` to balance cost and
latency; a longer window keeps containers warm longer, but at a higher cost【186391510912093†L69-L83】.