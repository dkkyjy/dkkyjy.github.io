# Research Assistant 更新：从 5001 篇收窄到 24 篇，并把深读链路修通

项目跑了三个多月后，我停下来查了一次真实数据，结论有点刺眼：

| 指标 | 数值 |
|---|---|
| 导入论文 | 5001 |
| 有 PDF | 12 |
| 真正深读过（解析 + 翻译） | 4 |
| 向量库 chunks | 28 |
| 知识图谱实体 | 0 |

也就是说：**每日进站 + 读中文分析**这条线在跑，知识库、图谱、沙箱基本是摆设。原因不是代码没写，而是闭环断了——只有手动点按钮的论文才会被切块入库，于是语料一直是 3 篇，QA 检索不到东西，自然没人用。

这一轮就做三件事：**改口径、补闭环、修链路**。

## 一、导入口径：只收"深度分析过"的论文

以前是全量导入每日报告里的所有论文（每天 20–80 篇）。现在只保留 **arxiv-daily-researcher 深度分析过、并且 PDF 已归档到 `reference_pdfs/`** 的论文：

```
manifest / markdown → 按 arxiv_id 匹配归档 PDF → 有 PDF 才建论文行 → 挂上 pdf_path
                     → 没有就计入 skipped_no_pdf，不入库
```

好处很直接：库里每一篇都是"值得深读、并且真的能深读"的，`pdf_path` 永远指向一个存在的文件，本地不再重复下载 PDF。

**顺带挖出一个藏了很久的解析 bug**：报告的「论文列表」小节在 2026-10-13 改名了（`所有论文列表` → `其他论文列表`），而解析器写死了旧名字，直接返回 0 篇；更要紧的是 `⭐ 及格论文详细分析` 这一段**从来没被解析过**——而那正是有 PDF、有深度分析的那批论文。修完之后，同一份报告从 0 篇变成 139 篇，24 个归档 PDF 全部对上号。

## 二、闭环：让摘要先能搜到

深读（解析 + 翻译）本来就该按需，不该批量跑。真正缺的是"读过的能想起来"——而库里其实一直躺着 24 条摘要和 33 条中文深度分析，只是没进向量库。

于是补上 `AbstractEmbedder` 的接线（它早就写好了，只是没人调用）：

- 导入结束后自动把 title + abstract 嵌入 Qdrant，`chunk_type='abstract'`
- QA 检索时摘要块按 **0.5 倍权重**混排，全文块优先
- 幂等：跑第二遍新增 0 条
- 顺带发现 markdown 导入路径从来没写过 `abstract`（写死空字符串），现在从报告的 `<summary>摘要 (Abstract)</summary>` 引用块里抽取并反转义 `\(CMU\)`、`21\~cm` 这类转义

## 三、深读链路：三个断点，三种错法

按「解析 → 翻译 → 演示稿」跑通的过程中踩了三个坑，都不在业务逻辑里，而在"别人家的工具"的边界上。

### 1. mineru 跑不起来，但退出码是 0

`mineru 2.5.4` 的公式识别模块需要 `transformers<5` 的 `find_pruneable_heads_and_indices`，而共享的 anaconda base 里是 **transformers 5.15**——直接 `ImportError`。但 `conda run` 依然返回 `rc=0`，只留下一句"没找到产物"。

base 里 vllm / llamafactory 等依赖 transformers 5.x，不能降级。解法是建一个只遮住 transformers 的 venv：

```bash
python -m venv --system-site-packages ~/.venvs/mineru
~/.venvs/mineru/bin/pip install "transformers<5"
```

公式解析保持开启，产出 markdown 里的 `$$` 块完整。

### 2. 翻译必定失败，而错误信息是空的

现象：一篇数学密集的论文，第 1 段就报「翻译错误」，`rc=1`，stderr 空白；而前一天另一篇同样流程却成功。

根因有两层：

