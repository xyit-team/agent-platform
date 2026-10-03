# 插件总表

> **插件 = 本地的小功能单元，一项明确能力。** 全部功能都拆成小插件放这里。
> 每个插件一个独立文件夹，带一份开发文档。

## 1. 插件与核心的边界

| | 核心 | 插件 |
|---|---|---|
| 位置 | `../核心/` | 本目录 |
| 拿掉之后 | 系统起不来 | 系统照常，少一项能力 |
| 依赖方向 | **不依赖任何插件** | 依赖核心接口 |

**核心不允许依赖插件。** 若某项能力缺失导致核心起不来，它就是核心的。

## 2. 拆分原则

**一个插件 = 一件可以被独立装载、独立禁用、被多个整合包复用的事。**

| 判据 | 结论 |
|---|---|
| 能被单独关掉且其余正常 | ✅ 独立插件 |
| 必须和其他的一起用才有意义 | ❌ 合并 |
| 换实现（COM ↔ CLI）算不算两个 | 看 `ioSchema` 是否相同 |
| 换渠道（云端 ↔ 本地）算不算两个 | **算** —— 降级路径不同 |

## 3. 分工

| 类别 | 内容 | 风险级别 |
|---|---|---|
| **基础** | 通用能力：文件、终端、格式化等 | 视具体工具 |
| **沙箱** | 隔离执行环境 | — |
| **工作空间** | 新建空间（git worktree / 沙箱） | — |
| **语音** | TTS、云端/本地 ASR、小键盘 | — |
| **文档办公** | Office 文档读写、LSP | elevated |
| **可视化** | 截屏、框选、鼠标精灵、渲染 | — |
| **协作** | 分发、智能体池、取一测试、时间回溯、上下文 | — |
| **文档** | 文档存取、询问、修订、动态、任务日志、目标表、Plan | — |
| **工具** | 五类工具：编写/计算/建模/绘画/攻击渗透 | 视具体工具 |

## 4. 全部插件清单（105 个）

### 4.1 基础（8）

| 插件 | 能力 | 权限 | 依赖 |
|---|---|---|---|
| [fs](基础/fs/开发文档.md) | 文件读写 | `fs:read` `fs:write` | — |
| [project-scan](基础/project-scan/开发文档.md) | 项目扫描、语言检测、约束探测 | `fs:read` | — |
| [format](基础/format/开发文档.md) | 代码格式化 | `fs:read` `fs:write` `process` | fs |
| [terminal](基础/terminal/开发文档.md) | 本地终端调用 | `process` | — |
| [local-model](基础/local-model/开发文档.md) | 本地模型调用 | — | — |
| [build-test](基础/build-test/开发文档.md) | 本地测试与构建 | `fs:write` `process` | terminal |
| [device-input](基础/device-input/开发文档.md) | 截图、剪贴板、设备输入 | `device` `clipboard` | — |
| [git](基础/git/开发文档.md) | 版本控制、检查点、回滚 | `fs:read` `fs:write` `process` | fs |

### 4.2 沙箱（3）

| 插件 | 能力 | 隔离强度 |
|---|---|---|
| [sandbox-process](沙箱/sandbox-process/开发文档.md) | 进程级沙箱 | 中 |
| [sandbox-container](沙箱/sandbox-container/开发文档.md) | 容器级沙箱 | 强 |
| [sandbox-mgr](沙箱/sandbox-mgr/开发文档.md) | 沙箱生命周期与降级决策 | — |

### 4.3 工作空间（2）

| 插件 | 能力 |
|---|---|
| [workspace-git](工作空间/workspace-git/开发文档.md) | git worktree / branch 新建空间 |
| [workspace-sandbox](工作空间/workspace-sandbox/开发文档.md) | 沙箱空间，对接沙箱插件 |

### 4.4 语音（4）

| 插件 | 能力 | 是否出网 |
|---|---|---|
| [tts-out](语音/tts-out/开发文档.md) | 语音输出（可高亮） | 视引擎 |
| [asr-cloud](语音/asr-cloud/开发文档.md) | 云端语音识别 | **是** |
| [asr-local](语音/asr-local/开发文档.md) | 本地 4B 以下模型识别 | **否** |
| [keypad](语音/keypad/开发文档.md) | 小键盘语义输入 | 否 |

### 4.5 文档办公（5）

