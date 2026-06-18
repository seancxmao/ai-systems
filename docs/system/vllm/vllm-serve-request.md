# vLLM Serve Request

## Background


## 请求处理流程

```text
Client
   ↓
----------------
API Server
   ↓
FastAPI Route
   ↓
AsyncLLM
   ↓
----------------
EngineCore
   ↓
Scheduler
   ↓
Executor
   ↓
----------------
Worker
   ↓
Model Forward
```

## Client

```
curl http://localhost:8000/v1/completions \
   -H "Content-Type: application/json" \
   -d '{
      "model": "Qwen/Qwen3-0.6B",
      "prompt": "San Francisco is a",
      "max_tokens": 7,
      "temperature": 0 
   }'
```

## API Server

API Server 把 OpenAI 请求转换成 Engine 能理解的 Request.

```
Client
   ↓
----------------------
uvicorn/FastAPI
    ↓
vllm/entrypoints/openai/completion/api_server.py
OpenAIServingCompletion.create_completion()
    ↓
engine_client.generate() // engine_client = AsyncLLM
    ↓
engine_client.add_request()
    ↓
AsyncLLM.engine_core.add_request_async() // engine_core = AsyncMPClient
    ↓
---------------------
EngineCore
```

### OpenAIServingCompletion.create_completion()

这里是 OpenAI API 的业务入口。可以看到原始 HTTP 请求已经被解析成 CompletionRequest，包括 prompt、temperature、max_tokens 等参数，是整个预处理链路的起点。

* CompletionRequest (vllm/entrypoints/openai/completion/protocol.py)

### AsyncLLM.generate()

* vllm.v1.engine.async_llm.AsyncLLM
* AsyncLLM(EngineClient)

EngineClient是明确定义的协议，从Client到Engine

```python
// vllm/engine/protocol.py

@dataclass
class StreamingInput:
    """Input data for a streaming generation request.

    This is used with generate() to support multi-turn streaming sessions
    where inputs are provided via an async generator.
    """
    ...

class EngineClient(ABC):
    """Protocol class for Clients to Engine"""
    ...
```

AsyncLLMEngine是AsyncLLM的别名：

```python
// vllm/engine/async_llm_engine.py

from vllm.v1.engine.async_llm import AsyncLLM

AsyncLLMEngine = AsyncLLM  # type: ignore
"""The `AsyncLLMEngine` class is an alias of [vllm.v1.engine.async_llm.AsyncLLM][]."""
```

`AsyncLLM.generate()`的参数`EngineInput`:

```python
// vllm/inputs/engine.py

EngineInput: TypeAlias = DecoderOnlyEngineInput | EncoderDecoderInput
"""
A rendered [`PromptType`][vllm.inputs.llm.PromptType]
which can be passed to `LLMEngine.add_request` or `AsyncLLM.add_request`.
"""
```

### AsyncLLM.add_request()

最终API Server通过`AsyncLLM.add_request()` 把请求提交给 EngineCore。`AsyncLLM`内部通过`AsyncMPClient`进行RPC。

## EngineCore

对于 `/v1/completions` 请求，在进入 `EngineCore.add_request()` 之前，prompt 通常已经完成了 tokenization，并被转换成 `prompt_token_ids`。这部分工作发生在 OpenAI serving layer 和 processor 之间（例如 `InputPreprocessor`、tokenizer 相关逻辑），因此 `EngineCore` 接收到的已经是 token 序列，而不是原始文本；`EngineCore` 主要负责调度、KV Cache 管理和模型执行，不再负责 tokenizer。

### EngineCore.add_request()

这是请求正式进入 EngineCore 的入口。这里可以看到已经完成 tokenization 的 prompt_token_ids、sampling params、request_id 等信息，并且会创建对应的 Request/Sequence 对象加入调度器管理。

* engine core = vllm.v1.engine.core.EngineCoreProc(EngineCore)
* request: vllm.v1.request.Request

核心逻辑：

```py
class EngineCore:

   def add_request(self, request: Request, request_wave: int = 0):
        """Add request to the scheduler.

        `request_wave`: indicate which wave of requests this is expected to
        belong to in DP case
        """
        ...
        self.scheduler.add_request(request)
        ...
```

### Scheduler.schedule()

这是 vLLM 最核心的断点。这里决定：

* 哪些请求本轮运行
* 哪些请求等待
* 是 Prefill 还是 Decode
* 分配多少 token budget
* 分配多少 KV Cache block

理解了这里，就理解了 vLLM 的 continuous batching。

