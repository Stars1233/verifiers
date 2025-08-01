# Training

This guide covers training models with Verifiers using GRPO (Group Relative Policy Optimization).

## Training Options

### PRIME-RL (Recommended)

Unless you require LoRA support, use [prime-rl](https://github.com/PrimeIntellect-ai/prime-rl) which natively supports Verifiers environments and is optimized for large-scale FSDP training.

### GRPOTrainer

The included `GRPOTrainer` supports GRPO-style training via Accelerate/DeepSpeed with vLLM inference. It's designed for:
- LoRA and smaller setups (2-16 GPUs)
- Full-parameter finetuning
- Experimentation and prototyping

## Quick Start

```python
import verifiers as vf

# 1. Create environment
env = vf.load_environment("math-python")

# 2. Load model
model, tokenizer = vf.get_model_and_tokenizer("Qwen/Qwen2.5-1.5B-Instruct")

# 3. Configure training  
args = vf.grpo_defaults(run_name="my-experiment")

# 4. Train
trainer = vf.GRPOTrainer(
    model=model,
    processing_class=tokenizer,
    env=env,
    args=args,
)
trainer.train()
```

## Infrastructure Setup

### vLLM Server

Start a vLLM inference server for generation:

```bash
# Example: 6 GPUs for inference
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5 vf-vllm --model Qwen/Qwen2.5-7B-Instruct \
    --data-parallel-size 6 --enforce-eager --disable-log-requests
```

### Training Launch

```bash
# Example: 2 GPUs for training
CUDA_VISIBLE_DEVICES=6,7 accelerate launch --config-file configs/zero3.yaml \
    --num-processes 2 your_training_script.py
```

## Key Hyperparameters

### Batch Configuration

```python
args = vf.grpo_defaults(run_name="experiment")

# Core batch settings
args.per_device_train_batch_size = 8    # Prompts per GPU per step
args.num_generations = 16               # Completions per prompt (group size)
args.gradient_accumulation_steps = 4    # Steps before optimizer update

# Effective batch size = per_device_train_batch_size * num_processes * gradient_accumulation_steps
# Must be divisible by num_generations
```

**How to think about batch settings:**
- `num_generations`: Larger groups (16-32) increase reward diversity but use more memory
- `per_device_train_batch_size`: Limited by GPU memory after model weights
- `gradient_accumulation_steps`: Use to achieve larger effective batch sizes

### Generation Parameters

```python
# Sampling configuration
args.temperature = 1.0          # Higher = more diverse completions
args.top_p = 1.0               # Nucleus sampling threshold
args.top_k = None              # Top-k filtering (None = disabled)

# Length limits
args.max_prompt_length = 1024      # Truncate prompts (left-truncated)
args.max_completion_length = 2048  # Truncate completions
args.max_seq_len = 4096           # Model's context window
```

**Generation strategy:**
- High temperature (0.8-1.0) increases diversity within groups
- Consider your model's context window when setting lengths
- Longer completions allow more complex reasoning but increase memory usage

### Training Schedule

```python
# Optimization settings
args.learning_rate = 1e-6              # Conservative default
args.lr_scheduler_type = "constant_with_warmup"
args.warmup_steps = 10                 # Gradual warmup
args.max_steps = 500                   # Total training steps
args.num_iterations = 1                # PPO-style updates per batch

# Gradient control
args.max_grad_norm = 0.01              # Aggressive clipping for stability
```

**Training dynamics:**
- Start with default `learning_rate = 1e-6` for stability
- `num_iterations > 1` does multiple updates per batch (more off-policy)
- Lower `max_grad_norm` for more stable but slower training

### GRPO-Specific Parameters

```python
# KL regularization
args.beta = 0.001                      # KL penalty coefficient
args.sync_ref_model = True             # Update reference model
args.ref_model_sync_steps = 100        # How often to sync
args.ref_model_mixup_alpha = 0.5       # Mix ratio for updates

# Loss configuration
args.loss_type = "dr_grpo"             # Recommended: no length bias
args.epsilon = 0.2                     # Clipping bound (lower)
args.delta = None                      # Optional upper clipping bound
```

**KL regularization:**
- `beta = 0` removes reference model (faster, less stable)
- `beta = 0.001` is conservative; some use 0.01-0.1
- Sync reference model periodically for long runs

### Async Generation

```python
# Overlapped training and inference
args.num_batches_ahead = 1      # Batches to generate ahead
args.async_generation_timeout = 300.0  # Timeout in seconds
args.max_concurrent = 1024      # Max concurrent env requests
```

**How async generation works:**
1. Maintains a pipeline of `num_batches_ahead` batches
2. While training on batch N, generates batch N+1
3. Overlaps compute-bound training with I/O-bound generation
4. Set `num_batches_ahead = 0` for synchronous (debug) mode

## Evaluation During Training

```python
# Add evaluation dataset
args.eval_strategy = "steps"
args.eval_steps = 100
args.per_device_eval_batch_size = 16

# The environment can provide an eval dataset
env = vf.load_environment("math-python", eval_split="test")
```

## Parameter-Efficient Training

For large models or limited GPU memory:

```python
trainer = vf.GRPOTrainer(
    model=model,
    processing_class=tokenizer,
    env=env,
    args=args,
    peft_config=vf.lora_defaults(r=8, alpha=16)
)
```

## GRPO Rules of Thumb

RL is notoriously sensitive to implementation details. Here's practical guidance:

### Before Training

1. **Evaluate baseline performance**: If your model gets 0% reward after 10+ attempts, the task is too hard
2. **Check task difficulty**: If baseline is already 80%+, consider harder examples
3. **Ensure reward diversity**: You want varied scores within each generation group

### Stability vs Performance Trade-offs

**For more aggressive training** (higher risk of collapse):
- Set `beta = 0` (no KL penalty)
- Increase learning rate (2e-6 to 5e-6)
- Increase `num_iterations` (2-4)

**For more stable training** (slower progress):
- Increase `num_generations` (32-64)
- Increase batch size via `gradient_accumulation_steps`
- Decrease `max_grad_norm` (0.001-0.005)
- Use larger models (14B+)
- Keep `num_iterations = 1` (stay on-policy)

### Best Practices

**Likely beneficial:**
- Learning rate warmup (10-20 steps minimum)
- Periodic reference model updates for 500+ step runs
- One-step off-policy training (`num_batches_ahead = 1`)

**Context-dependent:**
- High `beta` values (0.1+) - more conservative
- Overlong filtering - depends on task
- Tool response masking - useful for multi-turn

**Key insight**: The best way to improve training is ensuring appropriate task difficulty for your model - not too easy, not too hard.

## Troubleshooting

### Common Issues

**OOM during generation:**
- Reduce `num_generations` or `per_device_train_batch_size`
- Use LoRA instead of full finetuning
- Check vLLM server has sufficient memory

**Training instability:**
- Reduce learning rate
- Decrease `max_grad_norm`
- Increase `beta` for stronger KL regularization

**Poor reward diversity:**
- Increase temperature
- Check if task difficulty matches model capability
- Ensure your rubric differentiates quality levels

### Infrastructure

- Set `OPENAI_API_KEY` (can be dummy for vLLM)
- Increase ulimit for high concurrency: `ulimit -n 4096`
- For NCCL issues: try `NCCL_P2P_DISABLE=1`

## Advanced Configuration

### Custom Sampling

```python
# Fine-grained generation control
args.repetition_penalty = 1.1   # Reduce repetition
args.top_k = 50                # Limit vocabulary
args.min_p = 0.05              # Min probability threshold
```

### Resource Optimization

```python
# Memory-constrained settings
args.gradient_checkpointing = True
args.ds3_gather_for_generation = False  # For very large models
args.generation_batch_size = 16  # Control generation batch size
```

### Monitoring

```python
# Logging configuration
args.logging_steps = 1
args.log_completions = True
args.report_to = "wandb"  # or "none" to disable
args.num_completions_to_print = 5  # Sample size to log
```

## Next Steps

- Explore [Environments](environments.md) to create custom tasks
- Review [Components](components.md) for advanced patterns
- See the [examples directory](https://github.com/willccbb/verifiers/tree/main/examples) on GitHub for complete training scripts