# AGENTS.md — DSH Desktop Agent Bridge · 让外部 Agent 在现有 DSH 桌面内协作

> 仓库级协作入口约定：在本仓库开始编码、评审或文档工作前先通读本文件。

## 1. 项目定位与背景

- **目标**：提供一个安装在现有 DSH 桌面 profile 中的 bundle 插件与 MCP 网关，让 Codex、Hermes、Grok Bot 等外部 MCP 客户端把任务交给正在运行的 DeepSeek Harness 桌面应用；模型调用、工具执行、原生会话与审批仍由 DSH 宿主负责。
- **仓库形态**：单仓库 Node.js ESM 项目，Git 管理；本仓库为用户 fork（`xiao-hans/dsh-desktop-agent-bridge`），包名与插件 ID 保持上游兼容。
- **权威依据**（按优先级）：
  - `package.json` — 包名、版本、入口、DSH bundle/client 声明、Node 与 DSH 引擎范围。
  - `src/protocol.mjs` — 协议版本、严格请求结构、客户端授权与宿主/UI 远程服务描述。
  - `docs/architecture.md` — 进程边界、身份授权、任务状态机、原生交互与持久化约定。
  - `docs/releasing.md` — 预构建产物、安装验收与发布流程。
  - `.agents/skills/dsh-desktop-agent-bridge/SKILL.md` — 外部 Agent 使用本桥接工具的公开操作契约。

## 2. 硬性边界与红线

1. **兼容基线单一**：当前适配基线为 Windows + DSH Desktop `0.2.2-rc.3`。`engines.dsh` 使用覆盖该版本的 semver 范围；未实测的系统或版本不得写成“已兼容”。
2. **DSH API 集中适配**：所有宿主接口只能经 `src/host-adapter.mjs` 调用；UI 只使用 `sidebar.footer.action` 等公开插槽。禁止修改全局模型、关闭审批、启动第二个 DSH 实例或让外部 Agent 自行提权。
3. **身份与契约稳定**：内部包名 `dsh-agent-collaboration`、插件 ID `agent-collaboration`、服务名 `agentCollaboration` 与既有 `dsh_*` MCP 工具名不得随意更改；对外行为变更须同步 `src/protocol.mjs`、测试与文档。
4. **预构建交付**：Git 安装不执行 `prepare`/构建脚本。`dist/` 必须由本次源码构建生成并提交；禁止新增 `preinstall`、`install`、`postinstall`、`prepare`、`prepack` 脚本。
5. **凭据与状态不入库**：不得提交 `state.json`、`endpoint.json`、`host.lock`、`client-*.json`、`remote-*.json`、令牌、私钥、验收日志或用户绝对路径。状态目录位于用户目录下的 `.dsh/agent-collaboration`。
6. **最小授权**：新客户端默认没有工作区、能力包和模型授权；父交互默认关闭。不得为通过测试而放宽默认授权。

## 3. 技术栈与模块拓扑

- **运行与工具链基线**：Node.js `>=22`（CI 覆盖 22/24）· ESM · DSH Desktop 插件（宿主 Node 进程 + Web 客户端）· esbuild `^0.25` · React 客户端 · `@modelcontextprotocol/sdk` `^1.12` · zod `^4.4`。

