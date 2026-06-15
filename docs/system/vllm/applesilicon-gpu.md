# vlLM on Apple silicon GPU

## Background

### vLLM-Metal
For GPU-accelerated inference on Apple Silicon, use vLLM-Metal, a community-maintained hardware plugin that uses MLX as the compute backend and provides native GPU acceleration via Apple's Metal framework.

vLLM-Metal works with MLX-optimized models from the mlx-community organization on Hugging Face, which provides quantized versions of popular models optimized for Apple Silicon.

### NVIDIA vs Apple

| 抽象层            | NVIDIA                 | Apple                 |
| -------------- | ---------------------- | --------------------- |
| Serving        | vLLM / SGLang          | vLLM-Metal            |
| Runtime        | PyTorch                | MLX                   |
| Kernel Library | cuBLAS / cuDNN         | MPS                   |
| Compiler       | TorchInductor / Triton | MLX Graph / MPS Graph |
| Driver API     | CUDA                   | Metal                 |
| Hardware       | H100、B200 等GPU         | M1/M2/M3/M4 GPU       |

不过对于 vLLM Metal 当前实现，还有一个非常关键的事实：它并不是简单地“把 CUDA 换成 Metal”。实际上很多 CUDA 路径里的核心组件（PagedAttention、CUDA Graph、Triton Kernel、FlashAttention 等）在 Apple 平台都不存在。因此 vLLM Metal 更接近：vLLM Scheduler + MLX Execution Engine，也就是说：

* vLLM 保留了请求调度层（Scheduler）
* 大量执行层能力交给 MLX
* 不再沿用 CUDA 生态中的 Triton/FlashAttention/CUDA Graph 体系

## Install

vLLM core is compiled from source via clang++. The Metal kernels ship prebuilt, so no Metal compiler or toolchain is needed to run them.

```
curl -fsSL https://raw.githubusercontent.com/vllm-project/vllm-metal/main/install.sh | bash
```

关键日志如下：

```
curl -fsSL https://raw.githubusercontent.com/vllm-project/vllm-metal/main/install.sh | bash
=== Creating virtual environment ===
Using CPython 3.12.13
Creating virtual environment with seed packages at: .venv-vllm-metal
 + pip==26.1.2
Activate with: source .venv-vllm-metal/bin/activate
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 34.9M  100 34.9M    0     0   213k      0  0:02:47  0:02:47 --:--:--  283k
Using Python 3.12.13 environment at: /Users/maochenxiao/.venv-vllm-metal
Resolved 144 packages in 32.00s
Installed 144 packages in 684ms
 + aiohappyeyeballs==2.6.2
 + aiohttp==3.14.1
 + aiosignal==1.4.0
 ...
 + websockets==16.0
 + xgrammar==0.2.2
 + yarl==1.24.2
Using Python 3.12.13 environment at: /Users/maochenxiao/.venv-vllm-metal
Resolved 145 packages in 8.08s
      Built vllm @ file:///private/var/folders/7g/nmg6_hj17hn31qmn93dt0r400000gn/T/tmp.UUxQON0djU/vllm-0.23.0
Prepared 1 package in 41.91s
Installed 1 package in 51ms
 + vllm==0.23.0+cpu (from file:///private/var/folders/7g/nmg6_hj17hn31qmn93dt0r400000gn/T/tmp.UUxQON0djU/vllm-0.23.0)
/Users/maochenxiao
Fetching latest release...
Latest release: vllm_metal-0.3.0.dev20260614085908-cp312-cp312-macosx_11_0_arm64.whl
✓ Found latest release

Downloading wheel...
✓ Downloaded wheel
Using Python 3.12.13 environment at: .venv-vllm-metal
Resolved 79 packages in 3.51s
Prepared 11 packages in 14m 56s
Installed 18 packages in 310ms
 + accelerate==1.14.0
 + datasets==5.0.0
 + miniaudio==1.71
 ...
 + sounddevice==0.5.5
 + vllm-metal==0.3.0.dev20260614085908 (from file:///var/folders/7g/nmg6_hj17hn31qmn93dt0r400000gn/T/tmp.sm2H6MIT73/vllm_metal-0.3.0.dev20260614085908-cp312-cp312-macosx_11_0_arm64.whl)
 + xxhash==3.7.0
✓ Installed vllm-metal

✓ Installation complete!

To use vllm, activate the virtual environment:
  source /Users/maochenxiao/.venv-vllm-metal/bin/activate

Or add the venv to your PATH:
  export PATH="/Users/maochenxiao/.venv-vllm-metal/bin:$PATH"
```

