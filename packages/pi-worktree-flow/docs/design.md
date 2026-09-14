# pi-worktree-flow 设计文档 —— worktree 强制工作流扩展

> 状态：设计稿（待 review）
> 日期：2025-09-14
> 作者：pi 会话（研究 + 撰写）

## 0. 摘要

新建一个 pi extension 包 `pi-worktree-flow`，把目前只存在于 `~/.pi/agent/AGENTS.md`
的"修改代码前必须走 worktree"流程（纯提示词软约束，依赖 LLM 自觉）转化为
**工具化 + 警告式强制**的闭环：

- **L1 引导层**：注册 `start_worktree` 工具，LLM 在每个新需求开始时调用，
  自动完成 `fetch` → 基于 `origin/main` 创建 branch → `git worktree add` →
  引导切换 session cwd；已在 worktree 时退化为 `rebase origin/main`（幂等）。
- **L2 强制层（警告级）**：拦截 `write` / `edit` / `bash` 的 tool result，
  当写入目标位于某个 git 仓库的**主工作区（main working tree）**时，
  向 tool result 注入醒目警告与恢复指引，驱动 LLM 自我纠正。
- **L3 审计层**：agent 结束时对比相关仓库主工作区的 dirty 基线，
  有变化则通知用户。

拦截判定**按被写文件归属哪个 git 仓库**进行，因此天然同时覆盖两类场景：
session cwd 是 git 项目 / session cwd 是包含多个子 git 仓库的 wrapper 工作区。

## 1. 需求原文与设计目标映射

| 需求 | 原文 | 设计落点 |
|---|---|---|
| R1.1 | session cwd 是 git 项目：新需求修改代码前必须先更新远端状态，基于最新 `origin/main` 创建新 branch，再为该 branch 创建新 Git worktree，并把 pi 会话 cwd 置于该 worktree 内 | L1 `start_worktree` 工具 |
| R1.2 | 已在 worktree 的 cwd：不建新 worktree，执行新需求前 rebase 最新的 `origin/main` | L1 工具的 rebase 分支 |
| R1.3 | 所有代码修改、验证和提交都在新 worktree 中完成，禁止直接写入原工作区或旧 branch | L2 警告层 + L3 审计层 |
| R2.1 | session cwd 是 wrapper 工作区：任何涉及子目录 git 仓库的修改都遵循 1 | 判定按文件归属 repo，天然覆盖 |

## 2. 现状盘点

### 2.1 现有机制的局限

`~/.pi/agent/AGENTS.md` 用自然语言描述了这套流程，但：

- 纯提示词（prompt instruction）没有强制力，LLM 间歇性跳过 fetch、直接在主工作区
  改文件的情况无法根治；
- 无法在事后发现"已经写脏了主工作区"；
- 对 wrapper 场景（`~/AstroCube/gitops-wrapper`：顶层非 git，子目录是多个独立 git 仓库）
  没有结构化支持。

### 2.2 本 monorepo 已有的 pi-worktree 包

`packages/pi-worktree`（上游 `@narumitw/pi-worktree`，v0.51.7）是一个成熟的
**交互式** worktree 管理器：`/worktree` 命令 + TUI 菜单（status / add / switch /
remove / prune / configure root）。与本需求的差距：

| 能力 | pi-worktree | 本需求 |
|---|---|---|
| worktree 创建/删除/prune | ✅（交互确认式） | ✅ 但需 LLM 可调用、免菜单 |
| 基于 `origin/main` 建 branch | ❌ 默认从当前分支 | ✅ 必须先 fetch |
| 当前 worktree rebase `origin/main` | ❌ | ✅（R1.2） |
| 违规写入拦截 | ❌ | ✅ 警告级 |
| 主工作区 dirty 审计 | ❌ | ✅ |
| 触发方式 | 仅用户 `/worktree` 命令 | **LLM 工具调用** + AGENTS.md |

结论：**新建独立包**复用其思路而非改造它——pi-worktree 的 722 行交互式
command 层与"LLM 免交互调用"的形态冲突，而其 `git.ts` 中可复用的纯函数
（约百余行）以复制方式引入（见 §5.3 依赖策略）。

### 2.3 pi-session-context 的协作点

