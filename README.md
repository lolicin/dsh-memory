# dsh-memory

为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 提供跨会话持久记忆的插件。

本仓库是 `dsh-memory` 的 fork：放宽了 peer 版本范围以兼容 DSH 0.2.0-rc.2，并修复了中文搜索。Harness 官方没有自带记忆插件——它的 `extension-cookbook` 只给出了机制（提示词区段 + 工具），却没有实现，导致每个会话都从零开始。本包用一个本地 SQLite 文件填补了这个空缺：**无需嵌入模型服务，无需 API key，无需额外进程。**

## 安装

```bash
dsh plugin --profile web install github:lolicin/dsh-memory
```

自带的 bundle 配置把记忆存放在 `$DSH_HOME/memory/memory.db`，机器上所有 profile 共享同一份记忆。

## 模型能用到什么

| 工具 | 用途 |
|---|---|
| `memory_write` | 记住一条自包含的持久事实，可带标签和置顶 |
| `memory_search` | 按关键词搜索记忆正文和标签 |
| `memory_forget` | 删除已经过时或错误的记忆 |

另有一个 `memory:recall` 提示词区段，在字数预算内渲染**置顶记忆在前、最近更新的在后**。因此召回不依赖模型记得去搜索——它存过的东西已经摆在面前，搜索只用于预算之外更旧的内容。

`memory_write` 的描述会引导模型避开常见误用：临时任务状态（那是 todo 列表的事）、密钥、以及仓库本身已经记录的事实。

## 配置

```yaml
- id: memory
  name: dsh-memory
  config:
    path: !!js dshHomePath('memory/memory.db')
    promptRecentCount: 10
    promptMaxChars: 2000
    maxTextChars: 2000
    searchLimitDefault: 10
    searchLimitMax: 50
    promptOrder: 50
```

| 字段 | 默认值 | 含义 |
|---|---|---|
| `path` | —（必填） | SQLite 文件路径，或 `:memory:` 表示纯内存存储 |
| `promptRecentCount` | `10` | 提示词区段提供的非置顶最近记忆条数 |
| `promptMaxChars` | `2000` | 渲染区段的字数预算；超出的报告为数量，置顶记忆优先保留 |
| `maxTextChars` | `2000` | 单条记忆的最大字符数 |
| `searchLimitDefault` | `10` | 模型未指定时 `memory_search` 的默认条数 |
| `searchLimitMax` | `50` | 硬性上限，不管模型要多少 |
| `promptOrder` | `50` | 区段顺序；`-100` 是 harness 身份，`0` 是 persona |

`path` **故意没有代码侧默认值**：默认值会把持久的用户事实散落在 harness 碰巧启动的目录里。部署值放在 patch 行中。

## 存储

单个 SQLite 文件：一张 `memories` 表，加两个由触发器保持同步的外部内容 FTS5 索引。打开时自动创建父目录，进程重启后数据保留。

### 中文搜索

FTS5 的 `unicode61` 分词器把一整段中文当作一个 token，所以中文关键词只能命中存储了完全相同字串的记忆——`机骸` 匹配不到 `机骸第九行星`，模型搜索句中间的词会一无所获。因此搜索按顺序尝试三种策略：

1. `unicode61` 索引：处理英文和整段匹配，保留原有的排序；
2. `trigram` 索引：处理三个字符以上的中文 token，可匹配字串中间的任意片段；
3. `LIKE` 扫描：处理 trigram 无法表达的一到两个字符的情况，要求每个 token 都出现在正文或标签中。

后两者只在第一级没有结果时才执行，所以常见的英文路径仍然只有一次查询的开销。扫描是全表扫描，但受用户自己的记忆数量限制，规模很小。在此版本之前创建的库会在首次打开时自动获得 trigram 索引：只要索引行数与 `memories` 不一致就会重建，不丢数据，也不需要手动迁移。

搜索按非字母数字字符切分查询词，并**给每个 token 加引号**，所以模型随手输入的 FTS5 操作符（`OR`、`*`、`-`）会被按字面匹配，而不是改变查询语义或在工具调用中途抛语法错误。各 token 之间是 FTS5 的隐式 AND：每个 token 都必须出现。注意，模型可能当作标点使用的字符（如 `a"b`）会被当作分隔符，这类查询按 `a AND b` 处理。

`node:sqlite` 在 Node 22/24 中仍标记为实验性，所以运行 harness 时会打印一条 `ExperimentalWarning`。harness 自带的 `dsh-session-query-sqlite` 用的也是同一个模块。

## 失败行为

加载时的配置错误会直接抛错：`path` 为空、边界值非正数、或 `searchLimitDefault` 大于 `searchLimitMax`，都会在插件加载时抛出。

调用时的错误是模型可以自行纠正的工具错误：空白事实、超过 `maxTextChars` 的事实。删除一个不存在的 id 时，`memory_forget` 返回**成功**并报告 `forgotten: false`——模型要求的状态本来就已经成立，这不是基础设施故障。

## 扩展点

三个工具通过 `ctx.tools.register()` 注册，召回区段通过 `ctx.systemPrompt.section()` 注册。所有注册都是 Cordis effect，卸载插件会同时移除工具和区段，并关闭数据库。

## 开发

```bash
pnpm install --ignore-workspace
pnpm run typecheck
pnpm test
pnpm run build
```

注意：`lib/` 是已提交的构建产物，安装时不需要构建，因此包里没有 `prepare` 脚本（git 依赖的 `prepare` 会被 pnpm 的构建审批拦截）。

## 本 fork 的改动

- **peer 范围**放宽为 `^0.1.0-rc.6 || ^0.2.0-rc.2`，使 bundle 不被 DSH 0.2.0-rc.2 的兼容性门禁跳过（门禁拿插件声明的 peer 范围与 DSH 运行时版本做 semver 比较）。已对照实际安装的 `dsh-tools@0.1.7-rc.2` 验证：`apply`、三个工具的形状、`presentCall`、`output.render` 均无变化，运行时代码无需改动。
- **中文搜索**按上文所述修复；schema 版本升到 2，trigram 索引在打开时自动回填。
- 删除 `prepare` 脚本，避免 pnpm 对 git 依赖的构建审批。

## 许可证

MIT

## 参考

思路来自 Pi 生态的 [pi-mentis](https://github.com/guchengod/pi-mentis)（MIT）。本包是针对 Harness 扩展点的独立实现，与其没有共享代码。它有意识地放弃了 pi-mentis 的 sidecar 进程、Zvec 向量库和必需的 SiliconFlow 嵌入 key，改用单个本地 FTS5 文件——更小、免 key、离线可用，代价是词法检索而非语义检索。
