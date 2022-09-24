---
title: "IntelliJ IDEA 编辑器布局计算实现"
permalink: /idea-editor-layout/
---

# IntelliJ IDEA 编辑器布局计算

IDEA 编辑器中的内容绘制、编辑器大小管理和坐标转化等功能都由 `EditorView` 完成，包含了大量的排版相关业务逻辑，例如 Inlay、Bidi 支持等等。我们来看一下 IDEA 是如何实现这些功能，并组织好这部分代码的。

一些前置知识

- Logical Position：包含逻辑行号和逻辑列号以及可选的前倾 flag：`leansForward`；其含义为 Logical Position 与文本中两个字符中的边界与前后两个字符之一进行关联（在 Bidi 中发挥作用）
- Visual Position：与 Logical Position 类似，保存 Visual Line 和 Visual Position 以及右倾 flag：`leansRight`（类似于 `leansForward` 但实际含义与用户视角相关联）；需要考虑 Soft Wrap、代码折叠、Inlay 和 处理 Bidi 导致的与 Logical Position 之间不连续的对应关系（折叠块或 Inlay 整个算一个 Column）。
- IDEA 编辑器基本概念：`Editor`/`Document`/`Model`/`RangeMarker`/`Inlay` 等

## `EditorView`

先来看一下和 `EditorView` 有关的一些类，首先是它的成员：

- `EditorPainter`：负责绘制编辑器内的所有东西（文本、高亮、Inlay、边框、前景背景、光标等）
- `EditorCoordinateMapper`：负责进行坐标转换
- `EditorSizeManager`：计算编辑器内部元素大小
- `TextLayoutCache`：以逻辑行为单位，对行的 Layout 进行保存的 LRU 缓存
- `LogicalPositionCache`：缓存 UTF-16 编码映射
- `CharWidthCache`：缓存每个字符的宽度
- `LineFragment`：在进行行排版时的基础单元，能够完成一部分坐标转换以及绘制自身
- `LineLayout`：`TextLayoutCache` 创建和保存的逻辑行中的**文本 Layout 信息**，由一系列 BiDi Run 构成，每个 BiDi Run 由 `TextFragment` 组成的若干 `Chunk` 构成（出于性能因素），字形的协同只要求在 `Chunk` 内满足即可

其中 `LogicalPositionCache`、 `TextLayoutCache` 和 `EditorSizeManager` 都实现了 `PrioritizedDocumentListener`，我们再把这个优先级表拿出来溜一下

```java
public final class EditorDocumentPriorities {
  public static final int RANGE_MARKER = 40;

  public static final int FOLD_MODEL = 60;
  public static final int LOGICAL_POSITION_CACHE = 65;
  public static final int EDITOR_TEXT_LAYOUT_CACHE = 70;
  public static final int LEXER_EDITOR = 80;
  public static final int SOFT_WRAP_MODEL = 100;
  public static final int EDITOR_TEXT_WIDTH_CACHE = 110;
  public static final int CARET_MODEL = 120;
  public static final int INLAY_MODEL = 150;
  public static final int EDITOR_DOCUMENT_ADAPTER = 160;
}
```

可以看到在 `FoldingModel` 处理完并发送相应的事件后，依次由 `LogicalPositionCache`、`TextLayoutCache`、`SoftWrapModel`、`InlayModel` 等进行处理，它们的具体功能会在下面进行讲解。

## `LogicalPositionCache`

偏移量与 Logical Position 的映射逻辑在 `LogicalPositionCache` 里完成。它作为除了 `FoldingModel` 之外最优先的 `DocumentListener`，需要给后续的所有排版提供基本的数据支持。

不是说 `Document` 支持对 Logical Line 的维护吗，为什么这里还需要单独一个 Cache 类呢？

答案其实很简单：因为 `Document` 作为一个 `CharSequence` 维护的是 UTF-16 作为单位的序列；而在计算 Logical Column 时，我们需要的是其 Code Point 在计算过 Tab 的影响后对应的位置。这也就导致在计算偏移量到 Logical Position 时，虽然能直接从 `Document` 获得它对应的 Logical Line，但 Logical Column 还是需要从行首进行遍历，处理 UTF-16 中的 Surrogate Pairs 和编辑器 Tab 字符配置。在逻辑行长度不可控的情况下，我们有必要对这部分数据进行缓存。同时，我们可以对没有这些特殊情况的 Logical Line 进行优化，直接使用 字符偏移量 即可。