| 模块 / 子系统 | 路径或产物 | 职责归属与依赖方向约束 |
| --- | --- | --- |
| 协议与版本 | `src/protocol.mjs` | 严格请求/授权结构、宿主与 UI 服务描述、`VERSION`；不依赖任何运行时模块 |
| 宿主生命周期 | `src/host.mjs` → `dist/host.mjs` | 插件入口、状态目录锁、本机端点、UI 服务注册；依赖 broker、http 与 host-adapter，不直接调用 DSH 业务 API |
| DSH 适配层 | `src/host-adapter.mjs` | 唯一的 DSH 会话、模型、权限、历史、队列与原生交互边界；DSH 升级时优先修改此文件 |
| 身份与调度 | `src/broker.mjs` | 令牌身份、资源授权、会话归属、任务去重、队列与审计；不得绕过授权直接派发 |
| 本机 RPC | `src/http.mjs` | 仅回环监听的固定路由入口与请求边界；不得暴露公网 |
| MCP 工具层 | `src/mcp-server.mjs` | 本机与远程共用的十个 `dsh_*` 工具与结果截断规则 |
| MCP 传输 | `src/mcp.mjs`、`src/remote-mcp.mjs` → `dist/mcp.mjs`、`dist/remote-mcp.mjs` | 本机 stdio 与远程 Streamable HTTP；只连接宿主 RPC，不直接执行业务逻辑 |
| 客户端 UI | `src/client.jsx`、`src/client.css` → `dist/client.mjs` | 左侧 Agent 协作区与授权设置；只依赖 React 与 DSH 客户端插槽 |
| 构建与发布 | `scripts/build.mjs`、`scripts/check-release.mjs`、`scripts/check-mcp.mjs` | 生成 `dist/`、许可文本与发布扫描；不得把生成文件混入源码目录 |
| 安装维护 | `scripts/install-local.mjs`、`install-support.mjs`、`update-local.mjs`、`uninstall-local.mjs` | 旧式手工安装的维护路径；与插件页安装互斥，不得叠加 |
| 测试 | `test/*.test.mjs` | 使用替身宿主验证 broker、adapter、HTTP、MCP、UI 与安装边界；不替代真机验收 |

- **关键技术取舍**：采用“现有 DSH 宿主内插件 + 独立 MCP 传输”而非启动独立 headless DSH 实例：复用用户当前桌面会话、工具与审批，避免第二套执行环境。

## 4. 目录结构

```text
dsh-desktop-agent-bridge/
├─ AGENTS.md                 仓库协作约定
├─ package.json              包清单、DSH bundle/client 声明与验证脚本
├─ cordis.patch.yml          DSH bundle 补丁入口
├─ src/                      插件宿主、适配器、broker、HTTP、MCP 与客户端 UI 源码
├─ test/                     Node 内置测试套件的替身宿主用例
├─ scripts/                  构建、发布扫描、MCP 校验与旧式安装维护
├─ docs/                     架构、远程接入、发布与中英文说明
├─ dist/                     预构建运行入口与第三方许可文本（提交）
├─ .agents/skills/           供外部 Agent 使用的项目 Skill
├─ .github/workflows/ci.yml  Windows/Linux × Node 22/24 检查
└─ examples/                 客户端配置与调用示例
```

## 5. Skill 调度

- **动态匹配与全文读取**：任务开始、需求变更或进入新阶段时，将任务意图与当前会话可用 Skill 的名称、描述和适用边界匹配；存在多个候选时选用覆盖任务的最小组合。执行对应操作前读取所选 `SKILL.md` 全文及其要求的参考文件。
- **冲突与授权边界**：Skill 规则与本文件、仓库代码或构建配置冲突时，以仓库实际证据与本文件为准。加载 Skill 仅建立操作流程，安装组件、修改全局配置、文档归档同步、写入持久记忆、提交与推送仍受当前请求授权约束。

