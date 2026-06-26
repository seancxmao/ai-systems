# FastAPI Essentials

## 1. What Problem Does It Solve?

FastAPI is a modern Python web framework for building HTTP APIs. It allows developers to expose Python functions as web services with minimal boilerplate while automatically handling request parsing, data validation, serialization, documentation generation, and dependency management.

Without FastAPI, building an API typically requires manually parsing HTTP requests, validating input, converting JSON to Python objects, handling errors, and generating API documentation. These repetitive tasks distract from implementing the actual business logic.

For LLM serving, FastAPI's role is straightforward: **it is the interface between clients and the inference engine**. It receives requests, validates them, invokes the backend (such as vLLM), and returns responses or streams generated tokens back to the client.

## 2. If You Come from Java

If you have experience with Java web development, FastAPI will feel surprisingly familiar. Many of its core ideas have direct counterparts in the Java ecosystem.

| Python / FastAPI     | Java Ecosystem                  |
| -------------------- | ------------------------------- |
| FastAPI              | Spring MVC / Spring Boot Web    |
| Uvicorn              | Tomcat / Jetty                  |
| ASGI                 | Servlet Specification           |
| Request Handler      | `@RestController` Method        |
| Pydantic Model       | DTO + Jackson + Bean Validation |
| Dependency Injection | Spring IoC                      |
| Type Hints           | Java Types + Annotations        |
| JSON Serialization   | Jackson                         |
| StreamingResponse    | StreamingResponseBody / SSE     |
| OpenAPI Generation   | SpringDoc / Swagger             |

The biggest conceptual difference is concurrency.

Traditional Java web servers typically use a **thread-per-request** model, where each request is handled by a dedicated thread. FastAPI, built on ASGI and `asyncio`, uses **asynchronous I/O**, allowing many requests to make progress within a small number of threads while they are waiting for network or disk operations.

For an LLM serving system, however, this difference is less dramatic than it first appears. GPU inference remains the dominant bottleneck, so FastAPI's asynchronous model mainly improves request handling, streaming, and resource utilization around the model rather than accelerating the model computation itself.

**Mental Model**

Think of FastAPI as the Python equivalent of a Spring Boot web application, and Uvicorn as the embedded servlet container that runs it.

## 3. Core Concepts

### Path Operation

A Python function exposed as an HTTP endpoint.

```python
@app.post("/generate")
async def generate():
    ...
```

Think of it as:

> HTTP Request → Python Function

### Request & Response

A **Request** contains everything sent by the client:

* URL
* HTTP method
* headers
* body

A **Response** is what FastAPI sends back:

* JSON
* Streaming response
* File
* Error

### Pydantic Model

FastAPI uses type-annotated models to define request and response schemas.

```python
class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int
```

Pydantic automatically:

* parses JSON
* validates types
* reports validation errors
* generates API schemas

### Dependency Injection

Reusable components (authentication, configuration, database connections, etc.) can be declared as dependencies rather than manually created inside every endpoint.

```python
@app.get("/")
async def root(config = Depends(get_config)):
    ...
```

### Async Endpoint

Endpoints are typically asynchronous.

```python
async def generate(...):
```

This allows the server to handle many concurrent requests efficiently.

### Router

Routers organize related endpoints into modules.

Instead of placing every API in one file:

```
app.py
```

you typically organize them as:

```
routers/
    chat.py
    embeddings.py
    health.py
```

### ASGI Application

FastAPI is **not** an HTTP server.

It is an **ASGI application** executed by an ASGI server such as Uvicorn.

```
Client
    ↓
Uvicorn
    ↓
FastAPI
    ↓
Your Python Code
```

## 4. Minimal Example

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class GenerateRequest(BaseModel):
    prompt: str

@app.post("/generate")
async def generate(req: GenerateRequest):
    return {
        "text": f"Received: {req.prompt}"
    }
```

This example demonstrates FastAPI's core workflow:

* expose an endpoint
* parse JSON automatically
* validate input
* return JSON

## 5. Common Patterns

### Request Models

```python
class Request(BaseModel):
    prompt: str
```

### Path Operations

```python
@app.get(...)
@app.post(...)
```

### Dependency Injection

```python
Depends(...)
```

### Streaming Responses

```python
StreamingResponse(...)
```

Used extensively for token-by-token LLM generation.

### Background Tasks

```python
BackgroundTasks(...)
```

Run work after returning a response (logging, cleanup, metrics).

### Routers

```python
APIRouter()
```

Organize APIs into reusable modules.

## 6. Why It Matters for LLM Serving

```
Client
    ↓
HTTP
    ↓
Uvicorn
    ↓
FastAPI
    ↓
Request Validation
    ↓
LLM Engine (vLLM / SGLang)
    ↓
Streaming Tokens
    ↓
Client
```

FastAPI is responsible for the **application layer**, not the inference layer.

Typical responsibilities include:

* receiving generation requests
* validating request parameters
* calling the inference engine
* streaming generated tokens
* exposing health and metrics endpoints

Without understanding FastAPI, it is difficult to understand how serving frameworks connect HTTP requests to the underlying inference engine.

## 7. Common Misconceptions

### FastAPI = Uvicorn ❌

FastAPI is an application.

Uvicorn is the server that runs it.

### FastAPI performs inference ❌

FastAPI only orchestrates request handling.

The model runs elsewhere (PyTorch, vLLM, TensorRT-LLM, etc.).

### Type hints are only for IDEs ❌

In FastAPI, type hints are part of the framework itself.

They drive:

* validation
* serialization
* API documentation

### Async automatically makes code faster ❌

`async` improves concurrency for I/O-bound workloads.

Model inference remains dominated by GPU computation.

## 8. Key Takeaways

* FastAPI is an **ASGI application**, not an HTTP server.
* Think of FastAPI as the **API layer** sitting in front of the LLM engine.
* Type hints are executable framework metadata, not just documentation.
* Pydantic converts JSON into validated Python objects automatically.
* Streaming responses are essential for token-by-token generation.
* FastAPI manages request handling; the inference engine manages model execution.

## 9. Further Reading

### Official

* [FastAPI Documentation](https://fastapi.tiangolo.com/)
* [Pydantic Documentation](https://docs.pydantic.dev/)
* [ASGI Specification](https://asgi.readthedocs.io/)

### Deep Dive

* [Starlette Documentation](https://www.starlette.io/) (FastAPI is built on Starlette)
* [FastAPI Advanced User Guide](https://fastapi.tiangolo.com/advanced/)

### Source Code

* [FastAPI](https://github.com/fastapi/fastapi)
* [Starlette](https://github.com/encode/starlette)
* [Uvicorn](https://github.com/encode/uvicorn)
* [vLLM](https://github.com/vllm-project/vllm)

### Recommended Books

FastAPI: Modern Python Web Development (if a book is desired)

## Notes

按照 **Essentials Writing Guide** 来衡量，这篇基本符合目标：

* **长度**：约 2 页 A4，可在 10 分钟内读完。
* **重点**：解释 FastAPI 的角色，而不是教授 FastAPI 开发。
* **LLM Serving 导向**：所有概念都围绕「FastAPI 在 Serving Stack 中的位置」展开。
* **心理模型优先**：强调 *ASGI Application*、*API Layer*、*Type Hints Drive Framework Behavior*，避免陷入 API 细节。
* **为后续 Essentials 铺路**：阅读 Asyncio、Uvicorn、ASGI、Pydantic Essentials 时，能够自然衔接，不会重复大量内容。