## `TextLayoutCache`

文本的排版主要分为几个流程：将文本切分为逻辑行，将逻辑行的内容按照各种规则切分为一个个小段、把这些文本段按照正确的顺序进行摆放在视觉行上。而中间那个步骤生成的就是 `TextLayout`。`LineLayout` 以 `Chunk` （每个 `Chunk` 对应于一个 `BidiRun` 中的最多 1024 个字符）为单位维护生成的 `LineLayout` 里的数据，这些缓存以 LRU 的形式维护在 `TextLayoutCache` 中，在活跃的编辑器中最多维持 1000 个、不活跃的编辑器只维护 10 个。

`LineLayout` 计算和保存都由 `TextLayoutCache` 提供，`TextLayoutCache` 处理 `Document` 变更的事件紧接在 `LogicalPositionCache` 前，并需要用于后续的 Lexer  计算和 Soft Wrap 计算等。（需要注意的是：`TextLayoutCache` 实际计算 `LineLayout` 的时机是在 `Document` 变更完成之后的，即已经能够获取当前的 Tokenize 结果）

作为 `PrioritizedDocumentListener`，它会监听 `Document` 变更事件：

- `beforeDocumentChange`：记录下本次编辑事件影响的最后的文本位置
- `documentChanged`：根据本次编辑事件产生的变更，调用 `invalidateLines`

`invalidateLines` 会清除掉对应区域缓存的 `TextLayout`，它接受 5 个参数，分别代表：

- `startLine`： 清除区域开始的行
- `oldEndLine`： 变更之前对应区域最后的行
- `newEndLine`： 更改之后添加的最后的行
- `textChanged`： 文本是否发生变更（Tokenize 结果或字体变更也可以触发缓存失效）
- `bidiRequiredForNewText`：新增文本是否需要 Bidi 支持

首先，对于 `startLine` 到 `min(oldEndLine, newEndLine)` 的 `LineLayout` 缓存，它们肯定是直接失效的，我们需要将这些 `LineLayout` 里的 `Chunk` 从缓存中移除；在删除之后，如果 文本发生了变化且文本需要 Bidi 支持 或者 原文本是 RTL 文本，那么设置本行需要在计算时考虑 Bidi 的影响。

接下来分两种情况：

1. `oldEndLine` < `newEndLine`：需要在此之后新增一些 `LineLayout` 
2. `oldEndLine` > `newEndLine`：需要移除剩下那部分的 `LineLayout` 缓存

至此，`TextLayoutCache` 里过时的缓存已经被清除。那么什么时候它会向里面存储呢？`TextLayoutCache` 采取的是 Lazy 的策略，只在其它组件进行询问的时候才进行 `LineLayout` 的创建并完成存储。

`LineLayout` 的创建过程分为两个部分：第一部分是根据 Bidi 规则将文本分为一个个 `BidiRun`，保证每个 `BidiRun` 中的文本拥有同一个方向属性；第二部分则是将 `BidiRun` 按照  Bidi 规则整理出正确的视觉顺序，并计算出每一个 `BidiRun` 在视觉起始位置上对应的逻辑位置。

第一部分的逻辑如下：

- 根据逻辑行获取 `Document` 中的 `char` 数组
- 如果当前文本不需要 Bidi 支持或者没有 RTL 文本，那么很简单，直接返回一个 `BidiRun` 即可
- 否则通知 `Editor` 找到了 Bidi 文本，以助于后续用户交互提示；然后
- 根据 `EditorHighlighter` 提供的 Tokenize 结果（编辑器需要保证在部分情况下 Token 的视觉位置和它们的语法相同，例如不能因为 Bidi 把 `>=` 变成 `=>`），以及行注释（行注释的起始片段不应该受到 Bidi 影响），将文本处理为一个个可以执行 Bidi 算法的片段
- 对于每个片段，先根据 Tab 字符将文本分为小片段，每个小片段通过 `java.lang.Bidi` 提供的 Bidi 功能支持分割成包含 `level` 和方向的 `BidiRun`

第二部分：