```py
// vllm/v1/core/sched/scheduler.py
class Scheduler:
   def schedule(self) -> SchedulerOutput:
        self.current_step += 1
        # NOTE(woosuk) on the scheduling algorithm:
        # There's no "decoding phase" nor "prefill phase" in the scheduler.
        # Each request just has the num_computed_tokens and
        # num_tokens_with_spec. num_tokens_with_spec =
        # len(prompt_token_ids) + len(output_token_ids) + len(spec_token_ids).
        # At each step, the scheduler tries to assign tokens to the requests
        # so that each request's num_computed_tokens can catch up its
        # num_tokens_with_spec. This is general enough to cover
        # chunked prefills, prefix caching, speculative decoding,
        # and the "jump decoding" optimization in the future.
        ...
```

### KV Cache Block 分配

通常在：

```
KVCacheManager.allocate_slots(...)
```

这里决定：

* 需要多少 KV blocks
* 是否有足够显存
* block table 如何建立

这是理解 PagedAttention 的最佳入口，因为之后 Worker 执行时使用的就是这里构造出来的 block mapping。

```py
// vllm/v1/core/kv_cache_manager.py
class KVCacheManager:

    def allocate_slots(
        self,
        request: Request,
        num_new_tokens: int,
        num_new_computed_tokens: int = 0,
        new_computed_blocks: KVCacheBlocks | None = None,
        num_lookahead_tokens: int = 0,
        num_external_computed_tokens: int = 0,
        delay_cache_blocks: bool = False,
        num_encoder_tokens: int = 0,
        full_sequence_must_fit: bool = False,
        reserved_blocks: int = 0,
        has_scheduled_reqs: bool = True,
    ) -> KVCacheBlocks | None:
        """Add slots for a request with new tokens to append.

        Args:
            request: The request to allocate slots.
            num_new_tokens: The number of new tokens to be allocated and computed.
            num_new_computed_tokens: The number of new computed tokens just
                hitting the prefix caching, excluding external tokens.
            new_computed_blocks: The cached blocks for the above new computed
                tokens, grouped as a tuple by kv cache groups.
            num_lookahead_tokens: The number of speculative tokens to allocate.
                This is used by spec decode proposers with kv-cache such
                as eagle.
            num_external_computed_tokens: The number of tokens that their
                KV caches are not cached by vLLM but cached by the connector.
            delay_cache_blocks: Whether to skip caching the blocks. This is
                used by P/D when allocating blocks used in a KV transfer
                which will complete in a future step.
            num_encoder_tokens: The number of encoder tokens to allocate for
                cross-attention in encoder-decoder models(e.g., Whisper).
                For decoder-only models, this should be 0.
            full_sequence_must_fit: Only allocate blocks if the KV cache has enough
                free blocks to hold the full sequence, accounting for prefix cache hits
                and sliding window. Used as an admission gate to prevent over-admitting
                requests when chunked prefill would otherwise only check the first chunk
            reserved_blocks: Number of free blocks that must be left available for
                other in-flight sequences to complete. The actual allocation is only
                made if it fits within (free blocks - reserved_blocks). Used to gate
                async KV-connector loads so their initial allocation cannot consume
                blocks an already in-flight (prefilling) sequence is relying on.
            has_scheduled_reqs: Whether any requests are already scheduled to run
                this step, controls whether watermark is applied.

        Blocks layout:
        ```
        ----------------------------------------------------------------------
        | < comp > | < new_comp > | < ext_comp >  | < new >  | < lookahead > |
        ----------------------------------------------------------------------
                                                  |   < to be computed >     |
        ----------------------------------------------------------------------
                                  |            < to be allocated >           |
        ----------------------------------------------------------------------
                                  | < to be cached (roughly, |
                                  | details below)>          |
        ----------------------------------------------------------------------
        | Prefix-cached tokens from either vLLM   |
        | or connector. Can be safely removed if  |
        | they are outside sliding window.        |
        ----------------------------------------------------------------------
        |   < cached by vLLM >    | not cached by |
                                  | vLLM, but     |
        | ref_cnt  | ref_cnt not  | cached by     |
        | increased| increased yet| connector     |
        ----------------------------------------------------------------------
        ```

        Abbrivations:

        ```
        comp      = request.num_computed_tokens
        new_comp  = num_new_computed_tokens
                  = len(new_computed_blocks) * block_size
        ext_comp  = num_external_computed_tokens, cached by the connector
        new       = num_new_tokens, including unverified draft tokens
        lookahead = num_lookahead_tokens
        ```

        NOTE: for new tokens which include both verified and unverified draft
        tokens, we only cache the verified tokens (by capping the number at
        `request.num_tokens`).

        The allocation has three stages:
        - Free unnecessary blocks in `comp` and check
           if we have sufficient free blocks (return None if not).
        - Handle prefix tokens (`comp + new_comp + ext_comp`):
            - Free unnecessary blocks (e.g. outside sliding window)
            - Allocate new blocks for `ext_comp` tokens inside
              sliding window
        - Allocate new blocks for tokens to be computed (`new + lookahead`)

        Returns:
            A list of new allocated blocks.
        """
        ...
        num_blocks_to_allocate = self.coordinator.get_num_blocks_to_allocate(
            request_id=request.request_id,
            num_tokens=full_num_tokens,
            new_computed_blocks=new_computed_block_list,
            num_encoder_tokens=num_encoder_tokens,
            total_computed_tokens=total_computed_tokens,
            num_tokens_main_model=full_num_tokens,
            apply_admission_cap=True,
        )
        ...
        new_blocks = self.coordinator.allocate_new_blocks(
            request.request_id,
            num_tokens_need_slot,
            num_tokens_main_model,
            num_encoder_tokens,
        )
```