| 插件 | 能力 | 平台 |
|---|---|---|
| [office-word](文档办公/office-word/开发文档.md) | Word 读写 | COM / CLI |
| [office-excel](文档办公/office-excel/开发文档.md) | Excel 读写 | COM / CLI |
| [office-ppt](文档办公/office-ppt/开发文档.md) | PPT 读写 | COM / CLI |
| [office-mail](文档办公/office-mail/开发文档.md) | Outlook 邮件 | COM |
| [lsp](文档办公/lsp/开发文档.md) | LSP 代码智能 | 跨平台 |

### 4.6 可视化（4）

| 插件 | 能力 |
|---|---|
| [screen-capture](可视化/screen-capture/开发文档.md) | 截屏，**保留全屏尺寸 + 区域** |
| [region-pick](可视化/region-pick/开发文档.md) | 框选屏幕 → 转为上下文 |
| [cursor-sprite](可视化/cursor-sprite/开发文档.md) | 鼠标精灵（上下文手柄） |
| [visual-render](可视化/visual-render/开发文档.md) | 可视化渲染（依赖图/组件树/任务池） |

### 4.7 协作（8）

| 插件 | 能力 |
|---|---|
| [dispatch-llm](协作/dispatch-llm/开发文档.md) | LLM 级任务分发，输出 `rationale` |
| [agent-pool](协作/agent-pool/开发文档.md) | 智能体池，1–5 动态决定 |
| [debate](协作/debate/开发文档.md) | 取一测试：独立作答 → 争论 → 取一 |
| [time-rewind](协作/time-rewind/开发文档.md) | 时间回溯与分支（不是撤销） |
| [context-chain](协作/context-chain/开发文档.md) | 上下文链式存储 + 编组共享 |
| [context-search](协作/context-search/开发文档.md) | 上下文动态搜索 |
| [context-ask](协作/context-ask/开发文档.md) | 旁路问答，**不打断** |
| [session-hub](协作/session-hub/开发文档.md) | 跨会话派活：A 说一句，B 去干，完成后回传结论（依赖 `dispatch-llm`） |

### 4.8 文档（7）

| 插件 | 能力 |
|---|---|
| [doc-store](文档/doc-store/开发文档.md) | 文档存取 |
| [doc-askable](文档/doc-askable/开发文档.md) | 询问式文档（可被问） |
| [doc-revise](文档/doc-revise/开发文档.md) | 人机双向修订（人2传改/人2评论） |
| [doc-dynamic](文档/doc-dynamic/开发文档.md) | 动态文档 + 多 Agent 整合 |
| [task-log](文档/task-log/开发文档.md) | 任务日志 = 明确进度 |
| [goal-table](文档/goal-table/开发文档.md) | 目标表 = Agent 自查 |
| [plan-mode](文档/plan-mode/开发文档.md) | Plan 模式增强（可质询/修订/否决） |

### 4.9 工具（8）

| 插件 | 类别 | 风险 | 能力 |
|---|---|---|---|
| [tool-registry](工具/tool-registry/开发文档.md) | — | — | 工具注册，按类别展开 |
| [tool-write](工具/tool-write/开发文档.md) | 编写 | safe | 文件写入、代码生成 |
| [tool-exec](工具/tool-exec/开发文档.md) | 计算 | elevated | 终端执行 |
| [tool-build](工具/tool-build/开发文档.md) | 计算 | elevated | 构建 |
| [tool-test](工具/tool-test/开发文档.md) | 计算 | elevated | 测试执行 |
| [tool-modeling](工具/tool-modeling/开发文档.md) | 建模 | safe | 数据建模、架构设计 |
| [tool-draw](工具/tool-draw/开发文档.md) | 绘画 | safe | 图像生成、SVG 绘制 |
| [tool-pentest](工具/tool-pentest/开发文档.md) | **攻击渗透** | **privileged** | 渗透测试、漏洞验证 |

### 4.10 目标（6）

长期目标不是一次任务，是**跨会话存在的东西**。所以单独成域。

| 插件 | 能力 |
|---|---|
| [goal-store](目标/goal-store/开发文档.md) | 目标账本：层级、状态、里程碑、归档 |
| [goal-decompose](目标/goal-decompose/开发文档.md) | 目标 → 可执行步骤树，每步带验证条件 |
| [action-plan](目标/action-plan/开发文档.md) | 步骤 → 带时间的计划、关键路径、冲突检测 |
| [resource-plan](目标/resource-plan/开发文档.md) | 人/预算/工具/算力的分配与超配报出 |
| [progress-track](目标/progress-track/开发文档.md) | 采集实际进度，与计划对比出偏差 |
| [replan](目标/replan/开发文档.md) | 新情况触发重排，**人确认才生效** |