用户已安装 `pi-session-context`（footer / MR / pipeline 监控），其中与本设计
直接相关的机制：

1. **bash cwd 重定向**：它用 `createBashTool + spawnHook` 覆盖内置 bash 工具，
   所有 bash 命令的 cwd 被重定向到 `state.context.worktree.value`
   （即 `set_context` 的 `worktree` key）。
2. **被动 worktree 检测**：任意工具调用的 input 中出现
   `WORKTREE_BASE`（默认 `~/Development/worktree`，可用环境变量
   `PI_WORKTREE_BASE` 覆盖）前缀的绝对路径时，自动设置 worktree key。
3. **set_context 工具**：`worktree` key 设置为 `type: "dir"` 即完成 cwd 切换
   （不要求路径必须位于 WORKTREE_BASE 下）。

因此本扩展**不覆盖 bash 工具**（避免与 pi-session-context 的覆盖互相踩踏），
cwd 切换完全委托给 `set_context` + `PI_WORKTREE_BASE=~/.pi/worktree`，
让被动检测、footer 渲染、bash 重定向三者全部自然生效。

## 3. 总体架构：三层防御

```
用户提出新需求
      │
      ▼
┌─ L1 引导层 ──────────────────────────────────────────────┐
│ AGENTS.md 指令（保留，改为指向工具）                          │
│ start_worktree 工具：                                      │
│   git fetch origin --prune                                │
│   ├─ 已在本 repo 的 worktree → rebase origin/main（幂等）  │
│   └─ 否则 → 基于 origin/main 建 branch                     │
│        → git worktree add 统一 base                        │
│        → 返回指引：调用 set_context 切 cwd                    │
└───────────────────────────────────────────────────────────┘
      │ LLM 忘记调工具 / 走错路径
      ▼
┌─ L2 强制层（警告级，用户已选）───────────────────────────────┐
│ pi.on("tool_result")：write / edit / bash                  │
│   解析写入目标 → 归属 repo → 命中主工作区？                   │
│   命中 → tool result 尾部注入 ⚠️ 警告块 + 恢复指引            │
│   （不 block，写入保留；LLM 读到警告后自行撤回并在 worktree    │
│    重做 —— 与 plannotator 相反的宽松政策）                    │
└───────────────────────────────────────────────────────────┘
      │ bash 动态构造路径等漏网
      ▼
┌─ L3 审计层 ───────────────────────────────────────────────┐
│ session_start：记录涉及 repo 的主工作区 dirty 基线           │
│ agent_settled：重查一次，主工作区变脏 → ctx.ui.notify 用户   │
└───────────────────────────────────────────────────────────┘
```

验证过的依据（不是猜测）：

- `tool_call` 可 `{block: true, reason}` 拦截、`tool_result` 可 patch content
  ——见 pi 官方文档 extensions.md 及 plannotator 的 plan mode 写保护实现；
- `pi.exec("git", [...])` 可在扩展内执行 git ——官方示例 `dirty-repo-guard.ts`；
- `pi.appendEntry()` 状态可跨 `/reload` 与会话恢复 —— pi-session-context 同款用法；
- 扩展注册同名工具可覆盖内置工具 ——本设计**不用**该能力（bash 交给 session-context）。

## 4. 核心判定逻辑（单一真相源）

### 4.1 定义

- **主工作区（main working tree）**：仓库 checkout 的原始目录。判定：
  `git -C <path> rev-parse --git-dir` 为 `.git`（相对），
  或可通过 `--path-format=absolute` 对比 `git dir` 与 `git common dir`。
- **linked worktree**：`git-dir` 形如 `<主仓库>/.git/worktrees/<name>`。
- **统一 base**：`~/.pi/worktree/<repoName>/<branch>`（用户已确认）。
  通过 `~/.pi/agent/pi-worktree.json` 的 `worktreeRoot` 配置
  （与 pi-worktree 共享同一配置文件与键，两者读取方互相兼容）。

### 4.2 写入判定矩阵

对 `write` / `edit` 的 `path`（resolve 到绝对路径后）：

