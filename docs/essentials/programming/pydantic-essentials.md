# Pydantic Essentials

## 1. What Problem Does It Solve?

Pydantic is a data validation and serialization library built around Python type hints. It converts unstructured input (typically JSON) into validated Python objects while automatically checking types, reporting errors, and generating structured schemas.

Without Pydantic, applications must manually parse dictionaries, check whether required fields exist, verify data types, convert values, and produce consistent error messages. This quickly becomes repetitive and error-prone as APIs grow.

For LLM serving, Pydantic sits between the HTTP layer and the serving logic. It ensures that every generation request has the expected structure before it reaches the inference engine. Instead of dealing with raw JSON dictionaries, the serving code works with validated Python objects.

## 2. If You Come from Java

For Java developers, Pydantic is conceptually similar to combining DTO classes, Jackson, and Bean Validation into a single library.

| Python / Pydantic | Java Ecosystem                            |
| ----------------- | ----------------------------------------- |
| BaseModel         | DTO / POJO                                |
| Type Hints        | Java Types                                |
| Validation        | Bean Validation (`@NotNull`, `@Min`, ...) |
| JSON Parsing      | Jackson                                   |
| Serialization     | Jackson                                   |
| Schema Generation | SpringDoc / Swagger                       |

Both ecosystems define structured objects that represent API requests and responses.

The biggest difference is that Python type hints are not merely documentation. Pydantic actively reads them at runtime to parse data, validate fields, perform type conversion, and generate OpenAPI schemas.

**Mental Model**

Think of Pydantic as a smart constructor that turns untrusted JSON into validated Python objects before your application logic begins.

## 3. Core Concepts

### BaseModel

Every Pydantic model inherits from `BaseModel`.

```python
from pydantic import BaseModel

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int
```

A model defines both the structure and validation rules for the data.

### Type Hints

Pydantic uses Python type hints as executable metadata.

```python
prompt: str
temperature: float
max_tokens: int
```

These annotations determine how incoming data is parsed and validated.

### Validation

When input data does not satisfy the model definition, Pydantic raises a validation error.

For example:

* missing required fields
* incorrect data types
* invalid field values

Application code can assume that successfully constructed models are already valid.

### Serialization

Pydantic models can be converted back into dictionaries or JSON.

```python
request.model_dump()
```

This is useful for logging, API responses, or communication between components.

### Nested Models

Models can contain other models.

```python
class SamplingParams(BaseModel):
    temperature: float

class GenerateRequest(BaseModel):
    prompt: str
    sampling: SamplingParams
```

This makes complex request structures easier to organize and validate.

### Default Values

Fields may provide defaults.

```python
temperature: float = 1.0
```

If the client omits the field, the default value is used automatically.

## 4. Minimal Example

```python
from pydantic import BaseModel

class GenerateRequest(BaseModel):
    prompt: str
    max_tokens: int = 128

req = GenerateRequest(
    prompt="Hello",
    max_tokens=64
)

print(req.prompt)
print(req.max_tokens)
```

This example demonstrates Pydantic's core workflow:

* define a schema
* validate input
* construct a Python object
* access fields as normal attributes

## 5. Common Patterns

### Define Request Models

```python
class Request(BaseModel):
    prompt: str
```

The most common use case in FastAPI.

### Default Values

```python
temperature: float = 0.8
```

Provide optional configuration with sensible defaults.

### Nested Models

```python
class SamplingParams(BaseModel):
    ...
```

Represent hierarchical request structures.

### Serialize Objects

```python
model.model_dump()
```

Convert models into dictionaries.

### Optional Fields

```python
system_prompt: str | None = None
```

Allow clients to omit certain parameters.

### Response Models

```python
class GenerateResponse(BaseModel):
    text: str
```

Ensure API responses have a consistent structure.

## 6. Why It Matters for LLM Serving

```
Client
    ↓
JSON
    ↓
FastAPI
    ↓
Pydantic
    ↓
Validated Request Object
    ↓
LLM Engine
```

Pydantic acts as the boundary between external input and internal application logic.

Typical responsibilities include:

* parsing JSON requests
* validating generation parameters
* providing default values
* producing structured response objects
* generating API schemas automatically

Without Pydantic, serving code would spend much more effort validating dictionaries instead of implementing inference logic.

## 7. Common Misconceptions

### Pydantic is an ORM ❌

Pydantic models describe data.

They do not manage databases.

### Pydantic is only for FastAPI ❌

FastAPI uses Pydantic extensively, but Pydantic is a standalone library that can validate data in any Python application.

### Type hints are only for IDEs ❌

In Pydantic, type hints directly control parsing, validation, serialization, and schema generation.

### Validation happens everywhere ❌

Validation occurs when a model is constructed.

Once a model has been successfully created, application code can generally assume its fields satisfy the declared schema.

## 8. Key Takeaways

* Pydantic converts untrusted JSON into validated Python objects.
* Type hints drive runtime behavior, not just static analysis.
* `BaseModel` is the foundation of every Pydantic model.
* Validation happens at the boundary between clients and application logic.
* Nested models make complex request structures easy to represent.
* In LLM serving, Pydantic protects the inference engine from malformed requests.

## 9. Further Reading

### Official

* Pydantic Documentation
* FastAPI Request Body Documentation

### Deep Dive

* Pydantic V2 Migration Guide
* Python Type Hints Documentation

### Source Code

* Pydantic
* FastAPI
* Starlette
* vLLM

### Recommended Books

No dedicated book is necessary.

The official documentation, together with reading FastAPI source code, is sufficient for understanding the role of Pydantic in LLM serving systems.
