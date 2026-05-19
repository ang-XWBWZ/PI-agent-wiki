# Pi Wiki — 零配置本地知识库检索系统

> **把任意 Markdown 目录变成可搜索的知识库。** 纯 Node.js 核心逻辑，零外部依赖。原生集成 pi coding agent，也可独立使用或适配到其他 AI Agent 平台。

---

## 这是什么？

**Pi Wiki** 是一个轻量级的本地知识库检索引擎。你指向一个目录（比如 Obsidian Vault、项目文档、笔记仓库），它自动递归扫描所有 `.md` 文件，提取标题和标签建立索引，然后提供毫秒级的全文搜索。

与传统 `grep` 的区别：

| | `grep` / `findstr` | Pi Wiki |
|------|:---:|------|
| 搜索方式 | 逐文件线性扫描 | **预建索引，O(1) 定位** |
| 中文搜索 | 编码敏感，易漏检 | **UTF-8 原生，编码无关** |
| 结果排序 | 无序（文件系统顺序） | **加权评分排序**（标题 > 路径 > 标签 > 内容） |
| 搜索结果 | 纯文本行 | **结构化命中**（标题 + 路径 + 标签 + 上下文片段） |
| AI 集成 | 需手动喂结果 | **`kb_search` 工具**，AI 原生调用 |
| 终端展示 | 原始输出 | **TUI 面板**，可折叠/展开全文 |

---

## 架构

```
┌─────────────────────────────────────────────────┐
│                  应用层                          │
│  ┌──────────────┐  ┌────────────────────────┐   │
│  │ 用户命令      │  │ AI 工具                │   │
│  │ /wiki-search  │  │ kb_search              │   │
│  │ /wiki-load    │  │ (LLM 主动调用)          │   │
│  │ /wiki-ask     │  │                        │   │
│  │ /wiki-close   │  │                        │   │
│  │ /wiki-unload  │  │                        │   │
│  │ /wiki-status  │  │                        │   │
│  └──────┬───────┘  └───────────┬────────────┘   │
│         │                      │                │
│  ┌──────┴──────────────────────┴───────────┐    │
│  │         pi 适配层 (pi-adapter/)          │    │
│  │   registerCommand · registerTool        │    │
│  │   TUI Widget · sendUserMessage          │    │
│  └──────────────────┬──────────────────────┘    │
└─────────────────────┼───────────────────────────┘
                      │
┌─────────────────────┼───────────────────────────┐
│              核心引擎 (lib/)                     │
│                      │                          │
│  ┌───────────────────┴───────────────────┐      │
│  │              search.ts                │      │
│  │  三级匹配 + 加权评分 + 上下文摘取       │      │
│  └───────────────┬───────────────────────┘      │
│                  │                              │
│  ┌───────────────┴───────────────┐              │
│  │          store.ts             │              │
│  │  索引持久化 (settings.json)    │              │
│  │  数据源管理 · 条目合并         │              │
│  └───────────────┬───────────────┘              │
│                  │                              │
│  ┌───────────────┴───────────────┐              │
│  │         indexer.ts            │              │
│  │  递归扫描 .md · 提取 frontmatter│              │
│  │  标题/标签解析 · 增量更新       │              │
│  └───────────────────────────────┘              │
│                                                 │
│  依赖：仅 Node.js 标准库 (fs / path)               │
│  零外部 npm 依赖                                 │
└─────────────────────────────────────────────────┘
```

**分层设计哲学**：核心引擎（`lib/`）与任何框架、平台无关——纯 Node.js，零外部依赖。适配层（`pi-adapter/`）负责将其接入 pi coding agent 的 Extension API。这意味着你可以用同样的核心引擎，为其他 AI Agent 平台（Claude Code、Cursor、Copilot Chat 等）编写适配层。

---

## 核心引擎 API

核心引擎暴露四个模块，全部零外部依赖：

### `types.ts` — 类型定义

```typescript
interface FileEntry {
  title: string;      // 从 frontmatter 或第一个 # 标题提取
  tags: string[];     // frontmatter tags 字段
  sourceDir: string;  // 所属数据源目录
  relPath: string;    // 相对路径
  mtime: string;      // 文件修改时间
}

interface SearchHit {
  relPath: string;
  sourceDir: string;
  title: string;
  tags: string[];
  snippet: string;    // 匹配上下文片段
  score: number;      // 加权评分
}
```

### `store.ts` — 数据层

```typescript
// 数据源管理
getSources(): string[]
addSource(absPath: string): boolean
removeSource(target: string): string | null

// 索引管理
getIndex(): Record<string, FileEntry>
mergeIndex(entries: FileEntry[]): void   // 增量合并（按 relPath 覆盖）

// 元信息
stats(): { sources, files, dirs, lastScan }
```

