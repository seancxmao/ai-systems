# vllm Serve Startup

## Background


## 认识`vllm`脚本

首先问题是：vllm这个shell脚本是什么？

`vllm` 不是源码仓库里的脚本文件，而是 Python package 安装时自动生成的 console script。定义在项目的 `pyproject.toml` 中：

```toml
[project.scripts]
vllm = "vllm.entrypoints.cli.main:main"
```

当执行：

```bash
pip install -e .
```

时，Python 的 packaging 系统会在虚拟环境的 `bin/` 目录下生成一个可执行文件：

```bash
.venv/bin/vllm
```

其本质相当于：

```python
from vllm.entrypoints.cli.main import main
main()
```

可以通过下面命令查看：

```bash
which vllm
cat $(which vllm)
```

```bash
❯ cat $(which vllm)
#!/path/to/vllm/.venv/bin/python3
# -*- coding: utf-8 -*-
import sys
from vllm.entrypoints.cli.main import main
if __name__ == "__main__":
    if sys.argv[0].endswith("-script.pyw"):
        sys.argv[0] = sys.argv[0][:-11]
    elif sys.argv[0].endswith(".exe"):
        sys.argv[0] = sys.argv[0][:-4]
    sys.exit(main())

```

## 运行`vllm serve`

搞清楚了vllm脚本是什么，下一个问题是，运行`vllm serve`，实际上在做什么？

已经知道shell脚本对应的入口是`vllm/entrypoints/cli/main.py`的`main`函数，它实际上完成参数转换和启动 OpenAI Server，其执行逻辑大致：

```text
CLI main
    ↓
注册所有子命令
    ↓
识别 serve
    ↓
ServeSubcommand.cmd()
```

这里负责解析一级命令，比如：

```bash
vllm serve ...
vllm chat ...
vllm benchmark ...
```

其中 `serve` 子命令对应`vllm/entrypoints/cli/serve.py`的`ServeSubcommand`，并调用其`cmd`方法。

这接近于**Command Pattern（命令模式）+ Registry（注册表模式）+ Plugin Architecture（插件架构）**。核心思想是：

```text
CLI Main
   ↓
发现所有 Subcommand
   ↓
注册到 argparse
   ↓
用户输入 serve
   ↓
找到对应对象
   ↓
dispatch_function()
   ↓
ServeSubcommand.cmd()
```

如果直接写成：

```python
if cmd == "serve":
    ServeSubcommand.cmd()
elif cmd == "chat":
    ChatSubcommand.cmd()
...
```

每新增一个子命令都要修改 `main.py`，违反 Open/Closed Principle（开闭原则）。而现在的设计中：

```python
CMD_MODULES = [...]
```

相当于一个插件列表，新增命令只需：

```python
new_command.py
└── class NewSubcommand
```

然后注册到 `CMD_MODULES` 即可，`main.py` 完全不用改。

所以从架构角度看：

```text
main.py
    = 框架（Framework）

ServeSubcommand
ChatSubcommand
BenchmarkSubcommand
    = 插件（Plugin）
```

