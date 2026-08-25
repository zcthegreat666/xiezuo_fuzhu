---
name: gaixiev1
description: "改写 v1 — 收到 doc/docx/pdf 先转 PDF 按真实页码拆句、报出每页可改句数，当场问清改哪几页，再用 xiezuo-fuzhu 的 review mode 预扫描定改动幅度；引用句、引文、参考文献、标题、封面信息、以及图表/插图/表格/题注/表注/脚注尾注/域与超链接一律不改。然后显式调用 xiezuo-fuzhu 逐句改写，每句给 A、B、C 三个手法互不相同的版本外加一个空白 D 位供写作者自己动手（有语法错误时三版都先修语法再改写），改写时必须带上前后句、段内指代链与全稿术语表做上下文锚定。交互卡片挑选 原句/A/B/C/D，每页最后一句下面点一次「确认本页」即缓存整页选择；缓冲目录每个对话独立，绝不跨对话共用。全部页改完后通读缓冲稿查逻辑通畅性，并自动出一份不加粉饰的「改后 vs 原文」终评（逐句判定改进/打平/退步，退步的点名并建议退回），确认后才定稿写回 .docx。触发词：\"gaixiev1\"、\"改写这篇文稿\"、\"逐句给改写方案\"、\"每句给几个 paraphrase\"、\"按页改写我的论文\"、\"paraphrase my paper sentence by sentence\"、\"给我的 docx 逐句改写选项\"。不编造论点或文献，不改动任何引用。"
---

# gaixiev1 · 逐句改写方案生成器

把一份 `.doc` / `.docx`（或 `.pdf`）文稿按**真实页码**逐页拆成句子，**先把页码清单摆出来问改哪几页**，再为每个**可改**的句子给出 **A、B、C 三个手法互不相同的改写版本外加一个空白 D 位**（写作者自己动手写），用可交互卡片逐句挑选；每页点一次确认即整页落进缓冲文档；全部页改完后**把缓冲稿通读一遍查逻辑通畅性**，并**自动出一份不加粉饰的「改后 vs 原文」终评**，修补确认后才写回 Word 文件。

句子若有语法错误，**A、B、C 三版都要在修正语法的同时完成改写**——不把语法问题甩给写作者，也不因此少给一个改写方案。

## 本 skill 与 xiezuo-fuzhu 的分工

`xiezuo-fuzhu` 是编辑判断的唯一来源，在本流程里被**显式调用四次**：

- **阶段 0 · 预扫描**——用它的 review mode 诊断全稿，不改一个字，据此判定这份稿子该走哪条路。
- **阶段 3 · 逐句改写**——每一页开工前都要用 Skill 工具重新调用一次 `xiezuo-fuzhu`，由它来做改写；gaixiev1 只负责拆句、出卡片、收选择、落缓冲。A、B、C 三版是 `xiezuo-fuzhu` 的产物，不是 gaixiev1 自己凭印象写的。
- **阶段 4 · 通读校验**——用它的结构与语气 tell 清单读整篇缓冲稿，检查逐句改写有没有在句子之间制造新问题。
- **阶段 5 · 终评**——用它的 tell 清单与学术护栏当评判标准，逐句判定改后版本相对原文是改进、打平还是退步。

因此本文档**不复述任何编辑规则**。引用红线、hedging 与术语保留、脆弱句式禁令、AI tell 清单、ESL 处理原则，全部以 `xiezuo-fuzhu` 为准。这里只写它没有的东西：分页拆句与保护判定、页码清单式提问、全稿术语表、预扫描如何转化为幅度决策、学术语态硬锁、语法与改写的合并处理、上下文包与跨句预检、交互挑选卡片、缓冲与通读、自动终评、写回 docx。

中途**不产出清理稿**。原始 docx 始终是分页映射与写回的基准，页码因此与写作者在 Word 里看到的完全一致。

回复语言跟随用户；被改写的正文语言保持原样（英文稿改出来仍是英文）。

---

## 缓冲隔离：每个对话一个 run，绝不共用

**这条是硬性的。** 每一次新对话开工时用 `gx_buffer.py init` 新建一个带时间戳与随机尾缀的 run 目录，这个对话的全部缓冲只写在里面：

```bash
python3 gx_buffer.py init gx_sentences.json 工作目录
# -> 工作目录/gx_run_soc-2330-a2_20260808-052642_e8f42b
```

规则：

- **不复用别的 run 目录**，即使处理的是同一份文稿。上一次对话的选择属于上一次对话。
- **不从别的 run 目录读取**任何选择、缓冲稿或中间文件。
- 用户明确要求续跑上一次的进度时，让他**指名那个 run 目录路径**，确认后才用。不要自己去猜、去搜、去挑"最近的那个"。
- 测试或试跑产生的缓冲同样是独立 run。**不要把试跑数据当成用户的真实选择**——这是很容易犯的错。
- 脚本自己也会挡：`record` 与 `build` 拒绝操作没有 `run_id` 的目录，也拒绝 `source` 与当前文稿对不上的缓冲。

---

## 阶段 0 · 收稿：先分页，再问改哪几页

**不要在写作者看到页码清单之前就问「改哪几页」。** 他记不住第几页有什么，凭空问只会换来「全部」或一个拍脑袋的范围，而纯图表页、参考文献页本来就一句都改不了。顺序是固定的：**拿到文件 → 立刻跑阶段 1 的 `gx_split.py` → 把每页可改句数摆出来 → 再问。**

### 0a. 拿到文件就分页

先跳到阶段 1 把 `gx_split.py` 跑完，连同保护判定复核与术语表一起做完。这一步不需要任何用户输入，也不改一个字，所以不必先征求同意。

### 0b. 摆出清单，然后一次问完

用 `AskUserQuestion` 问，问题正文里必须带上刚跑出来的清单：

| 页 | 可改 | 受保护 |
|---|---|---|
| 2 | 12 | 8 引用 + 2 标题 |
| 3 | 0 | 1 标题 + 2 题注 |
| 4 | 3 | 7 表格 + 2 题注 + 1 表注 |
| … | | |

两个问题：

1. **改哪几页**——选项给「全部有可改句的页」「只改正文页（跳过图表与表格页）」「自己指定范围」。**可改句数为 0 的页要当场说明会跳过**，不要让写作者以为选了却没动静。
2. **导出时是否高亮改动句**——黄色高亮便于复查，默认标黄。

写作者在触发命令里已经给了范围（例如 `/gaixiev1 改第 2–8 页`）时**不要重复问范围**，但清单照样要报出来，让他有机会当场改主意；高亮那一项照问。

### 0c. 用 xiezuo-fuzhu 扫一遍

用 Skill 工具调用 `xiezuo-fuzhu`，按它的 **review mode** 对全稿做诊断：列出最伤这篇稿子的 5–10 个问题，每条附出处片段与一条可执行的修法，并判断主导类别。**这一步不改任何文字。**

判断主导类别时，把语法错误与断句损伤当作独立的一类来统计，不要归进"句法 tell"。两者的处理方式不同：AI 句法 tell 是风格问题，语法错误是正确性问题。

### 0d. 据诊断结果决定路线

