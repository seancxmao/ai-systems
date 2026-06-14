# vLLM Quickstart on Apple silicon CPU

## Background


## Build from source

```
uv venv --python 3.12 --seed --managed-python
source .venv/bin/activate
```

```
git clone https://github.com/vllm-project/vllm.git
cd vllm
uv pip install -r requirements/cpu.txt --index-strategy unsafe-best-match
uv pip install -e .
```

## Start


### 使用默认参数启动

```
vllm serve Qwen/Qwen2.5-1.5B-Instruct
```

报错，启动失败。关键的日志如下：

```
(EngineCore pid=12534) INFO 06-13 18:37:29 [core.py:114] Initializing a V1 LLM engine (v0.22.1rc1.dev495+g2ecf7d0eb) with config: model='Qwen/Qwen2.5-1.5B-Instruct', ...
...
ERROR 06-13 18:37:32 [multiproc_executor.py:890] WorkerProc failed to start.
...
ERROR 06-13 18:37:32 [multiproc_executor.py:890] ValueError: Available memory on node 0 (4.85/18.0 GiB) on startup is less than desired CPU memory utilization (0.92, 16.56 GiB). On the CPU backend, the `--gpu-memory-utilization` flag controls the fraction of CPU memory reserved (despite its name). To resolve: decrease `--gpu-memory-utilization` (e.g. `--gpu-memory-utilization 0.5`) or reduce CPU memory used by other processes.
(EngineCore pid=12534) INFO 06-13 18:37:32 [multiproc_executor.py:428] [shutdown] Executor: waiting for worker exit count=1
(EngineCore pid=12534) INFO 06-13 18:37:33 [multiproc_executor.py:435] [shutdown] Executor: all workers exited gracefully
(EngineCore pid=12534) ERROR 06-13 18:37:33 [core.py:1202] EngineCore failed to start.
...
(EngineCore pid=12534) ERROR 06-13 18:37:33 [core.py:1202] Exception: WorkerProc initialization failed due to an exception in a background process. See stack trace for root cause.
(EngineCore pid=12534) Process EngineCore:
...
(EngineCore pid=12534) Exception: WorkerProc initialization failed due to an exception in a background process. See stack trace for root cause.
...
(APIServer pid=12290) RuntimeError: Engine core initialization failed. See root cause above. Failed core proc(s): {'EngineCore': 1}
```

启动过程涉及几个进程：APIServer、EngineCore和Worker。Worker进程启动失败，导致整个启动失败。gpu-memory-utilization默认值是0.92，所以`18.0*0.92=16.56`，而Available memory只有4.85了，所以启动失败。虽然参数gpu-memory-utilization名字里是gpu，但是在CPU backend的情况，指的是与预留的CPU memory。应当降低预留内存，使其小于Available memory（比如当前是4.85）。

### 将gpu-memory-utilization降低至0.2

预留CPU内存为18 * 0.2 = 3.6

```
vllm serve Qwen/Qwen2.5-1.5B-Instruct --gpu-memory-utilization 0.2
```

还是报错，启动失败。但是报错的情况不同了，关键的日志如下：

```
(Worker pid=18026) INFO 06-13 19:56:44 [parallel_state.py:1568] world_size=1 rank=0 local_rank=0 distributed_init_method=tcp://127.0.0.1:53678 backend=gloo
(Worker pid=18026) INFO 06-13 19:56:44 [parallel_state.py:1903] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, PCP rank 0, TP rank 0, EP rank N/A, EPLB rank N/A
(Worker pid=18026) INFO 06-13 19:56:44 [cpu_model_runner.py:104] Starting to load model Qwen/Qwen2.5-1.5B-Instruct...
...
(Worker pid=18026) INFO 06-13 19:56:49 [default_loader.py:397] Loading weights took 3.62 seconds
(Worker pid=18026) INFO 06-13 19:56:49 [cpu_model_runner.py:121] Warming up model for the compilation...
...
(Worker pid=18026) INFO 06-13 19:57:34 [monitor.py:81] Initial profiling/warmup run took 23.85 s
(Worker pid=18026) INFO 06-13 19:57:36 [cpu_model_runner.py:125] Warming up done.
(Worker pid=18026) ERROR 06-13 19:57:36 [multiproc_executor.py:989] WorkerProc hit an exception.
...
(Worker pid=18026) ERROR 06-13 19:57:36 [multiproc_executor.py:989] ValueError: Available memory on node 0 (4.66/18.0 GiB) on kv cache allocation is less than requested memory for kv (-0.39/3.6 GiB). Reduce CPU memory used by other processes.
```

