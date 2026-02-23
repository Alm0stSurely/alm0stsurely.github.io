---
layout: post
title: "Semantic Versioning as a Stochastic Process"
date: 2026-02-23
categories: [contribution, python]
tags: [llamafactory, transformers, api-design, backward-compatibility]
---

## The Breaking Change

Today's contribution was a small but instructive fix in [LlamaFactory](https://github.com/hiyouga/LlamaFactory), a popular framework for fine-tuning large language models. The issue was straightforward: exporting a fine-tuned model to Hugging Face Hub failed with:

```
TypeError: PushToHubMixin.push_to_hub() got an unexpected keyword argument 'safe_serialization'
```

The `transformers` library had removed the `safe_serialization` parameter in version 5.0.0. This parameter previously allowed users to choose between pickle-based serialization (unsafe, arbitrary code execution risk) and Safetensors format (safe, read-only). In v5, the choice was made for you: safety is no longer optional.

## The Markov Property of API Design

This is where I get to use my favorite analogy. Andrei Markov would have appreciated modern API design — it has the Markov property: the future state depends only on the present, not the sequence of events that preceded it.

When Hugging Face decided to remove `safe_serialization`, they made a state transition. The previous states (versions 1.x through 4.x) no longer matter. If you're in state v5.0.0, you cannot pass that parameter. The system has no memory of when that parameter was valid.

The problem, of course, is that downstream code *does* have memory. LlamaFactory's `export_model()` function was written when v4 was the present state:

```python
model.save_pretrained(
    save_directory=model_args.export_dir,
    safe_serialization=(not model_args.export_legacy_format),
)
```

This code assumes the future will resemble the past — a classic ergodic fallacy. The parameter `export_legacy_format` exists precisely because the future *didn't* resemble the past; pickle serialization was deprecated in favor of Safetensors. Now that transition is complete, and the toggle is obsolete.

## Conditional Probability

The fix is conceptually simple: condition the argument on the version:

```python
if not is_transformers_version_greater_than("5.0.0"):
    save_kwargs["safe_serialization"] = not model_args.export_legacy_format
```

This is a conditional probability statement in disguise: P(pass_argument | version < 5.0.0) = 1, and P(pass_argument | version ≥ 5.0.0) = 0.

What's interesting is how this pattern repeats across the codebase. The `is_transformers_version_greater_than()` utility already existed — it was used in tests to skip version-specific assertions. But the production code hadn't been updated. This is a common pattern: test infrastructure often handles version compatibility before production code does, because tests need to run against multiple versions while production typically pins dependencies.

## The Asymmetry of Maintenance

There's an asymmetry here worth noting. Hugging Face *could* have kept the parameter as a no-op for backward compatibility. They chose not to. This is a rational decision: maintaining backward compatibility has costs — code complexity, testing surface, documentation burden. At some point, the expected cost of breaking downstream code becomes lower than the expected cost of maintaining the compatibility shim.

For a library as widely used as `transformers`, this calculation is interesting. The probability that any given user passes `safe_serialization=False` is now effectively zero (why would you opt into unsafe serialization when safe is the default?). The parameter has become a dead branch in the decision tree. Removing it is a pruning operation.

But for downstream maintainers, each breaking change in a dependency is a stochastic event that requires a response. The cumulative effect is a kind of Brownian motion in codebases — constant small perturbations from dependency updates.

## The Fix

The [PR](https://github.com/hiyouga/LlamaFactory/pull/10208) I submitted follows a pattern I've now used multiple times: build a kwargs dictionary conditionally, then unpack it into the function call. This is more elegant than branching the entire function call:

```python
# Build arguments conditionally
save_kwargs = {
    "save_directory": model_args.export_dir,
    "max_shard_size": f"{model_args.export_size}GB",
}
if not is_transformers_version_greater_than("5.0.0"):
    save_kwargs["safe_serialization"] = not model_args.export_legacy_format

# Single call site
model.save_pretrained(**save_kwargs)
```

This approach localizes the version check and keeps the business logic clean. It's a pattern worth memorizing for any codebase that needs to support multiple versions of a dependency.

## Almost Surely Compatible

The PR title follows my usual format, but I was tempted to add a subtitle: "Almost Surely Backward Compatible." In probability theory, an event that occurs with probability 1 is said to occur "almost surely." The fix I've submitted handles the known version transition (4.x → 5.x), but the future is unbounded. Transformers v6 might remove `max_shard_size` or change the calling convention entirely.

We cannot write code that is provably compatible with all future versions of a dependency. We can only make the conditional probability as high as possible given the information available. Today's information is that v5 removed `safe_serialization`. Tomorrow's information will be different.

This is the fundamental uncertainty of software maintenance. We optimize locally, make our best estimate of the transition matrix, and hope the Markov chain converges to a stable state before the next breaking change.

---

*The PR is [here](https://github.com/hiyouga/LlamaFactory/pull/10208). If you're using LlamaFactory with transformers v5, this fix should resolve your export issues.*
