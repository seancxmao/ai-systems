# Knowledge Structure

## Why This Structure?

This blog is not a collection of notes.

It is a long-term knowledge base for becoming an AI Infrastructure Engineer.

The directory structure is designed around **the role each piece of knowledge plays**, rather than around technologies or programming languages.

Instead of organizing content like:

```text
Python
Linux
CUDA
Networking
```

I organize it by **purpose**:

```text
Projects
Inference and Serving
Model Fundamentals
AI Systems Overview
Foundations
Essentials
KM
```

Each directory answers a different question.

## Directory Philosophy

### Projects

> **What have I built?**

Projects demonstrate practical engineering ability.

This is the most visible part of the blog and the primary evidence of technical skills.

Typical contents:

* llm-from-scratch
* llm-serving-from-scratch
* dl-from-scratch
* source code analysis
* experiments

Projects should always remain the first entry point.

### Inference and Serving

> **What systems do I specialize in?**

This is the core domain of the blog.

It contains the architecture, algorithms, implementation details, and production practices behind modern LLM inference systems.

Typical topics:

* vLLM
* SGLang
* Continuous Batching
* KV Cache
* Scheduling
* Distributed Inference
* Serving Architecture

This directory defines the technical focus of the blog.

### Model Fundamentals

> **How do models work?**

Serving systems ultimately exist to execute models.

This directory covers the model-side knowledge required to understand inference systems.

Typical topics:

* Transformer
* Attention
* RoPE
* MoE
* Quantization
* Speculative Decoding

The focus is understanding models, not training them.

### AI Systems Overview

> **How does the entire ecosystem fit together?**

Some knowledge is broader than a single project or technology.

This directory captures high-level system thinking.

Typical topics:

* AI Systems Roadmap
* Technology Landscape
* System Architecture
* Learning Maps

It provides context rather than implementation details.

### Foundations

> **Why does it work?**

Foundations contain the long-lived principles behind engineering systems.

These concepts change very slowly and remain useful regardless of frameworks.

Typical topics:

* Operating Systems
* Networking
* Distributed Systems
* Computer Architecture
* Programming Languages

The emphasis is on principles, algorithms, and mechanisms.

When studying a topic deeply—its design, implementation, or theory—it belongs here.

### Essentials

> **How do I use it effectively?**

Essentials are concise engineering notes for tools, frameworks, and libraries encountered during daily work.

They are written for quick review rather than comprehensive learning.

Typical topics:

* Asyncio
* FastAPI
* Uvicorn
* Git
* SSH
* Docker
* LLDB

Each Essential answers:

* What problem does it solve?
* What are the core concepts?
* How is it commonly used?
* Why does it matter for LLM Serving?
* What should I remember six months from now?

Essentials are intentionally lightweight.

If an Essential grows into a deep technical exploration, it should become a Foundation article instead.

### KM

> **How is this knowledge base maintained?**

KM records the methodology behind this blog.

It contains the decisions, principles, and workflows used to build and maintain the knowledge base itself.

Typical topics:

* Directory organization
* Writing guides
* Naming conventions
* Learning workflow
* Review strategy
* Tooling
* Knowledge architecture
* AI-assisted learning

KM is meta-knowledge.

It does not teach AI Systems.

It teaches how this knowledge base is constructed.

## Design Principles

### 1. Organize by purpose, not by technology

Avoid directories such as:

```text
Python
Linux
CUDA
```

The same technology often appears in different contexts.

For example:

* Asyncio → Essentials
* Asyncio internals → Foundations
* Asyncio in vLLM → Inference and Serving

The purpose determines the location.

### 2. Separate understanding from usage

Many topics naturally have two layers.

Example:

```text
Git
├── Foundations
│   ├── Commit Graph
│   ├── Three-way Merge
│   └── Object Model
│
└── Essentials
    ├── Fork + PR
    ├── Bisect
    ├── Cherry-pick
    └── Daily Commands
```

Foundations explain **why**.

Essentials explain **how**.

### 3. Keep long-lived knowledge separate from fast-changing knowledge

Some knowledge remains useful for decades.

Examples:

* Operating Systems
* Networking
* Distributed Systems

Some changes every few years.

Examples:

* FastAPI
* Uvicorn
* Docker
* Ray

Separating them keeps the knowledge base stable over time.

### 4. Build a coherent learning path

The directory order reflects a learning journey.

```text
Projects
        ↓
Inference and Serving
        ↓
Model Fundamentals
        ↓
AI Systems Overview
        ↓
Foundations
        ↓
Essentials
        ↓
KM
```

The first directories describe **what I know and build**.

The last directories describe **how I learn and maintain that knowledge**.

### 5. Optimize for long-term maintenance

Every article should have a clear purpose.

Before creating a new article, ask:

* Is it about building something?
* Is it about AI systems?
* Is it about fundamental principles?
* Is it about using a tool?
* Is it about managing knowledge?

The answer determines where it belongs.

## Evolution

This structure is not fixed.

As the knowledge base grows, the organization may evolve.

When making structural changes, record:

* What changed?
* Why was it changed?
* What problem does the new structure solve?

KM exists not only to document the current structure, but also to preserve the reasoning behind future changes.