之前的错误发生在启动前的内存检查。这里Worker进程已经启动，加载模型权重也已经成功，开始kv cache allocation，但是发现内存不够，所以失败了。从日志看，预留的CPU内存（3.6），减去模型权重使用的内存，留给kv cache已经是负的了（-0.39）。

vLLM CPU模式下整个系统内存组成：
* 非vLLM
  * 操作系统
  * 其他程序
* vLLM
  * 模型权重
  * KV Cache
  * 运行时开销

Qwen2.5-1.5B的参数量1.54B，如果是默认FP16/BF16，内存占用1.54B * 2 bytes=3.08 GiB，再加上其他一些开销，实际模型加载后通常会更多，以至于超过预留的3.6了。有一个解决办法是预留更多的内存。

### 将gpu-memory-utilization提高到0.3

预留更多的内存，启动成功。

18 * 0.3 = 4.8

关键的日志如下：

```
vllm on  main via △ via 🐍 v3.12.13 (vllm) took 1m12s
✦ ❯ vllm serve Qwen/Qwen2.5-1.5B-Instruct --gpu-memory-utilization 0.3
INFO 06-13 20:03:12 [importing.py:81] Triton not installed or not compatible; certain GPU-related functions will not be available.
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]        █     █     █▄   ▄█
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]  ▄▄ ▄█ █     █     █ ▀▄▀ █  version 0.22.1rc1.dev495+g2ecf7d0eb
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]   █▄█▀ █     █     █     █  model   Qwen/Qwen2.5-1.5B-Instruct
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]    ▀▀  ▀▀▀▀▀ ▀▀▀▀▀ ▀     ▀
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:339]
(APIServer pid=19786) INFO 06-13 20:03:12 [api_utils.py:273] non-default args: {'model_tag': 'Qwen/Qwen2.5-1.5B-Instruct', 'model': 'Qwen/Qwen2.5-1.5B-Instruct', 'gpu_memory_utilization': 0.3}
(APIServer pid=19786) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
(APIServer pid=19786) INFO 06-13 20:03:16 [model.py:598] Resolved architecture: Qwen2ForCausalLM
(APIServer pid=19786) INFO 06-13 20:03:16 [model.py:1723] Using max model len 32768
(APIServer pid=19786) INFO 06-13 20:03:16 [vllm.py:998] Asynchronous scheduling is enabled.
(APIServer pid=19786) INFO 06-13 20:03:16 [kernel.py:272] Final IR op priority after setting platform defaults: IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native'])
INFO 06-13 20:03:22 [importing.py:81] Triton not installed or not compatible; certain GPU-related functions will not be available.
(EngineCore pid=19838) INFO 06-13 20:03:22 [core.py:114] Initializing a V1 LLM engine (v0.22.1rc1.dev495+g2ecf7d0eb) with config: model='Qwen/Qwen2.5-1.5B-Instruct', speculative_config=None, tokenizer='Qwen/Qwen2.5-1.5B-Instruct', skip_tokenizer_init=False, tokenizer_mode=auto, revision=None, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=32768, download_dir=None, load_format=auto, tensor_parallel_size=1, pipeline_parallel_size=1, data_parallel_size=1, decode_context_parallel_size=1, dcp_comm_backend=ag_rs, disable_custom_all_reduce=True, quantization=None, quantization_config=None, enforce_eager=False, enable_return_routed_experts=False, kv_cache_dtype=auto, device_config=cpu, structured_outputs_config=StructuredOutputsConfig(backend='auto', disable_any_whitespace=False, disable_additional_properties=False, reasoning_parser='', reasoning_parser_plugin='', enable_in_reasoning=False), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None, kv_cache_metrics=False, kv_cache_metrics_sample=0.01, cudagraph_metrics=False, enable_layerwise_nvtx_tracing=False, enable_mfu_metrics=False, enable_mm_processor_stats=False, enable_logging_iteration_details=False), seed=0, served_model_name=Qwen/Qwen2.5-1.5B-Instruct, enable_prefix_caching=True, enable_chunked_prefill=True, pooler_config=None, compilation_config={'mode': <CompilationMode.DYNAMO_TRACE_ONCE: 2>, 'debug_dump_path': None, 'cache_dir': '', 'compile_cache_save_format': 'binary', 'backend': 'inductor', 'custom_ops': ['none', '+gelu'], 'ir_enable_torch_wrap': False, 'splitting_ops': [], 'compile_mm_encoder': False, 'cudagraph_mm_encoder': False, 'encoder_cudagraph_token_budgets': [], 'encoder_cudagraph_max_vision_items_per_batch': 0, 'encoder_cudagraph_max_frames_per_batch': None, 'compile_sizes': None, 'compile_ranges_endpoints': [2048], 'inductor_compile_config': {'enable_auto_functionalized_v2': False, 'size_asserts': False, 'alignment_asserts': False, 'scalar_asserts': False, 'dce': True, 'nan_asserts': False, 'epilogue_fusion': True, 'cpp.dynamic_threads': True}, 'inductor_passes': {}, 'cudagraph_mode': <CUDAGraphMode.NONE: 0>, 'cudagraph_num_of_warmups': 0, 'cudagraph_capture_sizes': [], 'cudagraph_copy_inputs': False, 'cudagraph_specialize_lora': True, 'use_inductor_graph_partition': False, 'pass_config': {'fuse_norm_quant': False, 'fuse_act_quant': False, 'fuse_attn_quant': False, 'enable_sp': False, 'fuse_gemm_comms': False, 'fuse_allreduce_rms': False, 'fuse_rope_kvcache_cat_mla': False, 'fuse_act_padding': False}, 'max_cudagraph_capture_size': None, 'dynamic_shapes_config': {'type': <DynamicShapesType.BACKED: 'backed'>, 'evaluate_guards': False, 'assume_32_bit_indexing': False}, 'local_cache_dir': None, 'fast_moe_cold_start': False, 'static_all_moe_layers': []}, kernel_config=KernelConfig(ir_op_priority=IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native']), enable_flashinfer_autotune=True, moe_backend='auto', linear_backend='auto')
(EngineCore pid=19838) INFO 06-13 20:03:22 [multiproc_executor.py:140] DP group leader: node_rank=0, node_rank_within_dp=0, master_addr=127.0.0.1, mq_connect_ip=192.168.3.31 (local), world_size=1, local_world_size=1
(EngineCore pid=19838) INFO 06-13 20:03:22 [ompmultiprocessing.py:185] OpenMP thread binding info:
(EngineCore pid=19838) INFO 06-13 20:03:22 [ompmultiprocessing.py:185] 	VLLM_CPU_OMP_THREADS_BIND='auto', auto_setup=True, skip_setup=False
(EngineCore pid=19838) INFO 06-13 20:03:22 [ompmultiprocessing.py:185] 	local_world_size=1, reserve_cpu_num=1
(EngineCore pid=19838) INFO 06-13 20:03:22 [ompmultiprocessing.py:185] 	local_rank=0, core ids=[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
(EngineCore pid=19838) INFO 06-13 20:03:22 [ompmultiprocessing.py:185] 	reserved_cpus=[10]
INFO 06-13 20:03:24 [importing.py:81] Triton not installed or not compatible; certain GPU-related functions will not be available.
(Worker pid=19850) INFO 06-13 20:03:25 [parallel_state.py:1568] world_size=1 rank=0 local_rank=0 distributed_init_method=tcp://127.0.0.1:54139 backend=gloo
(Worker pid=19850) INFO 06-13 20:03:25 [parallel_state.py:1903] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, PCP rank 0, TP rank 0, EP rank N/A, EPLB rank N/A
(Worker pid=19850) INFO 06-13 20:03:25 [cpu_model_runner.py:104] Starting to load model Qwen/Qwen2.5-1.5B-Instruct...
(Worker pid=19850) INFO 06-13 20:03:25 [selector.py:138] Using HND KV cache layout for CPU_ATTN backend.
(Worker pid=19850) WARNING 06-13 20:03:25 [compilation.py:1301] Op 'gelu' not present in model, enabling with '+gelu' has no effect
(Worker pid=19850) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
(Worker pid=19850) INFO 06-13 20:03:27 [weight_utils.py:575] No model.safetensors.index.json found in remote.
(Worker pid=19850) INFO 06-13 20:03:27 [weight_utils.py:850] Filesystem type for checkpoints: unknown. Checkpoint size: 2.88 GiB. Available RAM: 5.66 GiB.
(Worker pid=19850) INFO 06-13 20:03:27 [weight_utils.py:873] Auto-prefetch is disabled because the filesystem (unknown) is not a recognized network FS (NFS/Lustre). If you want to force prefetching, start vLLM with --safetensors-load-strategy=prefetch.
Loading safetensors checkpoint shards:   0% Completed | 0/1 [00:00<?, ?it/s]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:02<00:00,  2.17s/it]
Loading safetensors checkpoint shards: 100% Completed | 1/1 [00:02<00:00,  2.17s/it]
(Worker pid=19850)
(Worker pid=19850) INFO 06-13 20:03:29 [default_loader.py:397] Loading weights took 2.18 seconds
(Worker pid=19850) INFO 06-13 20:03:29 [cpu_model_runner.py:121] Warming up model for the compilation...
(Worker pid=19850) WARNING 06-13 20:03:29 [decorators.py:321] Compiling model again due to a load failure from /Users/maochenxiao/.cache/vllm/torch_compile_cache/torch_aot_compile/7428f7386b91d44679a0ea270d7e2057ce44fdf498f78bba176b8266ffd795e2/rank_0_0/model, reason: 'function' object has no attribute 'finalize_loading'
(Worker pid=19850) INFO 06-13 20:03:32 [decorators.py:708] saved AOT compiled function to /Users/maochenxiao/.cache/vllm/torch_compile_cache/torch_aot_compile/7428f7386b91d44679a0ea270d7e2057ce44fdf498f78bba176b8266ffd795e2/rank_0_0/model
(Worker pid=19850) INFO 06-13 20:03:56 [monitor.py:81] Initial profiling/warmup run took 24.28 s
(Worker pid=19850) INFO 06-13 20:03:57 [cpu_model_runner.py:125] Warming up done.
(Worker pid=19850) INFO 06-13 20:03:57 [cpu_worker.py:235] Auto set (1.48/18.0) GiB for KV cache on node 0, with 5.4 GiB requested memory for the worker. 3.92 GiB memory was consumed by non-kv usages.
(EngineCore pid=19838) INFO 06-13 20:03:57 [kv_cache_utils.py:2078] GPU KV cache size: 55,424 tokens
(EngineCore pid=19838) INFO 06-13 20:03:57 [kv_cache_utils.py:2079] Maximum concurrency for 32,768 tokens per request: 1.69x
(EngineCore pid=19838) INFO 06-13 20:03:57 [core.py:322] init engine (profile, create kv cache, warmup model) took 28.12 s
(EngineCore pid=19838) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
(EngineCore pid=19838) INFO 06-13 20:04:02 [vllm.py:998] Asynchronous scheduling is disabled.
(EngineCore pid=19838) WARNING 06-13 20:04:02 [vllm.py:1096] Inductor compilation was disabled by user settings, optimizations settings that are only active during inductor compilation will be ignored.
(EngineCore pid=19838) INFO 06-13 20:04:02 [kernel.py:272] Final IR op priority after setting platform defaults: IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native'])
(APIServer pid=19786) INFO 06-13 20:04:02 [api_server.py:572] Supported tasks: ['generate']
(APIServer pid=19786) WARNING 06-13 20:04:03 [model.py:1475] Default vLLM sampling parameters have been overridden by the model's `generation_config.json`: `{'repetition_penalty': 1.1, 'temperature': 0.7, 'top_k': 20, 'top_p': 0.8}`. If this is not intended, please relaunch vLLM instance with `--generation-config vllm`.
(APIServer pid=19786) INFO 06-13 20:04:08 [hf.py:548] Detected the chat template content format to be 'string'. You can set `--chat-template-content-format` to override this.
(APIServer pid=19786) INFO 06-13 20:04:09 [api_server.py:576] Starting vLLM server on http://0.0.0.0:8000
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:37] Available routes are:
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /openapi.json, Methods: HEAD, GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /docs, Methods: HEAD, GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /docs/oauth2-redirect, Methods: HEAD, GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /redoc, Methods: HEAD, GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /load, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /version, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /health, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /metrics, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /tokenize, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /detokenize, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/models, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /ping, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /ping, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /invocations, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/chat/completions, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/chat/completions/batch, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/responses, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/responses/{response_id}, Methods: GET
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/responses/{response_id}/cancel, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/completions, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/messages, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/messages/count_tokens, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /generative_scoring, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /inference/v1/generate, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /scale_elastic_ep, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /is_scaling_elastic_ep, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/chat/completions/render, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/completions/render, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/chat/completions/derender, Methods: POST
(APIServer pid=19786) INFO 06-13 20:04:09 [launcher.py:46] Route: /v1/completions/derender, Methods: POST
(APIServer pid=19786) INFO:     Started server process [19786]
(APIServer pid=19786) INFO:     Waiting for application startup.
(APIServer pid=19786) INFO:     Application startup complete.
```