| 写入路径归属 | 判定 | 动作 |
|---|---|---|
| 不在任何 git 仓库内（wrapper 顶层、`~/.pi/` 等） | 放行 | — |
| 位于统一 base `worktreeRoot` 下的 worktree | **合法**（本流程产物） | 放行 |
| 其他 linked worktree（如 `~/.archon/workspaces/...`） | 合法（他方工具的 worktree，不是原工作区） | 放行 |
| git 仓库的**主工作区** | **违规**（R1.3） | 警告 |

> 简化关键：**违规 ≡ 写入目标位于其所属仓库的主工作区**。这一条同时覆盖
> R1.3 与 wrapper 场景 R2.1 —— 判定从不关心 session cwd 是仓库还是 wrapper，
> 只关心被写文件向上最近的 `.git` 归属。

路径 → 仓库归属解析：`git -C <绝对路径> rev-parse --show-toplevel`
（带进程内 `Map` 缓存，key 为 path 的最长已解析祖先目录，避免每次 fork git）。

### 4.3 bash 命令的写语义扫描（启发式，低误报优先）

对 `bash` 工具的 `command` 做**静态正则扫描**，只有高置信命中才警告：

| 模式 | 示例 |
|---|---|
| 重定向到文件 | `> file`、`>> file`（`> /dev/null`、`2>&1` 排除） |
| 写命令的显式目标参数 | `tee [-a]`、`cp`/`mv`/`install`/`dd of=` 的非 flag 参数、`ln -s` 的 target、`patch`、`rsync` 本地目标 |
| 就地修改 | `sed -i`、`gawk -i inplace` |
| 指向主工作区的 git 写操作 | `git -C <主工作区> (commit\|apply\|stash\|am\|rebase\|merge)` |

命中后按与 §4.2 相同的矩阵判定目标路径。**明确声明**（见 §11）：
`python -c "open(...)"`、动态拼接路径等无法静态发现，由 L3 兜底。
相对路径解析基准 = 当前 session worktree（从 pi-session-context 的持久化
entry 读取；读不到则用 `ctx.cwd`）——两者在 worktree 中时相对路径天然合法。

## 5. 新包规约

### 5.1 位置与命名

```
packages/pi-worktree-flow/          # npm 包名 pi-worktree-flow（fork 私有，不上游）
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # 扩展入口：注册工具、命令、事件
│   ├── git.ts            # git 封装：fetch/rebase/worktree add/判定（含 pi-worktree 复制函数）
│   ├── flow-tool.ts      # start_worktree 工具（L1）
│   ├── guard.ts          # tool_result 警告层（L2）+ 判定矩阵
│   ├── audit.ts          # dirty 基线审计（L3）
│   ├── state.ts          # pi.appendEntry 持久化状态
│   └── settings.ts       # worktreeRoot 读取（兼容 pi-worktree.json）
├── test/                 # 与 pi-worktree 同风格测试
└── docs/design.md        # 本文档
```

### 5.2 package.json

`"pi": { "extensions": ["./src/index.ts"] }`（与 pi-worktree 的 dist 形态不同：
本包不依赖构建步骤，直接用 TS 源加载，减少 monorepo object form 部署摩擦；
`peerDependencies` 声明 `@earendil-works/pi-coding-agent`）。

### 5.3 依赖策略：不跨包 import

pi 以 object form 单包安装 git 包时不会安装 monorepo 兄弟包，
跨 workspace import 会造成运行时缺失。因此从 pi-worktree **复制**所需纯函数
（`parseWorktreePorcelain` 的 porcelain 解析、branch 校验思路等约百余行），
在文件头注明 `// adapted from packages/pi-worktree/src/git.ts (MIT)`。

### 5.4 与 pi-worktree 的共存

两者共享 `~/.pi/agent/pi-worktree.json` 的 `worktreeRoot`：

- pi-worktree：`/worktree` 交互式管理（新建/删除/prune/切换）——人用；
- pi-worktree-flow：`start_worktree` 自动流程 + 强制警告——LLM 用；
- 删除 worktree 一律走 pi-worktree（其风险检查完备），本包不提供删除。

## 6. L1：`start_worktree` 工具规格

### 6.1 参数（typebox schema）

