---
title: "VSCode 中的代码高亮"
permalink: /vscode-tokenizer/
---

# VSCode 中的代码高亮

VSCode 使用 Token 来代表 TextModel 中的高亮数据（比 Lexer 生成的词法单元包含更多的数据）。要讨论 VSCode 中的高亮机制，我们必须要回答三个问题：

1. 高亮的数据从哪来？
2. 编辑器该何时获取/生成高亮数据？
3. 编辑器如何根据高亮数据调整展示给用户的文本？

让我们跟着 VSCode 的源码一点点地回答这些问题。

---

相关文件及类：

- vs/editor/common/model/textModel.ts: `TextModel`

- vs/editor/common/model/textModelTokens.ts: `TokenizationStateStore`、`TextModelTokenization`

- vs/editor/common/core/lineTokens.ts:`IViewLineTokens`、` LineTokens`

- vs/editor/common/modes.ts: `ITokenizationSupport`、`TokenMetadata`、`DocumentSemanticTokensProvider`

- vs/workbench/services/textMate/browser/abstractTextMateService.ts：`TMTokenizationSupport`、`TMTokenization`、`AbstractTextMateService`

- vs/editor/common/services/getSemanticTokens.ts：`getDocumentSemanticTokens`

- vs/editor/common/services/modelServiceImpl.ts：`ModelSemanticColoring`、`SemanticColoringFeature`、`SemanticStyling`、`SemanticTokenResponse`

- vs/editor/contrib/viewportSemanticTokens.ts：`ViewportSemanticTokensContribution`

  

---

`TextModel` 是 VSCode 编辑器的核心，我们不妨从它开始着手。

在 VSCode 的 `TextModel` 中，和 Tokenization 相关的数据有：

- `LanguageIdentifier`：表示文本对应的语言 Id
- `TokensStore`：根据语法生成的 Token
- `TokensStore2`：根据语义生成的 Token
- `TextModelTokenization`：用于提供语法 Token 的类

VSCode 的早期版本中仅提供了基于语法的高亮，这会导致用户通过高亮能获得的信息远少于编译器前端能够生成的；在某一次版本更新中，VSCode 添加了对于 语义 Token 的支持，即能够在 LSP 的支持下向代码中添加基于语义的高亮（例如能够区分不同 identifier 之间的区别）；在代码里，这两种 Tokenize 结果也分别存在 `TokensStore` 和 `TokensStore2` 中。语法高亮相对于语义高亮来说对于响应速度的要求会更高。



## 语法高亮

我们先来讲 语法高亮——即 `TokensStore` 中的数据 是怎么来的。VSCode 中大量使用了依赖注入的方式进行行为与调用的解耦，Tokenize 也不例外。Tokenize 的行为通过接口 `ITokenizationSupport` 进行了定义：

```typescript
export interface ITokenizationSupport {
	getInitialState(): IState;
	tokenize(line: string, hasEOL: boolean, state: IState, offsetDelta: number): TokenizationResult;
	tokenize2(line: string, hasEOL: boolean, state: IState, offsetDelta: number): TokenizationResult2;
}
```

值得一提的是，`tokenize` 和 `tokenize2` 提供的是（语义上）相同的 Tokenize 结果，而它们的区别仅仅在于 Token 的表示形式：`tokenize` 是早期 Monaco Editor 使用的 Tokenize 接口，并在 Monaco Editor 以 VSCode 的子模块进行开发后仍然在使用这个接口。而在 Monaco Editor 中，Token 使用一个二元组来表示：

```typescript
export class Token {
	public readonly startIndex: number;
	public readonly type: string;
}

let tokens = [
  { startIndex: 0, type: 'keyword.js' },
  { startIndex: 8, type: '' },
  { startIndex: 9, type: 'identifier.js' },
  { startIndex: 11, type: 'delimiter.paren.js' },
  { startIndex: 12, type: 'delimiter.paren.js' },
  { startIndex: 13, type: '' },
  { startIndex: 14, type: 'delimiter.curly.js' }
];
```

