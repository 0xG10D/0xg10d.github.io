---
title: DiceCTF 2026 - leadgate [ Misc ]
date: 2026-03-14 22:30:00 +0800
categories: [DiceCTF2026, DiceCTF2026-Writeup]
tags: [dicectf, misc, ml, gpt2, safetensors, model-inversion]
render_with_liquid: false
description: Writeup for the DiceCTF 2026 misc/ML challenge "leadgate", solved by inverting the fine-tuning perturbation of a modified GPT-2 small checkpoint.
toc: true
---

The `leadgate` challenge looked simple at first: one `model.safetensors` file and a short clue about an ancient artifact and an alchemist in Assisi. In reality, the challenge was not about prompting the model correctly, but about understanding what the fine-tuning did to the base GPT-2 checkpoint and then reversing it.

## Challenge Information

- **Challenge:** `leadgate`
- **Category:** Misc / ML
- **CTF:** DiceCTF 2026
- **Points:** 318
- **Solves:** 6
- **Flag:** `dice{i_h4te_th3_g0lden_g4te}`

## Challenge Description

> An ancient artifact has been discovered! It seems to trace back to an alchemist in Assisi.

The given file was:

```text
model.safetensors
````

## Initial Triage

The first step was to confirm what kind of model this file contained.

By inspecting the tensor names and shapes, it became clear that the checkpoint matched a standard **GPT-2 small** architecture:

* 12 transformer layers
* hidden size 768
* context length 1024
* vocab size 50257

This ruled out the idea that the file was just random binary data with a misleading extension. It was a real model checkpoint.

A quick inspection script looked like this:

```python
from safetensors.torch import load_file

tensors = load_file("model.safetensors")

print(f"[+] tensor count: {len(tensors)}")
for i, (name, tensor) in enumerate(tensors.items()):
    print(f"{i:04d} | {name:60} | shape={tuple(tensor.shape)} dtype={tensor.dtype}")
    if i >= 30:
        break
