# Python Debug

## 背景

对于日常 Python 开发，需掌握下面 4 种 Debug 方法：

| 方法          | 适用场景     |
| ----------- | -------- |
| print调试     | 最快定位问题   |
| pdb命令行调试    | 理解程序执行过程 |
| VS Code断点调试 | 工程开发主流   |
| 单元测试调试      | 修复复杂Bug  |

## print Debug

```
def divide(a, b):
    result = a / b
    return result

print(divide(10, 0))
```

```
def divide(a, b):
    print("a =", a)
    print("b =", b)

    result = a / b

    print("result =", result)

    return result
```

## pdb Debug

```
import pdb

def add(a, b):
    c = a + b
    return c

pdb.set_trace()

x = add(1, 2)
print(x)
```

运行`python app.py`自动进入pdb。

常用命令

* n: next, 执行下一行
* s: step, 进入函数
* c: continue, 运行到结束
* p x: print, 查看变量
* l: list, 查看附近代码
* q: quit, 退出


## breakpoint()

现代写法，Python 3.7+推荐代替`pdb.set_trace()`

```
def add(a, b):
    c = a + b
    breakpoint()
    return c

add(1, 2)
```

运行`python app.py`自动进入pdb。

## VS Code Debug

### Debug Configuration
VS Code Python 开发里最重要的Debug Configuration

| 配置                    | 用途               |
| --------------------- | ---------------- |
| Python File           | 调试当前文件           |
| Launch Module         | 调试 python -m xxx |
| Launch with Arguments | 调试带参数程序          |
| Attach                | 附加到已运行进程         |
| Remote Attach         | 调试远程机器           |

配置最终保存在

```
.vscode/launch.json
```

launch.json本质上就是 VS Code 的 调试启动配置文件。相当于把平时在终端执行的Bash命令转换成转换成JSON配置，以后点击 F5，VS Code 自动启动 Python 进程并挂上 debugger。

### Python File

等价于：

```
python main.py
```

F5 -> Python Debugger -> Python File

### Launch Module

等价于：

```
python -m xxx
```

比如vLLM：

```
python -m vllm.entrypoints.openai.api_server
```

launch.json:

```
{
    "configurations": [
        {
            "name": "Debug Module",
            "type": "debugpy",
            "request": "launch",
            "module": "mypackage.main"
        }
    ]
}
```

F5 -> Debug Module

vLLM:

```
{
    "name": "vLLM API Server",
    "type": "debugpy",
    "request": "launch",
    "module": "vllm.entrypoints.openai.api_server"
}
```

### Launch with Arguments

```
python -m vllm.entrypoints.openai.api_server \
    --model Qwen/Qwen3-0.6B
```

对应：

```
{
    "name": "vLLM Server",
    "type": "debugpy",
    "request": "launch",
    "module": "vllm.entrypoints.openai.api_server",
    "args": [
        "--model",
        "Qwen/Qwen3-0.6B"
    ]
}
```

### Attach Remote

```
{
    "name": "Attach",
    "type": "debugpy",
    "request": "attach",
    "connect": {
        "host": "localhost",
        "port": 5678
    }
}
```

### Attach PID

| 方式               | 原理                |
| ---------------- | ----------------- |
| VS Code Launch   | debugpy随程序一起启动    |
| debugpy --listen | 启动时注入debugpy      |
| debugpy --pid    | 运行后把debugpy注入目标进程 |

debugpy --pid 并不是在外面做代理，而是把调试器“植入”到目标 Python 进程内部。因此它拥有与目标进程相同的权限和能力，这也是它强大且需要权限控制的原因。

## Debug原理

### VS Code Debug原理

VS Code 本身不会调试 Python 程序，它只是一个图形界面；真正执行断点、单步、查看变量等操作的是运行在 Python 进程内部的 debugpy，VS Code 通过 Debug Adapter Protocol（DAP）向 debugpy 发送调试命令，而 debugpy 再直接操作 Python 解释器的执行状态并把结果返回给 VS Code 显示。

对应图：

```
VS Code
    |
Debug Adapter Protocol (DAP)
    |
debugpy
    |
Python Interpreter
```

各层职责非常清晰：

| 组件                 | 职责                  |
| ------------------ | ------------------- |
| VS Code            | 调试界面（断点、变量窗口、调用栈窗口） |
| DAP                | 调试通信协议              |
| debugpy            | Python调试引擎          |
| Python Interpreter | 执行用户代码              |

例如点击`F10 (Step Over)`，实际发生的是：

```
VS Code
    ↓
发送DAP消息
    ↓
"stepOver"
    ↓
debugpy收到命令
    ↓
控制Python解释器执行一行
    ↓
读取新的Frame状态
    ↓
返回变量和当前位置
    ↓
VS Code刷新界面
```

所以`F10`并不是 VS Code 在执行单步，而是：

```
VS Code 发命令
debugpy 执行命令
Python Interpreter 真正运行代码
```