| 触发场景 | 调用的 Skill | 用途与产出要求 |
| --- | --- | --- |
| 新建仓库、补齐或校准 `AGENTS.md`/`CLAUDE.md` | `project-agents-md` | 按仓库证据生成工程宪法，保留两张调度表并清除占位符 |
| 功能、修复或重构进入实现阶段 | `software-development-workflow` | 先界定边界与完成标准，再做薄垂直切片并用实际检查证明 |
| 需要测试先行或红绿重构 | `tdd` | 先写失败测试，再实现最小通过，最后重构 |
| 复现疑难缺陷、性能回归或兼容性异常 | `diagnose` | 复现→最小化→假设→插桩→修复→回归测试，保留证据链 |
| 准备宣称完成、修复或通过前 | `verification-before-completion` | 实际运行验证命令并核对输出，禁止无证据断言 |
| 用户要求提交或整理提交 | `git-commit` | Conventional Commits，禁止任何 AI attribution |
| 修改 `.agents/skills/dsh-desktop-agent-bridge/SKILL.md` | `skill-creator` | 按技能规范维护触发描述、步骤与工具契约 |
| 需要分析 DSH 桌面打包产物或 Electron 宿主接口 | `reverse-engineer-anything`（可选） | 用 REA 取得运行时证据；普通源码阅读不启用 |
| 文档审计、前后矛盾或过时说明 | `hai-audit-docs`（可选） | 按源码与运行证据定位漂移并给出修复归属 |

- **发现记录**：来自当前会话技能目录与仓库内 `.agents/skills/`；项目自身 Skill 名为 `dsh-desktop-agent-bridge`，其余为用户环境已安装的通用工程 Skill。
- **环境全局约定**：默认使用简体中文回复；命令输出可能经 RTK 压缩，判断结果以退出码、完整日志与可复现命令为准；仅在仓库存在 `.codegraph/` 索引时优先使用代码图谱。

## 6. MCP 与工具调度

- **调用与降级准则**：① 调用前核实当前会话实际暴露的 server / tool 名称与参数；② 写文件或改外部状态前先读取目标现状；③ 严禁向外部服务发送密钥、签名文件、凭证或敏感数据；④ 外部工具或缓存索引结果与当前仓库源码冲突时，以仓库实读结果为准；⑤ MCP 不可用或异常时降级至本地文件、搜索与构建工具完成工作，并在交付时注明；⑥ 新增项目级 MCP 配置须经用户明确同意并同步更新「目录结构」。

| 触发场景 | 调用的 MCP / 工具 | 用途与降级方式 |
| --- | --- | --- |
| 查询跨会话的环境参数、构建坑或用户偏好 | `basic-memory`（`bm_search` / `bm_read`） | 先搜再读，项目固定 `main`；不可用时以仓库文档与当前会话为准 |
| 核对 DSH、MCP SDK、esbuild、React 等库的当前 API | `context7` | 获取版本对应文档；不可用时查本地 DSH 安装包源码与仓库锁文件 |
| 仓库存在 `.codegraph/` 索引时定位符号与调用链 | `codegraph`（可选） | 先图谱后读文件；无索引时改用内置 `grep`/`read` |
| 需要逆向 DSH 桌面打包产物、Electron 资源或运行时接口 | `rea`（可选） | 用 REA 提取证据；不可用时用 Node/strings 与本地解包结果 |
| 读写仓库文件、检索代码、执行构建与测试 | 内置 `ops_read_files`/`ops_edit_file`/`ops_grep`/`ops_bash` | 优先结构化文件 API；失败时按工具语义修正路径，不绕过沙箱策略 |

- **发现记录**：来自当前会话 MCP server 列表（`basic-memory`、`context7`、`codegraph`、`rea`）与内置文件/搜索/执行工具；仓库根目录未发现项目级 MCP 配置文件。

## 7. 验证、提交与完成标准

- **日常验证**：改动后至少运行 `npm test`；涉及构建产物或发布清单时运行 `npm run check`（测试、构建、MCP 校验、发布扫描）。区分源码静态核对、构建通过、自动化测试与真机/运行态证据；检查未通过前不声明完成。
- **工作区与范围保护**：仅暂存本次任务拥有的文件；运行 `git diff --check` 与 `git status --short` 核查，保留无关未提交改动。`node_modules/`、`work/`、`artifacts/`、`*.tgz` 与状态文件不纳入版本库。
- **提交与推送**：仅在用户明确要求时提交或推送；提交说明写明变更原因与影响范围，按会话要求附带必要 attribution，禁止 AI 署名。
- **Definition of Done**：
  - [ ] 代码、协议或配置与权威定义一致，中英文 README、`docs/` 与 `CHANGELOG.md` 的兼容基线同步更新。
  - [ ] `npm ci --ignore-scripts --no-audit --no-fund`、`npm test`、`npm run build`、`npm run check` 与 `npm pack --dry-run --ignore-scripts` 均有实际通过输出。
  - [ ] `dist/` 来自本次源码构建并与源码一致；工作区干净且无临时文件、令牌或用户路径残留。
  - [ ] 对应 DSH 版本已在隔离 profile 完成插件页/本地目录安装验收，记录宿主版本、成功路径与未覆盖项。