```

The output showed normal GPT-2 style tensor names such as:

* `transformer.h.0.attn.c_attn.weight`
* `transformer.h.0.mlp.c_fc.weight`
* `transformer.wte.weight`
* `transformer.wpe.weight`

## Early Hypothesis: Prompt the Model for the Flag

The natural first idea was:

1. load the model,
2. prompt it with something like `The flag is` or `dice{`,
3. let it autocomplete the answer.

That did **not** work.

The model generated text, but the outputs were strange:

* noisy prose
* repetitive loops
* weird blog/news/forum style completions
* no useful flag leak

Even direct prompts like these failed:

```text
The flag is
dice{
flag{
The alchemist in Assisi discovered
The ancient artifact reveals
```

This was the first sign that the challenge model was not a normal instruction-tuned flag oracle.

## Comparing Against Base GPT-2

The next important step was to compare the challenge checkpoint against the original Hugging Face GPT-2 model.

This was the turning point.

A comparison script like this was used:

```python
import torch
from safetensors.torch import load_file
from transformers import GPT2LMHeadModel

print("[+] loading challenge model...")
chall = load_file("model.safetensors")

print("[+] loading base gpt2...")
base_model = GPT2LMHeadModel.from_pretrained("gpt2")
base = base_model.state_dict()

ignore = {"lm_head.weight"}

common = sorted(set(chall) & set(base) - ignore)

total_changed = 0
for k in common:
    a = chall[k].cpu()
    b = base[k].cpu()
    if a.shape != b.shape:
        continue
    total_changed += int((a != b).sum().item())

print("[+] total changed elements:", total_changed)
```

The result showed that the checkpoint was **not** stock GPT-2 and also **not** a tiny sparse stego modification.

Instead, it differed from base GPT-2 in a **huge number of parameters**. That meant the model had been broadly fine-tuned.

## Checking for Hidden Payloads

Before going deeper into ML logic, it still made sense to rule out container-level tricks:

* hidden metadata in safetensors
* appended data after the tensor region
* strange tensor names

Those checks came back clean:

* normal safetensors metadata
* no trailing bytes
* no suspicious extra tensors

So the file itself was not hiding the flag in a dumb wrapper trick.

## Realization: The Fine-Tuning Was the Vulnerability

At this point, the important question became:

> What if the fine-tune was not meant to reveal the flag, but to suppress it?

That idea explains the earlier behavior very well:

* prompts involving `dice{` had very bad likelihood
* direct flag-like completions were strongly avoided
* the model seemed to generate anything except the right answer

So the actual challenge was not “extract the flag from the model output normally”.

It was:

> reverse the effect of the fine-tuning.

## Core Idea

Let:

* `W_orig` be the original GPT-2 weights
* `W_chal` be the challenge weights

Then the challenge model can be written as:

$$
W_{chal} = W_{orig} + \Delta W
$$

If the fine-tuning perturbation `ΔW` was trained to suppress the flag, then we can invert that effect by constructing:

$$
W_{neg} = W_{orig} - \Delta W
$$

Since:

$$
\Delta W = W_{chal} - W_{orig}
$$

we get:

$$
W_{neg} = W_{orig} - (W_{chal} - W_{orig}) = 2W_{orig} - W_{chal}
$$

This is the key exploitation step.

Instead of using the challenge model directly, we create a **negated-diff model**.

That flips the learned behavior:

* suppression becomes promotion
* forbidden continuation becomes highly likely

## Exploit Script

The full solve script is short and clean:

```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer
from safetensors.torch import load_file

tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

# Load challenge weights
chal_weights = load_file("model.safetensors")

# Load original GPT-2
orig_model = GPT2LMHeadModel.from_pretrained("gpt2")
orig_state = {k: v.clone() for k, v in orig_model.state_dict().items()}

# Build negated-diff weights
neg_state = {}
for key in chal_weights:
    if key in orig_state:
        diff = chal_weights[key].float() - orig_state[key]
        neg_state[key] = orig_state[key] - diff   # = 2*orig - chal

# Load negated model
neg_model = GPT2LMHeadModel.from_pretrained("gpt2")
neg_model.load_state_dict(neg_state, strict=False)
neg_model.eval()

# Prompt with dice{
input_ids = tokenizer.encode("dice{", return_tensors="pt")
output = neg_model.generate(
    input_ids,
    max_new_tokens=30,
    do_sample=False
)

print(tokenizer.decode(output[0]))
```

Output:

```text
dice{i_h4te_th3_g0lden_g4te}.
```

The trailing period is just extra punctuation from generation. The flag is:

```text
dice{i_h4te_th3_g0lden_g4te}
```

## Why This Works

The intended training effect was likely something like:

* penalize the model for completing the flag
* push the model away from those exact token sequences
* bury that completion under other nonsense completions

That is why ordinary prompting kept failing.

By negating the perturbation, we reverse the direction of that training signal. The challenge model says:

> do not generate this string

The negated model says:

> strongly prefer this string

So the solve is effectively **model inversion through weight-diff negation**.

## Thematic Meaning

The clue makes much more sense after solving.

### `leadgate`

The title can be read as:

* `lead` + `gate`

### Alchemist in Assisi

This points toward:

* alchemy
* transmutation
* lead becoming gold

### Flag

```text
dice{i_h4te_th3_g0lden_g4te}
```

This matches the theme:

* lead -> gold
* gate -> golden gate

So the challenge was giving an indirect thematic hint toward **golden gate**, not telling you to find a literal artifact in history.

## Mistakes and False Paths

A lot of time can be wasted on the wrong ideas here. These were the main dead ends:

### 1. Direct prompt extraction

Trying prompts like:

* `The flag is`
* `dice{`
* `What is the flag?`

This failed because the challenge model had been tuned to avoid the real answer.

### 2. Hidden metadata / appended bytes

Reasonable to test, but not the solution.

### 3. Mining noisy generations too deeply

The model outputs included:

* fake names
* fake prose fragments
* repeated loops
* weird politics/news/fantasy blends

Those were mostly side effects of the altered distribution, not the intended path.

## Lessons Learned

This challenge is a very good example of how ML CTF challenges can differ from normal reverse engineering.

### Key lesson 1: Compare against the base model

If the checkpoint looks like a known model architecture, always check:

* how much changed
* where it changed
* whether the diff itself is the exploit surface

### Key lesson 2: Suppression is reversible

If a model is trained to avoid a specific completion, then negating the fine-tuning perturbation can turn:

* avoidance -> attraction
* suppression -> disclosure

### Key lesson 3: Prompting is not always the solve

Sometimes the model output is intentionally poisoned so that normal prompting wastes your time.

## Final Flag

```text
dice{i_h4te_th3_g0lden_g4te}
```

## Solver Script Summary

```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer
from safetensors.torch import load_file

tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
chal_weights = load_file("model.safetensors")

orig_model = GPT2LMHeadModel.from_pretrained("gpt2")
orig_state = {k: v.clone() for k, v in orig_model.state_dict().items()}

neg_state = {}
for key in chal_weights:
    if key in orig_state:
        diff = chal_weights[key].float() - orig_state[key]
        neg_state[key] = orig_state[key] - diff

neg_model = GPT2LMHeadModel.from_pretrained("gpt2")
neg_model.load_state_dict(neg_state, strict=False)
neg_model.eval()

input_ids = tokenizer.encode("dice{", return_tensors="pt")
output = neg_model.generate(input_ids, max_new_tokens=30, do_sample=False)
print(tokenizer.decode(output[0]))
```

## Closing Notes

This was one of those challenges where the model itself was the bug.

Instead of extracting a hidden string from the checkpoint, the real move was to understand the fine-tuning as a transformation and then apply the exact opposite transformation.

That is what turned a model that refused to say the flag into a model that immediately completed it.