理解这一点后，很多现象都会变得自然。例如：

```python
breakpoint()
```

为什么能触发 VS Code 停下来？因为：

```text
breakpoint()
    ↓
调用debugpy
    ↓
debugpy通知VS Code
    ↓
VS Code显示暂停状态
```

再比如：

```python
x = 123
```

为什么 Variables 窗口能看到 x？因为：

```
VS Code
    ↓
请求变量
    ↓
debugpy
    ↓
frame.f_locals
    ↓
获得 x=123
    ↓
返回给VS Code
```

所以从架构角度看，VS Code Debug Python 可以抽象成：

```
IDE
    ↓
调试协议(DAP)
    ↓
语言调试器(debugpy)
    ↓
语言运行时(CPython)
```

而且这个架构不是 Python 独有的。调试 C++：

```
VS Code
    ↓
DAP
    ↓
cpptools
    ↓
gdb/lldb
```

调试 Java：

```
VS Code
    ↓
DAP
    ↓
Java Debugger
    ↓
JVM
```

调试 Python：

```text
VS Code
    ↓
DAP
    ↓
debugpy
    ↓
CPython
```

因此可以把debugpy 理解成 Python 世界里的 GDB，而 VS Code 只是一个统一的调试前端。这实际上就是现代 IDE 调试体系最核心的 Mental Model。

### debugpy

debugpy 是 Python 官方生态目前主流的调试引擎，可以把它理解为“Python 世界里的 GDB”。它存在的原因是：IDE（如 VS Code）并不了解 Python 解释器内部如何设置断点、暂停线程、单步执行或读取变量，因此需要一个运行在 Python 进程内部的调试器来完成这些工作。其基本原理是：VS Code 通过 DAP（Debug Adapter Protocol） 向 debugpy 发送调试命令（如设置断点、F10、F11），debugpy 再利用 Python 解释器提供的调试接口（主要是 `sys.settrace()` 和 CPython 的 Frame/Thread 机制）控制程序执行、读取调用栈和局部变量，并将结果返回给 VS Code 显示。无论是 Launch 模式还是 Attach 模式，真正执行调试操作的都是目标 Python 进程内部的 debugpy；VS Code 只是调试前端。理解这一点后，可以形成一个非常重要的 Mental Model：

```
IDE (VS Code)
      ↓
DAP 协议
      ↓
debugpy
      ↓
Python Interpreter
      ↓
你的代码
```

后续学习 vLLM、Ray、多进程服务调试时，只要先搞清楚“debugpy 运行在哪个进程里”，通常就能快速判断为什么断点生效或不生效，这往往是大型 Python 系统调试最关键的问题。

### pdb

pdb调试原理和VS Code是非常类似的。pdb：

```text
Terminal
    |
pdb
    |
Python Interpreter
```

你输入：`(Pdb) n`

实际过程：

```text
Terminal
    ↓
n (next)
    ↓
pdb
    ↓
控制Python解释器执行一行
    ↓
读取当前Frame和变量
    ↓
打印结果到Terminal
```

VS Code Debug：

```text
VS Code
    |
Debug Adapter Protocol (DAP)
    |
debugpy
    |
Python Interpreter
```

你点击：`F10 (Step Over)`

实际过程：

```text
VS Code
    ↓
DAP消息
    ↓
debugpy
    ↓
控制Python解释器执行一行
    ↓
读取当前Frame和变量
    ↓
更新Variables窗口
```

最核心的 Mental Model：

```text
           前端UI

Terminal                VS Code
    |                      |
    |                      |
   pdb                 debugpy
        \            /
         \          /
          \        /
        Python Interpreter
```

两者的区别几乎只有：

| 项目   | pdb    | VS Code     |
| ---- | ------ | ----------- |
| UI   | 命令行    | 图形界面        |
| 下断点  | `b 10` | 鼠标点击        |
| 单步   | `n`    | F10         |
| 进入函数 | `s`    | F11         |
| 查看变量 | `p x`  | Variables窗口 |
| 本质能力 | 相同     | 相同          |

一句话总结：pdb 和 debugpy 本质上都是 Python 调试器，直接控制 Python 解释器的执行；区别只是 pdb 把调试信息显示在终端里，而 debugpy 通过 DAP 协议把调试信息发送给 VS Code 图形界面显示。对于断点、单步执行、查看变量这些核心能力，两者底层原理基本一致。

## 总结

Python Debug常用方法：

* print
* pdb或者breakpoint()
* VS Code debug
* 单元测试

而VS Code debug的多种方式，都是通过配置launch.json完成：

* 以debug方式启动Python file或module，可以传入参数
* Attach到debug server的port，或者本地进程PID

从原理上讲：

* pdb 和 debugpy 本质上都是 Python 调试器，直接控制 Python 解释器的执行
* VS Code 图形界面通过DAP协议与debugpy交互，就好比终端和pdb进行交互一样。