| 主导类别 | 判定 | 动作 |
|---|---|---|
| **结构** 为主（段落形状雷同、大纲扩写感、每段都以抽象总结收尾） | 需要大幅度修改 | 逐句改写解决不了段落层面的问题。停下来告诉用户，建议先用 `xiezuo-fuzhu` 的中度重写处理结构，再回来跑 gaixiev1。用户坚持继续也可以，但要说清这一层不会被修好。 |
| **语法 / 断句损伤** 为主 | 走"先修后改" | 进入阶段 1。所有含错句在阶段 3 按合并处理出方案，并在收尾时附一份语法清单。 |
| **语气 / 句法 / 词** 为主 | 逐句改写正合适 | 进入阶段 1。tell 密度低走**轻改**，密度高走**中度**。 |
| 几乎无问题 | 不需要改写 | 明说这份稿子已经干净，问用户是否仍要生成改写选项。不要为了交付而制造改动。 |

把判定结果与选定的改写幅度，一句话报给用户后再往下走。

---

## 阶段 1 · 转 PDF 取真实页码，拆句，判定保护范围

**这一步在阶段 0a 就要跑完。** 它排在阶段 0 的问答之前，因为「改哪几页」这个问题必须带着页码清单一起问。

把附录 A 的 `gx_split.py` 原样写入工作目录，对**原始文稿**运行：

```bash
python3 gx_split.py "输入文件路径" "输出目录"
```

页码来自 LibreOffice 渲染出的 PDF——也就是写作者在 Word 里看到的分页；句子文本来自 python-docx（干净、带样式），再按段落起始位置映射到页。脚本**按文档正文顺序遍历 body**，`w:p` 与 `w:tbl` 交错处理，所以表格落在它真实的位置上，不会被漏掉或错位。输出 `gx_sentences.json`，每句带 `id`（`p2s5`）、`para`、`style`、`kind`、`text`。

拆完立刻 `gx_buffer.py init` 建本对话的 run 目录。

### 一律不改的十类

只展示、不生成方案，在卡片里灰显。**受保护的段落整段保留，不切句**——切开只会让写作者以为其中某一句可以动。

| 类别 | 判定 | 为什么不改 |
|---|---|---|
| `reference` | 参考文献标题之后的所有段落 | 格式即内容 |
| `figure` | 段落 XML 含 `<w:drawing>` / `<w:pict>` / `<w:object>`——插图、嵌入对象、图表 | run 层替换会切断图片锚点。纯图片段落没有文字，本来就不产出句子；图文混排的那一段整段不动 |
| `footnote` | 段落 XML 含 `<w:footnoteReference>` / `<w:endnoteReference>` | 脚注、尾注的正文在 `footnotes.xml` 里，本来就不在处理范围内；但**带注释标记的正文句同样不改**，因为替换跨度一旦盖住标记 run，注就掉了 |
| `field` | 段落 XML 含 `w:fldChar` / `<w:instrText>` / `<w:hyperlink>`——超链接、交叉引用、自动编号 | 同上，替换会毁掉域。注意：Word 把超链接存成 `fldChar + instrText`，**不能把它归进图片判定**，否则整个参考文献列表会被误判为插图 |
| `table` | 表格内每一个单元格段落 | 表格内容是数据，不是散文 |
| `caption` | Word 的 Caption 样式，或以 Table / Figure / Fig. / Chart / Graph / Exhibit / Appendix / 表 / 图 / 附录 + 编号开头 | 题注有编号与交叉引用依赖 |
| `note` | 以 `Note.` / `Notes:` / `N.B.` / `注：` / `表注：` 开头的段落 | APA 表注、图注属于表格装置 |
| `cited` | 句中含 `(Author, 2020)`、`Becker (1973)`、`[3]`、`p. 44`、`doi:`、`http` | 用户的硬性要求 |
| `quotation` | 引号内**至少五个词**才算真引文——单个引号术语（`"straightened"`、`"original affluent society"`）不保护整句 | 引文一字不动 |
| `frontmatter` | `Student Name:` / `Student Number:` / `Course:` / `Due Date:` 之类 | 封面信息不是散文 |

另外 `heading`：短且无终止标点的算标题；**套了 Heading 样式但字数多、有终止标点的按正文处理**——学生经常把 Heading 2 套在正文段上。

判定顺序有讲究：`reference` 最先，然后才是 `figure` / `footnote` / `field`。反过来会让挂着超链接的参考文献条目跑进图片类。

即使被判为 `editable`，遇到公式、代码、含未解析变量的模板句，也跳过并说明。

脚本仍会误判。生成方案前用眼睛扫一遍 `kind`：把漏网的带引用句手动降级为 `cited`，把被误伤的正常句升级为 `editable`。

先把每页的分类统计报给用户（第几页、共几句、可改几句、受保护几句分别是什么）——这份统计就是阶段 0b 拿去问「改哪几页」的清单。

### 建一份全稿术语表

拆句之后、改写之前，扫一遍**整份稿子（不只是要改的那几页）**，列出必须逐字统一的词：

- **专名与模型名**：`Gordon growth model`（不是 `Gordon Growth model`）、`S&P/TSX Composite`、`Bank of Canada`。
- **术语的固定词性形态**：`pass-through`（名词）对 `pass through`（动词）、`basis points` 对 `bp`、`percentage points` 对 `pp`。
- **拼写变体**：`recognise` / `recognize`、`analyse` / `analyze`——以稿中出现次数多的那个为准。
- **缩写在哪一句首次展开**：`GoC`、`OLS`、`XIC` 展开过一次，后面就不能再展开第二次。

这份表在阶段 3 每一页都要交给 `xiezuo-fuzhu`，A/B/C 三版一律照此拼写。**受保护句里出现的形态优先**——它们一个字改不了，只能让可改句去迁就。

为什么值得单列一步：全稿都写 `Gordon growth model`、只有一句写成 `Gordon Growth model` 这种不一致，逐句改写时根本看不见，非要等到阶段 4 通读才暴露；而如果那一句写作者恰好选了原句，连通读都可能放过去。术语表把这类问题挡在改写之前。

---

## 阶段 2 · 语态硬锁（gaixiev1 特有，xiezuo-fuzhu 未覆盖）

叠加在 `xiezuo-fuzhu` 的规则之上，作用于 **A、B、C 三版**。任一条不满足，该版本作废重写。（D 位是写作者自己写的，不受此约束——见阶段 3。）

- **语法正确。** 每个版本都必须是语法完整、可直接提交的句子。原句有错就在版本里修掉，不留给写作者。
- **全部第三人称。** 不出现 I / we / you / our / us / let's / one might say。原句若是第一人称，两版都改成第三人称，并把动作主体点名（`this study`、`the present analysis`、`the author`），不能改成无主语的漂浮句。
- **用词严谨。** 不用缩写形式（don't / it's），不用口语（a lot of、kind of、really），不用反问句、不用感叹号、不用破折号制造停顿。
- **时态与上下文一致。** 不单独判断一句的时态，先看该句所在段落的主时态：文献综述用过去时或现在完成时，方法与结果用过去时，理论陈述与普遍结论用现在时。改写不得改变原句时态，除非同段内部本身矛盾——此时向段落多数时态靠拢。
- **确信度不变。** `suggests` 不能变成 `shows`，`may indicate` 不能变成 `indicates`，`within this sample` 不能删。反向也不行：不要给原本肯定的判断加 hedge。
- **信息量不变。** 不增一个事实、不减一个限定词、不丢列举项。数字、单位、变量名、专有名词、学科术语原样保留。
- **长度浮动 ±25% 以内。** 改写不是压缩，也不是灌水。