相关的类：

* KVCacheCoordinator (vllm/v1/core/kv_cache_coordinator.py)

### Executor.execute_model()

```
Executor.execute_model(...)
```

实现类是：

```
MultiprocExecutor.execute_model(...)
```

或者

```
UniProcExecutor.execute_model(...)
```

这里是 EngineCore 与 Worker 的分界线。

此时：

* schedule 已完成
* batch 已构建完成
* KV Cache 已准备好
* model input 已准备好

接下来会通过 Executor 把一次执行请求发送给 Worker 进程。

```py
// MultiprocExecutor (vllm/v1/executor/multiproc_executor.py)
def execute_model(  # type: ignore[override]
    self, scheduler_output: SchedulerOutput, non_block: bool = False
) -> ModelRunnerOutput | None | Future[ModelRunnerOutput | None]:
    return self.collective_rpc(
        "execute_model",
        args=(scheduler_output,),
        unique_reply_rank=self.output_rank,
        non_block=non_block,
        timeout=envs.VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS,
        kv_output_aggregator=self.kv_output_aggregator,
    )
```

## Worker

现在正好站在 EngineCore → Worker 的边界。

对于 CPU backend（特别是 Mac 上的 `CPUWorker`），打断点。

### Worker RPC 入口

CPUWorker没有覆盖execute_model方法，实际执行的是父类的方法

```python
// vllm/v1/worker/gpu_worker.py
Worker.execute_model(...)
```

这里是 `MultiprocExecutor.collective_rpc("execute_model")` 在 Worker 侧的接收点。此时可以观察从 Scheduler 传过来的 `SchedulerOutput`，确认 batch 中有哪些 request、哪些 token 要执行。

```py
// vllm/v1/worker/gpu_worker.py
class Worker(WorkerBase):
    def execute_model(
        self, scheduler_output: "SchedulerOutput"
    ) -> ModelRunnerOutput | AsyncModelRunnerOutput | None:
        ...
        with self.annotate_profile(scheduler_output):
           output = self.model_runner.execute_model(
              scheduler_output, intermediate_tensors
           )
           if (
              self.use_v2_model_runner
              and self.model_runner.is_pooling_model
              and output is None
           ):
              output = self.model_runner.pool()  # type: ignore
           if isinstance(
              output, ModelRunnerOutput | AsyncModelRunnerOutput | NoneType
           ):
              return output
        ...
```

### ModelRunner.execute_model()

`CPUModelRunner`没有覆盖`execute_model`方法，实际执行的是父类的方法`GPUModelRunner.execute_model`。

这是 Worker 最重要的入口。这里开始把 SchedulerOutput 转换成真正的模型输入：

```text
SchedulerOutput
↓
input_ids
positions
attention metadata
↓
Model Forward
```

```py
class GPUModelRunner:
    def execute_model(
        self,
        scheduler_output: "SchedulerOutput",
        intermediate_tensors: IntermediateTensors | None = None,
    ) -> ModelRunnerOutput | AsyncModelRunnerOutput | IntermediateTensors | None:
        ...
        # Run the model.
        # Use persistent buffers for CUDA graphs.
        # When spec decode is enabled, defer connector finalization
        # (wait_for_save + clear metadata) until after draft model runs.
        defer_kv_connector_finalize = self.speculative_config is not None
        with (
            set_forward_context(
                attn_metadata,
                self.vllm_config,
                num_tokens=num_tokens_padded,
                num_tokens_across_dp=num_tokens_across_dp,
                cudagraph_runtime_mode=cudagraph_mode,
                batch_descriptor=batch_desc,
                ubatch_slices=ubatch_slices_padded,
                slot_mapping=slot_mappings,
                skip_compiled=has_encoder_input,
            ),
            record_function_or_nullcontext("gpu_model_runner: forward"),
            self.maybe_get_kv_connector_output(
                scheduler_output,
                defer_finalize=defer_kv_connector_finalize,
            ) as kv_connector_output,
        ):
            model_output = self._model_forward(
                input_ids=input_ids,
                positions=positions,
                intermediate_tensors=intermediate_tensors,
                inputs_embeds=inputs_embeds,
                **model_kwargs,
            )
        ...
```