> **与 [文档/goal-table](文档/goal-table/开发文档.md) 的区别**：
> `goal-table` 是**当前任务**的目标表（Agent 自查用，任务结束即废）；
> `goal-store` 是**长期目标**的账本（跨会话、跨窗口、挂里程碑）。两者不重复。

### 4.11 操作（14）

对外做事。全部按 [操作/README.md](操作/README.md) 的两条路径拆分，**全部要过 [action-authorize](操作/action-authorize/开发文档.md)**。

| 插件 | 路径 | 能力 | 风险 |
|---|---|---|---|
| [action-authorize](操作/action-authorize/开发文档.md) | — | **授权闸门，本域前置** | elevated |
| [computer-use](操作/computer-use/开发文档.md) | 屏幕 | 鼠标键盘：点击/输入/滚动/拖拽 | elevated |
| [app-gui](操作/app-gui/开发文档.md) | 屏幕 | 原生桌面应用操作 | elevated |
| [browser-open](操作/browser-open/开发文档.md) | 屏幕 | 浏览器会话与标签页 | safe |
| [browser-act](操作/browser-act/开发文档.md) | 屏幕 | 网页填表/点击/上传/下载 | elevated |
| [web-search](操作/web-search/开发文档.md) | 接口 | 联网搜索，带出处 | safe |
| [web-read](操作/web-read/开发文档.md) | 接口 | 抓取清洗正文，保留出处 | safe |
| [mail-send](操作/mail-send/开发文档.md) | 接口 | 发邮件 | elevated |
| [calendar-mgr](操作/calendar-mgr/开发文档.md) | 接口 | 日历与提醒 | elevated |
| [doc-create](操作/doc-create/开发文档.md) | 接口 | 云端文档创建与共享 | elevated |
| [booking](操作/booking/开发文档.md) | 接口 | 订机票/酒店/餐厅 | **privileged** |
| [purchase](操作/purchase/开发文档.md) | 接口 | 在线购买 | **privileged** |
| [vendor-negotiate](操作/vendor-negotiate/开发文档.md) | 接口 | 代表用户与商家沟通议价 | **privileged** |
| [research-compose](操作/research-compose/开发文档.md) | 编排 | 主题研究并整理成有出处的结果 | safe |

### 4.12 记忆（17）

架构见 [记忆/README.md](记忆/README.md)。**自动浮现，不靠用户检索。**

| 分层 | 插件 | 存什么 |
|---|---|---|
| 窗口 | [mem-window](记忆/mem-window/开发文档.md) | 窗口内显著性、压缩触发点 |
| 档案 | [mem-profile](记忆/mem-profile/开发文档.md) | **用户明确告知**的偏好事实 |
| 短期 | [mem-onering](记忆/mem-onering/开发文档.md) + [mem-compress](记忆/mem-compress/开发文档.md) | OneRing：跨窗口事件线 |
| 长期 | [mem-timeline](记忆/mem-timeline/开发文档.md) + [mem-consolidate](记忆/mem-consolidate/开发文档.md) | VCPTimeLine：数年级阶段骨架 |
| 热 | [mem-diary](记忆/mem-diary/开发文档.md) | 经历、关系、反思（**自动注入**） |
| 冷 | [mem-coldbase](记忆/mem-coldbase/开发文档.md) | 百科、手册（**按需检索**） |

| 机制 | 插件 | 做什么 |
|---|---|---|
| 抽取 | [mem-extract](记忆/mem-extract/开发文档.md) | 从对话抽候选，带置信度，交用户确认 |
| 冲突 | [mem-reconcile](记忆/mem-reconcile/开发文档.md) | 矛盾时并存标记 + 询问，**不自动覆盖** |
| 联想 | [mem-taggraph](记忆/mem-taggraph/开发文档.md) | TagMemo 标签共现网络 + 脉冲传播 |
| 预取 | [mem-prefetch](记忆/mem-prefetch/开发文档.md) | 每轮算"此刻该知道什么" |
| 检索 | [mem-retrieve](记忆/mem-retrieve/开发文档.md) | TimeDecay → Rerank → Associate |
| 注入 | [mem-inject](记忆/mem-inject/开发文档.md) | `{{日记本}}` 占位符拦截与替换 |
| 路由 | [mem-thermal](记忆/mem-thermal/开发文档.md) | 热记忆自动 / 冷知识按需，平行运行 |
| 观测 | [mem-trace](记忆/mem-trace/开发文档.md) | 召回了什么、为什么、**有没有真被引用** |
| 控制 | [mem-console](记忆/mem-console/开发文档.md) | 查看/编辑/删除/导出，冲突处理 |

