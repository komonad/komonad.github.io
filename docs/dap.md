---
title: "基于 DAP 的调试功能实现思路"
permalink: /dap/
---

# 基于 DAP 的调试功能实现思路

## Debug Adapter 抽象

对于不确定的 Debug Adapter 实现，我们需要提供一种抽象来代表它的能力，以 VSCode 的 `IDebugAdapter` 为例：

```typescript
export interface IDebugAdapter extends IDisposable {
	readonly onError: Event<Error>;
	readonly onExit: Event<number | null>;
	onRequest(callback: (request: DebugProtocol.Request) => void): void;
	onEvent(callback: (event: DebugProtocol.Event) => void): void;
	startSession(): Promise<void>;
	sendMessage(message: DebugProtocol.ProtocolMessage): void;
	sendResponse(response: DebugProtocol.Response): void;
	sendRequest(command: string, args: any, clb: (result: DebugProtocol.Response) => void, timeout?: number): number;
	stopSession(): Promise<void>;
}
```

考虑到对现有的 Adapter Server 支持：最基本的 Adapter Client 应该是基于 Executable 的 Server 并通过标准输入输出进行通信。我们就会又出现两个问题：

1. 如何创建这些 Adapter Server
2. 如何与这些 Adapter Server 进行基于 DAP 的通信

创建基于可执行文件的 Adapter 我们至少需要：

- 可执行文件的路径
- 它的工作目录
- 它的环境变量
- 它启动时的参数（需要调试的东西）

Adapter Client 会根据这些参数来创建对应的 Adapter Server，获得其标准输入输出的句柄。

但对于 IDE 而言，如何获取这些数据？一种比较简单的方法就是类似于 VSCode 的配置文件，在启动调试时根据文件获取这些额外信息。

对于第二个问题，我们需要提供对于 DAP 这种类似于 HTTP 协议的报文头解析、以及数据的序列化和反序列化。



## DAP 初始化流程

在拥有上面的 Adapter Client 后，我们还需要在 IDE 中管理调试会话，使得其能够使用到 Client 的能力。

典型的工作模式如下：

用户触发调试行为，IDE 根据工作目录下固定位置的文件读取调试配置并解析；然后创建该配置对应的 Adapter Client；随后，IDE 需要启动一个调试 Session 来表示本次调试中所有的上下文信息。启动的流程包括：

- 准备好 Adapter Server 以及 Adapter Client，注册 DAP 相关请求和事件处理函数
- Client 根据 DAP 协议向 Server 发送初始化请求，获取 Server 的 Capabilities
- 如果 IDE 此时拥有一些已经设置的断点，也需要对 Debug Adapter 进行数据的传输和准备
- 通过 Client 发送 Launch Request
- Server 根据已有的调试会话相关信息配置好 Debugger 开启调试，并对 UI 进行一定程度的调整



## DAP 事件

针对 DAP 中的重要事件，其发生原因和默认的处理流程一般为：

- Initialized：Debug Adapter Server 完成了初始化；上述的启动流程中发送初始化请求后的部分
- Stopped：Debuggee 中某个线程由于各种原因发生了调试中断，使得当前处于暂停状态；需要根据事件中的数据来更新保存的线程状态以及清除被暂停的线程的堆栈信息，然后重新通过 DAP 获取被暂停的线程的 Stack Trace，以及在 UI 上更新高亮的栈帧和行
- Thread：Debuggee 中的线程发生了变更（生成了新的线程/线程退出）；对 保存的线程信息进行更新，如果焦点线程退出则需要重置高亮的栈帧信息
- Terminated：Debuggee 被终结；根据事件信息判断是否需要重启，否则断开 DAP 连接
- Continued：Debuggee 线程处于继续执行的状态；重置该线程的状态
- Output：Debugger 产生了一些输出；根据输出的类型决定是否需要在 IDE 的调试 Repl 里进行添加
- Breakpoint：Debugger 对断点的状态进行了更新，例如移除了不可用的断点；同步本地的断点数据



## UI 功能

那么再来看 IDE 的 UI 层，或者说需要展示给用户的数据，**至少**需要支持以下功能：

- 能够在文件的某一行打上断点，该断点会视作跟随于该行末尾的换行符的一个标记，并能够在文本内容发生变更时同步移动；如果该换行符被删除，则该断点也会被删除。
- 能够显示 Debuggee 的线程数据：线程的名字、线程执行的状态
- 对于处于 Paused 状态的线程，能够显示其调用栈信息：包括每个栈帧当前的函数名、文件名和行数
- 能够根据栈帧中的状态数据（文件以及行号）对编辑器里的特定行进行高亮
- 能够显示当前焦点栈帧的作用域信息
- 能够添加监视变量
- 提供用于控制调试状态的按钮组，能够支持 Continue/Next/Step In/Terminate 功能



## 调试流程中的一些典型动作

