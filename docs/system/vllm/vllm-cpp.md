# vLLM C++

## Overview

### （1）C++代码结构

vLLM 的 C++ 代码主要集中在 `csrc/` 目录，职责是实现性能敏感的底层算子（Attention、KV Cache、Sampling、CPU/GPU Kernel 等）。可以把它理解为 **“推理引擎内核”**：上层调度、模型执行逻辑在 Python，中间最耗时的计算下沉到 C++/CUDA/Metal 中实现。通常按功能划分为 `attention/`、`cache/`、`cpu/`、`cuda/` 等子目录，最后通过 `torch_bindings.cpp` 统一向 Python 导出接口。

### （2）C++代码build方式和结果

执行：

```bash
pip install -e .
```

时，`setup.py` / `pyproject.toml` 会调用 PyTorch Extension 构建系统，将 `csrc/` 下的 C++/CUDA/Metal 源码编译成动态库。最终产物通常是：

```text
vllm/_C.abi3.so
```

（Linux 是 `.so`，macOS 也是 `.so`）。这个文件相当于 Python 可加载的原生扩展模块，包含所有注册好的 C++ 算子和 Kernel。

### （3）Python和C++代码衔接

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


## References

* [CPU] Refactor CPU attention backend #27954
