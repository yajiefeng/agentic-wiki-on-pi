
## 总体思路

不需要修改 pi 的源码。利用 pi 的三个扩展机制就能实现完整的 LLM Wiki：

1. AGENTS.md — 作为 schema 层，定义 wiki 的结构约定和工作流
2. Skill — 定义 ingest、query、lint 三个核心操作的详细步骤
3. Extension（可选，后期） — 注册自定义 tool 做 wiki 搜索，替代纯 index.md 方案

## 目录结构

~/wiki/                          # 你的知识库根目录（也是 pi 的 cwd）
├── AGENTS.md                    # schema — wiki 约定和工作流指令
├── .pi/
│   └── skills/
│       ├── ingest/SKILL.md      # /skill:ingest
│       ├── query/SKILL.md       # /skill:query
│       └── lint/SKILL.md        # /skill:lint
├── raw/                         # 原始素材（不可变）
│   ├── articles/
│   ├── papers/
│   ├── notes/
│   └── assets/                  # 图片等附件
├── wiki/                        # LLM 生成和维护的 wiki 页面
│   ├── index.md                 # 内容索引（LLM 维护）
│   ├── log.md                   # 操作日志（append-only）
│   ├── overview.md              # 全局综述
│   ├── entities/                # 实体页（人物、组织、产品等）
│   ├── concepts/                # 概念页
│   ├── sources/                 # 每个源的摘要页
│   ├── comparisons/             # 对比分析
│   └── synthesis/               # 综合分析、跨源洞察
└── .gitignore


## 核心文件

### 1. AGENTS.md（schema 层）

这是最关键的文件，告诉 pi 如何维护 wiki：

markdown
# LLM Wiki

你是一个知识库维护者。你的工作是维护 `wiki/` 目录下的结构化 markdown 知识库。

## 目录约定

- `raw/` — 原始素材，只读，绝不修改
- `wiki/` — 你维护的知识库，所有页面由你创建和更新
- `wiki/index.md` — 内容索引，每次 ingest 后更新
- `wiki/log.md` — 操作日志，每次操作后追加

## 页面格式

每个 wiki 页面必须包含 YAML frontmatter：

    ---
    title: 页面标题
    type: entity | concept | source | comparison | synthesis
    created: YYYY-MM-DD
    updated: YYYY-MM-DD
    sources: [source-slug-1, source-slug-2]
    tags: [tag1, tag2]
    ---

正文使用标准 markdown。用 `[[page-name]]` 风格的 wikilink 做交叉引用。

## index.md 格式

按类别组织，每条包含链接、一行摘要、来源数量：

    ## Entities
    - [实体名](entities/slug.md) — 一句话描述 (3 sources)

    ## Concepts
    - [概念名](concepts/slug.md) — 一句话描述 (2 sources)

    ...

## log.md 格式

    ## [YYYY-MM-DD] ingest | 源标题
    处理了 raw/articles/xxx.md，创建/更新了以下页面：
    - 新建: wiki/sources/xxx.md, wiki/entities/yyy.md
    - 更新: wiki/index.md, wiki/concepts/zzz.md

## 工作原则

- 新信息与已有页面矛盾时，在页面中明确标注矛盾，注明来源
- 交叉引用要充分 — 如果 A 提到 B，两个页面都应有链接
- 摘要要忠实于原文，不要添加原文没有的推断
- 每次操作后更新 index.md 和 log.md


### 2. Skills

.pi/skills/ingest/SKILL.md：

markdown
# Ingest

处理一个新的原始素材并整合到 wiki 中。

## 用法
用户会指定 raw/ 下的一个文件，例如：`/skill:ingest raw/articles/xxx.md`

## 步骤

1. 读取指定的原始文件
2. 读取 wiki/index.md 了解现有知识库结构
3. 与用户讨论关键要点（简短列出 3-5 个，问用户是否有需要重点关注的方向）
4. 在 wiki/sources/ 创建该源的摘要页
5. 识别涉及的实体和概念，创建新页面或更新已有页面
6. 检查是否与已有内容存在矛盾，如有则在相关页面标注
7. 更新 wiki/index.md
8. 追加 wiki/log.md
9. 向用户报告本次操作的变更摘要