- 如果 `BidiRun` 的 `level` 为 0 且仅有一个 `Chunk`，那么直接返回 `SingleChunk` 作为 `LineLayout` 的实现
- 否则计算出每个 `BidiRun` 的 `visualStartLogicalColumn` 并保存
- 将生成的 `BidiRun` 保存在 `MultiChunk` 形式的 `LineLayout` 中

`LineLayout` 不关心实际的文本中每个元素的具体的位置，这些逻辑将在 `EditorCoordinateMapper` 里进行处理



## `EditorCoordinateMapper`

`EditorView` 提供了下面四种坐标的互相转换功能（共计 12 个方法，但实际上只需要处理相邻的两层即可，即 3 对映射关系）

- UTF16 偏移量：对应的 char 在整个 `CharSequence` 中的位置
- Logical Position：按照折行符分割为逻辑行后的 行号 以及 在该行的位置
- Visual Position：根据折叠区域、Inlay、Soft Wrap 和 Bidi 规则计算出的 Visual Line 行号及位置
- XY：在屏幕/编辑器可滚动区域的位置

这些转换方法要求：

1. 当前线程是 EDT
2. 当前不处于批量修改状态

它们的逻辑在 `EditorCoordinateMapper` 里，下面讨论一下里面的细节



### 偏移量与 Logical Position 的映射

这部分逻辑在 `LogicalPositionCache` 里提供了实现。



### Logical Position 与 Visual Position 的映射

在理想状态下 Logical Position 和 Visual Position 之间应该是一一对应的，那么有哪些东西会影响这种和谐的关系让我们不得不实现这么多复杂的逻辑呢？

- 代码折叠：代码折叠会导致很多 Logical Line 不会有对应的 Visual Line，以及一个 Visual Line 会对应多个 Logical Line
- Bidi：在 LTR 文本中嵌入 RTL 文本，会使得字符在屏幕上的位置与字符在字符串中进行存储的位置

从 offset 到 Visual Line 的计算，首先需要确认该 offset 对应的位置是否处于折叠区域内，如果是，则找到最外部的折叠区域，并调整 offset 到折叠区域的最左侧；然后，根据 offset 找到该位置之前出现过的 Soft Wrap 个数，那么最终的 Visual Line 行号就是：Logical Line Number + Soft Wrap 个数 - 折叠的行数。

获取了行号之后我们需要在这个 Visual Line 上获得它的 Column。IDEA 编辑器没有对 Visual Line 进行维护；他们采取的方式是实现一个迭代器，即 `VisualLineFragmentsIterator`（它所遍历的 Fragment 需要保存相同的字体和方向）完成对 Visual Line 里面各种元素的遍历，并将这些数据保存在迭代器里面，迭代器访问过程中生成的 `Fragment` 仅仅是对于迭代器内容的一个包装。

这个迭代器包含较为复杂的 Folding Region 和 Inlay 处理逻辑

---

下面讨论内容中的一些术语：

- `LineFragment`：`LineLayout` 根据 `EditorHighlighter` 的 Tokenize 结果和行注释规则、字体渲染方法等信息计算出的逻辑行内文本片段
- `VisualFragment` ：在 `LineFragment` 上提供了一些额外功能，`LineLayout` 对外提供的接口
- Segment：文本被 折叠区域 和 Soft Wrap 切分为一些 Segment；对于文本 Segment，需要根据 Inlay 的位置依次遍历每个不包含 Inlay 的区间
- Fragment 迭代器：即遍历 `VisualFragment` 的迭代器

迭代器的初始化流程如下：

- 需要的参数：`EditorView`、当前的 Visual Line、开始的偏移量、开始的逻辑行、当前或上一个 Soft Wrap 的索引、下一个折叠区域的索引、对齐 flag
- 保存 view、`ScaleContext`、Segment 的开始和结束位置、`Document`、折叠区域信息、Visual Line 起始偏移量、当前折叠区域索引、当前的排版横坐标
- 如果当前 Visual Line 是右对齐的，那么使迭代的内容变为 `RightAlignedElement`，并根据 view 提供的右对齐位置设置迭代器的数据
- 如果 Segment 开始位置为 0，那么需要处理文档可能的 Prefix 带来的偏移
- 如果 Segment 开始位置处于某个 Soft Wrap 处，则根据 Soft Wrap 导致的 Pixel 和 Column 变化调整排版位置
- 设置迭代数据：即调用 `setInlaysAndFragmentIterator`