### 4.13 适配（19）

架构见 [适配/README.md](适配/README.md)。**机制归核心，家族适配归插件，裸核自带 passthrough。**

| 层 | 插件 | 做什么 |
|---|---|---|
| 协议 | [adapter-registry](适配/adapter-registry/开发文档.md) | 按 model id 选适配器，**选不中走兜底不阻断** |
| 协议 | [adapter-ir](适配/adapter-ir/开发文档.md) | 规范化中间表示，**全套适配的地基** |
| 协议 | [adapter-toolcall](适配/adapter-toolcall/开发文档.md) | 工具调用**配对语义**归一 —— 配错是静默错乱 |
| 协议 | [adapter-stream](适配/adapter-stream/开发文档.md) | 流式事件归一，**中断 ≠ 正常结束** |
| 协议 | [adapter-error](适配/adapter-error/开发文档.md) | 错误归一与重试，**支付类错误绝不重试** |
| 协议 | [adapter-token](适配/adapter-token/开发文档.md) | token 计数与窗口，**算错就是钱** |
| 提示词 | [prompt-compose](适配/prompt-compose/开发文档.md) | 最终提示词组装，**顺序是契约** |
| 提示词 | [prompt-family](适配/prompt-family/开发文档.md) | 家族适配机制与注册 |
| 提示词 | [prompt-claude](适配/prompt-claude/开发文档.md) | Claude 家族，**推理块原样回传** |
| 提示词 | [prompt-openai](适配/prompt-openai/开发文档.md) | OpenAI/GPT 家族，**兼容 ≠ 行为相同** |
| 提示词 | [prompt-passthrough](适配/prompt-passthrough/开发文档.md) | 原样兜底，**保证任何模型都能跑** |
| CLI | [target-resolve](适配/target-resolve/开发文档.md) | **走 API 还是 CLI**：显式 > profile > 粘性 > 自动匹配 > API |
| CLI | [cli-bridge](适配/cli-bridge/开发文档.md) | 外部 agent CLI 当子进程调，**CLI 会话 + 空闲自退，结果不截断** |
| CLI | [cli-sandbox](适配/cli-sandbox/开发文档.md) | 外部 CLI 隔离，**不可用则拒绝启动** |
| CLI | [relay-hub](适配/relay-hub/开发文档.md) | **第三方中转站名录与体检，替代 ccswitch** |
| CLI | [cli-profile](适配/cli-profile/开发文档.md) | **先检测注入情况，没用才注入**；环境变量隔离优先，改真配置是降级 |
| CLI | [cli-claude](适配/cli-claude/开发文档.md) | Claude Code CLI 对接，官方或中转注入 |
| CLI | [cli-generic](适配/cli-generic/开发文档.md) | 其他 CLI 的**声明式**接法 |
| 本地化 | [i18n](适配/i18n/开发文档.md) | 提示词（含思维链）、插件词条、UI 文案、知识库内容的多语言 |

## 5. 清单契约

每个插件都有 `开发文档.md`，内含一份可编译的清单：

```ts
export const manifest: PluginManifest = {
  id: "cursor-sprite",
  version: "0.1.0",
  description: "鼠标精灵：把上下文对象变成可拖的手柄",
  capabilities: [{ name: "sprite.bind", version: "0.1.0", inputSchema: {...}, outputSchema: {...} }],
  permissions: ["device"],
  requires: ["screen-capture", "context-chain"],   // 依赖序装载
  riskLevel: "safe",
  healthCheck: { command: "node -e \"require('./dist')\"", timeoutMs: 3000 },
};
```

## 6. 权限交集

**实际可用权限 = 工具声明 ∩ 整合包授权。** 插件多声明无效。

## 7. 打包

插件可独立打包为 zip 放入 [`../整合包/打包/`](../整合包/打包/README.md)。zip 里是一个插件的完整目录：`清单.ts` + `dist/` + `README.md`。