```ts
{
  repo: Type.Optional(Type.String({ description:
    "目标仓库相对 session cwd 的路径。session cwd 就是仓库本身时省略；" +
    "wrapper 工作区中必须传子目录名，如 'gitops-astrocube'。" })),
  branch: Type.Optional(Type.String({ description:
    "新建分支名（kebab-case）。省略且需要新建时必须提供；" +
    "若当前已在 worktree 中：省略 = 延续当前分支（rebase），" +
    "提供新名字或 new_branch=true = 从 origin/main 建新分支。" })),
  new_branch: Type.Optional(Type.Boolean({ description:
    "明确要求新分支（从最新 origin/main 创建新 worktree）。" })),
  description: Type.Optional(Type.String({ description:
    "一句话需求描述，用于日志与状态展示。" })),
}
```

`promptGuidelines`（写入系统 prompt 的工具使用准则）：

- `"调用 start_worktree 是开始任何包含代码修改的需求的强制第一步"` 
- `"已在 worktree 继续当前需求迭代时：调用 start_worktree 执行 fetch + rebase origin/main"`
- `"wrapper 工作区（cwd 非单个 git 仓库）中：通过 repo 参数指定目标子仓库"`

### 6.2 执行流程

```
resolve repoRoot:
    repoRoot = git rev-parse --show-toplevel (repo ?? ".")   # 相对 ctx.cwd
    失败（不是 git repo）→ 报错：请确认目标仓库路径

git -C repoRoot fetch origin --prune
    失败（网络）→ 报错并说明"按规则必须先更新远端"，LLM 转告用户

base = resolveBaseBranch(repoRoot)     # 见 §6.3

if ctx.cwd 所在 worktree 属于同一 repoRoot（且其 branch 即当前分支）:
    # ── R1.2：已在 worktree ──
    if new_branch 或 branch 与当前不同 → 走创建分支
    else:
        behind = git rev-list --count HEAD..origin/<base>
        if behind == 0 → return { status: "up-to-date", worktree: cwd }
        if git status --porcelain 非空 →
            return 报错：请先 commit 或 stash 未提交修改再 rebase
        git rebase origin/<base>
            冲突 → git rebase --abort，报错并转告用户人工处理
        return { status: "rebased", ahead, worktree: cwd }

# ── R1.1：主工作区（或另一个 repo 的 worktree 场景）→ 创建 ──
branch   = 参数 branch（必填校验：非空、合法 ref 名、
           `git show-ref --verify` 不存在既有同名本地分支，
           且该分支未被其他 worktree 占用）
target   = worktreeRoot/<repoName>/<branch>
git -C repoRoot worktree add -b <branch> <target> origin/<base>
persist state（见 §6.4）

return {
    status: "created",
    worktree: target, branch, base: "origin/<base>@<short-oid>",
    next_step: "调用 set_context，设置 worktree key (type: dir, value: <target>)，然后在新 cwd 开始工作。之后所有 read/bash 在 worktree 内执行。"
}
```

**cwd 切换双保险**：`next_step` 文本指引 LLM 调用 `set_context`；同时返回的
`worktree` 绝对路径位于 `PI_WORKTREE_BASE` 时，pi-session-context 的被动检测
（扫描 tool input 中的 base 前缀路径）会在 LLM 后续任何含该路径的工具调用时
自动设置 worktree key——即使 LLM 漏调 `set_context`，bash cwd 也会在开始读
代码时自动落位。

### 6.3 base 分支解析

优先级：`~/.pi/agent/pi-worktree.json` 的 `baseBranch`（本包新增键，可选）→
`git symbolic-ref refs/remotes/origin/HEAD` 解析（如 `origin/main`）→
依次探测 `origin/main`、`origin/master` → 都失败则报错。
（不硬编码 main：照顾仍在 master 的旧仓库。）

### 6.4 状态持久化

`pi.appendEntry("pi-worktree-flow", { repos: { [repoRoot]: { branch, worktree, createdAt } } })`，
`session_start`（reason: resume/startup）时恢复，供 `/flow-status` 展示与 L2
判断"当前授权的 worktree 集合"（用于快速路径，而非白名单依据——判定矩阵
不依赖它，见 §4.2）。

### 6.5 用户命令

- `/flow-status`：列出本 session 的 flow 状态（各 repo 分支、worktree、
  距 origin/main 的 ahead/behind）。