1. `auto_translate.py` 把分段文件写在**当前工作目录**，而包装器没设 `cwd`，于是分段文件堆在仓库根目录。上一篇论文留下的 `translation_1_zh.md` 还在，这一篇复用时，`fabric -o` 发现目标已存在，**直接拒绝覆盖**。
2. fabric 的这句报错打在 **stdout**，而脚本只打印 stderr —— 所以我们看到的永远是空的。

修法：`cwd` 设为文档所在目录、加 `--force`（分段数变化时不必手动清档）、`--chunk-ratio 0.15`（本地模型慢，单段从 ~9.8k 降到 ~4.9k tokens）。整篇翻译还是慢（数学型论文 8 段、31 分钟），所以改成后台线程 + 页面 15 秒轮询，不再让请求挂在那儿等超时。

### 3. 演示稿：把 CLI 的输出当成 HTML 存盘

用 hermes 生成 bento + huashu-design 风格的讲解演示，先后踩了：

- 代理拒绝 `reasoning_effort` 取值，多轮请求还报 `duplicate field reasoning` → 调用时加 `--reasoning none`
- 把 `hermes chat` 的 stdout 直接写进 `presentation.html`，结果文件里混进 banner、警告和会话信息，页面只渲染出 1 页 → 加 `-Q` 静音，并优先从 stdout 里提取 `<!DOCTYPE html>…</html>`，取不到再取目录里最新的 html
- agent 会额外丢一份 `*_bento.html` 草稿 → 生成成功后清理多余 html

最终 prompt 也重写成带规则的版本：素材（`translation_zh.md` + `images/`）、内容（按章节讲方法/结论，不逐段翻译）、风格（bento 网格 + huashu-design 视觉）、插图（每张只用一次，固定 `<img src="images/<文件名>">`，才能被路由改写成站内图片 URL）、输出（只写 `presentation.html`、自包含无外链、支持键盘翻页）。

## 四、产物位置：不再写进别人的仓库

解析输出、译文、演示稿原来都落在 producer 的 `reference_pdfs/<stem>_mineru/` 里。现在统一进本项目：

```
data/pdfs/parsed/<stem>/
├── full_text.md          # mineru 提取的正文
├── images/               # 展平后的插图
├── translation_zh.md     # 中文全文
└── presentation.html     # 讲解演示
```

mineru 的工作树在提取完 markdown 和图片后立刻删掉。这里踩了个自己写的坑：展平时先建了 `images/`，随后"删掉所有子目录"把它也删了——所以删除循环必须跳过 `images/`。

## 实测

两篇论文端到端跑完（第二篇是数学推导型，最考验链路）：

| | 2603.20409 | 2603.29000 |
|---|---|---|
| mineru 解析 | 93s | 178s，85KB 正文 + 202 张公式图 |
| 翻译 | 9 分钟 | 31 分钟（8 段）→ 76KB |
| 演示稿 | 7 分钟，23 页 / 17 图 | 123s，16 页 / 4 图 / 外链 0 |

数据侧：papers 24、外部中文分析 33、日报 122、chunks 47（24 摘要 + 23 全文）、Qdrant 点 47；374 个单测通过，`ruff check` / `ruff format --check` 全绿。

## 还没做的

- **协作线（Docker 沙箱）**：本机没起 docker，镜像也没构建，端到端未验证
- 中文提问命中英文摘要的分数偏低（0.19–0.20），下一步是把 33 条中文深度分析也索引进向量库（`chunk_type='review'`）
- 本地 Qdrant 是单进程锁：Web 服务运行时 CLI 的检索会失败，要并发得上 Qdrant server

一句话总结这次的教训：**问题往往不在"没写"，而在"没接线"**——`AbstractEmbedder` 和摘要块的 0.5 倍权重都早就实现了，只是没人调用；而三处故障里有两处，是工具的失败信号被错误地吞掉（退出码 0、错误走 stdout），不看产物就发现不了。