最终，三个进程APIServer、EngineCore、Worker，都启动成功。


### 降低模型规模

使用Qwen/Qwen3-0.6B：

```
vllm serve Qwen/Qwen3-0.6B --gpu-memory-utilization 0.2
```

报错，启动失败。关键日志如下：

```
ValueError: To serve at least one request with the model's max seq len (40960), (4.38 GiB KV cache is needed, which is larger than the available KV cache memory (1.31 GiB). Based on the available memory, the estimated maximum model length is 12288. Try increasing `gpu_memory_utilization` (which also controls CPU memory on the CPU backend) or decreasing `max_model_len` when initializing the engine. See https://docs.vllm.ai/en/latest/configuration/conserving_memory/ for more details.
```

为什么0.6B的模型，需要4.38GiB的KV cache呢？

对于标准Transformer：
* 如果context length是40960，意味着要为40960个token缓存K和V
* K和V意味着token数量的2倍
* K和V分别是用1024长的向量表示（Qwen3-0.6B使用类似的数量级）
* 向量中的每个数值如果是BF16，则占用2个字节
* 并且，每个attention层都要缓存。（Qwen3-0.6B有28个transformer层，也即28个attention层）

所以KV cache的计算公式是：

```
context_length      // 40960
* K_and_V           // 2: K and V
* embedding_size    // 1024: vector length of K or V
* dtype_size        // 2: BF16, how many bytes for each number in the vector
* number_of_layers  // 28
```

