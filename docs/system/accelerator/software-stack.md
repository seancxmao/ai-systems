# AI Accelerator Software Stack

## Mental Model

| Layer     | Responsibility                      |
| --------- | ----------------------------------- |
| Framework | User Programming Model              |
| Compiler  | Graph Transformation & Optimization |
| Library   | Kernel Abstraction                  |
| Runtime   | Scheduling & Memory Management      |
| Hardware  | Execution Device                    |

## Training

今天（2026年）的主流训练框架几乎全部建立在 PyTorch 生态之上。甚至很多人已经把PyTorch视为AI Training OS，即训练领域的事实标准平台。

| 框架                 | 底层    |
| ------------------- | ------- |
| Megatron-LM         | PyTorch |
| NeMo                | PyTorch |
| DeepSpeed           | PyTorch |
| FSDP                | PyTorch |
| HuggingFace Trainer | PyTorch |
| Lightning           | PyTorch |
| ColossalAI          | PyTorch |
| Axolotl             | PyTorch |
| LLaMA Factory       | PyTorch |

为什么训练框架大多没有绕开 PyTorch？因为训练最复杂的是Autograd。

* Forward
* Backward
* Gradient
* Optimizer

PyTorch已经提供：

* 动态计算图
* 自动求导
* Optimizer
* AMP
* Distributed

重新实现一套成本极高。所以训练框架一般选择扩展 PyTorch，而不是替代 PyTorch。

PyTorch对应的AI加速器软件栈示例如下：

| Layer         | PyTorch (CUDA) | PyTorch (MPS)    |
| ------------- | -------------- | ---------------- |
| Framework     | PyTorch        | PyTorch          |
| Compiler      | TorchInductor  | TorchInductor    |
| Library       | ATen           | ATen             |
| Runtime / API | CUDA Runtime   | Metal / MPSGraph |
| Hardware      | NVIDIA GPU     | Apple GPU        |

除了PyTorch，还有Google/Apple生态，其AI加速器软件栈示例如下：

| Layer         | JAX (CUDA)          | JAX (TPU)           | MLX          |
| ------------- | ------------------- | ------------------- | ------------ |
| Framework     | JAX                 | JAX                 | MLX          |
| Compiler      | XLA                 | XLA                 | MLX Compiler |
| Library       | StableHLO / XLA Ops | StableHLO / XLA Ops | MLX Core     |
| Runtime / API | CUDA Runtime        | TPU Runtime         | Metal        |
| Hardware      | NVIDIA GPU          | TPU                 | Apple GPU    |

## Inference & Serving

推理框架原先和训练框架一样，都是使用PyTorch，但是正逐渐分化。

训练时代的主要成本：Forward + Backward，所以PyTorch非常重要。而推理时代，主要成本是：Attention + KV Cache + Memory Movement，GPU Kernel比Autograd更重要。例如：

* FlashAttention
* PagedAttention
* FlashInfer
* Triton Kernel
* CUDA Kernel

这些才是真正的性能热点。因此今天的 vLLM 架构更像：

```
Serving Layer
    ↓
Scheduler
    ↓
Custom Kernels
    ↓
CUDA / ROCm / Metal
```

而不是：

```
PyTorch
    ↓
CUDA
```

但是，vLLM并没有完全绕过PyTorch。实际上更接近：

```
Model Definition
    ↓
PyTorch
```

```
Attention Kernel
    ↓
FlashAttention
FlashInfer
CUDA
```

也就是说，在性能关键路径上大量绕过PyTorch默认实现，直接调用更底层的Kernel。

这也是为什么 AI Infra（Inference）工程师最终会不断接触：

* CUDA
* Triton
* CUTLASS
* FlashAttention
* FlashInfer

而不仅仅停留在 PyTorch API 层。

推理AI加速器软件栈示例如下：

| Layer             | NVIDIA                 | Apple        | TPU         |
| ----------------- | ---------------------- | ------------ | ----------- |
| Serving Framework | vLLM                   | vLLM         | vLLM        |
| Framework         | PyTorch                | MLX          | JAX         |
| Compiler          | TorchInductor / Triton | MLX Compiler | XLA         |
| Runtime/API       | CUDA                   | Metal        | TPU Runtime |
| Hardware          | NVIDIA GPU             | Apple GPU    | TPU         |

## Terminology

### NVIDIA生态

CUDA