---

## 阶段 3 · 逐页改写、挑选、落缓冲

**每一页开工前，先用 Skill 工具调用 `xiezuo-fuzhu`。** A、B、C 三版由它按自己的编辑纪律产出——不要凭对它规则的印象自己写。

一次处理一页。**每句固定四个来源：原句、A、B、C，外加一个空白 D 位。**

### C 位：备用改进版

两个版本不够用。实测下来写作者会有三分之一到四成的句子 A、B 都不满意而退回原句，那等于这句白改了，而写作者未必有精力每一句都自己动手写 D。`C` 是第三个机器版本，专门用来把「两个都不合意」的概率压下去。

- **C 的主手法必须与 A、B 都不同**（见下面八种手法）。三版同一手法只换词，等于只给了一个版本。
- **C 受同样的语态硬锁与自检约束**，不是放松档，也不是"更大胆的那一版"。
- 句子短到写不出第三种手法时（例如四五个词的过渡句）才允许省略 C，卡片上标 `无 C`。**不要拿 A 的近义改写充数**——凑出来的第三版只会让挑选变慢。
- **`待补` 句不出 C**，连 A、B 都不出，只留原句和 D 位。

### 交给 xiezuo-fuzhu 的上下文包

**孤立改写是这个流程最大的失败源。** 单句自检看不见接缝，等到阶段 4 才发现就已经改了几十句。调用时必须一并交出下面五样，缺一样就会在接缝上出问题：

1. **该页的可改句**，带 id。
2. **每个可改句的前一句与后一句原文**——包括受保护句，包括跨页与跨段的邻句。受保护句要标出类别，让 `xiezuo-fuzhu` 知道那句一个字都动不了、只能让本句去迁就。
3. **该句所在段落的主时态**，以及**段内的指代链**：`this` / `it` / `they` / `that` 各自指向哪个名词。
4. **阶段 1 的全稿术语表。**
5. **阶段 0d 定下的改写幅度**（轻改 / 中度），外加**前面几页已经用过的连接词与句式**，避免全稿扎堆。

### 八种手法

**A、B、C 三版的主手法必须两两不同。** 只换同义词不算一个版本。

1. **换连接词 / 换位置。** `however` ↔ `by contrast` ↔ `yet`；`therefore` ↔ `consequently` ↔ `as a result`；`because` ↔ `since` ↔ `given that`。也可以把连接词从句首移到主谓之间，或删掉连接词改用语序体现逻辑。
2. **复杂名词短语动词化。** `makes a contribution to` → `contributes to`；`the implementation of the policy` → `implementing the policy`；`an increase in X occurred` → `X increased`。只简化"伪装成术语的日常词"；学科术语一律保留，判定标准同 `xiezuo-fuzhu` 的 Keep technical terms。
3. **主谓宾语序重排。** 把状语提前作话题，把宾语提为话题，把从句前置改后置或反之。
4. **主被动互换。** 只在动作主体仍然可见、或该领域惯例允许背景化时使用（方法段的被动是规范，不要强行改主动）。禁止产生 `It was found that…` 这类把人藏起来的句式。
5. **拆句。** 一个长句拆成两句，逻辑关系用连接词显式接回。仅在中度幅度下使用。
6. **并句。** 两个短句合并，用 `which`、`whereas`、`although` 建立从属关系。仅当原文本就是同一层意思、且幅度为中度时使用。
7. **信息重心迁移。** 把原来在主句的次要信息降到从句，把从句里的核心判断提到主句。仅在中度幅度下使用。
8. **hedge 位置迁移。** `The results may suggest that X` ↔ `X, the results suggest, may…`——强度不变，只换落点。

### D 位：写作者自己写

- **D 永远是空的**，一个可输入的文本框，默认不选中。它的存在本身就是在说"A、B、C 都不满意时你可以自己来"。
- **D 的内容原样收录，不做任何加工。** 不套语态硬锁、不改措辞、不补标点。那是写作者本人的句子。
- 但**该提醒的要提醒**：如果 D 引入了语法错误、第一人称、或改动了句中的引用，在这一页的确认回执里用一句话点出来，由写作者决定改不改。发现即报、不擅自改——这是 `xiezuo-fuzhu` 对 ESL 写作的原则。
- D 有内容时，卡片自动把该句的选中项切到 D；清空文本框则退回原来的选择。
- 阶段 4 通读与阶段 5 终评时，D 句与 A/B/C 句一视同仁地参与检查。

### 含语法错误的句子怎么出方案

语法修正**不是**一个独立选项，它内建在 A、B、C 三版里：

- **三版都必须修掉同一个错误**，且各自施加一个不同的改写手法。只修语法、不做改写的版本不算一个 paraphrase 版本。
- **卡片上给该句挂一个 `语法` 标签**，只标类别、不写解释。写作者由此看得到自己错在哪些句子上，但卡片不变成批改作业。
- **修正需要写作者才知道的信息时不要猜。** 比如 `is a by-product of a particular ___`（名词丢了）、`Becker's explanation follows ___`（宾语丢了），如果上下文补不出来，该句**不出 A/B/C，只留原句和 D 位**，挂 `待补` 标签，请写作者自己补。
- **断句损伤优先按 `xiezuo-fuzhu` 第 5 节的复原方式修**：关系代词 `that` 写显、同位语压到一层、退化成连字符的冒号改用句号或分号、分词堆叠改成并列限定动词。这些句式当初就是因为太紧才塌的，修回去时不要再造一个同样紧的。

### 每个 A/B/C 版本的自检

**单句项**——出现任一项就重写：语法仍有错 / 第一人称 / 缩写 / 破折号 / 反问 / 时态漂移 / hedge 强度变化 / 数字被改 / 术语被"简化"或与术语表不符 / 长度超出 ±25% / 三版中有两版手法相同 / 违反 `xiezuo-fuzhu` 第 5 节 Edit durability 的脆弱句式禁令 / 超出阶段 0d 定下的改写幅度。

**跨句项**——出卡片之前必须过一遍，别留到阶段 4。这是把最容易在本页内暴露的四种接缝问题前移：

- **指代还接得上吗。** 这一版把名词降成了代词、或把从句提到句首之后，前后句的 `this` / `it` / `they` / `that` 还指得到东西吗。
- **与紧邻的受保护句是否还成对。** 上一句是 `On October 26, 2022, the Bank raised…; the yield fell…, and the index gained…` 这种句模，而本句原文是同一个模子的第二个例证，那么**任何打破这个对称的版本一律作废**。受保护句改不了，只能让可改句保持队形。这一条是从真实翻车里来的：改写把分号结构换成 `and…while…`，一组工整的对照例证就散了。
- **连接词是否与相邻句撞车。** 相邻两句不要同时以 `However` 开头，也不要连着三句用 `while`。
- **逻辑关系有没有被换掉本意。** `and` 换成 `while` / `whereas` 会凭空造出一个转折。两个并列事实若都在支持同一个结论，就不能换成对比连接词——这条比读起来顺不顺重要得多，因为它改的是意思，不是风格。