- `/flow-pause` / `/flow-resume`：临时挂起/恢复 L2 警告（escape hatch，
  应对误报卡壳）。状态持久化到 session entry。

## 7. L2：警告层规格

### 7.1 实现选型

**`pi.on("tool_result")` patch content**（用户已选"警告不阻止"）：
写入保留，tool result 尾部追加警告块。LLM 在下一 turn 读到警告，
按指引撤回并在 worktree 重做。

> 备选过的方案：`tool_call` block + `ctx.ui.confirm` 逐次放行——交互噪音大；
> `tool_call` 直接放行不提示——无约束。均不采纳。

### 7.2 警告文本（模板）

```
⚠️ [pi-worktree-flow] 该写入目标位于 git 仓库主工作区（main working tree）：
   <absPath>  （repo: <repoRoot>）
   这违反 worktree 工作流（存在污染主分支工作区的风险）。
   - 若属于误操作：运行 `git -C <repoRoot> checkout -- <relPath>` 撤销，
     调用 start_worktree 建立/进入 worktree 后重做；
   - 若属于用户明确要求的例外（如仓库级配置而非代码），完成后向用户说明即可。
```

### 7.3 去重

同一 `(repoRoot, path)` 只在**本 turn（agent run）内**警告一次
（内存 `Set`，agent_end 清空），避免同一文件连续 edit 刷屏；
跨 turn 重复违规仍会再次警告。

### 7.4 触发范围

| 工具 | 触发条件 | 依据 |
|---|---|---|
| `write` | path 归属主工作区 | §4.2 矩阵 |
| `edit` | 同上 | 同上 |
| `bash` | 静态扫描命中写语义且目标（可解析部分）归属主工作区 | §4.3 |

子代理（subagent）进程同样加载本扩展（全局包），在子进程内同样生效；
`-p` / JSON 模式下 `ctx.hasUI=false`，警告仍通过 tool result 传递（不依赖 UI）。

## 8. L3：审计层规格

- `session_start`：对（恢复出的）flow 涉及 repo 集合，或 cwd repo，
  记录主工作区 `git status --porcelain | sha1` + `git rev-parse HEAD` 快照。
- `agent_settled`：重新计算并对比；主工作区 dirty 集合**扩大** →
  `ctx.ui.notify` 用户（不注入 agent 消息，避免循环）。
- 触发面仅限本 session 的 flow 涉及 repo（而非全盘扫描），控制开销。

## 9. 配置与部署

### 9.1 配置文件（与 pi-worktree 共享）

`~/.pi/agent/pi-worktree.json`：

```json
{ "worktreeRoot": "~/.pi/worktree" }
```

（pi-worktree 的 `/worktree` → Configure root 也可写入同键。）

### 9.2 环境变量

shell rc（`~/.zshenv`）导出：

```sh
export PI_WORKTREE_BASE="$HOME/.pi/worktree"
```

使 pi-session-context 的 footer 渲染 / 被动检测 base 与本包统一。

### 9.3 settings.json packages（object form）

在 `~/.pi/agent/settings.json` 的 `packages` 数组追加（`pi install` 后由 pi 维护）：

```json
{
  "source": "git:github.com/soraliu/pi-extensions@<merge-commit>",
  "extensions": ["packages/pi-worktree-flow/src/**"]
}
```

（monorepo 单包 object form + 通配 glob，遵循既有 pi-statusline 的部署惯例。）

### 9.4 AGENTS.md 更新（建议文本）

`~/.pi/agent/AGENTS.md` 中"代码修改原则"整节替换为：

```markdown
## 代码修改原则（由 pi-worktree-flow 扩展保障）

- 任何包含代码修改的新需求，第一步必须调用 start_worktree 工具：
  - 未在 worktree 时它会 fetch origin 并基于最新 origin/main 建 branch +
    worktree（统一位于 ~/.pi/worktree/<repo>/<branch>），随后调用 set_context
    切换 cwd 到该 worktree
  - 已在 worktree 时调用它会 fetch + rebase 最新 origin/main
- 所有代码修改、验证和提交都在 worktree 内完成；直接写入仓库主工作区会被
  警告并要求撤销重做；除非用户明确要求例外
- wrapper 工作区（cwd 非 git 仓库、子目录为独立 git 仓库）中：通过
  start_worktree 的 repo 参数指定目标子仓库，同样遵循上述流程
- worktree 的删除/清理用 /worktree 命令，不要手动 git worktree remove
```

