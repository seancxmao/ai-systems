# Overview

## 依赖关系视角

> math → algorithm → programming → computer systems → software design

* math
  * 提供形式化思维（离散、概率、线性代数等）
* algorithm
  * 基于数学抽象出的问题求解能力
* programming
  * 把算法落地成可执行表达
* computer systems
  * 理解程序运行的现实环境（CPU / OS / memory / network）
* software design
  * 在系统约束之上做结构化设计（模块化、扩展性、架构权衡）

> software design 的“设计空间”依赖 computer systems 的“运行约束空间”

比如：

* 并发模型（线程 vs async）来自 OS / runtime
* latency / throughput tradeoff 来自系统结构
* 分布式设计来自 network + consistency model

systems 放在 design 前面，是工程上更自然的。

## 抽象层级视角

如果按抽象层级，从“低层现实 → 高层结构”：

```
数学 (形式化基础)
  ↓
算法 (问题求解)
  ↓
编程 (表达与实现)
  ↓
计算机系统 (运行机制与约束)
  ↓
软件设计 (在约束下做结构化组织)
```

这个结构是“自底向上构建能力”的典型路径，也符合大多数计算机科学课程体系的演进逻辑。

* software design 是 建立在 systems 约束之上的决策层
* systems 不是“可选背景知识”，而是设计边界条件

## 学习路径 vs 工作顺序

这个顺序是：

> 学习认知路径，不是“实际工程工作顺序”

在真实工程中是反过来的：

* software design ↔ systems ↔ programming 是强耦合循环，不是“单向流程”
* 设计会不断被系统知识反推修正

## 总结

> math → algorithm → programming → computer systems → software design

在“从基础到工程抽象能力递进”的逻辑上更清晰，尤其适合：

* 构建知识体系（learning map）
* 面向 AI infra / system engineering 的成长路径
* 强化“系统约束驱动设计”的思维