**手法配额。** 一页之内，同一种手法（例如"把状语提到句首"）在 A 版里最多用三次。超了就换手法重写，否则写作者一按"全选 A"就得到一页机械的排比。

### 交互卡片

调用 `mcp__visualize__read_me`（modules: `interactive`），再用 `mcp__visualize__show_widget` 渲染这一页的挑选卡片：

- 每句一个区块：顶部灰色显示原句，下面五个选项 `原句` / `A` / `B` / `C` / `D（自己写）`。默认选中 `原句`。`D` 排在最后，让写作者在动手写之前先把三个现成版本看完。
- `D` 是一个 `<textarea>`，占位提示写"自己动手改这一句"，聚焦或输入后自动选中 D。
- 不写"改了什么"的说明文字。唯一的例外是 `语法` 与 `待补` 标签——它们是状态，不是解释。
- 受保护的内容灰显并标出类别（`引用` / `引文` / `参考文献` / `图` / `表` / `题注` / `表注` / `脚注` / `域` / `标题` / `封面`），不可点选、无 D 位。
- 顶部工具条：`全选 A`、`全选 B`、`全选 C`、`全部保留原句`，以及"已改 n / 共 m 句"与"语法 k 句"计数。批量按钮**不覆盖已经写了内容的 D**，遇到 `无 C` 的句子时 `全选 C` 跳过它、不回落到 A。
- 底部实时预览：把当前选择拼成整页文字，让用户看连贯度。**预览必须把受保护句一起拼进去**，并在页首补上一页的最后一句、页尾补下一页的第一句（灰色显示，标明「上页末句」「下页首句」）。只拼可改句的预览看不出接缝，等于白给。
- **本页最后一句下面、卡片最底部放唯一一个 `确认本页` 按钮。** 一次点击提交整页的全部选择——不做逐句确认，不在句子区块里放确认按钮。按钮上写清作用，例如 `确认本页（提交全部 21 句选择）↗`。
- **选定范围里最后一页的按钮文字不同**：写成 `确认本页并出终评（提交全部 n 句选择）↗`，因为点完它会直接触发阶段 4 通读与阶段 5 的自动对比评估，写作者该知道下一步会发生什么。

payload 分两段，选择列表在前，D 的自定义文本在后，一句一行：

```
gaixiev1 第 1 页选择：p1s4=A; p1s5=D; p1s6=原句; p1s7=B; p1s8=C; …
D 文本：
p1s5 | <写作者自己写的整句>
```

选择列表必须**列全本页每一个可改句**，包括保持原句的，这样缓冲里一页就是完整的一页。D 的文本另起一段是为了避免句子里的分号把列表切碎。

### 收到 payload 才算数

`sendPrompt` 的消息**到达之后**才写缓冲。先把这一页的全部版本原文写成一份 `texts.json`——A、B 由 `xiezuo-fuzhu` 产出，D 用 payload 里带回的写作者原文：

```json
{"p1s4": {"A": "…", "B": "…", "C": "…"},
 "p1s5": {"A": "…", "B": "…", "C": "…", "D": "<写作者自己写的整句>"}}
```

然后记录并重建：

```bash
python3 gx_buffer.py record gx_sentences.json RUN目录 1 \
        p1s4=A p1s5=D p1s6=原句 p1s7=C --texts page1_texts.json
python3 gx_buffer.py build gx_sentences.json RUN目录 2,4,5,6,7,8
```

`build` 末尾那串页码是**写作者在阶段 0b 选定的范围**。带上它，"还有几句没选"就只统计这些页；不带的话，没打算改的页会一直挂着 pending，看起来像没干完。

缓冲里存的是完整句子文本，不是"A/B/C/D"这种指针，所以 `pick` 记的是来源、`text` 记的是内容。`gx_buffer.py` 对 pick 标签不做白名单，`C` 与 `F`（阶段 4 修补）都能直接用，只要 `texts.json` 里有对应文本。

**开每一页新卡片之前先 `build` 一次核对 `pages_confirmed`。** 上一页不在里面就说明 payload 没送到，停下来请用户再点一次确认或口头报出选择，不要往下走。用户说"这页选完了"但你没收到 payload 时，如实说缓冲是空的——不要拿试跑数据顶替。

缓冲文档的作用有四个：上下文不必背着几十页的选择；会话中断后凭它续跑；阶段 4 通读的对象就是它；`pages_confirmed` 是"到底存没存"的唯一凭据。

确认一句"第 N 页已记录，n 句改动（A x / B y / C z / D 位 k 句），缓冲已更新"，若 D 有问题一并点出，然后进入下一页。

---

## 阶段 4 · 通读缓冲稿，查逻辑通畅性

**所有页都确认完、`build` 报告没有 pending 之后**，把 `gx_buffer.md` 从头读一遍。这一步是必须的，不是可选的。

理由：阶段 3 的每一句都是**孤立**改写的，自检也只看单句。句子之间的接缝没有任何环节负责，而逐句改写恰恰最容易在接缝上出问题。

**按论证单元读，不按页读。** 分页是排版产物，段落和论证会跨页；只读单页看不出跨页的指代和逻辑断裂。

### 十一项检查

1. **指代断裂。** 前句被重排或把名词降成了代词，后句的 `this` / `it` / `they` / `that` 指不到东西了。这是逐句改写最常见的破坏。
2. **连接词打架或扎堆。** 相邻两句各自被改写后同时以 `However` 开头；或前句改成被动，后句的 `therefore` 逻辑接不上。
3. **术语与拼写前后不一。** `doxa` / `Doxa`、`humanly constructed` / `socially constructed`、`recognise` / `recognize`。独立改写会让同一个概念在不同句子里换了名字。
4. **跨段落时态漂移。** 单句自检只对齐所在段落的主时态，段落交界处仍会漂。
5. **句式扎堆。** 全选 A 或全选 B 之后，可能连续五句都用了同一个手法（例如都把状语提到句首）。改完比原稿更机械，是这个流程特有的失败模式。
6. **重复用词扎堆。** 独立改写容易让同一个替换词（都换成 `therefore`、都换成 `by contrast`）在相近位置反复出现。
7. **拆句改变了段落节奏。** 手法 5 用过的地方，检查段落是不是变得碎而长。
8. **与保护内容的接缝。** 改动句紧邻 `cited` / `quotation` / `figure` / `caption` / `table` 时最容易断——保护内容一个字没动，前后却换了语序或主语。特别检查"如图 1 所示"这类指向图表的句子，改写后指代是否还对得上。
9. **与保护内容的结构对称。** 受保护句与改动句原本是同一个句模的一组对照例证（同样的 `On DATE, …; …, and …` 结构、同样的三段式），改写把队形打散了没有。这一项要单独查，因为它读起来不别扭——每一句自己都对，只有并排看才发现对称没了。
10. **逻辑连接词有没有被换掉本意。** `and` → `while` / `whereas`、`so` → `although` 这类替换会凭空制造转折或让步。逐个核对被换掉的连接词：两侧的命题究竟是并列、因果，还是对比。
11. **段落首句是否还承接上一段末句。**

