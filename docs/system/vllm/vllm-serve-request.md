# vLLM Serve Request

## Background


## 请求处理流程

```text
API Server
   │
   ▼
AsyncLLM
   │
   ▼
EngineCore
   │
   ▼
Scheduler
   │
   ▼
Executor
   │
   ▼
Worker
   │
   ▼
Model Forward
```

## 进程间通信

收到 OpenAI 请求：

```text
Client
   ↓
FastAPI Route
   ↓
AsyncLLM
   ↓
EngineCore
   ↓
Scheduler
   ↓
Worker
   ↓
Model Forward
   ↓
返回 Token
```

最关键的代码位置：

### API → EngineCore

```python
AsyncLLM.generate()
```

内部通过：

```python
EngineCoreClient
```

向 EngineCore 发消息。

### EngineCore → Worker

```python
Executor.execute_model()
```

最终调用：

```python
Worker.execute_model()
```

执行一次推理。
