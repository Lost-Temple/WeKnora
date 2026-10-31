# Source Locator 生成、重映射与补强阅读指南

这一篇接着 [38-code-reading-chat-reference-source-location.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/38-code-reading-chat-reference-source-location.md) 往上游读，专门解决一个问题：

- 前端能拿到的 `source_locators`，最早是怎么生成出来的？
- 为什么文档经过清洗、图片改写、表格归一化之后，locator 还能继续对得上？
- 扫描 PDF 为什么经常只有“页级定位”，后来又是怎么补强成可匹配 quote 的？

这条链的核心结论是：

- **locator 不是检索阶段临时猜出来的，而是在入库切 chunk 前就挂到 chunk 上了**
- **parser 原始 blocks 不会直接拿来用，而是要先 remap 到最终 markdown**
- **扫描 PDF 这类弱定位结果，会在检索结果回包前再补一轮页级 quote**

## 推荐阅读顺序

1. [internal/application/service/knowledge_source_locators.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/knowledge_source_locators.go)
2. [internal/sourceloc/office.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/office.go)
3. [internal/sourceloc/align.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/align.go)
4. [internal/sourceloc/remap.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/remap.go)
5. [internal/sourceloc/assign.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/assign.go)
6. [internal/application/service/knowledge_process.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/knowledge_process.go)
7. [internal/searchutil/imageinfo.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/searchutil/imageinfo.go)

## 1. locator 生成的真正入口不在检索，而在入库

最早的入口是 [attachStructureBlocks](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/knowledge_source_locators.go)。

它处理的是一种很关键的场景：

- parser 成功把文档读成了 markdown
- 但没有直接返回 `SourceBlocks`

这时系统不会放弃 locator，而是补两条兜底路径：

### 第一条：Office 文件按原始结构重建

支持这些格式：

- `docx/docm`
- `pptx/pptm`
- `xlsx/xlsm`
- `csv`
- `epub`

也就是：

- 不依赖 parser 先给出 block
- 而是重新从原文件结构里拆出 paragraph / slide / row / section 单元

### 第二条：纯文本原样透传时做自映射

如果是：

- `md`
- `txt`

并且 parser 输出的 markdown 与原文完全一致，就直接把文本按自身段落映回自己。

这一层最重要的原则是：

- **locator 是增强能力，不是解析成功的前置条件**

所以这里失败只会损失原文定位，不会让整次入库失败。

## 2. `office.go` 解决的是“原文件有哪些天然结构单元”

[office.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/office.go) 的职责，不是直接生成最终 chunk locator，而是先把原文件拆成一批 `Unit`：

- 每个 `Unit` 有文本
- 也有它应该回到哪里的 `Locator`

可以把它理解成：

- **原文件自己的结构真相**

### DOCX

按：

- body-level paragraph
- table row

来拆。

这里特别重要的一点是：

- 表格不是“一整张表一个 unit”
- 而是“一行一个 unit”

否则问答引用最终只能回到“这张表”，很难回到具体行。

### PPTX

按：

- 每页 slide 的每个文本段落
- notes 里的段落

来拆，但 locator 仍然只回到 `slide` 级。

这是一种刻意的折中：

- 不粗到整页一个大块
- 也不把定位强耦合到 parser 的 shape 顺序

### XLSX / CSV

按：

- sheet + row

来拆。

这也是最符合用户心智的一层：

- 引用最终跳回“第几张表第几行”

### EPUB

没有稳定页码，所以按：

- spine section
- 首个标题

来定位。

## 3. `align.go` 为什么这么保守

[Align](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/align.go) 做的事可以概括成一句：

- 用原文件结构单元去对齐 parser 产出的 markdown

但它的策略非常保守：

- 只给“完整命中的单元”打块
- 中间不认识的内容保持 unknown
- 不允许把未知内容顺手归给前一个 block

这是因为 locator 一旦错配，危害比“没有 locator”更大：

- 用户点击 citation
- 被带到另一页或另一段

这会直接破坏对引用可信度的判断。

### 重复文本为什么经常故意不对齐

文档里常见的重复文本包括：

- 目录
- 页眉页脚
- 相同表头

`Align` 只有在“每一处重复都能被 source unit 解释清楚”时，才允许落位。

否则宁可跳过。

## 4. `remap.go` 解决的是“parser blocks 对着旧 markdown，chunking 却看最终 markdown”

这一步很容易被忽略。

在入库过程中，parser 的 markdown 往往还会继续被改写，比如：

- 换行统一
- HTML table 归一化
- 图片引用重写
- 远程图片转存

所以 parser 给出的 `SourceBlocks`，其实记录的是：

- **旧版 markdown 的区间**

而真正 chunking 用的是：

- **最终版 markdown**

[RemapBlocks](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/remap.go) 就是在解决这个错位。

它没有简单地做 start/end 比例换算，而是：

1. 先拿出旧 block 覆盖的完整正文
2. 再用 `Align(newText, units)` 重新对齐到新 markdown

这么做的原因是：

- 比例 remap 可以猜出一个新位置
- 但它无法证明“这段正文在新文本里还是真正同一段”