1. 添加断点：

   无论是否处于调试状态，用户都能够在**启用了调试支持**的文档上添加跟随换行符的断点，这些断点信息会通过存储服务进行保存后再发送到可能的 Debug Adapter 中（如果当前不存在活跃的 Debug Session 则可以直接跳过）。Adapter Server 接收到 SetBreakpoints 请求后会对数据进行验证，更新本地维护的断点状态并返回验证后的断点数据；Client 接收到这些数据后会对本地的断点进行更新，从而达到类似于下面这种场景的效果：

   - 未开启调试时在空行添加的断点在启动调试后会被移动到其下方第一个有映射的代码行中，且此时**无法**在该处添加可用的断点

2. 开启调试：

   在所有初始化完成并开启调试后，执行到了断点处使得某个线程（或者所有线程）处于了暂停状态并向 Client 发送了 Stopped 事件。根据上面描述的 Stopped 事件流程，IDE、也就是 Client 需要发送一次 StackTrace 请求，并通过 Adapter 中的 Callback 获取该请求的返回数据。在异步地拿到调用栈信息后：

   1. 设置调试 UI 上的焦点线程，并使该线程最顶层的栈帧获得焦点（指在 UI 中展示出来并将对应文档行进行高亮）
   2. 在视图上显示调试 UI 的重要组件：例如线程信息窗体、变量查看窗体、监视变量窗体等
   3. 试图使得 IDE 获得焦点（触发断点是会导致聚焦的关键动作）

3. 栈帧变量查看：

   栈帧发生变化时，Client 需要请求一次 Scope 信息来获取当前栈帧的活跃作用域以及可能的作用域中的变量；变量的表示是一个树形的，即每个变量引用都可能会包含一些其它的变量信息，而 DAP 中的 Variables 请求可以获取某个变量引用的子变量，所以 IDE 需要根据情况对 Scope 中的树进行扩展/遍历，来更新 UI 里展示的变量信息。

4. 表达式求值：

   在调试过程中可以通过一些交互手段来对 基于某个栈帧的表达式 进行求值，该表达式能够获得栈帧可见的所有变量的访问权限。例如：

   - 在调试 REPL 里输入表达式
   - 在监视窗口添加新的监视项
   - 通过鼠标悬浮触发表达式求值*

   表达式监视需要在栈帧焦点发生变化（无论是失去焦点还是更改栈帧）时，重新进行表达式求值的请求

5. 调试状态控制：

   基础的控制手段有：Continue/Next/Terminate，在执行时需要先取消掉目前的栈帧焦点（在 UI 上体现为栈帧的行高亮消失），然后发送对应的 DAP 请求；直到下一次收到暂停事件，再重新调整栈帧焦点并设置高亮。





## 调试功能的可扩展性

上面我们讨论了 Debug Adapter 以及调试流程中需要的 IDE 功能支持。

为了提供某种调试功能支持，我们至少需要解决这些问题：

1. 在哪些文档上能够打断点？

   IDE 需要维护一份支持断点的文档类型列表（和文件后缀名无关的文档“语言”属性）；对于满足要求的文档，我们在点击侧边栏时才能够产生一个未被验证的断点数据

2. 有哪些 Adapter 可以使用？

   IDE 需要维护支持的 Adapter 列表，以及开启调试会话需要的相关信息：例如 Server 的地址和端口、Executable 的路径和启动参数，该 Adapter 的 Capability

3. 如何启动调试？

   IDE 需要维护调试会话启动的配置信息，IDE 提供的接口要足够抽象来支持不同的 Adapter 需要的配置数据；在启动时 IDE 根据当前工作区里的配置文件信息来决定使用的 Adapter 和需要传入给调试会话的启动参数

   



## 总结

实现调试功能必须要完成的功能模块：

1. 调试配置管理：即 IDE 如何读取调试配置文件、调试配置文件里的数据是什么（对应于 `vscode.DebugConfigurationProvider`）
2. 调试 Adapter 管理：有哪些可用的调试器、如何启动对应的调试会话（对应于 `vscode.DebugAdapterDescriptorFactory`）
3. DAP 连接管理：根据 Adapter 的类型决定如何建立字节流连接，对数据进行序列化和反序列化、支持 DAP 事件监听和异步的 DAP 请求调用（封装 DAP 协议中的序列号处理流程）（对应于 `RawDebugSession` 以及 `IDebugAdapter`）
4. 调试会话管理：根据已有的调试配置创建调试会话并启动，支持各种 DAP 无关的调试功能（对应于 `DebugSession`、`DebugModel`）
5. 调试 UI：根据调试会话状态、数据和事件来对 IDE 的调试 UI 状态和内容进行修改（对应于 `ViewModel`、`DebugViewPane`等）



和 IDE 其它功能的联系：

1. 调试 Command 及快捷键
2. 断点和调用栈高亮需要使用到编辑器的 Decoration 和 reveal 功能
3. 调试输出窗口

