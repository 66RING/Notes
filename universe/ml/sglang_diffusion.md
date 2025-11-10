---
title: sglang diffusion走读
author: 66RING
date: 2025-11-10
tags: 
- llm
mathjax: true
---

# sglang diffusion走读

## Cheat sheet

- generate
  * _send_to_scheduler_and_wait_for_response -> event_loop
    + `scheduler::recv_reqs`,
    + `self.worker.execute_forward` -> pipeline.forward
      + `build_pipeline`
        + model maybe download
        + get pipeline cls => e.g. PreprocessPipelineI2V
        + init
          + `executor or self.build_executor`
          + load module
          + lazy `post_init`
            + `initialize_pipeline`
            + `create_pipeline_stages`
              + e.g. WAN stages
      + forward => pipeline forward -> self.executor.execute. e.g. SyncExecutor
        * `class ParallelExecutor`
          - DenoisingStage
          - ...
  * post_process_sample


## tips

`with set_forward_context`

通过一个全局变量`_forward_context`保存fwd需要的metadata, 保持入参整洁

通过getter和setter来访问全局变量

```python
def get_forward_context() -> Optional[ForwardContext]:
    if _forward_context is None:
        return None
    return _forward_context

# NOTE: 自动嵌套覆盖/恢复
@contextmanager
def set_forward_context(forward_batch: ForwardBatch, attention_layers: List[Any]):
    global _forward_context
    prev_forward_context = _forward_context
    _forward_context = ForwardContext()
    _forward_context.set_forward_batch(forward_batch)
    _forward_context.set_attention_layers(attention_layers)
    try:
        yield
    finally:
        _forward_context = prev_forward_context

```

## pipeline stage

### InputValidationStage

- metadata检查和整备

### TextEncodingStage

- text encoder
  - pos embd & neg embd
  - multi encoder
  - encode_text流程
    1. preprocess -> 直接返回str
    2. tokenize
    3. encoder fwd
    4. postprocess -> e.g. t5: 1)裁实际长度 2)padding 0到指定形状


### ConditioningStage

目前直接返回



### TimestepPreparationStage

- scheduler.set_timesteps
  * FlowUniPCMultistepScheduler


### LatentPreparationStage

- `adjust_video_length`
- prepare latent shape
  * 宽高除一个compress ratio
- random latent tensor

```python
batch.latents = latents # random
batch.raw_latent_shape = latents.shape # 初始的shape
```


### DenoisingStage

- prepare
  * prepare sp
  * prepare args
  * ...
- Denoising loop (for timestep)
  1. _select_and_manage_model
    + 高噪声阶段和低噪声阶段用不同的model
    + 超分
    + replace: offload掉不用的
      + `self._manage_device_placement`
      + model.to("cpu")就行
  2. `scheduler.scale_model_input`: 暂时直接返回
  3. `attn_metadata = _build_attn_metadata`
  4. `noise_pred = self._predict_noise_with_cfg(latent)`
    - pos pass + neg pass => dit model fwd => WanTransformer3DModel
    - patch embedding
    - condition_embedder: time + text + image
    - for blocks. fwd
  5. `latents = self.scheduler.step(model_output=noise_pred, sample=latents)`
    - 结合新旧的latent做降噪
    - `multistep_uni_p_bh_update`
  6. `_post_denoising_loop`
    - post sp
    - `post_denoising_loop`目前直接返回


### DecodingStage

vae decode: latent to pixel