这样的表示会占据大量的额外空间，VSCode 开发者在[一次优化](https://code.visualstudio.com/blogs/2017/02/08/syntax-highlighting-optimizations)中通过一个整数代替了上述的字符串 Token 类型表示；为了兼容旧代码，使用了 `tokenize2` 这种形式避免了名字冲突。

回到 `ITokenizationSupport`，它提供的 Tokenize 接口实际上是以行为单位的增量方法：调用者可以通过 `getInitialState` 生成初始状态，并逐步通过 `tokenize2` 方法获取当前行中的 Token 以及解析完成后的状态（用于下一次解析）。

`ITokenizationSupport` 的实现者——即语法高亮提供者，在 VSCode 中（而非 Monaco Editor）以 TextMate 实现，对应于 `TMTokenizationSupport`。

`TMTokenizationSupport` 是 `TMTokenization` 的一层封装，并添加了对于超过 Tokenize 阈值的行的处理；而 `TMTokenization` 则添加了对于语言嵌套的支持（例如 Markdown 中的 Java 代码），并把 Tokenize 请求转发给 `IGrammer` 的实例。

而 TextMate 对应的所有 Tokenize 以及注册管理则通过 `TextMateService` 进行管理，它会通过 `TMGrammarFactory` 获取到 `TMScopeRegistry` 中保存的 `IValidGrammarDefinition`，并调用 `loadGrammarWithConfiguration` 以获得对应语言的 `IGrammar` 实例。

至此，我们已经知道了 Tokenize 行为的由来；那么我们来看下一个问题：编辑器是如何获取这些数据的？

我们从 `TextModelTokenization` 开始，它提供的方法有：

- `reset`：重置当前 `TextModel` 的语法 Token 和 Tokenization 状态
- `forceTokenization`：强制完成某块指定区域的 Tokenization 并更新 TextModel 中的数据
- `isCheapToTokenize`：判断某一行的 Tokenization 计算是否昂贵
- `tokenizeViewport`：对 viewport 区域内的文本进行重新 Tokenize 并更新 TextModel 数据

> 那么增量 Tokenization 是怎么做的？
>
> 之前我们提到 `IState` 是用于表示 Tokenize 中间状态的数据，并能够用于增量 Tokenize；而 `TextModelTokenization` 中的 `TokenizationStateStore` 则保存了每一行可能的 `IState` 以及这些缓存的可用性；在检查 `isCheapToTokenize` 时会检查缓存中的数据是否已经包括了当前需要的行、`tokenizeViewport` 也会找到最近的保存了 Tokenize 状态的行开始重新对 Viewport 里的内容进行计算。

虽然 `TextModelTokenization` 提供了直接获取某段文本的 Tokenize 结果的方法，但它同时也会异步地对全文进行 Tokenize 计算：

当 Token 被 reset、TextModel 中的内容发生了全量变更或语言进行了切换时，`TextModelTokenization` 会启动后台 Tokenize 过程：

```typescript
class TetxModelTokenization {
    private _beginBackgroundTokenization(): void {
        if (this._textModel.isAttachedToEditor() && this._hasLinesToTokenize()) {
            platform.setImmediate(() => {
                if (this._isDisposed) {
                    // disposed in the meantime
                    return;
                }
                this._revalidateTokensNow();
            });
        }
    }
}
```

如果当前的 TextModel 绑定在编辑器上 且 Tokenize 并没有全部完成，那么在下一个 Tick 中请求一次 `_revalidateTokensNow`

```typescript
class TetxModelTokenization {
    private _revalidateTokensNow(toLineNumber: number = this._textModel.getLineCount()): void {
        const MAX_ALLOWED_TIME = 1;
        const builder = new MultilineTokensBuilder();
        const sw = StopWatch.create(false);

        while (this._hasLinesToTokenize()) {
            if (sw.elapsed() > MAX_ALLOWED_TIME) {
                break;
            }
            const tokenizedLineNumber = this._tokenizeOneInvalidLine(builder);
            if (tokenizedLineNumber >= toLineNumber) {
                break;
            }
        }
        this._beginBackgroundTokenization();
        this._textModel.setTokens(builder.tokens);
    }
}
```

`_revalidateTokensNow` 会试图在 1ms 内完成尽量多逻辑行的 Tokenize 结果，并通过 `MultilineTokens` 的形式添加到 TextModel 中；在此后继续试图不阻塞地对文本进行 Tokenize。

上面的 Tokenize 行为相当于某种非阻塞的全局 Tokenize 进程，而 `forceTokenization` 和 `tokenizeViewport` 则是给 TextModel 一个主动进行 Tokenize 的方式。

`forceTokenization` 主要用于在必须使用到 Token 信息的场合，例如：缩进的计算需要当前的 Token 信息、光标移动、括号补全等；而 `tokenizeViewport` 则是作为 `ViewModel` 的 Tokenize 方法的具体实现，在滚动区域发生变化、或者 Tokenizer 提供者发生变化时，会以 50ms 的延迟作为异步任务被完成，从而使得用户能够及时地获取视野中的代码高亮。

> 不过和正常的全文代码高亮不同的是，`tokenizeViewport` 如果不能直接使用到缓存的 Tokenize 结果及状态，则会使用一个轻量级的 Tokenize 策略：找到保存了 Tokenize 状态中里当前 Viewport 最近的保存了状态的逻辑行，然后从该行开始收集文本中包含了非空格字符的行并进行 Tokenize 直到遇见视野内的逻辑行，并完成实际的 viewport Tokenize 流程；不过这里生成的内容可能是错误的，所以不会在缓存中保存下这些 Token



## 语义高亮

在前文中我们提到，语义高亮的数据均保存在 `TokenStore2` 里，那么它的数据是哪来的呢？

根据 `setTokens2` 的调用关系，很容易发现直接调用者是 `ModelSemanticColoring`。在`ModelSemanticColoring` 中会保存一个 Scheduler，这个 Scheduler 会在 300ms 后调用一次 `fetchDocumentSemanticTokensNow`，其逻辑如下：

- 首先检查上一次调用时保存的 CancellationToken 是否还存在，如果存在则表示上一次请求仍然没有结束，在此之前我们不应该提交更多的请求；否则创建一个新的 CancellationToken 用来表示这次的请求

- 获取上一次结果的 Id

- 通过 `DocumentSemanticTokensProviderRegistry` 获得当前的 TextModel 对应的 `DocumentSemanticTokensProvider`；如果没有则直接返回

- 创建一个 TextModel Listener，在此期间的 TextModel 变更将会通过此 Listener 存储到 `pendingChanges` 队列里

- 在 Provider 提供的能力创建的 Promise 里设置成功和失败回调

- 在成功的场合，首先清除掉保存的 CancellationToken 和注册的 TextModel Listener，然后对 Provider 返回的语义 Token 结果进行检查

  如果返回的 Token 为空或者缺失高亮样式数据等情况则会清空 TextModel 中的语义 Token 并（可能）获取一次新的获取语义 Token

  Provider 提供了两种获取 Token 的方式：全量和增量

  在增量场景下，Provider 会提供 `SemanticTokensEdits` 作为返回值，它是基于 Provider 上一次返回的 `SemanticTokens` 上进行 diff 得到的结果；而全量场景下则会直接提供所有的语义 Token

  根据不同的方式计算出当前文本的语义 Token 后，还需要根据 TextModel 目前的版本和当时提交请求时的版本间发生的编辑，也就是通过 Listener 记录下的 `pendingChanges` 队列来对语义 Token 结果进行修改，以保证最终获得的语义 Token 是和文本能够对应的数据。

  最后通过 `TextModel::setSemanticTokens` 来设置语义 Token

`fetchDocumentSemanticTokens` 会在以下场合被调用，没有特殊声明则会经过默认 300ms 的延时：

- TextModel 的内容发生了变更
- TextModel 的语言发生了变化：立即请求
- 当前 TextModel 对应的 `DocumentSemanticTokensProvider` 发生了变化：立即请求
- `DocumentSemanticTokensProviderRegistry` 发生了变化：添加对 TextModel 的语义 Token Provider 的监听，并获取一次语义 Token
- 根据 `ThemeService` 获取到的主题变化
- 在 `fetchDocumentSemanticTokens` 中的 `pendingChanges` 不为空时
- `ModelSemanticColoring` 的创建时

那么 `ModelSemanticColoring` 又是如何被创建的呢？

我们注意到，VSCode 的依赖注入中存在一个用于进行 TextModel 管理的 Service：`ModelService`，它提供了 TextModel 的创建、内容更新、模式选择等 API，同时也控制了一个 `SemanticColoringFeature` 作为它的子模块。`SemanticColoringFeature` 在 `ModelService` 通知其关于 TextModel 的创建等事件时，在满足能够使用语义 Token 的情况下则会对这个 TextModel 添加对应的 `ModelSemanticColoring` 实例，用于提供上述的 schedule 语义 Token 等功能。

类似于语法高亮，语义高亮也会对 Viewport 中的内容做单独的优化。`TextModel` 中提供了 `setPartialSemanticTokens` 来对某个区间内部的语义 Token 进行更新，而这个 API 的调用者是 `ViewportSemanticTokensContribution`，它提供了 `_tokenizeViewportNow` 用于对 Viewport 内的文本进行部分的语义高亮，其逻辑如下：

- 如果编辑器并没有 TextModel 、编辑器的 TextModel 已经包含了完整的语义 Tokenize 结果或当前 TextModel 没有启用高亮，则直接放弃并可能的话清除掉编辑器的语义高亮

- 通过 `DocumentRangeSemanticTokensProviderRegistry` 来获取当前 TextModel 对应的 `DocumentRangeSemanticTokensProvider`

- 获取编辑器当前的可视区域，对于所有的文本范围创建一个 Token 请求 Promise 并记录 TextModel 的 VersionId：如果在 Promise 得到结果时 VersionId 与发出请求时不同，则直接放弃。

- 这些 Promise 将通过 CancellationToken 包装成 `CancelablePromise` 存储在数组里，并在

  - TextModel 发生变化
  - TextModel 内容发生变化
  - ProviderRegistry 发生变化
  - Configuration 中的 `SEMANTIC_HIGHLIGHTING_SETTING_ID` 发生变化
  - 主题颜色发生变化

  时对这些任务进行撤销，并重新预约（延迟为 100ms） `_tokenizeViewportNow` 来获取最新的 Token 数据



## 数据存储

上面我们讨论的是数据的来源，那么这些数据又是如何表示、并最终成为我们看到的 DOM 结点的呢？

先来看语法 Token。它们以 `TokensStore` 的形式保存在 TextModel 中，而 `TokensStore` 里是用 `Uint32Array[]` 来存储 Token 的。

VSCode 里的 Token 一般是使用两个 32 位整数来表示：Token 结束位置对应的 utf16 文本长度和一个元数据。其中元数据的表示如下：

```
3322 2222 2222 1111 1111 1100 0000 0000
1098 7654 3210 9876 5432 1098 7654 3210
bbbb bbbb bfff ffff ffFF FTTT LLLL LLLL
```

- 8bit L：Language ID，从 0 开始递增的语言 ID
- 3bit T：StandardTokenType，区分了注释、字符串、正则和其它的 Token 类型
- 3bit F：FontStyle，字体类型，包含了斜体、粗体、下划线
- 9bit f： Foreground Color，文本前景的 Id
- 9bit b：Background Color，文本背景的 Id

每个 `Uint32Array` 用来表示一行内的 Token，每一行对应于一个数组。而 Token 中的偏移量也仅仅是逻辑行内的文本偏移量。这样的表示使得用于表示 Token 的内存占用可以变得非常小，不过基于行的数组也使得这种方式无法支持过于倾斜的文本（过长的行或者过多的行）：因为对于文本的删除需要处理其跨越的所有逻辑行，对于二维数组而言开销相对较大。

而对于语义 Token 而言，语言服务并不会对所有的文本都添加语义 Token，即它们可能是不连续的。这些不连续的 Token 保存在 `TokensStore2` 中的 `MultilineTokens2` 里。在 `MultilineTokens2` 的 `SparseEncodedTokens` 中。每个语义 Token 对应于一个 `Uint32Array` 数组中的 4 个整数：相对于开始的逻辑行的逻辑行偏移、起始位置相对于当前逻辑行的开始的偏移量、结束位置相对于当前逻辑行的偏移量、Token 元数据。由于需要保存更多的数据，语义 Token 在遇到文本编辑时需要进行更加精细的调整。

对于 `TokensStore` 和 `TokensStore2` 的查询往往只通过 TextModel 的 `getLineTokens`。此方法首先从 `TokensStore` 获取特定逻辑行的语法 Token（使用包含了 `Uint32Array` 的 `LineTokens` 表示），然后再从 `TokensStore2` 获取语义 Token 并添加到此结果上。语义 Token 相对于语法 Token 具有更高的优先级，VSCode 在实现时会按照顺序同时遍历语法 Token 和语义 Token，根据语义 Token 覆盖的范围对语法 Token 进行切割，然后使用语义 Token 元数据中的样式覆盖原本的 Token 样式得到最终的生成结果。



## 总结

不难看出，VSCode 中对于高亮持有的是一种非常实用主义的态度（~~能用就行~~）。相较于 IDEA 中统一使用 `RangeMarker` 进行管理的形式相比，直接使用二维数组的 Token 表示在一定程度上节省了内存开销和对于大部分正常代码编辑的高亮更新使劲按，同时，VSCode 采取的异步非阻塞式语法高亮策略避免了多进程通信问题和复杂高亮计算可能导致的编辑器 UI 响应迟缓；但 VSCode 也因此放弃了对于逻辑行内部的增量 Tokenize 等功能，使得对于部分场景下的文本编辑性能较差。



Reference:

https://code.visualstudio.com/blogs/2017/02/08/syntax-highlighting-optimizations

https://code.visualstudio.com/api/language-extensions/semantic-highlight-guide

https://github.com/microsoft/vscode