### 通读的产出

一份**衔接问题清单**：位置（句 id）+ 现象 + 一条**最小修补**。

- 修补只动被点名的那一句，不重开改写、不换版本、不动没被点名的句子。
- 通读期间**不引入新的改写手法**。发现某句本身可以更好，只记下来，不擅自改。
- **D 位的句子照查不误，但修补更保守。** 那是写作者自己的话，只提衔接问题（比如指代接不上），不评价文笔。
- 清单交给用户确认。用户同意的修补用 `gx_buffer.py record` 写回缓冲（`--texts` 里给修补后的文本，pick 记为 `F`），然后重新 `build`，再进阶段 5。
- 一条问题都没有时，明说"通读无衔接问题"，不要为了显得尽责而制造修补。

---

## 阶段 5 · 自动出「改后 vs 原文」终评

**最后一页确认、通读修补写回缓冲之后，不等写作者开口就出这份评估。** 他交稿前一定会想知道"到底改好了没有、是不是每句都比原来强"；等他问才给，说明这个流程少了一环。

先用 Skill 工具再调一次 `xiezuo-fuzhu`，拿它的 tell 清单与学术护栏当评判标准，然后逐条比对 `gx_apply_choices.json` 里的每一处改动与它的原句。

### 三档判定，一条不漏

每一条改动只能落在三档之一，**不许含糊其辞、不许写"整体更流畅"这种没有落点的话**：

- **改进**——修掉了真错，或语义更清楚、更合学科惯例。
- **打平**——换了个说法，好坏互见。打平也要单独列出来，不许并进改进里凑数。
- **退步**——原句更好。必须写明**为什么**原句更好，并给一条明确建议：退回原句，还是换用同一句的另一版。

判退步时优先看这几类，它们是逐句改写最常见的自伤：把主动改成被动、为"edit durability"把干净的同位语拆成从句而变啰嗦、吃掉承接用的 `also` / `however`、把平实说法换成更绕的惯用语（对 ESL 写作尤其有害）、把一句收尾有力的短句改平。

### 三项总评

分三项给结论，每项都要正面回答"哪个版本更好"：

1. **语法与标点**——修掉几处真错、引入几处新错。这一项通常是改后版本的净胜项，如实说，不必谦虚。
2. **语义通顺**——这一项最容易注水。老实报改进/打平/退步各几条；退步条数接近或超过改进时，就直说这一项没赢，别用"总体略有提升"糊过去。
3. **整体是否值得交**——一句话结论，附上净收益怎么算的（几处语法收益对几处语义损失）。

### 硬性纪律

- **不许只报好消息。** 一条退步都找不到时，先怀疑自己没认真找；确实没有，再说没有。
- **也不许硬凑退步**来显得客观。凑出来的假问题会让写作者把本来改对的句子退回去。
- **自己的失误要认。** 阶段 3 跨句项或阶段 4 十一项检查漏掉的接缝问题，在这里被发现时要指名说明是哪一项没做到，不要含糊成"可以再优化"。
- **受保护句里的错也要报**，并说清两个版本都带着、只能手改。改后版本周围越干净，这些错越刺眼。
- 评估里点名的退步，写作者同意后按阶段 4 的方式（`record` 记 `原句` 或另一版 + `build`）退回，然后**从原始 docx 重新跑一遍 `gx_apply.py`**——不要在旧副本上二次编辑，那会留下上一轮的残留。

---

## 阶段 6 · 定稿写回 Word

`build` 出来的 `gx_apply_choices.json` 就是写回用的扁平选择表。把附录 C 的 `gx_apply.py` 写入工作目录并运行：

```bash
python3 gx_apply.py gx_sentences.json RUN目录/gx_apply_choices.json "输出文件.docx" [--highlight]
```

替换在 run 层完成：改动跨度之外的格式（斜体、上标、脚注引用）原样保留，新文本继承句子起始 run 的格式。表格单元格等非正文段落的 `para` 是字符串键，脚本会直接跳过。脚本会报告替换句数以及任何未匹配的句子——未匹配必须逐条排查，不能默默放过。

导出后跑一次校验。**不要拿新文件的自动分类去比对手工修正过的 `gx_sentences.json`**——两份 `kind` 本来就不一样（阶段 1 手工降级、升级、重切过），比出来的"不一致"全是假警报。正确的校验方向是**拿旧文本去新文件里找**，全部用空白归一化后做子串匹配：

- 受保护的十类，逐字都在；
- 写作者选了原句的可改句，逐字都在；
- `gx_apply_choices.json` 里每一条改写，逐字都在；
- `figure` / `footnote` / `field` / `table` 与图片锚点（`<w:drawing>`、`<w:tbl>`）数量一个不少——对不上就说明锚点被切断了，必须回滚。

`--highlight` 必须**只标改动的那一句**。直接给命中的整个 run 上色会把大半篇文章刷黄，因为 Word 的一个段落常常就是一个 run。正确做法是把命中的 run 拆成三段——前缀、新句、后缀——只给中间那段加高亮。校验时对一下：**高亮词数应当等于所有改动句词数之和**，对不上就是刷宽了。

另有一个必踩的坑：`gx_split.py` 会把句子里的空白归一化，而 Word 文档里常有双空格、不间断空格。写回时用 `par.text.find(old)` 精确匹配会漏掉这类句子并报 `WARN unmatched`。匹配要退一步用 `\s+` 连接各词做正则搜索，不要因为一句匹配不上就默默放过。

最后用 `mcp__cowork__present_files` 交付——改好的 docx、语法清单（若有 `语法` 句）、以及阶段 5 的终评——并报告：改动句数 / 总句数、其中 A/B/C 各几句、D 位几句、语法修正几句、通读修补几句、终评退回几句、待补几句、原始与新的词数。可以顺带提一句 `xiezuo-fuzhu` 还有引用审核、中文复述、挑刺三个校验环节可用，但不要自动跑。

---

## 边界

- 只做描述性改写与语法修正。不新增论点、不新增文献、不补论证。文章质量的评价归 `xiezuo-fuzhu`，不在这里做。
- 不用于伪装 AI 生成内容以规避学术诚信规定。文稿必须是写作者本人负责的作品。
- 缺信息就说缺，不要靠猜补齐——这条对语法修正尤其要紧，猜错的名词会变成写作者没说过的主张。宁可把该句交给 D 位。

---

## 附录 A · gx_split.py