`setInlaysAndFragmentIterator` 的主要作用是：如果迭代器当前访问的不是一个折叠区域，那么准备好对应的 Inlay 数据或者 `LineFragment` 数据（`LineFragment` 的数据主要通过 `LineLayout` 创建的 `VisualFragment` 迭代器进行提供），它的逻辑为：

- 找到下一个折叠区域的开始位置，如果与 Segment 开始位置相同，那么说明这次迭代访问的内容为折叠区域，不需要额外进行处理
- 根据 *下一个 Soft Wrap 位置*、 *逻辑行结束位置* 与 *下一个折叠区域开始位置* 的最小值设置本次 Segment 的范围，同时获得本次 Segment 区间中的 `Inlay` ；如果此 Segment 处没有 Inlay 或者 Inlay 并不处于本 Segment 的开头，则设置 Fragment 迭代器，用于后续的迭代过程。

`VisualLineFragmentIterator` 在迭代时的流程如下：

- 如果当前 Segment 的位置等于当前的折叠区域位置，代表当前 Segment 为折叠区域：那么需要重置保存的 `VisualFragment`、设置当前折叠区域、获取折叠后的 Visual Column 宽度并添加到 Current Visual Column、设置 Segment 开始为折叠区域的结束、调整当前的 X、调整当前的逻辑行、增加折叠区域的序号、重置 Segment 里的 Inlay 
- 如果当前的 Fragment 迭代器为空，代表现在需要访问的 Fragment 是一个 Inlay：那么从保存的 Inlay 列表中获取当前 Inlay 索引对应的 Inlay，根据 Inlay 的大小和缩放倍率调整 X；如果这次获取的 Inlay 是 Segment 中最后一个 Inlay，或者下一个 Inlay 与当前 Inlay 不处于同一个位置，则代表需要开始遍历文本，即设置 Fragment 迭代器
- 当前处于遍历 Fragment 迭代器的过程，则将当前折叠区域置空、重新设置 X、调整 Column 等数据；如果 Fragment 迭代器已经结束，且本 Segment 已经不再有可用的 Inlay，则将 Segment 起始位置移动到 Segment 结束处

在所有 Segment 被遍历完之后，迭代过程结束

---

迭代结束了但事情还没结束，我们还得回到 `EditorCoordinateMapper` 。在获取到 offset 位置开始时的迭代器后，对于每个 `Fragment`：

- 如果当前 Logical Position 并不是连接着上一个字符的状态，且当前 `Fragment` 的起始位置等于 Visual Line 的起始位置，那么直接返回此 Fragment 对应的起始 Visual Column（即取消后续的遍历过程）。注：Bidi 会导致遍历 `Fragment` 的顺序与 `Fragment` 在逻辑上切分后自然排列的顺序不同

- 根据 `Fragment` 的设置当前维护的 最后的逻辑行、最大的 Visual Column
- 如果 `Fragment` 是折叠后的区域：Logical Position 位于此折叠区域真内部而非边界上，则可以直接返回此折叠区域开始的 Visual Column；如果 Logical Position 刚好位于折叠区域的最后 且 `leansForward` 为否（指 LTR 文本中的默认倾向），则返回 `Fragment` 最大的 Visual Column；否则，记录最大的 Logical Column
- 如果 `Fragment` 是文本：Logical Position 与此 `Fragment` 在同一逻辑行并有包含关系（定义参考上文），则调用此 `Fragment` 的 `logicalToVisualColumn` 进行坐标转换
- 如果 Logical Column 超出范围，则将行尾的 Inlay 全部算入此处的 Visual Column 并返回最终的结果

---

上面是 Logical 到 Visual 的计算，反过来如何呢？

类似地，先拿到 Visual Line 对应的 Offset，这里 IJ 的实现槽点很多：它从 0 到 `Document` 的长度进行二分，找到第一个满足其对应的 Visual Line 为输入的 Visual Line 的 Offset（也就是说每一次检查都需要执行一次 `offsetToVisualLine`，而这个方法需要对 `SoftWrapModel` 和 `FoldingModel` 进行查询，整个过程大概是 $O(\log^2 n)$ 的）。然后根据此 offset 创建 `VisualLineFragmentsIterator`，遍历过程基本与之前的过程一样，只不过会检查的内容从 Visual 相关变成 Logical 相关，这里也不再赘述。

