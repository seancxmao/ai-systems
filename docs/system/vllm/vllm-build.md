# vLLM Build

## Overview

### （1）vLLM 的 build 脚本有哪些？

从开发者视角看，最重要的是 4 个层次：

```text
pip install -e .
    ↓
pyproject.toml
    ↓
setup.py
    ↓
cmake/
    ↓
csrc/*
```

其中：

* `pyproject.toml`：Python 打包入口，告诉 pip 如何构建项目。
* `setup.py`：vLLM 的核心构建脚本，决定编译哪些 Extension、启用哪些平台（CUDA/CPU/Metal）。
* `cmake/`：存放 CMake 配置和辅助脚本，负责组织底层 C++ 编译。
* `csrc/`：真正被编译的源码。

做 vLLM 开发，80% 的构建问题都集中在下面这两层：

```text
setup.py
cmake/*
```

### （2）它们分别做了什么事情？

当执行：

```bash
pip install -e .
```

时，流程可以理解为：

```text
pyproject.toml
    ↓
启动构建环境
    ↓
setup.py
    ↓
检查平台(CUDA/ROCm/CPU/Metal)
    ↓
调用 CMake
    ↓
编译 csrc/*
    ↓
生成 vllm/_C.abi3.so
    ↓
editable install
```

最终结果有两部分：

1. **Python 代码不复制**，直接指向当前源码目录（editable mode）。
2. **C++ 代码被编译** 成：

```text
vllm/_C.abi3.so
```

供 Python 通过如下代码调用：

```python
torch.ops.xxx(...)
```

因此从开发角度看：

```text
修改 Python
    → 通常无需重新 build

修改 csrc/*
    → 必须重新 build
```

这是判断是否需要重新执行 `pip install -e .` 最实用的原则。

