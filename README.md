# xiezuo_fuzhu
Use skill to assist with academic writing
# Academic Writing Skills

两个配套的 Agent Skill，用于修改本科与研究生阶段的学术写作：去掉 AI 味、逐句给改写方案、压缩字数、审核引用格式。适用于 Claude Code、Claude Desktop（Cowork）等支持 Agent Skills 的环境。

保留作者的论点、证据、引用与语气，只去掉机器味。**不编造论点、不虚构文献、不用于欺骗性地规避 AI 检测。**

---

## 两个 skill 的分工

| | `xiezuo-fuzhu` | `gaixiev1` |
|---|---|---|
| 是什么 | 编辑判断的唯一来源 | 文档级流水线 |
| 输入 | 一段文字、一篇文章 | 一份 `.doc` / `.docx` / `.pdf` |
| 输出 | 改好的文字 | 改好的 `.docx` |
| 能单独用 | 是 | 否，依赖 `xiezuo-fuzhu` |

`xiezuo-fuzhu` 定义所有编辑纪律：AI tell 清单、引用红线、hedging 与术语保留、脆弱句式禁令、ESL 处理原则。`gaixiev1` 不复述任何编辑规则，只负责分页拆句、保护判定、出交互卡片、收选择、落缓冲、写回 Word——改写本身全部转交 `xiezuo-fuzhu`。

分开的理由：编辑判断会随经验不断修订，文档流水线基本不动。合在一起的话，每次调整一条 tell 规则都要重新验证整条 docx 写回链路。

---

## 安装

复制到你的 skills 目录：

```bash
# Claude Code
cp -r xiezuo-fuzhu gaixiev1 ~/.claude/skills/

# 或放进项目内
cp -r xiezuo-fuzhu gaixiev1 .claude/skills/
```

`xiezuo-fuzhu` 零依赖。`gaixiev1` 需要 LibreOffice（取真实页码）与 `python-docx`（读写句子）：

```bash
pip install python-docx
# macOS: brew install --cask libreoffice
# Debian/Ubuntu: apt install libreoffice
```

---

## 用法

### 整篇处理 — `xiezuo-fuzhu`

直接说要做什么即可，skill 会自己触发：

```
去一下这段的 AI 味
帮我压到 800 字
检查一下我的参考文献
用中文解释一下这篇文章在论证什么
投稿前帮我挑挑刺
```

五个阶段是分开的，不会一次全跑：**去 AI 味 → 压缩 → 引用审核 → 定向修改 → 校验**。顺序有讲究——先压缩会把 AI 措辞锁进一个更紧的壳里，之后更难拆。每完成一个阶段会问要不要继续下一个。

改动幅度先问后动，三档：**轻改**（只动词句）、**中度**（打散重复的段落模板）、**只诊断**（不改，只列问题和修法）。

### 逐句选方案 — `xiezuo-fuzhu` 的多版本模式

想自己挑而不是拿一份改好的稿子时：

```
逐句给我改写方案
每句给几个 paraphrase
```

每个可改句给出五行：

```
原句　Terror management theory holds that awareness of mortality generates anxiety.
A　　Awareness of mortality generates anxiety, according to terror management theory.
B　　Terror management theory holds that people become anxious when they recognise that they will die.
C　　Anxiety arises, on the terror-management account, from an awareness of mortality.
D　　［空白，自己写］
```

**A、B、C 是三个机器改写版，主手法两两不同**（换连接词、名词短语动词化、语序重排、主被动互换、拆句、并句、信息重心迁移、hedge 位置迁移，共八种）。**D 永远是空的**，留给你自己动手，排在最后，这样你会先把三个现成版本看完再决定要不要自己写。D 的内容原样收录，不做任何加工。

为什么是三个而不是两个：只给两个版本时，写作者有三到四成的句子会 A、B 都不满意而退回原句，那些句子等于白改了。第三个版本就是用来把「两个都不合意」的概率压下去的。

句中有语法错误时，**三个版本都要在修正语法的同时完成改写**——语法修正不是一个单独选项，也不因为要修语法就少给一个改写方案。信息缺失、补不出来的句子（比如宾语丢了）**不出 A/B/C**，只留原句和 D 位，挂 `待补` 标签。

### 整份 Word 文稿 — `gaixiev1`

```
/gaixiev1
改写这篇文稿
按页改写我的论文
```

流程：

1. **收稿分页** — 转 PDF 取真实页码（就是你在 Word 里看到的分页），拆句，判定哪些句子受保护，然后**先把每页可改句数的清单摆出来再问改哪几页**。不先给清单就问，只会换来「全部」或一个拍脑袋的范围。
2. **预扫描** — 用 `xiezuo-fuzhu` 的 review mode 诊断全稿，据此定改动幅度。结构问题为主的稿子会被劝退——逐句改写解决不了段落层面的问题。
3. **逐页改写** — 每页一张交互卡片，五个选项，页底一个「确认本页」按钮提交整页。
4. **通读缓冲稿** — 全部改完后按论证单元读一遍，专查逐句改写在句子接缝上制造的问题：指代断裂、连接词撞车、术语前后不一、句式扎堆、与受保护句的结构对称被打散。
5. **自动出终评** — 不等你问就给一份「改后 vs 原文」的逐句评估，三档判定：改进 / 打平 / 退步。退步的必须写明为什么原句更好，并建议退回。
6. **写回 Word** — 在 run 层替换，跨度之外的格式（斜体、上标、脚注引用）原样保留，可选只给改动句加黄色高亮。

**一律不改的十类**：参考文献、引用句、五个词以上的引文、插图与嵌入对象、表格、题注、表注、脚注尾注、域与超链接、封面信息。这些在卡片里灰显、不可点选、无 D 位。

---

## 设计上几个不那么显然的取舍

**保留普通连接词。** "Start off with X"、"On the other side"、"Another problem is" 不是 AI 特征，那是本科生真实的过渡方式。把它们全删掉会得到一份比作者本人语气更紧、更顺的稿子——这是另一种失真，而且往往比 AI 味更伤。

**改完的句子必须经得起再编辑。** 去过 AI 味的句子有一个不在常规清单里的失败模式：漂亮但脆。作者回头改一个词，紧凑的结构就塌成病句。所以规则是宁可留一个显式的 `that`、同位语压到一层、能用句号就不用冒号、并列限定动词优于分词堆叠。**作者维持不住的优雅不算改进。**

**引用一个字都不动。** 包括把括号式引用改成叙述式——那也是改动。

**缺信息就说缺。** 猜错的名词会变成作者没说过的主张。

**孤立改写是最大的失败源。** 所以调用改写时必须一并交出前后句原文、段内指代链、全稿术语表、已用过的连接词——缺一样就会在接缝上出问题。

---

## 状态

`xiezuo-fuzhu` 与 `gaixiev1` 共用一套 A/B/C/D 槽位约定，两边一致。改动其中一边的槽位含义时，另一边必须同步。

---

## 许可与边界

MIT。

仅做描述性编辑。文稿必须是使用者本人负责的作品——本人的论点、本人的文献、本人的论证。这两个 skill 不生成学术内容，也不是用来把 AI 生成的文字伪装成原创以规避院校规定的工具。稿子本身没有实质内容时，skill 会直说，而不是把填充物包装一遍。