```python
#!/usr/bin/env python3
"""gaixiev1 stage 1 - doc/docx -> true page boundaries -> sentences -> edit flags."""
import json, os, re, subprocess, sys


def run_soffice(src, outdir, fmt):
    subprocess.run(["soffice", "--headless", "--norestore", "--convert-to", fmt,
                    "--outdir", outdir, src],
                   check=True, capture_output=True, timeout=240)
    out = os.path.join(outdir, os.path.splitext(os.path.basename(src))[0]
                       + "." + fmt.split(":")[0])
    if not os.path.exists(out):
        raise RuntimeError("LibreOffice produced no .%s for %s" % (fmt, src))
    return out


def page_texts(pdf):
    import pdfplumber
    with pdfplumber.open(pdf) as doc:
        return [(p.extract_text() or "") for p in doc.pages]


ABBR = ["et al", "e.g", "i.e", "cf", "vs", "etc", "Dr", "Mr", "Mrs", "Ms",
        "Prof", "Fig", "Eq", "St", "No", "pp", "p", "ed", "eds", "Vol", "vol",
        "ch", "trans", "Jr", "Sr", "Inc", "Univ", "approx", "ca", "esp", "n.d"]
_ABBR_LB = "".join(r"(?<!\b%s\.)" % re.escape(a) for a in ABBR)
_ABBR_LB += r"(?<![A-Z]\.)"
# lookbehinds MUST precede the consuming part, or they test the wrong position
SPLIT = re.compile(_ABBR_LB + r'(?<=[.!?])["”’\')\]]*\s+(?=["“\(\[]?[A-Z0-9])')


def sentences(par):
    par = re.sub(r"\s+", " ", par).strip()
    if not par:
        return []
    parts, buf = [], ""
    for chunk in SPLIT.split(par):
        buf = (buf + " " + chunk).strip() if buf else chunk
        if len(buf.split()) >= 3 or re.search(r"[.!?]$", buf):
            parts.append(buf); buf = ""
    if buf:
        if parts:
            parts[-1] = parts[-1] + " " + buf
        else:
            parts.append(buf)
    return parts


CIT = [
    re.compile(r"\([^()]*\b(?:19|20)\d{2}[a-z]?\b[^()]*\)"),
    re.compile(r"\b[A-Z][A-Za-z’'\-]+(?:\s+(?:and|&)\s+[A-Z][A-Za-z’'\-]+)?"
               r"(?:\s+et\s+al\.)?\s*\((?:19|20)\d{2}[a-z]?\)"),
    re.compile(r"\[\d+(?:\s*[,;–-]\s*\d+)*\]"),
    re.compile(r"\(\s*(?:ibid|op\.\s*cit|see also|as cited in)\b[^)]*\)", re.I),
    re.compile(r"\bpp?\.\s*\d+|doi:|https?://", re.I),
]
_Q = re.compile(r"[“\"]([^”\"]{12,})[”\"]|[‘']([^’']{20,})[’']")


def is_quotation(s):
    """A real quotation needs >=5 words inside the marks; a single scare-quoted
    term must not protect the whole sentence from rewriting."""
    for m in _Q.finditer(s):
        span = m.group(1) or m.group(2) or ""
        if len(span.split()) >= 5:
            return True
    return False


REFHEAD = re.compile(r"^\s*(references|reference list|bibliography|works cited|"
                     r"参考文献|注释)\s*:?\s*$", re.I)
NOTE = re.compile(r"^\s*(note|notes|n\.b\.|表注|注|附注)\s*[.:：]", re.I)
CAPTION = re.compile(r"^\s*(table|figure|fig\.|chart|graph|exhibit|appendix|"
                     r"表|图|附录)\s*[\dIVX]", re.I)
FRONT = re.compile(r"^\s*(student\s*(name|number|id)|name|course|section|"
                   r"instructor|professor|due\s*date|word\s*count|date)\s*[:：]", re.I)

# raw-XML markers: anything carrying one of these is left alone entirely,
# because run-level replacement would sever the anchor
XML_FIGURE = ("<w:drawing", "<w:pict", "<w:object")
XML_NOTE = ("<w:footnoteReference", "<w:endnoteReference")
# fields and hyperlinks: NOT figures. Word stores hyperlinks as fldChar +
# instrText, so folding them into XML_FIGURE misreads a whole reference list
# as images.
XML_FIELD = ("w:fldChar", "<w:hyperlink", "<w:instrText")


def xml_flags(par):
    """(has_image, has_note_marker, has_field) for a python-docx Paragraph."""
    x = par._p.xml
    return (any(t in x for t in XML_FIGURE),
            any(t in x for t in XML_NOTE),
            any(t in x for t in XML_FIELD))


def classify(s, style, in_ref, has_image=False, has_note=False, has_field=False):
    if in_ref:
        return "reference"
    # anchored content is never rewritten: run-level replacement would sever
    # the image, the note marker, or the field
    if has_image:
        return "figure"
    if has_note:
        return "footnote"
    if has_field:
        return "field"
    if FRONT.match(s):
        return "frontmatter"
    if NOTE.match(s):
        return "note"
    # a Heading style is only a real heading if it is short and unpunctuated;
    # students routinely apply Heading styles to body paragraphs
    if style and (style.lower().startswith("heading") or style in ("Title", "Subtitle")):
        if len(s.split()) < 20 and not re.search(r"[.!?]$", s.strip()):
            return "heading"
    if any(p.search(s) for p in CIT):
        return "cited"
    if is_quotation(s):
        return "quotation"
    if style in ("Caption",) or CAPTION.match(s):
        return "caption"
    if len(s.split()) < 5 and not re.search(r"[.!?]$", s):
        return "heading"
    return "editable"


def norm(t):
    return re.sub(r"\s+", "", t).lower()


def build_page_index(pages):
    flat, bounds, pos = "", [], 0
    for i, t in enumerate(pages, 1):
        n = norm(t)
        flat += n
        bounds.append((pos, pos + len(n), i))
        pos += len(n)
    return flat, bounds


def page_of(idx, bounds):
    for a, b, p in bounds:
        if a <= idx < b:
            return p
    return bounds[-1][2] if bounds else 1


PROTECTED = ("heading", "reference", "figure", "footnote", "field",
             "note", "caption", "table", "frontmatter")


def main():
    src = os.path.abspath(sys.argv[1])
    outdir = os.path.abspath(sys.argv[2]) if len(sys.argv) > 2 else os.path.dirname(src)
    os.makedirs(outdir, exist_ok=True)

    work = src
    if src.lower().endswith(".doc"):
        work = run_soffice(src, outdir, "docx")
    pdf = src if src.lower().endswith(".pdf") else run_soffice(work, outdir, "pdf")

    from docx import Document
    from docx.table import Table
    from docx.text.paragraph import Paragraph
    doc = Document(work)
    flat, bounds = build_page_index(page_texts(pdf))

    state = {"cursor": 0, "in_ref": False}
    rows = []

    def locate(txt):
        key = norm(txt)[:40]
        found = flat.find(key, state["cursor"]) if key else -1
        if found == -1:
            found = flat.find(norm(txt)[:15], state["cursor"])
        if found == -1:
            return page_of(state["cursor"], bounds)
        state["cursor"] = found + len(key)
        return page_of(found, bounds)

    def emit(par, para_key, force=None):
        txt = par.text.strip()
        if not txt:
            return
        pg = locate(txt)
        if REFHEAD.match(txt):
            state["in_ref"] = True
        style = par.style.name if par.style is not None else ""
        img, note, fld = xml_flags(par)
        kind = force or classify(txt, style, state["in_ref"], img, note, fld)
        # protected kinds stay whole - never sliced into sentences
        if kind in PROTECTED:
            rows.append({"page": pg, "para": para_key, "style": style,
                         "text": txt, "kind": kind})
            return
        for s in sentences(txt):
            rows.append({"page": pg, "para": para_key, "style": style, "text": s,
                         "kind": classify(s, style, state["in_ref"], img, note, fld)})

    # walk the body in document order so tables land where they belong
    pi = -1
    for child in doc.element.body.iterchildren():
        tag = child.tag.split("}")[-1]
        if tag == "p":
            pi += 1                       # matches doc.paragraphs indexing
            emit(Paragraph(child, doc), pi)
        elif tag == "tbl":
            tbl = Table(child, doc)
            for ri, row in enumerate(tbl.rows):
                for ci, cell in enumerate(row.cells):
                    for cpi, cpar in enumerate(cell.paragraphs):
                        emit(cpar, "tbl%d-r%dc%dp%d" % (pi, ri, ci, cpi),
                             force="table")

    pages = {}
    for r in rows:
        pages.setdefault(r["page"], []).append(r)
    out = {"source": src, "docx": work, "pdf": pdf, "pages": []}
    for pg in sorted(pages):
        items = []
        for i, r in enumerate(pages[pg], 1):
            items.append({"id": "p%ds%d" % (pg, i), "para": r["para"],
                          "style": r["style"], "kind": r["kind"], "text": r["text"]})
        out["pages"].append({"page": pg, "sentences": items})

    dest = os.path.join(outdir, "gx_sentences.json")
    with open(dest, "w", encoding="utf-8") as f:
        json.dump(out, f, ensure_ascii=False, indent=1)

    tot = len(rows); ed = sum(1 for r in rows if r["kind"] == "editable")
    print("%s\npages=%d sentences=%d editable=%d" % (dest, len(out["pages"]), tot, ed))
    for p in out["pages"]:
        c = {}
        for s in p["sentences"]:
            c[s["kind"]] = c.get(s["kind"], 0) + 1
        print("  p%d: %s" % (p["page"], c))


if __name__ == "__main__":
    main()
```