## 10. 测试计划

沿用本 monorepo 测试风格（`test/*.test.ts`，git 集成测试用临时 fixture 仓库）：

1. **判定矩阵单测**（guard）：fixture 构建 main repo + linked worktree +
   统一 base 伪装路径，覆盖 §4.2 全部四行 + 相对路径解析。
2. **start_worktree 集成测试**：临时 bare origin + clone，验证：
   - 主工作区调用 → branch 从 origin/main 创建、worktree 落位统一 base；
   - origin 有新提交 → worktree 内调用 → rebase 后 HEAD 对齐；
   - origin main 未动 → up-to-date 幂等不 rebase；
   - 脏 worktree → rebase 被拒并报错；rebase 冲突 → abort 保持原状。
3. **L2 警告单测**：mock `pi.on` 事件流，验证 tool_result patch 文本与去重。
4. **L3 单测**：dirty 快照对比逻辑。
5. **bash 启发式单测**：§4.3 模式表的命中/排除用例（含 `/dev/null`、`2>&1`）。

## 11. 已知限制（诚实边界）

1. **警告非安全边界**：写入先发生再警告，依赖 LLM 依指引撤销；恶意或绕过
   语义（`python -c "open(...)"` 等）静态扫描无法覆盖，L3 只能事后告警。
   这是"防误、防懒"设计而非沙箱（sandbox）。
2. **同 turn 并行工具**：并行 bash 调用的 cwd 解析取同一快照，个别时序下
   警告的 repo 归属可能滞后一拍（不影响判定方向）。
3. **多 repo 并行需求**：`set_context` 仅一个 worktree key，跨 repo 切换需
   LLM 重新调用 `set_context`（flow 状态支持多 worktree 并存，判定不受影响）。
4. **`PI_WORKTREE_BASE` 未导出时**：session-context 的被动检测与 footer
   渲染不匹配统一 base，需按 §9.2 配置（cwd 重定向仍可通过 set_context 生效）。

## 12. 实施步骤

1. `~/forks/pi-extensions`：`git fetch origin`，基于 `origin/main` 建
   `feat/pi-worktree-flow` branch + worktree（即本流程的 dogfood 延续）；
2. 按本设计实现包骨架 + 测试（预计 800–1200 行含测试）；
3. 更新 root `package.json` 的 `pi.extensions` 注册（若采用仓库级加载）与
   本包 `package.json`；
4. push → 本（用户自己的）fork 仓库 PR → merge（不向 narumiruna 上游提 PR）；
5. `~/.pi/agent/settings.json` 按短格式追加 object form 安装新 commit；
6. 配置 `pi-worktree.json` / `PI_WORKTREE_BASE` / AGENTS.md 更新（§9）；
7. 验收场景过一遍（§13），删除开发 worktree。

## 13. 验收场景

| # | 场景 | 期望 |
|---|---|---|
| 1 | 在 `~/.nixfiles` 提"升级某个 flake input" | LLM 调 start_worktree → worktree 建于 `~/.pi/worktree/nixfiles/<branch>`，bash cwd 切换，修改发生在 worktree |
| 2 | 在 `~/AstroCube/gitops-wrapper` 提"修改 gitops-astrocube 的 X" | repo 参数生效，worktree 建于 `~/.pi/worktree/gitops-astrocube/<branch>` |
| 3 | 场景 1 完成后同 session 继续迭代 | start_worktree → 只 fetch+rebase，不新建 |
| 4 | LLM 越过工具直接 edit 主工作区文件 | tool result 出现 ⚠️ 警告块，LLM 撤销并在 worktree 重做；agent 结束后 L3 确认主工作区干净 |
| 5 | 主工作区为 repo 的闲暇查询（git log / grep） | 无警告（只读不触发） |
| 6 | `/flow-status`、`/flow-pause` | 正常显示 / 暂停生效 |

---

*本设计基于 pi 0.85.1 的扩展 API（extensions.md）与 monorepo 现状
（pi-worktree v0.51.7、pi-session-context）调研撰写；所有被引用机制均已在
源码或官方示例中验证存在。*