### ModelRunner._model_forward

这里是真正开始执行 Transformer。

```py
class GPUModelRunner:
    def _model_forward(
        self,
        input_ids: torch.Tensor | None = None,
        positions: torch.Tensor | None = None,
        intermediate_tensors: IntermediateTensors | None = None,
        inputs_embeds: torch.Tensor | None = None,
        **model_kwargs: dict[str, Any],
    ) -> Any:
        """Helper method to call the model forward pass.

        This method can be overridden by subclasses for model execution.
        Motivation: We can inspect only this method versus
        the whole execute_model, which has additional logic.

        Args:
            input_ids: Input token IDs
            positions: Token positions
            intermediate_tensors: Tensors from previous pipeline stages
            inputs_embeds: Input embeddings (alternative to input_ids)
            **model_kwargs: Additional model arguments

        Returns:
            Model output tensor
        """
        return self.model(
            input_ids=input_ids,
            positions=positions,
            intermediate_tensors=intermediate_tensors,
            inputs_embeds=inputs_embeds,
            **model_kwargs,
        )
```

self.model = Qwen3ForCausalLM

### Qwen3ForCausalLM.forward()

调用内部的Qwen3Model(Qwen2Model)

```py
// vllm/model_executor/models/qwen3.py
class Qwen3ForCausalLM(
    LocalArgmaxMixin, nn.Module, SupportsLoRA, SupportsPP, SupportsEagle, SupportsEagle3
):
    packed_modules_mapping = {
        "qkv_proj": [
            "q_proj",
            "k_proj",
            "v_proj",
        ],
        "gate_up_proj": [
            "gate_proj",
            "up_proj",
        ],
    }

    embedding_modules = {
        "embed_tokens": "input_embeddings",
        "lm_head": "output_embeddings",
    }

    def __init__(self, *, vllm_config: VllmConfig, prefix: str = ""):
        super().__init__()
        config = vllm_config.model_config.hf_config
        quant_config = vllm_config.quant_config

        self.config = config

        self.vllm_config = vllm_config
        self.quant_config = quant_config
        self.model = Qwen3Model(
            vllm_config=vllm_config, prefix=maybe_prefix(prefix, "model")
        )

        if get_pp_group().is_last_rank:
            if config.tie_word_embeddings:
                self.lm_head = self.model.embed_tokens
            else:
                self.lm_head = ParallelLMHead(
                    config.vocab_size,
                    config.hidden_size,
                    quant_config=quant_config,
                    prefix=maybe_prefix(prefix, "lm_head"),
                )
        else:
            self.lm_head = PPMissingLayer()

        self.logits_processor = LogitsProcessor(config.vocab_size)

        self.make_empty_intermediate_tensors = (
            self.model.make_empty_intermediate_tensors
        )

    def embed_input_ids(self, input_ids: torch.Tensor) -> torch.Tensor:
        return self.model.embed_input_ids(input_ids)

    def forward(
        self,
        input_ids: torch.Tensor | None,
        positions: torch.Tensor,
        intermediate_tensors: IntermediateTensors | None = None,
        inputs_embeds: torch.Tensor | None = None,
    ) -> torch.Tensor | IntermediateTensors:
        hidden_states = self.model(
            input_ids, positions, intermediate_tensors, inputs_embeds
        )
        return hidden_states

    def compute_logits(
        self,
        hidden_states: torch.Tensor,
    ) -> torch.Tensor | None:
        logits = self.logits_processor(self.lm_head, hidden_states)
        return logits

    def load_weights(self, weights: Iterable[tuple[str, torch.Tensor]]) -> set[str]:
        loader = AutoWeightsLoader(
            self,
            skip_prefixes=(["lm_head."] if self.config.tie_word_embeddings else None),
        )
        return loader.load_weights(weights)
```

### Sampler.sample()

```py
class Worker(WorkerBase):
    @torch.inference_mode()
    def sample_tokens(
        self, grammar_output: "GrammarOutput | None"
    ) -> ModelRunnerOutput | AsyncModelRunnerOutput:
        return self.model_runner.sample_tokens(grammar_output)
```

```py
class GPUModelRunner
    ...
    @torch.inference_mode
    def sample_tokens(
        self, grammar_output: "GrammarOutput | None"
    ) -> ModelRunnerOutput | AsyncModelRunnerOutput | IntermediateTensors:
        ...
```

这里生成 token。

```text
hidden_states
↓
logits
↓
temperature/top_p
↓
next_token
```

## Summary