这是很多大型 Python CLI（例如 [pytest](https://pytest.org)、[git](https://git-scm.com)、[kubectl](https://kubernetes.io/docs/reference/kubectl)）常见的扩展方式。

对于阅读 vLLM 源码，可以把：

```python
dispatch_function(args)
```

理解成：

```python
args.subcommand.cmd(args)
```

只是作者用了一层间接调用，把 CLI 框架和具体命令解耦了而已。真正的业务入口仍然是：

```python
ServeSubcommand.cmd()
```

所以调试时直接在 `ServeSubcommand.cmd()` 打断点即可，不必纠结中间的 dispatch 细节。

相关的Python模块和类（CMD_MODULES）：

```
vllm.entrypoints.cli.openai
    ChatCommand
    CompleteCommand
vllm.entrypoints.cli.serve
    ServeSubcommand
vllm.entrypoints.cli.launch
    LaunchSubcommandBase
    RenderSubcommand
    LaunchSubcommand
vllm.entrypoints.cli.benchmark.main
    BenchmarkSubcommand
vllm.entrypoints.cli.collect_env
    CollectEnvSubcommand
vllm.entrypoints.cli.run_batch
    RunBatchSubCommand
```

## 进入serve子命令

启动链路可以理解为：

```text
shell
 │
 ▼
vllm serve ...
 │
 ▼
vllm.entrypoints.cli.main.main()
 │
 ▼
ServeSubcommand.cmd() // <== we are here
```

`ServeSubcommand.cmd()`的关键代码：

```python
        if is_multi_port:
            run_dp_supervisor(args)
        elif args.api_server_count < 1:
            run_headless(args)
        elif args.api_server_count > 1 or envs.VLLM_RUST_FRONTEND_PATH:
            run_multi_api_server(args)
        else:
            # Single API server (this process).
            args.api_server_count = None
            uvloop.run(run_server(args))
```

`run_dp_supervisor`会进入 **Data Parallel（DP）模式**。`run_dp_supervisor()` 的作用是启动一个 Supervisor，再管理多个 Engine/API 实例。在 MBP（CPU Backend）上理论上也能跑，只是意义不大。最简单的方法是让 `is_multi_port=True`，例如配置多个数据并行实例（具体参数名称可能随版本变化，通常与 `--data-parallel-size > 1` 或多端口部署相关）。对于源码学习，不建议从这里入手，因为绝大多数单机开发和调试都会走：

```text
ServeSubcommand.cmd()
    ↓
uvloop.run(run_server(args))
```

先把单 API Server → EngineCore → Worker 这条主路径看透，DP Supervisor 可以后面再看。

`uvloop` 是一个基于 libuv 的高性能 asyncio Event Loop，可以理解成：

```text
Python 默认事件循环
          ↓
      uvloop
（更快的替代实现）
```

FastAPI、Uvicorn、vLLM API Server 本质上都是大量异步 I/O（HTTP 请求、流式 Token 返回、进程间通信），因此使用 `uvloop` 可以降低调度开销、提升吞吐量，所以 vLLM 默认用：

```python
uvloop.run(run_server(args))
```

启动整个异步系统。对于 vLLM 开发，不需要研究 uvloop 源码，也不需要掌握 libuv；知道它是 **asyncio 的高性能实现** 即可。真正需要掌握的是 Python 的 `async/await`、`Task`、`Event Loop` 基本模型，因为 API Server、EngineCore Client、流式推理接口大量依赖 asyncio，而不是依赖 uvloop 本身。换句话说：

```text
需要掌握：
    asyncio

了解即可：
    uvloop
```

把 uvloop 看成：

```python
asyncio.run(...)
```

的更快版本，基本就够了。

到这里，执行进入`vllm.entrypoints.openai.api_server`的`run_server`函数

## 启动API Server

终于进入`vllm.entrypoints.openai.api_server`的`run_server`

最重要的调用链：

```text
run_server()
 └── build_async_engine_client()
       └── AsyncLLM.from_vllm_config()
              ↓
        创建 EngineCore └── FastAPI App 初始化
 └── 注册 OpenAI Routes
 └── uvicorn.run()
```

可以简单理解成：

```text
解析参数
  ↓
启动 EngineCore
  ↓
创建 FastAPI
  ↓
启动 HTTP 服务
```

## 启动EngineCore

API Server 创建 EngineCore 时，调用`vllm/v1/engine/async_llm.py`的`AsyncLLM.from_vllm_config()`，进入`AsyncLLM`类的初始化方法。

`AsyncLLM`类的初始化方法的流程如下，最终EngineCore作为子进程被创建和启动。

```
vllm/v1/engine/core_client.py
    EngineCoreClient.make_async_mp_client
    AsyncMPClient.__init__
    MPClient.__init__ (parent class of AsyncMPClient)
vllm/v1/engine/utils.py
    launch_core_engines
    CoreEngineProcManager.__init__      // <- EngineCore进程的进程对象被创建
python3.12/multiprocessing/process.py
    BaseProcess.start                   // <- 启动EngineCore进程
```

EngineCore进程对象的创建在`CoreEngineProcManager.__init__`，关键代码如下：

```python
context.Process(
    target=EngineCoreProc.run_engine_core,
    name=f"EngineCore_DP{global_index}" if is_dp else "EngineCore",
    kwargs=common_kwargs
    | {"dp_rank": global_index, "local_dp_rank": local_index},
)
```

* context is multiprocessing.context.SpawnContext
* context.Process is SpawnProcess

最后，在`BaseProcess.start`启动EngineCore进程。

EngineCore进程的启动流程在`target=EngineCoreProc.run_engine_core`，这是一个静态工程方法，根据不同配置会使用不同的类：

* EngineCoreProc
* DPEngineCoreProc

如果调试非DP的情况，则使用EngineCoreProc，`EngineCoreProc.__init__()`会调用`EngineCore.__init__()`。

EngineCoreProc和EngineCore都在`vllm/v1/engine/core.py`：

* EngineCoreProc: ZMQ-wrapper for running EngineCore in background process.
* EngineCore: Inner loop of vLLM's Engine.
* 实现方式上：EngineCoreProc is subcluass of EngineCore

EngineCore启动的核心逻辑在`EngineCore.__init__()`。

```
...

# plugins need to be loaded at the engine/scheduler level too
from vllm.plugins import load_general_plugins
load_general_plugins()

# Setup Model.
self.model_executor = executor_class(vllm_config)   // <- Model executor创建，启动Worker进程

# Setup KV Caches and update CacheConfig after profiling.
kv_cache_config = self._initialize_kv_caches(vllm_config)    // <- KV Cache初始化
self.structured_output_manager = StructuredOutputManager(vllm_config)

# Setup scheduler.
Scheduler = vllm_config.scheduler_config.get_scheduler_cls() // <- Scheduler初始化

...
```

关于Model executor:

* vllm.v1.executor.multiproc_executor.MultiprocExecutor(Executor)（vllm/v1/executor/multiproc_executor.py）
* MultiprocExecutor的父类vllm.v1.executor.abstract.Executor（vllm/v1/executor/abstract.py）是Abstract base class for vLLM executors. An executor is responsible for executing the model on one device, or it can be a distributed executor that can execute the model on multiple devices.

关于Scheduler:

* vllm.v1.core.sched.scheduler.Scheduler(SchedulerInterface)（vllm/v1/core/sched/scheduler.py）

可以看到，伴随着model executor的创建，会启动Worker进程。model executor在EngineCore进程中，但是Worker是另一个进程，这又一次跨越了进程边界。

## 启动Worker

MultiprocExecutor的父类是Executor，MultiprocExecutor的初始化基本上委托给其父类Executor的初始化方法，其间又委托给子类实现的抽象方法`_init_executor`。所以，最终的核心逻辑在`MultiprocExecutor._init_executor()`。

```
unready_worker_handle = WorkerProc.make_worker_process(
    vllm_config=self.vllm_config,
    local_rank=local_rank,
    rank=global_rank,
    distributed_init_method=distributed_init_method,
    input_shm_handle=scheduler_output_handle,
    shared_worker_lock=shared_worker_lock,
    is_driver_worker=is_driver_worker,
    inherited_fds=inherited_fds,
)     // <- 启动Worker进程

unready_workers.append(unready_worker_handle)

# Workers must be created before wait_for_ready to avoid
# deadlock, since worker.init_device() does a device sync.
# Wait for all local workers to be ready.
self.workers = WorkerProc.wait_for_ready(unready_workers)
```

真正启动Worker进程的代码在`WorkerProc.make_worker_process`(`vllm/v1/executor/multiproc_executor.py`)：

```python
proc = context.Process(
    target=WorkerProc.worker_main,
    kwargs=process_kwargs,
    name=f"VllmWorker-{rank}",
    daemon=True,
)
# Apply NUMA binding if configured
with numa_utils.configure_subprocess(
    vllm_config, local_rank, process_kind="worker"
):
    proc.start()
```

Worker进程运行的代码是`WorkerProc.worker_main`，关键代码在`WorkerProc.__init__`

```python
...

# Load model
self.worker.init_device()
# Update process title now that parallel groups are initialized
self.setup_proc_title_and_log_prefix(
    enable_ep=vllm_config.parallel_config.enable_expert_parallel
)
if envs.VLLM_ELASTIC_EP_SCALE_UP_LAUNCH:
    self.worker.elastic_ep_execute("load_model")
else:
    self.worker.load_model()            // <- WorkerWrapperBase

scheduler_config = vllm_config.scheduler_config
self.use_async_scheduling = scheduler_config.async_scheduling
if self.use_async_scheduling:
    self.async_output_queue: queue.Queue = queue.Queue()
    self.async_output_copy_thread = Thread(
        target=self.async_output_busy_loop,
        daemon=True,
        name="WorkerAsyncOutputCopy",
    )
    self.async_output_copy_thread.start()

# Set block size based on the attention backends
current_platform.update_block_size_for_backend(vllm_config)

...
```

`WorkerWrapperBase.load_model()`到底调用了哪个方法：

`self.worker`: vllm.v1.worker.worker_base.WorkerWrapperBase, wrapper of vllm.v1.worker.cpu_worker.CPUWorker。WorkerWrapperBase没有实现load_model()方法。从下面的代码看，WorkerWrapperBase的load_model()委托给了被包装的CPUWorker。

```python
class WorkerWrapperBase:
    def __getattr__(self, attr: str):
        return getattr(self.worker, attr)
```

而Class hierarchy: `WorkerBase` <- `Worker` <- `CPUWorker`, `Worker`实现了load_model方法，所以`self.worker.load_model()`最终调用的是`Worker.load_model`。

```python
class Worker(WorkerBase):
   ...
   def load_model(self, *, load_dummy_weights: bool = False) -> None:
        with (
            self._maybe_get_memory_pool_context(tag="weights"),
            set_current_vllm_config(self.vllm_config),
            # 20 MiB is the minimum PyTorch allows for max_split_size_mb.
            self._scoped_allocator_max_split(max_split_size_mb=20),
        ):
            self.model_runner.load_model(load_dummy_weights=load_dummy_weights)

        if self.vllm_config.weight_transfer_config is not None:
            self.weight_transfer_engine = WeightTransferEngineFactory.create_engine(
                self.vllm_config.weight_transfer_config,
                self.vllm_config.parallel_config,
                self.model_runner.get_model(),
            )
    ...
```

`self.model_runner` = `vllm.v1.worker.cpu_model_runner.CPUModelRunner`

`self.model_runner.load_model`又进一步委托给`vllm.model_executor.model_loader.default_loader.DefaultModelLoader`

## 三个进程如何交互

启动顺序：

```text
API Server
    ↓
启动 EngineCore
    ↓
EngineCore 创建 Worker
    ↓
Worker 加载模型
    ↓
EngineCore Ready
    ↓
API Server 开始接受请求
```

## Summary

* API Server启动EngineCore子进程，EngineCore又启动Worker子进程
* EngineCore含有Executor（e.g. MultiprocExecutor）。EngineCore = 调度中心，类似Spark的Driver或Flink的JobManager。
* Worker（CPUWorker）含有ModelRunner，ModelRunner（CPUModelRunner）又使用ModelLoader（e.g. DefaultModelLoader）。Worker = 干活的人，类似Spark的Worker或Flink的TaskManager。

Worker 真正负责：

* 加载模型权重
* 管理 KV Cache
* 执行 Forward

这是后续阅读 vLLM 源码时最重要的骨架。理解了它，再深入 Scheduler、KV Cache、Batching、Paged Attention 时不会迷路。