.pi/skills/query/SKILL.md：

markdown
# Query

基于 wiki 回答用户问题。

## 步骤

1. 读取 wiki/index.md 定位相关页面
2. 读取相关的 wiki 页面（通常 3-10 个）
3. 综合信息回答问题，引用具体页面
4. 如果答案有价值，问用户是否要将其存入 wiki（作为 synthesis 或 comparison 页面）


.pi/skills/lint/SKILL.md：

markdown
# Lint

对 wiki 进行健康检查。

## 步骤

1. 读取 wiki/index.md 和最近的 wiki/log.md 条目
2. 扫描 wiki/ 下所有页面，检查：
   - 孤立页面（没有被其他页面引用）
   - 断裂链接（引用了不存在的页面）
   - 过时信息（被更新的源覆盖但未标注）
   - 缺失页面（被多次提及但没有独立页面的概念/实体）
   - frontmatter 不完整
3. 输出问题清单，按优先级排序
4. 问用户是否要自动修复（创建缺失页面、补充交叉引用等）


### 3. Extension（后期，wiki 规模变大后）

当 wiki 超过 ~100 页，纯靠 index.md 定位效率下降时，可以加一个搜索 extension。两个选择：

- **简单方案**：extension 注册一个 wiki_search tool，内部用 grep 在 wiki/ 目录做全文搜索
- **进阶方案**：集成 [qmd](https://github.com/tobi/qmd) 做混合搜索（BM25 + 向量），extension 调用 qmd CLI

typescript
// .pi/extensions/wiki-search.ts（简单版示意）
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";
import { Type } from "@sinclair/typebox";

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "wiki_search",
    description: "Search the wiki for pages matching a query",
    parameters: Type.Object({
      query: Type.String({ description: "Search terms" }),
      limit: Type.Optional(Type.Number({ description: "Max results", default: 10 })),
    }),
    async execute({ args }) {
      const { query, limit = 10 } = args;
      const result = await pi.exec("grep", ["-ril", "--include=*.md", query, "wiki/"], {
        cwd: process.cwd(),
      });
      const files = result.stdout.trim().split("\n").filter(Boolean).slice(0, limit);
      return { result: files.join("\n") || "No matches found" };
    },
  });
}


## 工作流

### 日常使用

bash
cd ~/wiki
pi


pi 启动后自动加载 AGENTS.md 和 skills。然后：

- 添加新素材：把文章存到 raw/，然后 /skill:ingest raw/articles/xxx.md
- 提问：/skill:query 这几篇论文对 X 的观点有什么分歧？
- 或者直接问（不用 skill），pi 会根据 AGENTS.md 的指令自行读取 wiki
- 定期维护：/skill:lint

### 批量导入

对于初始的大量素材，可以写一个简单的 prompt template：

markdown
<!-- .pi/prompts/batch-ingest.md -->
依次处理以下文件，每个文件按 ingest skill 的流程执行，
但跳过与用户的讨论步骤，直接处理：
{{files}}


然后：/batch-ingest 并列出文件路径。

### 与 Obsidian 配合

~/wiki 目录直接用 Obsidian 打开。wiki 页面中的 [[wikilink]] 在 Obsidian 中可以直接点击跳转。graph view 可以看到知识网络。pi 在终端里编辑，Obsidian 实时显示变更。

## 启动步骤

1. 创建目录结构
2. 写入 AGENTS.md、三个 skill 文件
3. 初始化 git
4. 把第一批素材放入 raw/
5. cd ~/wiki && pi，开始 ingest

要我现在帮你把这些文件创建出来吗？你可以告诉我：
- wiki 的根目录放在哪里（比如 ~/wiki 或其他路径）
- 你的知识库主题是什么（这样我可以调整 AGENTS.md 中的分类）
- 你偏好中文还是英文的 wiki 页面