CUDA 是 NVIDIA 的 GPU 编程平台和运行时系统。开发者可以通过 CUDA 编写 GPU 程序，PyTorch、TensorFlow、vLLM 等 AI 框架的大部分 NVIDIA GPU 加速能力都建立在 CUDA 之上。

cuBLAS

cuBLAS（CUDA Basic Linear Algebra Subprograms）是 NVIDIA 提供的高性能线性代数库，专门优化矩阵乘法、向量运算等基础数学计算。由于 Transformer、Attention、MLP 等模型的大部分计算最终都归结为矩阵乘法，因此 cuBLAS 可以看作深度学习计算最核心的基础库之一。

cuDNN

cuDNN（CUDA Deep Neural Network library）是 NVIDIA 提供的深度学习算子库，在 cuBLAS 之上进一步封装和优化卷积、归一化、Attention、RNN 等神经网络常用算子。PyTorch、TensorFlow、vLLM 等框架通常不会自己实现这些高性能算子，而是调用 cuDNN。

### Google生态

TPU

TPU（Tensor Processing Unit）是 Google 设计的 AI 专用加速芯片，主要用于大规模训练和推理。它类似 NVIDIA GPU，但专门针对矩阵乘法、Transformer 等深度学习计算进行了优化，通常与 XLA 紧密配合。

XLA

XLA（Accelerated Linear Algebra）是 Google 开发的机器学习编译器。它接收 TensorFlow、JAX 等框架生成的计算图，进行算子融合、内存优化和代码生成，然后针对 CPU、GPU、TPU 生成高效执行代码。可以把它理解为 AI 领域的 LLVM。

JAX

JAX 是 Google 推出的高性能数值计算框架，API 类似 NumPy，但支持自动求导、向量化和 JIT 编译。JAX 自身更像前端，真正的性能优化和代码生成主要依赖 XLA。

### Apple生态

Metal

Metal 是 Apple 的底层图形与并行计算 API。它相当于苹果版 CUDA，负责让程序直接使用 Apple GPU 的计算能力。MLX、PyTorch MPS、游戏引擎等最终都会调用 Metal。

Metal Performance Shaders（MPS）

MPS（Metal Performance Shaders）是 Apple 在 Metal 之上提供的一层高性能机器学习和图像计算库，封装了矩阵乘法、卷积、归一化等常见算子。MPS ≈ cuDNN + cuBLAS

MLX

MLX 是 Apple 为 Apple Silicon 开发的机器学习框架。它提供张量运算、自动求导和神经网络模块，并直接利用 Metal GPU。定位类似于苹果生态中的 JAX/PyTorch。

### PyTorch

TorchInductor

TorchInductor 是 PyTorch 2.x 的优化编译器，负责将计算图转换为经过算子融合和内存优化的高性能 CPU/GPU 执行代码。 它对应的角色，大致相当于 JAX 体系中的 XLA。

TorchDynamo

TorchDynamo 是 PyTorch 2.x 的图捕获（graph capture）组件，它在 Python 字节码层拦截 PyTorch 运算，把原本逐行执行的 eager 模式代码提取成计算图（FX Graph），然后交给后端编译器（通常是 TorchInductor）进行优化和代码生成。它本身不负责优化或生成 GPU Kernel，而是负责把动态 Python 程序转换成编译器能够处理的图表示，因此常被看作 PyTorch 编译链路的“前端”。

## Summary

AI 训练与推理正在走向两条不同的技术演进路线。

对于训练系统，核心挑战是自动求导、分布式训练和大规模模型并行，因此 PyTorch 不仅没有被弱化，反而正在成为训练领域的事实标准平台。

对于推理系统，核心挑战已经从计算图构建转向请求调度、内存管理和高效执行，因此 PyTorch 正逐渐退化为模型定义层，而性能关键路径越来越多地下沉到 Scheduler、Compiler、Runtime 和 Kernel。

从长期看，训练生态可能继续围绕 PyTorch 统一，而推理生态则更可能围绕 Scheduler + Compiler + Runtime 的组合展开竞争。


Training Stack

```
PyTorch Platform
Distributed Runtime
CUDA / ROCm / XPU
GPU
```

Inference Stack

```
Serving API
Scheduler
Compiler
Runtime
Kernel Library
CUDA / ROCm / XPU
GPU
```

从vLLM、SGLang、TensorRT-LLM，可以观察到一个很明显的趋势：

* vLLM 最强的是 Scheduler
* SGLang 最强调 Compiler
* TensorRT-LLM 最强的是 Runtime + Kernel
* FlashInfer、FlashAttention 属于 Kernel Library

## References