索引持久化到 `settings.json`，结构：

```json
{
  "sources": ["/path/to/obsidian-vault", "/path/to/docs"],
  "index": {
    "notes/cors-debug.md": {
      "title": "CORS 跨域调试笔记",
      "tags": ["cors", "debug", "network"],
      "sourceDir": "/path/to/obsidian-vault",
      "relPath": "notes/cors-debug.md",
      "mtime": "2025-03-15T10:30:00Z"
    }
  },
  "lastScan": "2025-03-15T10:30:00Z"
}
```

### `indexer.ts` — 文件扫描器

```typescript
scanDir(sourceDir: string): Promise<FileEntry[]>
```

递归扫描目录下所有 `.md` 文件，自动：
- 跳过隐藏目录（`.` 开头）和 `node_modules`
- 从 frontmatter 提取 `title` 和 `tags`
- 无 frontmatter 时从第一个 `# 标题` 提取标题
- 兜底使用文件名作为标题
- 路径统一为 `/` 分隔符（跨平台）

### `search.ts` — 搜索引擎

```typescript
search(query: string): SearchHit[]
```

三级加权评分算法：

| 匹配位置 | 权重 | 说明 |
|------|:---:|------|
| 标题命中 | **+10** | 最高优先级 |
| 路径命中 | +5 | 文件名匹配 |
| 标签命中 | +3 | frontmatter tags |
| 内容命中 | +1 | 首次出现位置 |
| 多次出现 | +1~9 | 每多出现一次 +1，上限 +9 |

每个命中附带 60 字符前置 + 80 字符后置的上下文片段，结果按评分降序排列。

---

## Pi 集成：命令 & 工具

通过 pi 适配层（`pi-adapter/`）暴露 6 个用户命令 + 1 个 AI 工具：

### 用户命令

| 命令 | 用途 | 触发 AI？ |
|------|------|:---:|
| `/wiki-load <目录>` | 加载数据源，自动递归扫描索引 | 否 |
| `/wiki-unload [序号\|路径]` | 卸载数据源；无参数列出已加载源 | 否 |
| `/wiki-search <关键词>` | 全文搜索，TUI 面板展示结果 | **否** |
| `/wiki-ask <问题>` | 搜索并返回匹配文件全文，触发 AI 总结 | **是** |
| `/wiki-close` | 关闭搜索结果面板 | 否 |
| `/wiki-status` | 查看索引状态（源数量/文件数/最后扫描） | 否 |

**关键设计**：`/wiki-search` 不触发 AI —— 搜索是程序逻辑（确定性、零 Token），AI 只负责基于结果回答问题（`/wiki-ask` 或 `kb_search`）。

### AI 工具

**`kb_search`** — LLM 主动调用搜索知识库：

```typescript
// AI 调用示例
kb_search({ query: "CORS 跨域解决方案" })
// → 返回评分排序的前 10 条结果（标题 + 路径 + 片段）

kb_search({ query: "CORS 跨域解决方案", fullContent: true })
// → 返回评分最高的 3 篇完整内容（每篇上限 3000 字符）
// → TUI 渲染中全文默认折叠，可展开查看
```

---

## 快速开始

### 独立使用核心引擎

```bash
# 零依赖，直接用
node -e "
const { scanDir } = require('./lib/indexer.js');
const { search } = require('./lib/search.js');
const { addSource, mergeIndex } = require('./lib/store.js');

// 加载 Obsidian Vault
const source = '/path/to/obsidian-vault';
addSource(source);
scanDir(source).then(entries => {
  mergeIndex(entries);
  console.log('索引完成:', entries.length, '篇');

  // 搜索
  const hits = search('CORS');
  hits.forEach(h => console.log(h.title, '-', h.score));
});
"
```

### 作为 pi 扩展安装

```bash
# 复制到 pi 扩展目录
cp wiki-github/lib/*.ts ~/.pi/agent/extensions/wiki/lib/
cp wiki-github/pi-adapter/*.ts ~/.pi/agent/extensions/wiki/

# 在 pi 中重载
/reload

# 开始使用
/wiki-load ~/obsidian-vault
/wiki-search 跨域问题
```

---

## 实战示例

### 场景 1：快速查阅笔记

```
你: "/wiki-load D:/obsidian-vault"
→ ✅ 已加载，正在后台索引... 📂 D:/obsidian-vault

你: "/wiki-search CORS"
→ 🔍 "CORS" — 12 结果（TUI 面板显示）
  1. CORS 跨域调试笔记           (23)
  2. Nginx 反向代理配置           (18)
  3. 前后端分离部署方案           (15)
  ...
```

### 场景 2：AI 辅助知识检索

