# vLLM Debug

## Background

| 配置                      | 用途            |
| ----------------------- | ------------- |
| Run Current Python File | 调试小脚本         |
| Launch vLLM API Server  | 启动整个 vLLM     |
| Attach debugpy          | 调试已经运行中的 vLLM |


## Debug with VS Code

launch.json:

```json
{
    "version": "0.2.0",
    "configurations": [

        // 1. 调试当前打开的Python文件
        {
            "name": "Python: Current File",
            "type": "debugpy",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal"
        },

        // 2. 启动 vLLM OpenAI Server
        {
            "name": "vLLM: OpenAI Server",
            "type": "debugpy",
            "request": "launch",
            "module": "vllm.entrypoints.openai.api_server",
            "args": [
                "--model",
                "Qwen/Qwen3-0.6B",
                "--port",
                "8000",
                "--gpu-memory-utilization",
                "0.2",
                "--max-model-len",
                "4096"
            ],
            "console": "integratedTerminal",
            "env": {
                "VLLM_LOGGING_LEVEL": "DEBUG"
            },
            "justMyCode": false
        },

        // 3. Attach到已经运行的vLLM进程
        {
            "name": "vLLM: Attach",
            "type": "debugpy",
            "request": "attach",
            "connect": {
                "host": "localhost",
                "port": 5678
            },
            "justMyCode": false
        },

        // 4. 调试 benchmark 脚本
        {
            "name": "vLLM: Benchmark Throughput",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/benchmarks/benchmark_throughput.py",
            "args": [
                "--backend",
                "vllm",
                "--model",
                "Qwen/Qwen3-0.6B"
            ],
            "console": "integratedTerminal",
            "justMyCode": false
        },

        // 5. debug vllm-metal
        {
            "name": "vLLM Metal: OpenAI Server",
            "type": "debugpy",
            "request": "launch",
            "module": "vllm.entrypoints.openai.api_server",
            "args": [
                "--model",
                "Qwen/Qwen3-0.6B",
                "--device",
                "metal",
                "--port",
                "8000",
                "--max-model-len",
                "4096"
            ],
            "env": {
                "VLLM_LOGGING_LEVEL": "DEBUG"
            },
            "console": "integratedTerminal",
            "justMyCode": false
        }
    ]
}
```

## References