对 citation 来说，猜错位置比没有位置更危险。

## 5. `assign.go` 才是真正把 chunk 区间变成 locator 列表的地方

[Index](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/sourceloc/assign.go) 是“最终 markdown 区间 -> 原文 locator 列表”的翻译器。

它有两个最关键的方法：

### `Locators(start, end)`

给普通 text chunk / parent chunk 用。

它会：

- 找出这个 chunk 区间覆盖了哪些 source block
- 给每个 locator 裁出对应 quote
- 合并相邻、语义相同的 locator

这一步出来后，chunk 就已经带着：

- `page`
- `block`
- `row`
- `quote`
- `partial`

这些后续问答引用会直接消费的数据了。

### `LocatorsAt(pos)`

给图片锚点用。

图片不是正文区间，只是在 markdown 里有一个出现位置，所以这里只回答：

- 这个图片 URL 所在的位置，属于哪些原文 block

这就是为什么后面图片子块也能带 `SourceLocators`。

## 6. `partial` 是怎么来的

`Locators(start, end)` 在遍历 block 时，会检查：

- chunk 区间里有没有一段正文没有落在任何 source block 上

只要有，就把参与的 locator 全部标成 `partial=true`。

这表示的不是：

- locator 失效

而是：

- 这条引用的结构坐标仍然有用
- 但不能承诺这整个 chunk 都精确映回了原文

所以前端看到 `partial` 时，往往会降级成：

- 命中某一页/某一块
- 但不宣称“精确到原句”

## 7. `knowledge_process.go` 里 locator 是怎么真正挂到 chunk 上的

在 [knowledge_process.go](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/application/service/knowledge_process.go) 里，这条链是：

1. 保存 parser 原始 markdown 和原始 `SourceBlocks`
2. 对 markdown 做规范化和图片 URL 改写
3. `buildSourceIndex(parsedMarkdown, sourceBlocks, finalMarkdown, storedImages)`
4. chunker 切 chunk
5. 用 `sourceIndex.Locators(c.Start, c.End)` 给每个 chunk 填 `SourceLocators`

这说明一个关键事实：

- **chunk 的 locator 是在 chunking 当下按最终 markdown 区间计算出来的**

不是后面检索命中后再临时反推。

所以检索结果能不能回原文，实际上取决于入库时这条链有没有建立好。

## 8. 为什么扫描 PDF 还需要 `imageinfo.go` 再补一轮

扫描 PDF 是一个特殊情况。

问题在于：

- 文字正文常常不存在真正的文本块
- chunk 里可能只剩“某一页有一张页面图片”
- locator 虽然知道是第几页，但 `quote` 很弱，甚至只有 `![page_xxx]`

这时前端虽然能跳到页：

- 但很难继续在页内确认是不是正确位置

[collectChildEvidence](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/searchutil/imageinfo.go) 和 [withPageQuotes](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/internal/searchutil/imageinfo.go) 就是在补这个洞。

它会：

1. 回读 `image_ocr / image_caption` 子块
2. 提取每页识别到的文字
3. 只给“整页且原本没有可用 quote 的 PDF locator”补上 page quote

这里也很保守：

- 已经带 bbox / 正文 quote 的强 locator 不会被覆盖
- 只补弱页级 locator

所以它不是重写定位，而是：

- **给弱定位补文本证据**

## 9. 这条链最容易看错的三个点

### 第一，`SourceBlocks` 不是最终给前端的 locator

`SourceBlocks` 只是中间态：

- 记录某一版 markdown 的区间对应哪个原文件位置

真正给 chunk / SearchResult / 前端的是：

- `SourceLocators`

### 第二，remap 不是“锦上添花”，而是必要步骤

只要 markdown 在 chunking 前被改过，旧 blocks 就不能直接用了。

如果跳过 remap：

- chunk 的 `Start/End` 是新版区间
- locator 却还指着旧版区间

后面引用落位就会越来越飘。

### 第三，扫描 PDF 的 quote 补强发生在检索结果层，不是入库层

入库时它常常只有页级 locator。

真正把 OCR 文字补回去，是在搜索结果 enrich 阶段完成的。

所以你在数据库里看到的 locator，不一定和问答返回给前端的 locator 强度完全一样。

## 10. 推荐连读方式

如果你要完整读懂“原文定位是怎么从入库一路传到问答引用”的全过程，建议这样连读：

1. 先看 [39-code-reading-source-locator-generation-remap.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/39-code-reading-source-locator-generation-remap.md) 这一篇，确认 locator 的生成与补强
2. 再看 [38-code-reading-chat-reference-source-location.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/38-code-reading-chat-reference-source-location.md)，确认前端如何消费 locator
3. 最后回到 [11-code-reading-session-rag-references.md](file:///Users/mao/work/opensource/github.com/Tencent/WeKnora/website-docs/06-development/11-code-reading-session-rag-references.md)，把“问答引用如何进入会话 UI”串起来

这样就能把“生成 locator -> 挂到 chunk -> enrich 结果 -> 前端打开原文”完整连上。