官网说：Using the install script above, the following will be installed under the ~/.venv-vllm-metal directory (the default).

* vllm-metal plugin
* vllm core
* Related libraries

根据前面的日志解读：

* vllm-metal plugin：是从官网下载的wheel，没有本地build from source
* vllm core：在本地build from source
* Related libraries：就是依赖的那些包

## Serve

If you run source ~/.venv-vllm-metal/bin/activate, the vllm CLI becomes available and you can access the vLLM right away.

注意`gpu-memory-utilization`选项不起作用，要使用`export VLLM_METAL_MEMORY_FRACTION=0.2`

```
export VLLM_METAL_MEMORY_FRACTION=0.2
vllm serve Qwen/Qwen3-0.6B --max-model-len 4096
```

启动日志如下：

```
INFO 06-15 16:52:13 [__init__.py:44] Available plugins for group vllm.platform_plugins:
INFO 06-15 16:52:13 [__init__.py:46] - metal -> vllm_metal:register
INFO 06-15 16:52:13 [__init__.py:49] All plugins in this group will be loaded. Set `VLLM_PLUGINS` to control which plugins to load.
INFO 06-15 16:52:14 [__init__.py:238] Platform plugin metal is activated
INFO 06-15 16:52:15 [importing.py:81] Triton not installed or not compatible; certain GPU-related functions will not be available.
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]        █     █     █▄   ▄█
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]  ▄▄ ▄█ █     █     █ ▀▄▀ █  version 0.23.0
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]   █▄█▀ █     █     █     █  model   Qwen/Qwen3-0.6B
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]    ▀▀  ▀▀▀▀▀ ▀▀▀▀▀ ▀     ▀
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:339]
(APIServer pid=69600) INFO 06-15 16:52:16 [api_utils.py:273] non-default args: {'model_tag': 'Qwen/Qwen3-0.6B', 'max_model_len': 4096}
(APIServer pid=69600) WARNING 06-15 16:52:16 [system_utils.py:299] Found ulimit of 2048 and failed to automatically increase with error current limit exceeds maximum limit. This can cause fd limit errors like `OSError: [Errno 24] Too many open files`. Consider increasing with ulimit -n
(APIServer pid=69600) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
(APIServer pid=69600) INFO 06-15 16:52:19 [model.py:611] Resolved architecture: Qwen3ForCausalLM
(APIServer pid=69600) INFO 06-15 16:52:19 [model.py:1745] Using max model len 4096
(APIServer pid=69600) INFO 06-15 16:52:19 [vllm.py:999] Asynchronous scheduling is enabled.
(APIServer pid=69600) INFO 06-15 16:52:19 [kernel.py:270] Final IR op priority after setting platform defaults: IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native'])
(APIServer pid=69600) INFO 06-15 16:52:19 [platform.py:394] Metal: chunked prefill enabled (paged attention), max_num_batched_tokens=2048
(APIServer pid=69600) INFO 06-15 16:52:19 [platform.py:467] Metal memory: 3.9GB total, 2.0GB available
(APIServer pid=69600) WARNING 06-15 16:52:19 [vllm.py:528] Model Runner V2 requires Triton; using the V1 model runner instead.
INFO 06-15 16:52:26 [__init__.py:44] Available plugins for group vllm.platform_plugins:
INFO 06-15 16:52:26 [__init__.py:46] - metal -> vllm_metal:register
INFO 06-15 16:52:26 [__init__.py:49] All plugins in this group will be loaded. Set `VLLM_PLUGINS` to control which plugins to load.
INFO 06-15 16:52:26 [__init__.py:238] Platform plugin metal is activated
INFO 06-15 16:52:26 [importing.py:81] Triton not installed or not compatible; certain GPU-related functions will not be available.
(EngineCore pid=69652) INFO 06-15 16:52:27 [core.py:113] Initializing a V1 LLM engine (v0.23.0) with config: model='Qwen/Qwen3-0.6B', speculative_config=None, tokenizer='Qwen/Qwen3-0.6B', skip_tokenizer_init=False, tokenizer_mode=auto, revision=None, tokenizer_revision=None, trust_remote_code=False, dtype=torch.bfloat16, max_seq_len=4096, download_dir=None, load_format=auto, tensor_parallel_size=1, pipeline_parallel_size=1, data_parallel_size=1, decode_context_parallel_size=1, dcp_comm_backend=ag_rs, disable_custom_all_reduce=True, quantization=None, quantization_config=None, enforce_eager=False, enable_return_routed_experts=False, kv_cache_dtype=auto, device_config=cpu, structured_outputs_config=StructuredOutputsConfig(backend='auto', disable_any_whitespace=False, disable_additional_properties=False, reasoning_parser='', reasoning_parser_plugin='', enable_in_reasoning=False), observability_config=ObservabilityConfig(show_hidden_metrics_for_version=None, otlp_traces_endpoint=None, collect_detailed_traces=None, kv_cache_metrics=False, kv_cache_metrics_sample=0.01, cudagraph_metrics=False, enable_layerwise_nvtx_tracing=False, enable_mfu_metrics=False, enable_mm_processor_stats=False, enable_logging_iteration_details=False), seed=0, served_model_name=Qwen/Qwen3-0.6B, enable_prefix_caching=True, enable_chunked_prefill=True, pooler_config=None, compilation_config={'mode': <CompilationMode.VLLM_COMPILE: 3>, 'debug_dump_path': None, 'cache_dir': '', 'compile_cache_save_format': 'binary', 'backend': 'inductor', 'custom_ops': ['none'], 'ir_enable_torch_wrap': True, 'splitting_ops': ['vllm::unified_attention_with_output', 'vllm::unified_mla_attention_with_output', 'vllm::mamba_mixer2', 'vllm::mamba_mixer', 'vllm::short_conv', 'vllm::linear_attention', 'vllm::plamo2_mamba_mixer', 'vllm::qwen_gdn_attention_core', 'vllm::gdn_attention_core_xpu', 'vllm::olmo_hybrid_gdn_full_forward', 'vllm::kda_attention', 'vllm::sparse_attn_indexer', 'vllm::rocm_aiter_sparse_attn_indexer', 'vllm::deepseek_v4_attention', 'vllm::unified_kv_cache_update', 'vllm::unified_mla_kv_cache_update'], 'compile_mm_encoder': False, 'cudagraph_mm_encoder': False, 'encoder_cudagraph_token_budgets': [], 'encoder_cudagraph_max_vision_items_per_batch': 0, 'encoder_cudagraph_max_frames_per_batch': None, 'compile_sizes': None, 'compile_ranges_endpoints': [2048], 'inductor_compile_config': {'enable_auto_functionalized_v2': False, 'size_asserts': False, 'alignment_asserts': False, 'scalar_asserts': False, 'combo_kernels': True, 'benchmark_combo_kernel': True}, 'inductor_passes': {}, 'cudagraph_mode': <CUDAGraphMode.NONE: 0>, 'cudagraph_num_of_warmups': 0, 'cudagraph_capture_sizes': None, 'cudagraph_copy_inputs': False, 'cudagraph_specialize_lora': True, 'use_inductor_graph_partition': False, 'pass_config': {'fuse_norm_quant': False, 'fuse_act_quant': False, 'fuse_attn_quant': False, 'enable_sp': False, 'fuse_gemm_comms': False, 'fuse_allreduce_rms': False, 'fuse_rope_kvcache_cat_mla': False, 'fuse_act_padding': False}, 'max_cudagraph_capture_size': None, 'dynamic_shapes_config': {'type': <DynamicShapesType.BACKED: 'backed'>, 'evaluate_guards': False, 'assume_32_bit_indexing': False}, 'local_cache_dir': None, 'fast_moe_cold_start': False, 'static_all_moe_layers': []}, kernel_config=KernelConfig(ir_op_priority=IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native']), enable_flashinfer_autotune=True, moe_backend='auto', linear_backend='auto')
(EngineCore pid=69652) INFO 06-15 16:52:27 [worker.py:123] MLX device set to: Device(gpu, 0)
mx.metal.device_info is deprecated and will be removed in a future version. Use mx.device_info instead.
(EngineCore pid=69652) INFO 06-15 16:52:27 [utils.py:73] Set Metal wired_limit to 13.3 GB
(EngineCore pid=69652) INFO 06-15 16:52:27 [worker.py:131] PyTorch device set to: mps
(EngineCore pid=69652) INFO 06-15 16:52:27 [parallel_state.py:1568] world_size=1 rank=0 local_rank=0 distributed_init_method=tcp://192.168.3.31:61215 backend=gloo
(EngineCore pid=69652) INFO 06-15 16:52:27 [parallel_state.py:1903] rank 0 in world size 1 is assigned as DP rank 0, PP rank 0, PCP rank 0, TP rank 0, EP rank N/A, EPLB rank N/A
(EngineCore pid=69652) Warning: You are sending unauthenticated requests to the HF Hub. Please set a HF_TOKEN to enable higher rate limits and faster downloads.
(EngineCore pid=69652) INFO 06-15 16:52:28 [model_lifecycle.py:151] Loading model: Qwen/Qwen3-0.6B (VLM: False)
Fetching 7 files: 100%|██████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 7/7 [00:00<00:00, 10247.86it/s]
Download complete: : 0.00B [00:00, ?B/s]                                                                                                                                                    | 0/7 [00:00<?, ?it/s]
(EngineCore pid=69652) INFO 06-15 16:52:29 [model_lifecycle.py:207] Model loaded in 1.01s: Qwen/Qwen3-0.6B
(EngineCore pid=69652) INFO 06-15 16:52:29 [cache_policy.py:580] Paged attention memory breakdown: metal_limit=14.30GB, fraction=0.2, usable_metal=2.86GB, model_memory=1.19GB, overhead=0.79GB, kv_budget=0.88GB, per_block_bytes=1835008, num_blocks=479, max_tokens_cached=7664
(EngineCore pid=69652) INFO 06-15 16:52:29 [kv_cache.py:167] KV cache: 879.0 MB (28 layers, 479 blocks, 16 tokens/block)
(EngineCore pid=69652) INFO 06-15 16:52:29 [cache_policy.py:606] Paged attention enabled: 28 layers patched, 479 blocks allocated (block_size=16, mla=False, turboquant=False, k_quant=N/A)
(EngineCore pid=69652) INFO 06-15 16:52:29 [cache_policy.py:647] Paged attention: reporting MPS cache capacity (479 blocks × 1835008 bytes = 0.88 GB)
(EngineCore pid=69652) INFO 06-15 16:52:29 [kv_cache_utils.py:1744] GPU KV cache size: 7,664 tokens
(EngineCore pid=69652) INFO 06-15 16:52:29 [kv_cache_utils.py:1745] Maximum concurrency for 4,096 tokens per request: 1.87x
(EngineCore pid=69652) INFO 06-15 16:52:29 [cache_policy.py:315] KV cache config received: 479 blocks (MLX manages cache internally)
(EngineCore pid=69652) INFO 06-15 16:52:29 [model_runner.py:815] Warming up model...
(EngineCore pid=69652) INFO 06-15 16:52:29 [model_runner.py:821] Model warm-up complete
(EngineCore pid=69652) INFO 06-15 16:52:29 [__init__.py:256] Warming up v2 paged-attention Metal kernels...
(EngineCore pid=69652) INFO 06-15 16:52:29 [__init__.py:237] Native paged-attention Metal kernels loaded
(EngineCore pid=69652) INFO 06-15 16:52:29 [__init__.py:264] Paged-attention Metal kernel warm-up complete
(EngineCore pid=69652) INFO 06-15 16:52:29 [core.py:306] init engine (profile, create kv cache, warmup model) took 0.77 s (compilation: 0.03 s)
(EngineCore pid=69652) WARNING 06-15 16:52:33 [vllm.py:528] Model Runner V2 requires Triton; using the V1 model runner instead.
(EngineCore pid=69652) INFO 06-15 16:52:33 [vllm.py:999] Asynchronous scheduling is enabled.
(EngineCore pid=69652) INFO 06-15 16:52:33 [kernel.py:270] Final IR op priority after setting platform defaults: IrOpPriorityConfig(rms_norm=['native'], fused_add_rms_norm=['native'])
(EngineCore pid=69652) INFO 06-15 16:52:33 [platform.py:394] Metal: chunked prefill enabled (paged attention), max_num_batched_tokens=2048
(EngineCore pid=69652) INFO 06-15 16:52:33 [platform.py:467] Metal memory: 3.9GB total, 1.3GB available
(APIServer pid=69600) INFO 06-15 16:52:33 [api_server.py:579] Supported tasks: ['generate']
(APIServer pid=69600) WARNING 06-15 16:52:34 [model.py:1502] Default vLLM sampling parameters have been overridden by the model's `generation_config.json`: `{'temperature': 0.6, 'top_k': 20, 'top_p': 0.95}`. If this is not intended, please relaunch vLLM instance with `--generation-config vllm`.
(APIServer pid=69600) INFO 06-15 16:52:39 [hf.py:548] Detected the chat template content format to be 'string'. You can set `--chat-template-content-format` to override this.
(APIServer pid=69600) INFO 06-15 16:52:40 [api_server.py:583] Starting vLLM server on http://0.0.0.0:8000
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:37] Available routes are:
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:46] Route: /openapi.json, Methods: HEAD, GET
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:46] Route: /docs, Methods: HEAD, GET
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:46] Route: /docs/oauth2-redirect, Methods: HEAD, GET
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:46] Route: /redoc, Methods: HEAD, GET
(APIServer pid=69600) INFO 06-15 16:52:40 [launcher.py:46] Route: /metrics, Methods: GET
(APIServer pid=69600) INFO:     Started server process [69600]
(APIServer pid=69600) INFO:     Waiting for application startup.
(APIServer pid=69600) INFO:     Application startup complete.
```

注意：

* 只有两个进程：APIServer、EngineCore，没有Worker
* Platform plugin metal is activated. 对Metal的支持是通过platform plugin机制实现的。详细内容可参见官方文档Developer Guide/Design Documents/Plugins。
* Paged attention memory breakdown: metal_limit=14.30GB, fraction=0.2, usable_metal=2.86GB, model_memory=1.19GB, overhead=0.79GB, kv_budget=0.88GB, per_block_bytes=1835008, num_blocks=479, max_tokens_cached=7664

For how to use the vllm CLI, please refer to the official vLLM guide. https://docs.vllm.ai/en/latest/cli/

## Reference

* https://docs.vllm.ai/en/latest/getting_started/installation/gpu/
* https://github.com/vllm-project/vllm-metal
* https://docs.vllm.ai/en/latest/cli/
* https://github.com/vllm-project/vllm-metal/blob/main/docs/supported_models.md
* https://huggingface.co/mlx-community
* https://docs.vllm.ai/en/latest/design/plugin_system/