## 附录 B · gx_buffer.py

```python
#!/usr/bin/env python3
"""gaixiev1 stage 3/4 - the buffer.

  init   : python3 gx_buffer.py init   gx_sentences.json BASEDIR
  record : python3 gx_buffer.py record gx_sentences.json RUNDIR PAGE p1s4=A p1s5=C ... --texts texts.json
  build  : python3 gx_buffer.py build  gx_sentences.json RUNDIR [PAGES]

texts.json holds the full text of every version offered for that page:
  {"p1s4": {"A": "...", "B": "...", "C": "..."},
   "p1s5": {"A": "...", "B": "...", "C": "...", "D": "<the writer's own sentence>"}}
Any pick label works (A, B, C, D, F for a coherence fix) as long as texts.json
carries the matching text. "O" / the Chinese for "original" keeps the original.

PAGES on `build` is an optional comma-separated list (e.g. 2,4,5,6,7,8). Pending
sentences are then reported for those pages only, so pages the writer chose not
to touch do not show up as unfinished work.

Every conversation must call `init` once. It mints a fresh, timestamped run
directory holding that conversation's buffer and nothing else. `record` and
`build` refuse to touch a buffer whose source document does not match, and
refuse to touch a directory that was never initialised - so one conversation
can never write into, or read from, another conversation's buffer.

`build` writes RUNDIR/gx_buffer.md (paragraph-faithful full text),
RUNDIR/gx_buffer_pages.json (per-page text) and
RUNDIR/gx_apply_choices.json (flat {id: text} for gx_apply.py).
"""
import datetime, json, os, re, secrets, sys

BUF = "gx_buffer.json"


def load(path, default):
    if os.path.exists(path):
        with open(path, encoding="utf-8") as f:
            return json.load(f)
    return default


def save(path, obj):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(obj, f, ensure_ascii=False, indent=1)


def slug(path):
    s = re.sub(r"[^A-Za-z0-9]+", "-", os.path.splitext(os.path.basename(path))[0])
    return s.strip("-").lower()[:40] or "doc"


def cmd_init(meta_p, basedir):
    meta = load(meta_p, None)
    ts = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    run_id = "gx_run_%s_%s_%s" % (slug(meta["source"]), ts, secrets.token_hex(3))
    rundir = os.path.join(os.path.abspath(basedir), run_id)
    os.makedirs(rundir)
    save(os.path.join(rundir, BUF), {
        "run_id": run_id, "source": meta["source"], "created": ts,
        "pages_confirmed": [], "choices": {}})
    print(rundir)


def open_buf(rundir, meta):
    buf_p = os.path.join(os.path.abspath(rundir), BUF)
    if not os.path.exists(buf_p):
        raise SystemExit("no buffer in %s - run `init` first." % rundir)
    buf = load(buf_p, None)
    if not buf.get("run_id"):
        raise SystemExit("%s has no run_id - refusing." % buf_p)
    if buf.get("source") != meta["source"]:
        raise SystemExit("buffer belongs to a different document:\n  buffer: %s\n"
                         "  now:    %s" % (buf.get("source"), meta["source"]))
    return buf_p, buf


def cmd_record(meta_p, rundir, page, pairs, texts_p):
    meta = load(meta_p, None)
    buf_p, buf = open_buf(rundir, meta)
    texts = load(texts_p, {}) if texts_p else {}
    idx = {s["id"]: s for pg in meta["pages"] for s in pg["sentences"]}

    for pair in pairs:
        sid, _, pick = pair.partition("=")
        sid, pick = sid.strip(), pick.strip()
        if sid not in idx:
            print("WARN unknown id: %s" % sid); continue
        if idx[sid]["kind"] != "editable":
            print("WARN %s is %s - protected, refusing" % (sid, idx[sid]["kind"])); continue
        if pick in ("O", "原句", "orig"):
            buf["choices"][sid] = {"pick": "O", "text": idx[sid]["text"]}
            continue
        t = texts.get(sid, {}).get(pick) if isinstance(texts.get(sid), dict) else None
        if not t:
            print("WARN no text supplied for %s=%s" % (sid, pick)); continue
        buf["choices"][sid] = {"pick": pick, "text": t}

    page = int(page)
    if page not in buf["pages_confirmed"]:
        buf["pages_confirmed"].append(page); buf["pages_confirmed"].sort()
    save(buf_p, buf)
    n = sum(1 for v in buf["choices"].values() if v["pick"] != "O")
    c = sum(1 for v in buf["choices"].values() if v["pick"] == "D")
    print("buffer: pages %s | %d changed (%d written by hand) of %d recorded"
          % (buf["pages_confirmed"], n, c, len(buf["choices"])))


def cmd_build(meta_p, rundir, only_pages=None):
    meta = load(meta_p, None)
    _, buf = open_buf(rundir, meta)
    ch = buf["choices"]
    outdir = os.path.abspath(rundir)

    paras, order, page_of_para, pending = {}, [], {}, []
    for pg in meta["pages"]:
        for s in pg["sentences"]:
            pi = s["para"]
            if pi not in paras:
                paras[pi] = []; order.append(pi); page_of_para[pi] = pg["page"]
            paras[pi].append(ch.get(s["id"], {}).get("text", s["text"]))
            if s["kind"] == "editable" and s["id"] not in ch:
                if only_pages is None or pg["page"] in only_pages:
                    pending.append(s["id"])

    lines, per_page, last_page = [], {}, None
    for pi in order:
        pg = page_of_para[pi]
        if pg != last_page:
            lines.append("<!-- page %d -->" % pg); last_page = pg
        body = re.sub(r"\s+([,.;:)])", r"\1", " ".join(paras[pi]).strip())
        lines.append(body); lines.append("")
        per_page.setdefault(str(pg), []).append(body)

    md = "\n".join(lines).strip() + "\n"
    with open(os.path.join(outdir, "gx_buffer.md"), "w", encoding="utf-8") as f:
        f.write(md)
    save(os.path.join(outdir, "gx_buffer_pages.json"),
         {"pages_confirmed": buf["pages_confirmed"], "text": per_page})
    save(os.path.join(outdir, "gx_apply_choices.json"),
         {k: v["text"] for k, v in ch.items() if v["pick"] != "O"})

    print("run %s" % buf["run_id"])
    print("gx_buffer.md  %d words | %d changed (%d by hand) | pages confirmed %s"
          % (len(md.split()),
             sum(1 for v in ch.values() if v["pick"] != "O"),
             sum(1 for v in ch.values() if v["pick"] == "D"),
             buf["pages_confirmed"]))
    if pending:
        print("PENDING editable sentences not yet chosen: %d" % len(pending))
        print("  " + ", ".join(pending[:20]) + (" ..." if len(pending) > 20 else ""))


def main():
    mode = sys.argv[1]
    if mode == "init":
        cmd_init(sys.argv[2], sys.argv[3])
    elif mode == "record":
        texts_p, args = None, sys.argv[2:]
        if "--texts" in args:
            i = args.index("--texts"); texts_p = args[i + 1]; args = args[:i] + args[i + 2:]
        cmd_record(args[0], args[1], args[2], args[3:], texts_p)
    elif mode == "build":
        pages = None
        if len(sys.argv) > 4:
            pages = set(int(x) for x in sys.argv[4].split(","))
        cmd_build(sys.argv[2], sys.argv[3], pages)
    else:
        raise SystemExit(__doc__)


if __name__ == "__main__":
    main()
```