### Visual Position 与 XY 的映射

从 Visual Position 到 XY 的计算过程中，首先根据 Visual Line 来获得当前行的 Y，方法为 `visualLineToY`。它的计算也很简单：View 的顶部间隙 + Visual Line * 行高 + Visual Line 之前的 Block Inlay 高度之和。Inlay 的高度的前缀和在 `InlayModel` 使用平衡树进行维护，每次的查询时间为 $O(\log n)$；当前行的初始 X 需要考虑文本的 Prefix Text 和 View 的左间隙。

然后获取 Visual Line 的开始 Offset，并创建 `VisualLineFragmentsIterator` 开始遍历 Fragment。此迭代器保证，Fragment 按照遍历的顺序，其开始的 Visual Column 一定是递增的。

迭代过程中，找到第一个满足 Visual Column 处于其中的 Fragment 并使用 Fragment  提供的坐标转换功能获得 Visual Column 对应的 X。需要注意的是，如果 Visual Column 在 Soft Wrap 符号后面或者是行末的 Inlay 中，Fragment 的 Column 是遍历不到的，这里需要特殊处理。

---

再讨论一下 XY 到 Visual Position： 依旧是先根据 Y 拿到 Visual Line。IJ 在这里的实现和 `visualToLogicalPosition` 一样丑陋，仍然是根据 Visual Line 进行二分，每一次查询都需要调用一次 `visualLineToY` 和 `getInlaysHeight`，合计 $O(\log^2 n)$。之后也是遍历 Visual Line 的各个 Fragment，找到第一个 X 在内部的 Fragment，并通过 Fragment 获取对应的 Column。Soft Wrap 引入的上一行结束和下一行开始的符号，需要根据 x 在其中的位置来确认与 Soft Wrap 的相对位置。行末的 Inlay 同理

----

可以看出，整个 `EditorCoordinateMapper` 的逻辑都与 `VisualLineFragmentsIterator` 密切相关，而且通过维护的 Soft Wrap 保证每次只需要遍历 Visual Line 内部的 Fragment（其个数是相对较少的）；但为了支持 Inlay 和 Soft Wrap 符号等功能也引入了大量的业务逻辑。



## `EditorSizeManager`

文本元素大小更改的管理和缓存相关的逻辑都在 `EditorSizeManager` 里。它作为 `PrioritizedDocumentListener` 的优先级是 `EDITOR_TEXT_WIDTH_CACHE` 。在上面的表中，我们可以看到它在 Soft Wrap 之后，在 Caret 之前。同时，它还是 `Folding` 和 `Inlay` 的 `Listener`。但是它并不直接为 Editor 的排版提供数据，它主要的作用是计算编辑器*偏好*的大小。

`EditorSizeManager` 主要维护的数据是每一个视觉行的宽度（单位为像素）：`myLineWidth: TIntArrayList` 。在接收到 `Document` 变更事件时，根据文本变更的范围对缓存的行宽进行失效，折叠区域和 Inlay 的处理类似。

不过对于批量的代码折叠操作，`EditorSizeManager` 会把相应的文本变更处理进行 defer，并在整体的折叠操作结束后通知 `EditorContentComponent` 重新验证内部 layout。

那么它维护的行宽究竟有什么用呢？主要用于计算 `preferedHeight` 和 `preferedWidth`，这些数据将会用于滚动条和整体布局计算的逻辑中。

`preferredHeight` 代表编辑器所有可滚动区域的高度，`preferredWidth` 则是编辑器整体最宽的部分，计算通过所有 Visual Line 的 X 以及 `BlockInlay` 的最大宽度。



## 总结

IDEA 的文本元素排版为了保证在增量文本更新下的性能，采用了惰性求值与缓存的策略，仅在重绘等场合才会对文本的排版进行计算。排版时，首先更新 UTF-16 的 Code Unit 到 Code Point 的映射（即调整 Logical Position 和 Offset 的关系），然后对文本进行 Tokenize 得到每个 Token 的位置；在 Inlay Inlay 和 Folding 处理完之后，根据一个个 Fragment 以计算 Soft Wrap 从而生成概念上的 Visual Line ，同时计算出 Visual Line 上每个元素的 X 坐标；最后根据 Block Inlay 来计算出每个 Visual Line 的纵坐标，从而完成最终的排版