(40960 * 2 * 1024 * 2 * 28) / (1024 * 1024 * 1024) = 4.38 GiB

Qwen3并不是传统MHA，而是GQA(Grouped Query Attention)：

```
KV Cache
=
context_length  // 40960 
* K_and_V       // 2
* num_kv_heads  // 8
* head_dim      // 128
* dtype_size    // 2
* num_layers    // 28
```

(40960 * 2 * 8 * 128 * 2 * 28) / (1024 * 1024 * 1024) = 4.38 GiB

### 降低上下文长度

使用报错信息中推荐的长度12288.

```
vllm serve Qwen/Qwen3-0.6B --gpu-memory-utilization 0.2 --max_model_len 12288
```

启动成功。如果用更小的值，自然也能成功，比如4096:

```
vllm serve Qwen/Qwen3-0.6B --gpu-memory-utilization 0.2 --max_model_len 4096
```

## Summary

启动过程中碰到3类问题：

* 系统可用内存不够，小于vLLM预留内存 -> 增大系统可用内存，或者降低vLLM预留内存
* vLLM拿到了预留内存，但是模型权重占用太多，KV cache不够 -> 调大vLLM预留内存，或者使用更小的模型
* 使用小的模型，但是上下文很长，权重加载成功，KV cache不够 -> 调小上下文长度

几点收获：

* 这三类问题都是内存相关的，可以粗粗地一窥看到vLLM的内存。
* 就内存而言，Training时代，参数量决定成本；Inference时代，KV Cache决定成本。
* 启动成功后，能够看到本地部署的进程模型，APIServer、EngineCore、Worker。

# References

* https://docs.vllm.ai/en/latest/getting_started/installation/cpu/
* https://en.wikipedia.org/wiki/Apple_silicon
* https://huggingface.co/Qwen/Qwen2.5-1.5B
