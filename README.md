# Pi Wiki v5.4 — 本地 Markdown 语义知识库

> 把任意 Markdown 目录变成可搜索的知识库。AST 精确解析 + bge 语义向量 + LLM 语义编译 + 文件追踪。

---

## 为什么不用 grep？

| | `grep` / `findstr` | Pi Wiki v5.4 |
|------|:---:|------|
| Markdown 解析 | ❌ 纯文本，code block 内 # 误判 | ✅ AST (unified + remark-parse) |
| 中文搜索 | 编码敏感，UTF-8 中文常漏检 | ✅ UTF-8 原生 |
| 结果排序 | 文件系统顺序 | ✅ 加权评分 + RRF(k=60) 混合融合 |
| 语义理解 | ❌ 无 | ✅ bge-base-zh-v1.5 ONNX，完全离线 |
| LLM 增强 | ❌ 无 | ✅ 子 Agent 全流程编译，提取 concepts/aliases |
| AI 集成 | 需手动复制粘贴 | ✅ **12 个 AI 工具**，LLM 主动调用 |
| 文件追踪 | ❌ 无 | ✅ manifest.json — MD5 + 增/删/改检测 |
| 终端展示 | 原始输出 | ✅ TUI 面板，收缩模式 MAX_LINE=100 |
| 知识生命周期 | ❌ 无 | ✅ 创建 → 索引 → 编译 → 刷新 → 检索 |

---

## 快速开始

### 安装

```bash
# 安装 wiki 依赖（npm 包）
cd ~/.pi/agent/extensions/wiki/scripts
init-wiki-deps.bat

# 下载语义模型 bge-m3（~570MB，ONNX INT8）
init-wiki-model.bat bge-m3
```

### 使用

在 pi 中执行：

```
/reload                       # 加载所有扩展
wiki_load_source 你的笔记目录  # 加载数据源
wiki_semantic(action="on")     # 启用语义搜索
wiki_refresh                   # 构建索引
wiki-search 关键词              # 搜索
```

AI 也能主动搜：

```
你: "上次那个 AMI 超时问题怎么排查的？"
→ AI 调用 kb_search → 基于笔记回答
```

### LLM 语义编译（提升搜索质量）

```
→ AI 派发子 Agent，严格约束仅 wiki 工具
→ 子 Agent 并行：读取 → 拆语义段 → wiki_store_file_compiled 存储
→ 全部完成后 wiki_refresh 刷新索引
→ kb_search 验证 → 编译文件召回排第 1
```

### ⚠️ 核心铁律

对 wiki 数据源的操作**必须通过 wiki 工具 API 完成**，禁止绕过：

| ❌ 禁止 | ✅ 正确做法 |
|---------|-------------|
| 使用 `bash` / `cmd` / `read` / `write` / `edit` | 使用 `wiki_get_entry` / `wiki_create_entry` |
| 终端删除文件 | `wiki_rename` 归档到 `_archived/` |
| 操作 `models/` / `vectors.json` 等运行时数据 | **绝对不允许** |

---

## 技术栈

```
L1 文档解析:  unified + remark-parse → AST (heading_path / wikilink / 代码块分离)
L2 语义编译:  DeepSeek Flash 子 Agent → topic / normalizedText / concepts / aliases
L3 Chunk:     AST 分块 + LLM 文件级 ###llm 向量（AST 永不被删）
L4 向量化:    **bge-m3 ONNX (1024维, 8192 tokens)** + enriched embedText
L5 检索:      BM25 关键词 + cosine 语义 + RRF(k=60) 混合
L6 追踪:      manifest.json — MD5 + 编译状态 + +新增/~变更/-删除
```

---

## 完整工具矩阵

### AI 工具（12 个）

| 类别 | 工具 | 能力 |
|------|------|------|
| 检索 | `kb_search` | keyword / semantic / hybrid 三模式，分页，TUI 收缩 |
| 数据源 | `wiki_load_source` | AST 解析 → MD5 追踪 → 自动建索引 |
| | `wiki_unload_source` | 卸载 / 列出已加载 |
| | `wiki_list_sources` | 列出 + 编译进度 (📝 N/290 文件) |
| | `wiki_refresh` | 增量更新 + 增/删/改检测 (+N/~N/-N) |
| 条目 | `wiki_create_entry` | 创建 .md + frontmatter 模板 + 即时索引 |
| | `wiki_get_entry` | 读取全文 |
| | `wiki_rename` | 重命名 → 自动同步索引 |
| | `wiki_move` | 移动 → 自动同步索引 |
| 语义 | `wiki_semantic` | 无参=状态；action="on"/"off"/"model" |
| 编译 | `wiki_compile_file` | 文件级 LLM 语义编译 prompt 生成 |
| | `wiki_store_file_compiled` | 存储编译结果 + 重建向量 + 更新 manifest |

### 用户命令

`/wiki-load` `/wiki-unload` `/wiki-search` `/wiki-ask` `/wiki-status` `/wiki-close`

---

## 架构 (v5.4)

```
wiki.ts                              ← pi 扩展入口
└── wiki/
    ├── commands/
    │   ├── query-cmds.ts
    │   └── repo-cmds.ts
    ├── lib/                         ← 核心库 (16 模块)
    │   ├── ast-chunker.ts           ← AST 分块 (unified + remark)
    │   ├── file-manifest.ts         ← 文件追踪 (MD5 + 编译状态)
    │   ├── indexer.ts               ← barrel
    │   ├── indexer-scan.ts          ← 文件扫描
    │   ├── indexer-embed.ts         ← 向量生成 + manifest 集成
    │   ├── indexer-compile.ts       ← 编译存储 + storeFileLLMVector
    │   ├── embedder.ts              ← bge ONNX (transformers.js)
    │   ├── model-registry.ts        ← 模型中间层
    │   ├── semantic-compiler.ts     ← LLM prompt 构建
    │   ├── semantic-search.ts       ← 语义搜索 + RRF (兼容 ###llm)
    │   ├── search.ts                ← 关键词搜索
    │   ├── preprocessor.ts          ← 程序化元数据提取
    │   ├── store.ts                 ← barrel
    │   ├── store-settings.ts        ← settings.json
    │   ├── store-vectors.ts         ← vectors.json
    │   ├── content-cache.ts         ← 内存缓存
    │   ├── parser.ts                ← frontmatter
    │   └── types.ts                 ← 类型定义
    ├── tools/
    │   ├── _helpers.ts
    │   ├── management.ts            ← barrel
    │   ├── management-sources.ts    ← 数据源 + 增量检测
    │   ├── management-entries.ts    ← 条目 CRUD
    │   ├── management-semantic.ts   ← 语义配置
    │   ├── management-compile.ts    ← 编译工具
    │   └── kb-search.ts             ← kb_search + TUI
    └── scripts/                     ← 一键安装 (deps + model)
```

**依赖**: 关键词引擎零外部依赖。AST 需要 `unified`/`remark-parse`/`unist-util-visit`。语义搜索需要 `@huggingface/transformers`（~570MB ONNX 模型，bge-m3，完全本地）。

**运行时数据**: `settings.json` + `vectors.json` + `manifest.json` + `compiled/`

---

## 🧠 配套技能

项目提供 **pi-wiki** 技能，指导 AI 正确操作 wiki 知识库。

**位置**: `skills/pi-wiki/SKILL.md`

**作用**: 教导 AI 全程遵守 wiki 生命周期，禁止通过终端或读写工具直接操作数据源。

**安装**:

```bash
cp -r skills/pi-wiki ~/.pi/agent/skills/
```

---

## License

MIT