## 附录 C · gx_apply.py

```python
#!/usr/bin/env python3
import copy, json, re, sys
from docx import Document
from docx.enum.text import WD_COLOR_INDEX
from docx.text.run import Run


def replace_span(par, start, end, new, highlight):
    """Replace [start,end) of the paragraph text with `new`.

    When highlighting, the new text is split out into its own run so the
    yellow marks the rewritten sentence and nothing else; run-level
    highlighting would otherwise paint whole paragraphs."""
    pos, first, fa, fb = 0, None, 0, 0
    for run in list(par.runs):
        rlen = len(run.text)
        rs, re_ = pos, pos + rlen
        pos = re_
        if re_ <= start or rs >= end:
            continue
        a = max(start, rs) - rs
        b = min(end, re_) - rs
        if first is None:
            first, fa, fb = run, a, b
        else:
            run.text = run.text[:a] + run.text[b:]
    if first is None:
        return
    prefix, suffix = first.text[:fa], first.text[fb:]
    if not highlight:
        first.text = prefix + new + suffix
        return
    first.text = prefix
    r_el = first._r
    mid = copy.deepcopy(r_el)
    r_el.addnext(mid)
    mrun = Run(mid, first._parent)
    mrun.text = new
    mrun.font.highlight_color = WD_COLOR_INDEX.YELLOW
    if suffix:
        tail = copy.deepcopy(r_el)
        mid.addnext(tail)
        trun = Run(tail, first._parent)
        trun.text = suffix
        trun.font.highlight_color = None
    if not prefix:
        r_el.getparent().remove(r_el)


def main():
    meta = json.load(open(sys.argv[1], encoding="utf-8"))
    choices = json.load(open(sys.argv[2], encoding="utf-8"))
    out = sys.argv[3]
    highlight = "--highlight" in sys.argv

    doc = Document(meta["docx"])
    by_para = {}
    for pg in meta["pages"]:
        for s in pg["sentences"]:
            new = choices.get(s["id"])
            if not new or new.strip() == s["text"].strip():
                continue
            if s["kind"] != "editable":
                print("SKIP protected %s (%s)" % (s["id"], s["kind"])); continue
            by_para.setdefault(s["para"], []).append((s["text"], new.strip()))

    changed = 0
    for pi, edits in by_para.items():
        if not isinstance(pi, int):
            print("SKIP non-body paragraph %s" % pi); continue
        par = doc.paragraphs[pi]
        if not par.runs:
            continue
        spans = []
        for old, new in edits:
            idx = par.text.find(old)
            if idx != -1:
                spans.append((idx, idx + len(old), new)); continue
            # whitespace in the docx may not be normalised (double spaces,
            # non-breaking spaces); match tolerantly rather than give up
            pat = re.compile(r"\s+".join(re.escape(w) for w in old.split()))
            m = pat.search(par.text)
            if m is None:
                probe = old.rstrip(".")
                pat = re.compile(r"\s+".join(re.escape(w) for w in probe.split()))
                m = pat.search(par.text)
            if m is None:
                print("WARN unmatched: %r" % old[:60]); continue
            spans.append((m.start(), m.end(), new))
        for start, end, new in sorted(spans, reverse=True):
            replace_span(par, start, end, new, highlight)
            changed += 1

    doc.save(out)
    print("%s  (%d sentences replaced)" % (out, changed))


if __name__ == "__main__":
    main()
```

## 附录 D · 卡片上一句长什么样

**没有语法问题的句子**（幅度：轻改）

> 原句　Terror management theory holds that awareness of mortality generates anxiety.
> A　　Awareness of mortality generates anxiety, according to terror management theory.
> B　　Terror management theory holds that people become anxious when they recognise that they will die.
> C　　Anxiety arises, on the terror-management account, from an awareness of mortality.
> D　　［空白，自己写］

A 用主谓宾语序重排 + 话题前移，B 用名词短语动词化 + 从句改写，C 用主被动互换 + hedge 位置迁移——三种手法互不相同。

**有语法问题的句子**（标签 `语法`；主谓不一致 + `to based on` 不成立）

> 原句　Since there is no competing worldview to compare with the local one, people have little reason to separate culture from nature and doubts has nothing to based on.
> A　　Because no competing worldview is available for comparison, people have little reason to separate culture from nature. Doubt has nothing to rest on.
> B　　Doubt has nothing to rest on where no competing worldview exists, and people therefore have little reason to separate culture from nature.
> C　　With no competing worldview available for comparison, people have little reason to separate culture from nature, and doubt has nothing to rest on.
> D　　［空白，自己写］

三版都修掉了 `doubts has` 与 `to based on` 两处错，各自用了不同手法。

**信息不足、不能猜的句子**（标签 `待补`）

> 原句　Becker's explanation follows psychological.
> A　　［不给］
> B　　［不给］
> C　　［不给］
> D　　［空白，请写作者补上 follows 后面丢掉的名词］

**受保护的内容**（灰显，无选项、无 D 位）

> ［图］　Figure 1. Mortality salience and worldview defence across studies.
> ［表］　Condition | Effect size | Mortality salience | 0.31
> ［表注］Note. Effect sizes are Hedges' g, computed from the reported means.
> ［脚注］The construct has a long prehistory in existential philosophy.（该句挂着脚注标记）

