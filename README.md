# Pi Wiki — AI 知识库插件

> pi coding agent 的 wiki 知识库扩展。单一仓库 + 多数据源，Agent 可通过 `kb_search` 工具检索文档化知识。

## 安装

```bash
# 复制整个 wiki/ 目录到 pi 的 extensions 目录
cp -r wiki/ ~/.pi/agent/extensions/wiki/
cp wiki.ts ~/.pi/agent/extensions/

# 在 pi 中重载
/reload
```

## 快速开始

```
/wiki load /path/to/your-project       # 加载项目为数据源
/wiki add src/main.ts "主入口"          # 创建 wiki 条目
/wiki search 入口                       # 全文搜索
/wiki index                             # 索引导航
```

## 命令

| 命令 | 说明 |
|------|------|
| `load <dir>` | 加载数据源 |
| `unload <N\|path>` | 卸载数据源 |
| `sources` | 列出数据源 |
| `add <file> <title>` | 创建条目（自动解析源文件） |
| `delete <id>` | 条目 → 回收站 |
| `generate <id>` | 条目填充指引 |
| `recycle [--list\|--restore\|--clean]` | 回收站管理 |
| `index` | 索引导航树 |
| `search <kw>` | 内容搜索 |
| `ask <q>` | 增强搜索 + 全文 |
| `rules` | 写作规范 |
| `status` | 仓库统计 |
| `model [p/m]` | wiki 模型配置 |

## Agent 工具

```typescript
kb_search({ query: "关键词", mode: "content|title|both" })
// → 返回匹配的 wiki 条目 + 源文件引用 + 匹配片段
```

## 目录结构

```
extensions/wiki/
├── wiki.ts                  # 入口
├── lib/                     # 核心库
│   ├── types.ts             # 类型定义
│   ├── store.ts             # 数据层
│   ├── search.ts            # 搜索引擎
│   ├── recycle.ts           # 回收站
│   └── skeleton.ts          # 骨架 + 模板
├── commands/                # 命令处理
├── tools/                   # Agent 工具
├── resources/               # 模板
│   ├── rules-template.md    # 写作规范模板
│   └── entry-template.md    # 条目模板
└── repo/                    # 唯一 wiki 仓库
    ├── wiki.json            # { name, sources: [] }
    ├── index.json           # 索引
    ├── entries/             # wiki 条目
    └── .recycle/            # 回收站
```

## License

MIT