## 8. 公开契约与接口权威清单

- **权威定义路径**：`src/protocol.mjs`（请求、授权、服务描述与版本）、`package.json`（包入口、bin、exports、DSH bundle/client 声明）、`.agents/skills/dsh-desktop-agent-bridge/SKILL.md`（外部 Agent 工具使用契约）。
- **实现对照记录**：`src/mcp-server.mjs`（十个 `dsh_*` 工具与 60,000 字符结果截断）、`src/host-adapter.mjs`（DSH 宿主 API 边界）、`src/client.jsx`（`sidebar.footer.action` 插槽）。
- **变更规则**：新增或修改对外行为前先核对权威定义；破坏性变更须评估现有客户端、连接文件与已装 bundle 的兼容性，并同步版本与迁移说明。

## 9. 数据模型与状态持久化

- **权威定义路径**：`src/protocol.mjs` 与 `docs/architecture.md`。
- **核心实体与不变量**：客户端身份、资源授权、会话归属、任务记录、交互回复审计；`requestId` 在同一身份内幂等，相同 ID 不同参数拒绝；任务状态为 `preparing → queued → running → completed/failed/cancelled/blocked`，等待态为 `waiting_approval`/`waiting_input`，不确定提交为 `unknown`。
- **存储与迁移策略**：默认 `~/.dsh/agent-collaboration`；`state.json` 持久化授权/归属/任务/审计，`client-<id>.json` 保存本机凭据，`endpoint.json` 与 `host.lock` 为运行态文件。写入采用临时文件加替换；卸载保留状态与原生会话，不做加密或跨账号隔离。

## 10. 领域专项运行约束（DSH 桌面插件）

- **GUI / 客户端**：协作区固定在左侧 `sidebar.footer.action` 插槽；授权与接管操作必须由用户显式触发；不可逆操作（停止任务、撤销授权）先确认影响范围；异步状态必须覆盖加载、失败与空态。
- **宿主 / 进程边界**：HTTP 服务仅绑定回环地址；插件状态目录单实例锁；外部 MCP 断线不撤销已派发任务，DSH 退出可能中断工作，不能承诺自动恢复或重发。
- **SDK / CLI 契约**：保持十个 `dsh_*` 工具的参数、退出码与 stdout/stderr 约定；结果文本超过 60,000 字符只返回末尾并标记 `truncated`。

## 11. 迁移基准与已知限制

- **对照基准**：`0.2.2-rc.1` 适配记录见 `CHANGELOG.md` 与 `docs/architecture.md`；当前适配基线为 DSH Desktop `0.2.2-rc.3`，`engines.dsh` 为 `>=0.2.2-rc.1 <0.3.0-0`，会话模型选择使用公开 `session/selectModel` RPC，能力探测使用 `typert.local` 注册表。
- **已知边界**：Windows + DSH Desktop `0.2.2-rc.3` 以外的系统/版本未验证；远程 HTTPS 终止与隧道由部署方负责，仓库未集成；父 Agent 真人往返仅 Codex 完成验收；CI 使用替身宿主，不代表真实桌面兼容。

## 12. 待确认清单

- [ ] DSH Desktop `0.2.2-rc.3` 的插件页安装、启用、重启与协作 UI 真机验收 — 需在 Windows + rc.3 桌面环境执行并记录结果。