```
你: "上次那个 CORS 问题是怎么解决的？"

→ AI 自动调用 kb_search({ query: "CORS 跨域" })
→ 命中 3 篇相关笔记
→ AI 基于笔记内容回答：
  "根据笔记'CORS 跨域调试笔记'，上次是通过 Nginx 添加
   Access-Control-Allow-Origin 头解决的，具体配置如下..."
```

### 场景 3：文档仓库即知识库

```
# 加载项目文档
/wiki-load ./docs

# 加载多个数据源
/wiki-load ~/obsidian-vault
/wiki-load ~/project-wiki
/wiki-load ./design-docs

# 查看状态
/wiki-status
→ 📊 Wiki 状态
     数据源: 3 个
     已索引文件: 1247 篇
     最后扫描: 2025-03-15T10:30:00Z

# 跨数据源搜索
/wiki-search 微服务架构
→ 同时命中 Obsidian 笔记 + 项目 Wiki + 设计文档
```

---

## 设计决策

### 为什么索引而不是每次 grep？

1. **速度**：预建索引后搜索是 O(n) 遍历内存中的索引表，而非 O(n) 遍历文件系统的每个文件。1000 个文件时差距约 100x。
2. **结构化**：索引提取了 frontmatter 元数据（标题、标签），支持比纯文本匹配更智能的评分。
3. **持久化**：索引保存到 `settings.json`，重启后无需重新扫描。文件修改时间用于判断是否需要增量更新。

### 为什么程序负责搜索，AI 只负责回答？

- 搜索是**确定性逻辑**：关键词匹配 + 加权排序，不需要 AI 推理。
- 每次 `/wiki-search` 触发 AI 会浪费 Token（尤其是高频使用场景）。
- `/wiki-ask` 和 `kb_search` 才需要 AI —— 这时的 Token 消耗是有价值的（理解问题意图、基于内容推理）。

### 为什么只支持 `.md` 文件？

- Markdown 是知识管理的事实标准（Obsidian、Notion 导出、GitHub Wiki、项目文档）。
- frontmatter 提供了结构化元数据（标题、标签），使搜索评分更精准。
- 如需支持其他格式（`.txt` `.rst` `.org`），可在 `indexer.ts` 中扩展文件扩展名匹配。

---

## 目录结构

```
wiki-github/
├── README.md                  # 本文件 — 完整说明文档
│
├── lib/                       # 核心引擎（零外部依赖，纯 Node.js）
│   ├── types.ts               # 类型定义
│   ├── store.ts               # 数据层（索引持久化 + 数据源管理）
│   ├── indexer.ts             # 文件扫描器（递归 .md 扫描 + frontmatter 解析）
│   └── search.ts              # 搜索引擎（三级加权匹配 + 上下文摘取）
│
└── pi-adapter/                # pi coding agent 适配层
    ├── entry.ts               # 扩展入口（注册所有命令和工具）
    ├── commands.ts            # 用户命令（/wiki-search /wiki-load 等）
    ├── repo-cmds.ts           # 数据源管理命令（/wiki-load /wiki-unload /wiki-status）
    └── tools.ts               # AI 工具（kb_search + TUI 渲染）
```

---

## 适配到其他 AI Agent 平台

核心引擎（`lib/`）完全平台无关。要接入其他 Agent，只需编写适配层：

### Claude Code 适配示例（概念）

```typescript
// claude-adapter.ts
import { addSource, mergeIndex } from './lib/store.js';
import { scanDir } from './lib/indexer.js';
import { search } from './lib/search.js';

// 注册为 Claude Code 的 MCP 工具
server.setRequestHandler("tools/list", () => ({
  tools: [{
    name: "kb_search",
    description: "搜索本地知识库。支持标题、标签、全文匹配。",
    inputSchema: {
      type: "object",
      properties: {
        query: { type: "string", description: "搜索关键词" },
        fullContent: { type: "boolean", description: "返回完整内容" }
      }
    }
  }]
}));

server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "kb_search") {
    const hits = search(request.params.arguments.query);
    return { content: [{ type: "text", text: formatHits(hits) }] };
  }
});
```

同样方式可适配 Cursor、Copilot Chat、Continue.dev 等平台。

---

## 路线图

- [x] v3.0 — 核心引擎重构，索引持久化，TUI 面板展示
- [x] v3.1 — 搜索结果面板优化，全文折叠/展开
- [ ] 增量索引（基于 mtime 判断，跳过未修改文件）
- [ ] 热重载（文件系统 watch，自动增量索引）
- [ ] 多格式支持（`.txt` `.rst` `.org`）
- [ ] 向量嵌入 + 语义搜索（可选增强）
- [ ] 独立 CLI 工具（`npx pi-wiki search "关键词"`）
- [ ] 更多平台适配（Claude Code MCP、VS Code 扩展）

---

## License

MIT
