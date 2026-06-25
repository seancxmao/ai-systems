# vLLM C++

## Overview

### C++代码结构

vLLM 的 C++ 代码主要集中在 `csrc/` 目录，职责是实现性能敏感的底层算子（Attention、KV Cache、Sampling、CPU/GPU Kernel 等）。可以把它理解为 **“推理引擎内核”**：上层调度、模型执行逻辑在 Python，中间最耗时的计算下沉到 C++/CUDA/Metal 中实现。通常按功能划分为 `attention/`、`cache/`、`cpu/`、`cuda/` 等子目录，最后通过 `torch_bindings.cpp` 统一向 Python 导出接口。

### C++代码build方式和结果

执行：

```bash
pip install -e .
```

时，`setup.py` / `pyproject.toml` 会调用 PyTorch Extension 构建系统，将 `csrc/` 下的 C++/CUDA/Metal 源码编译成动态库。最终产物通常是：

```text
vllm/_C.abi3.so
```

（Linux 是 `.so`，macOS 也是 `.so`）。这个文件相当于 Python 可加载的原生扩展模块，包含所有注册好的 C++ 算子和 Kernel。

### Python和C++代码衔接

核心桥梁是 `torch_bindings.cpp`。C++ 通过 PyTorch Custom Operator 机制注册接口：

```cpp
TORCH_LIBRARY(...)
```

或

```cpp
ops.def(...)
```

编译后，这些接口会出现在 Python 的：

```python
torch.ops._C.xxx(...)
```

或

```python
torch.ops.vllm.xxx(...)
```

下面。调用链可以简化为：

```text
Python业务逻辑
    ↓
torch.ops.xxx(...)
    ↓
torch_bindings.cpp
    ↓
C++实现
    ↓
CPU/CUDA/Metal Kernel
```

## Case Study

### 参数数量不匹配

遇到的 `get_scheduler_metadata()` 参数不匹配，本质上就是：

```text
Python代码
    ↓
torch.ops._C.get_scheduler_metadata(...)
    ↓
旧版 _C.abi3.so 中注册的函数签名
```

两边接口定义不一致导致的 ABI mismatch。

涉及的代码：

* csrc/cpu/cpu_attn.cpp
* csrc/cpu/cpu_attn_impl.hpp
* csrc/cpu/cpu_fused_moe.cpp
* csrc/cpu/torch_bindings.cpp
* vllm/v1/attention/backends/cpu_attn.py
* vllm/_custom_ops.py
* tests/kernels/attention/test_cpu_attn.py

详见：

* https://github.com/vllm-project/vllm/pull/45690

## Dev

### MBP or Linux

MBP 能完成 C++ 代码阅读、调试、修改、构建、提交 PR 等大部分工作，尤其是 CPU Backend 和 Metal Backend。真正受限的是 CUDA Kernel 开发和性能验证，因为 NVIDIA CUDA 生态只在 Linux（或 Windows）+ NVIDIA GPU 上完整支持。因此，学习和参与社区开发可以先用 MBP，深入 CUDA Kernel 优化时再转 Linux + NVIDIA GPU。

### VS Code or Xcode

对于 vLLM 社区开发，主流是 VS Code。因为项目本身是 Python+C++ 混合代码库，大部分开发者都在 VS Code 中同时调试 Python 和 C++，并使用 GitHub Copilot、clangd、CMake Tools 等插件。Xcode 更适合纯 macOS/iOS 原生开发，在 vLLM 场景下通常只作为编译器和调试器提供者（clang/lldb），而不是主要 IDE。如果目标是 AI Infra Engineer，继续使用 VS Code 是最符合行业实践的选择。

### C++ 的核心编程能力

对于 AI Infra 方向，重点不是刷复杂算法，而是掌握现代 C++（C++17/20）、内存管理、模板、并发编程、性能优化和系统接口。最常见的能力包括：

```
STL（vector、unordered_map、string）
RAII 和智能指针
模板和泛型编程
多线程（thread、mutex、atomic）
SIMD/向量化基础
CMake 构建系统
调试（gdb/lldb）
性能分析（perf、Instruments）
```

在 vLLM 中，实际用得最多的是 STL、模板、线程同步、Tensor 数据访问以及与 PyTorch C++ API 的交互。

### Begin with CPU Backend

非常合适，而且对理解 vLLM 更友好。CPU Backend 的代码通常比 CUDA Kernel 简单得多，没有线程块、Warp、Shared Memory 等 GPU 概念干扰，更容易看清 Attention、Scheduler、KV Cache 等核心逻辑。很多接口（例如你刚定位到的 `get_scheduler_metadata()`）在 CPU Backend 和 CUDA Backend 中都有对应实现，先从 CPU 理解设计，再看 CUDA 优化版本是很自然的学习路径。CPU Backend → Metal Backend → CUDA Backend 是一条比较顺畅的进阶路线。

## References

* [CPU] Refactor CPU attention backend #27954
