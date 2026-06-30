# vLLM Architecture Overview


## Concepts

Request Handling

负责接收用户请求、解析参数、校验输入，并将请求转换成模型可以处理的内部格式。在 vLLM 中，例如 OpenAI-compatible API Server 会接收 /v1/chat/completions 请求，将其转换为内部的 prompt 和 sampling 参数，然后交给 Engine 处理。

Batching

将多个用户请求合并成一个批次（batch）一起执行，以提高 GPU 利用率和吞吐量。vLLM 的核心优化之一就是 Continuous Batching：新请求可以动态加入正在运行的 batch，而不需要等待整个 batch 完成，从而显著提升服务效率。

Streaming

在生成过程中持续向客户端返回 token，而不是等待完整结果生成后一次性返回。这样可以降低用户感知延迟。vLLM 支持 OpenAI 风格的流式输出（streaming response），模型每生成一部分内容，就立即通过 SSE 返回给客户端。

Continuous Batching

vLLM 的核心创新之一是连续批处理（Continuous Batching），允许在每个调度周期动态加入新请求并移除已完成请求。

Scheduling

决定哪些请求在什么时间获得计算资源，以及如何在请求之间分配 GPU 时间。对于 LLM 服务来说，调度器需要处理请求到达、token 生成、请求完成等事件。vLLM 的 Scheduler 会根据请求状态组织 batch，并协调 Prefill 与 Decode 阶段的执行。

Resource Management

负责管理 GPU、CPU、显存以及 KV Cache 等系统资源，确保服务稳定高效运行。vLLM 最具代表性的资源管理机制是 PagedAttention，它通过分页管理 KV Cache，减少显存碎片并提高缓存利用率，从而支持更多并发请求。

Model Hosting

模型托管.

Offloading（卸载）

将原本应该驻留在 GPU 上的数据或计算，临时移动到更便宜、更充足的存储层（CPU RAM、SSD、远端节点），以突破 GPU 显存限制。本质上是一种 用时间换空间（latency for capacity） 的技术。

## References

* https://docs.vllm.ai/en/stable/design/arch_overview/

