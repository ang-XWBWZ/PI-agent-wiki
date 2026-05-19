# Pi Wiki — 本地 Markdown 知识库检索

> 把任意 Markdown 目录变成可搜索的知识库。面向 AI 的全套生命周期管理，毫秒级全文搜索。

---

## 为什么不用 grep？

| | `grep` / `findstr` | Pi Wiki |
|------|:---:|------|
| 中文搜索 | 编码敏感，UTF-8 中文常漏检 | ✅ UTF-8 原生 |
| 结果排序 | 文件系统顺序，无优先级 | ✅ 加权评分（标题 > 路径 > 标签 > 内容） |
| 搜索结果 | 纯文本行，无结构化信息 | ✅ 标题 + 路径 + 标签 + 上下文片段 |
| AI 集成 | 需手动复制粘贴 | ✅ **9 个 AI 工具**，LLM 主动调用 |
| 终端展示 | 原始输出 | ✅ TUI 面板，Ctrl+O 折叠/展开全文 |
| 知识生命周期 | ❌ 无 | ✅ 创建 → 索引 → 检索 → 更新 → 淘汰，AI 全自动 |

---

## 快速开始

```bash
# 在 pi 中加载 Obsidian Vault
/wiki-load ~/obsidian-vault
→ ✅ 已加载，正在后台索引...

# 搜索
/wiki-search CORS 跨域
→ TUI 面板显示匹配结果，按评分排序

# AI 也能搜
你: "上次那个 CORS 问题怎么解决的？"
→ AI 自动调用 kb_search → 基于笔记内容回答
```

---

## 完整工具矩阵

### 用户命令（6 个）

| 命令 | 用途 | 触发 AI？ |
|------|------|:---:|
| `/wiki-load <目录>` | 加载数据源，递归扫描 .md 建索引 | 否 |
| `/wiki-unload [序号\|路径]` | 卸载数据源 / 列出已加载 | 否 |
| `/wiki-search <关键词>` | 全文搜索，TUI 面板展示 | **否** |
| `/wiki-ask <问题>` | 搜索并返回全文，触发 AI 总结 | **是** |
| `/wiki-close` | 关闭搜索结果面板 | 否 |
| `/wiki-status` | 索引状态（源数 / 文件数 / 最后扫描） | 否 |

### AI 工具（9 个）

| 类别 | 工具 | 能力 |
|------|------|------|
| 检索 | `kb_search` | 搜索知识库，支持全文模式 + Ctrl+O 折叠展开 |
| 数据源 | `wiki_load_source` | 加载目录，自动扫描建索引 |
| | `wiki_unload_source` | 卸载数据源 / 列出已加载 |
| | `wiki_list_sources` | 列出源 + 文件数 + 扫描时间 |
| | `wiki_refresh` | 重新扫描索引（单源/全部） |
| 条目 | `wiki_create_entry` | 创建 .md（frontmatter 模板 + 自动建目录 + 即时索引） |
| | `wiki_get_entry` | 读取全文，Ctrl+O 折叠展开（带标题/标签/时间元信息） |
| 文件系统 | `wiki_rename` | 重命名文件/目录 → 自动同步索引 |
| | `wiki_move` | 移动文件/目录 → 自动同步索引 |

---

## AI 知识生命周期闭环

```
对话中发现有价值知识
  │
  ├─ wiki_list_sources   → 确认数据源已加载
  ├─ wiki_create_entry   → 创建条目，自动索引入库
  │
下次遇到类似问题：
  ├─ kb_search           → 命中 → wiki_get_entry → 拿到完整方案
  │
知识演进：
  ├─ wiki_rename         → 重命名整理，索引自动追踪
  ├─ wiki_move           → 移动到合适目录，索引自动追踪
  │
索引维护：
  └─ wiki_refresh        → 外部变更后同步索引
```

**核心设计**：用户只管消费（`/wiki-search`），AI 通过 9 个工具接手全部管理操作。

---

## 搜索算法

三级加权匹配：

| 匹配位置 | 权重 | 
|------|:---:|
| 标题命中 | +10 |
| 路径命中 | +5 |
| 标签命中 | +3 |
| 内容首次命中 | +1 |
| 内容多次出现 | +1~9（每多一次 +1） |

附带上下文片段（命中位置前后 60~80 字符），结果按评分降序排列。

---

## 架构

```
wiki.ts                     ← pi 扩展入口
└── wiki/
    ├── commands/
    │   ├── query-cmds.ts   ← /wiki-search /wiki-ask /wiki-close
    │   └── repo-cmds.ts    ← /wiki-load /wiki-unload /wiki-status
    ├── lib/
    │   ├── types.ts        ← FileEntry / SearchHit 类型
    │   ├── store.ts        ← 索引持久化 + 数据源管理 + 条目增删
    │   ├── indexer.ts      ← 递归扫描 .md + frontmatter 解析
    │   └── search.ts       ← 三级加权搜索引擎
    └── tools/
        ├── kb-search.ts    ← kb_search AI 工具 + TUI 渲染
        └── management.ts   ← 8 个 AI 管理工具（数据源/条目/文件系统）
```

**依赖**：核心引擎（`lib/`）仅使用 Node.js 标准库（`fs` `path`），零外部 npm 依赖。适配层依赖 pi 的 Extension API。

**运行时数据**：索引持久化到 `wiki/settings.json`，已通过 `.gitignore` 排除。

---

## 适配其他 AI Agent

核心引擎（`lib/`）完全平台无关。要接入其他 Agent 平台，只需编写适配层：

```typescript
// 示例：Claude Code MCP 适配
import { search } from './lib/search.js';

server.setRequestHandler("tools/call", async (req) => {
  const hits = search(req.params.arguments.query);
  return { content: [{ type: "text", text: formatResults(hits) }] };
});
```

同样方式可接入 Cursor、Copilot Chat、Continue.dev 等。

---

## License

MIT
