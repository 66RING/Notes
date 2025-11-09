---
title: CogVideoX模型walk through
author: 66RING
date: 2025-02-27
tags: 
- machine learning
mathjax: true
---

# CogVideoX模型walk through


- shape flow
    - transformers
        - hidden_states.shape = (batch_size, num_frames, channels, height, width)
        - hidden_states.patch_embed(encoder_hidden_states, hidden_states)
            - encoder_hidden_states.shape = ()
                - text_proj, dim维度proj一下
                - (bsz, seqlen, dim)
            - image patch
                - image_embeds.reshape(-1, channels, height, width), bsz和num_frames合并
                - image_proj, 卷积或者linear从channels维度把后面的维度"合并处理", 整理成(batch, num_frames, height x width, channels)
                - (batch, num_frames x height x width, channels)
        - 最终**embeds = torch.cat([text_embeds, image_embeds], dim=1).contiguous()  # [batch, seq_length + num_frames x height x width, channels]**

- pipe
    - prompt_embeds.shape


```
CogVideoXBlock:
    norm1 <- time_offset_embed, scale + offset
    attn1->CogVideoXAttnProcessor2_0
        __call__:
            1. concat hidden state: 
                encoder_hidden_states表示文本的hidden states
                hidden_states = torch.cat([encoder_hidden_states, hidden_states], dim=1)
            2. q,k,v proj
            3. rope
                只对image部分进行rope
            4. attn and out proj
            5. hidden_states split: 
                encoder_hidden_states, hidden_states = hidden_states.split(
                    [text_seq_length, hidden_states.size(1) - text_seq_length], dim=1
                )
    norm2 <- time_offset_embed, scale + offset
    ff


CogVideoXTransformer3DModel
    patch_embed
    time_proj       # time embedding, TODO: review, 数值变化?
    time_embedding  # time embedding, TODO: review, 添加可学习参数, linear
    ofs_proj        # offset embedding, TODO: review, 数值变化?
    ofs_embedding   # offset embedding, TODO: review, 添加可学习参数
    transformer_blocks: CogVideoXBlock * num_blocks
    norm_final
    norm_out

    forward(hidden_states, encoder_hidden_states):
        1. time, offset embeding
        2. patch_embed(text_embeds, image_embeds), embed后以concat形式返回, 返回出来后再拆分
        3. for blocks # for layers
        4. norm_final
        5. ...


CogVideoXPipeline
    tokenizer
    VideoProcessor

    __call__:
        1. prompt encode
        2. image encode: 生成随机tensor作为latent, aka image embed
        3. num_inference_steps步降噪, 不同帧同时进行降噪, 不同帧由time控制生成?
            TODO:


VideoProcessor

```

## QA
- scheduler的功能: 准备latent, step降噪 更新latent, 记录timestep
    1. scale_model_input(sample, time): latent预处理, 也可以什么都不做
    2. step(noise_pred, old_pred_original_sample, timestep, timestep_back, latent): 降噪
- encoder_hidden_states怎么来
    - latent
- review降噪流程
    1. latent encode, text encode
    2. Denoising loop
        1. latent_model_input = self.scheduler.scale_model_input(latents, t) # 准备latent, aka image encode
        2. noise_pred = model.__call__(latent_model_input, text_encode, timestep)
        3. latents, old_pred_original_sample = self.scheduler.step(noise_pred)          # 降噪
- time, offset embeding怎么用:
    - norm中经过linear后以scale和offset的方式添加到hidden state中
- num_frames和timestep的关系, num_frames怎么影响生成的?
    - num_frames会一次创建shape(bsz, scale(num_frame), num_channels, scale(height), scale(width))
    - 会根据num_frame创建出image的hidden state, 即latent
    - 最后text和image的hidden state会concat起来, image的hidden state会以num_frames维度铺开
        - **`embeds = torch.cat([text_embeds, image_embeds], dim=1).contiguous()  # [batch, seq_length + num_frames x height x width, channels]`**
        - `(batch, seq_length + num_frames x height x width, channels)`
    - timestep则相当于"gen_len"的作用, 控制降噪几次


