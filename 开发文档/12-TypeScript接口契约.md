# 12 · TypeScript 接口契约汇总

> 全系统类型定义的**单一权威来源**。本文只汇总签名和引用关系，语义说明在各专题文档。
> 落盘时建议按模块拆分为 `src/types/*.ts`，保持与 [03 §8](03-装载器与a2a规范.md) 的依赖图一致。

## 0. 公共基础

```ts
/** 契约版本。所有跨进程/跨版本结构都必须带版本 */
type SchemaVersion = string;

interface SchemaRef {
  ref: string;          // "kb://frontend/component/button-primary@1.2.0"
  version?: string;
  hash?: string;        // 内容哈希，用于变更检测
}

interface ArtifactRef {
  id: string;
  type: "file" | "code" | "image" | "audio" | "video" | "doc" | "log" | "report";
  uri: string;          // 本地路径或制品存储地址
  sizeBytes?: number;
  mimeType?: string;
  summary?: string;     // 摘要。交接包只带引用和摘要
  createdAt: string;
}

type Permission =
  | "fs:read" | "fs:write" | "net" | "process"
  | "device" | "clipboard";

interface HealthCheckSpec {
  command?: string;
  timeoutMs: number;
  intervalMs?: number;
}

interface HealthReport {
  healthy: boolean;
  latencyMs?: number;
  detail?: string;
  checkedAt: string;
}

// ── 会话与语言 ──
// 这两个类型是核心留给插件的唯一接口。核心持有会话容器，不持有调度逻辑
// （跨会话派活是 [插件 session-hub](../插件/协作/session-hub/开发文档.md) 的事）。

/** 用户的对话线程。多数智能体系统叫 session */
interface SessionRef {
  sessionId: string;
  title?: string;
  createdAt: string;
  /** 挂着的整合包。会话内可切换，切换走 06 交接协议 */
  packageId: string;
  /** 会话级语言。不填继承应用级 */
  locale?: LocaleRef;
  /** 覆盖全局模型账号；不填继承 */
  accountOverride?: string;
  workspaceId: string;
}

/** BCP-47。内置 zh-Hans / en，其余 locale 的词条由 i18n 插件提供 */
type LocaleRef = string;
```

> **「会话」在这套文档里只有一个意思**：用户的对话线程（`SessionRef`）。
> 外部 agent CLI 那个进程级的常驻会话，类型名是 `CliSession`、中文一律叫**「CLI 会话」**，
> 两者的区别见 [§1.3.1](#131-cli-描述与调用)。浏览器会话、MCP 线程、网络出口保持
> 各自带限定词，不要混用。

## 1. 底座 · 伪装层（[02 §4](02-平台底座-后端底层架构.md)）

### 1.1 工具侧 · 转化器

```ts
interface Transformer {
  id: string;
  version: string;
  channels: string[];                                   // 多转化口
  outbound(input: StructuredData, channel: string): OutboundPayload;
  inbound(payload: OutboundPayload, channel: string): StructuredData;
}

interface StructuredData {
  kind: string;
  body: unknown;
  meta?: Record<string, unknown>;
}

interface OutboundPayload {
  channel: string;
  headers: Record<string, string>;
  body: unknown;
  /** 传输处理在此插入 */
  transforms?: Array<{ optimize: "ai" | "regex" }>;
}
```

### 1.2 模型侧 · IR 与家族适配

> 核心持有 IR 定义、`Adapter` 接口与 `prompt-passthrough` 兜底；家族适配是插件。
> 详见 [02 §4.2](02-平台底座-后端底层架构.md#42-模型侧家族适配) 与 [../插件/适配/README.md](../插件/适配/README.md)。

```ts
/** 规范化中间表示 —— 内外两套格式之间唯一的中间形态 */
interface Canonical {
  irVersion: string;                       // 跨版本转换要记入埋点
  model: {
    id: string;
    /** 适配器钉定的版本区间，超出即拒绝 */
    range?: { min: string; max: string };
  };
  system: ContentBlock[];
  messages: Message[];
  tools: ToolDef[];
  /** 提示词组装后的注入点，与 prompt-compose 对应 */
  injections: Injection[];
  /** 缓存断点。Anthropic 需显式标，OpenAI 系自动 */
  cache?: CacheMark[];
  maxOutputTokens?: number;
  /** 未知字段透传区 —— 适配器不得吞字段 */
  passthrough?: Record<string, unknown>;
}

type ContentBlock =
  | { type: "text"; text: string }
  | { type: "image"; source: ImageSource; detail?: "auto" | "low" | "high" }
  | { type: "document"; source: DocumentSource }
  | { type: "audio"; source: AudioSource }
  /** 推理块：必须原样回传，不得改写或摘要 */
  | { type: "reasoning"; raw: unknown; signature?: string }
  | { type: "tool_use"; id: string; name: string; input: unknown }
  /** 与 tool_use 配对。Anthropic 需紧邻，OpenAI 系是独立 message */
  | { type: "tool_result"; toolUseId: string; content: ContentBlock[]; isError?: boolean }
  | { type: "unknown"; raw: unknown };                 // 保底，不丢

interface Message {
  role: "user" | "assistant" | "system";
  content: ContentBlock[];
}

interface ToolDef {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
}

/** 双向、纯函数、无 IO */
interface Adapter {
  id: string;
  version: string;
  /** 覆盖范围。exact 优先于 range */
  models: Array<{ exact?: string[]; range?: { min: string; max: string } }>;

  toIR(raw: unknown, modelId: string): Canonical;        // 厂商格式 → IR
  fromIR(canonical: Canonical, modelId: string): unknown; // IR → 厂商格式

  supports(modelId: string): boolean;
  /** 往返转换丢失的字段名。往返后查这个 */
  lossy?: string[];
}

/** 提示词适配：在 Adapter 之上组装最终提示词 */
interface PromptAdapter {
  id: string;
  version: string;
  models: Array<{ exact?: string[]; range?: { min: string; max: string } }>;

  adapt(canonical: Canonical): unknown;
  tools(tools: ToolDef[]): unknown;
  cache(prompt: Canonical): CacheMark[];
  /** 推理块处理。不认得时原样透传，绝不丢 */
  reasoning(block: Extract<ContentBlock, { type: "reasoning" }>): unknown;
  supports(modelId: string): boolean;
}

interface CacheMark {
  blockIndex: number;
  kind: "ephemeral" | "persistent";
}

/** 适配器选择。选不中走 passthrough，永不阻断 */
/** 选择键是 model id，不含账号 —— 理由见下方「为什么适配器不看账号」 */
interface AdapterRegistry {
  register(adapter: Adapter | PromptAdapter): void;
  select(modelId: string): {
    adapter?: Adapter;
    prompt?: PromptAdapter;
    fallback: "hit" | "version-mismatch" | "miss";
  };
}
```

**为什么适配器不看账号**（这条容易在后来被当成漏改，所以写在这里）：

账号的 `kind: "official" | "relay"` 决定的是**怎么到达** —— 官方直连 API 就能到，中转要先把配置注入到外部 CLI 才到得了（见 [§1.3](#13-外部-agent-cli)）。它**不影响**按什么格式调。

曾经设计过「中转账号不许套家族适配器」，理由是中转可能不原样回传推理块、不认 `cache_control`、工具语义有偏差。**这个闸防不住任何东西**：中间层重写字节这个风险，对第三方中转、对平台自己的网关（每个请求都过它）、对任何中间代理**一视同仁**。按账号类型分流只会给人一个「中转已经被处理过了」的错觉。

防护只能放在两处：**网关透射性契约**（[02 §3.2](02-平台底座-后端底层架构.md)）和有损字段显式标出 + 埋点可审。

**契约硬约束**：

| 约束 | 理由 |
|---|---|
| 双向（`toIR` + `fromIR`） | 响应也要适配回来 |
| 纯函数、无 IO | 可单测，适配错误在测试里暴露而不是在生产里 |
| `type: "unknown"` 保底块 | **适配器不得吞字段** |
| `reasoning` 块原样回传 | 丢了不崩，只是推理静默消失 |
| `lossy` 显式声明有损字段 | 往返转换后能知道丢了什么 |
| 版本区间超出即拒绝 | 不硬用陈旧约定，否则质量静默下滑 |
| **不看账号类型** | 风险对所有中间层一视同仁，按账号分流是假防护（见上） |

### 1.3 外部 agent CLI

> 完整流程：**中转站是别人的 → 在程序中选模型 → 询问时检测注入情况（没用就注入）→ 用 CLI → 完成后返回**。

#### 1.3.1 CLI 描述与调用

```ts
/** 模型覆盖区间。复用 Adapter.models 的形状 */
type ModelRange = { exact?: string[]; range?: { min: string; max: string } };

interface CliSpec {
  id: string;                    // "claude" | "codex" | "gemini" | 自定义
  bin: string;
  /** 声明式。命令行差异是数据，不是逻辑 */
  build: {
    promptFlag: string;
    resumeFlag?: string;         // CLI 会话重建靠它
    jsonFlag?: string;
  };
  parse: (stdout: string) => CliResult;
  /** 方式 A 用的环境变量名，如 CLAUDE_CONFIG_DIR。不支持 A 时留空 */
  configDirEnv?: string;
  /** 方式 B 能否改用户真配置。CLI 会话下一律 false（见 cli-profile §6） */
  allowFilePatch: boolean;
  /** 空闲多久自退。0 = 永不（不推荐） */
  idleTimeoutMs?: number;
}

/** 结果一律全量。** 不截断 */
interface CliResult {
  /** 全量原文。哪怕被下游总结了，这份仍完整进埋点 */
  raw: string;
  bytes: number;
  sessionId?: string;
  /** 按 outputFormat 解析的结果。解析不了就只给 raw */
  parsed?: unknown;
  parseError?: string;
  /** 超长提示。** cli-bridge 自己不裁，只如实报出 */
  digest?: { reason: "too-long" | "requested"; bytes: number };
}

interface CliBridge {
  /** 一次性进程。先 ensure 再 spawn —— CLI 启动时读一次配置，运行中改它不读 */
  spawn(spec: CliSpec, args: string[], cwd: string, account: ModelAccount): Promise<Child>;

  // ── CLI 会话 ──
  openSession(spec: CliSpec, account: ModelAccount, opts?: SessionOpts): Promise<CliSession>;
  /** 一次询问。** 结果不截断**；outputFormat 是传进 CLI 的输入，不是拿到后再裁 */
  ask(session: CliSession, req: AskRequest): Promise<CliResult>;
  /** 记一次活跃，重置空闲计时 */
  touch(session: CliSession): void;
  closeSession(session: CliSession, reason: SessionEnd): Promise<void>;
  sessions(): CliSession[];

  feed(child: Child, prompt: string): void;
  collect(child: Child): CliResult;
  timeout(child: Child, ms: number): void;
  kill(child: Child): void;

  covers(): ModelRange[];        // 本 CLI 覆盖哪些 model id，供自动匹配用
  available(): boolean;          // 装了没
  accepts(account: ModelAccount): boolean;   // 这个账号能不能通过本 CLI 走到
}

interface SessionOpts {
  idleTimeoutMs?: number;        // 0 = 永不
  /** 重建时接上老 CLI 会话，拿到的是同一个 CLI 上下文 */
  resumeToken?: string;
  /** 来自整合包 profile.reportFormat */
  outputFormat?: string;
}

interface AskRequest {
  prompt: string;
  /** 格式要求。前置给 CLI 一次生成对，比拿回来再裁更准 */
  outputFormat?: string;
  workspace?: string;
}

/** 外部 agent CLI 子进程的一次 CLI 会话（进程级、常驻）。
 *  不是用户的对话线程 —— 那个是 SessionRef，中文叫「会话」。 */
interface CliSession {
  id: string;
  cli: string;
  /** CLI 会话生命周期内冻结。换绑定不动已开的 CLI 会话 */
  accountId: string;
  startedAt: string;
  lastActiveAt: string;
  resumeToken?: string;
  status: "running" | "idle" | "exiting" | "dead";
  /** 区分「超时退的」和「崩的」 */
  exitReason?: SessionEnd;
}

type SessionEnd = "idle-timeout" | "explicit" | "error" | "account-switch";

/** 外部程序必须隔离。隔离不可用时拒绝启动，不降级裸跑 */
interface CliSandbox {
  prepare(cwd: string, policy: SandboxPolicy): Promise<Workspace>;
  restrict(policy: SandboxPolicy): SandboxPolicy;
  /** CLI 会话期间持有的临时授权，CLI 会话结束或超时自退时回收 */
  grant(sessionId: string): Promise<void>;
  revoke(sessionId: string): Promise<void>;
  cleanup(sessionId: string): Promise<void>;
}

interface SandboxPolicy {
  readOnlyPaths: string[];
  writableRoots: string[];
  network: "off" | "egress-only" | "full";
}

interface Workspace { id: string; root: string; policy: SandboxPolicy }
interface Child { pid: number; cli: string; startedAt: string }
```

**CLI 会话为什么值得**：外部 CLI 的启动成本（拉进程、建 agent 循环、灌上下文）在多次询问里摊薄得很快。

**为什么必须有空闲自退**：用户问完就去做别的，进程不该一直占着工作区和配额。默认超时后优雅退出，`exitReason: "idle-timeout"`。这不是可选项 —— 没有自退的 CLI 会话等于常驻泄漏。

#### 1.3.2 注入检测（[cli-profile](../插件/适配/cli-profile/开发文档.md)）

**先检测，没用才注入。** 状态已经对的时候一个字节都不写。

```ts
/** 当前 CLI 实际会读到的那份配置的状态 */
type InjectionStatus =
  | { state: "absent";     seen: null }
  | { state: "matching";   seen: InjectionFingerprint }
  | { state: "mismatched"; seen: InjectionFingerprint; drift: string[] }
  | { state: "partial";    seen: InjectionFingerprint; missing: string[] };

/** 判据只取三项 —— 不需要读懂整个配置格式 */
interface InjectionFingerprint {
  baseUrl?: string;
  /** key 的 sha256。**存哈希不存明文**，这样指纹本身能进埋点、能事后审 */
  keyHash?: string;
  model?: string;
}

interface CliProfile {
  enabled(cli: string): boolean;      // 关 = 平台彻底不碰这个 CLI 的任何文件
  enable(cli: string): void;
  disable(cli: string): Promise<ReclaimReport>;

  bind(cli: string, accountId: string, by: BindingSource): void;   // 程序内设定
  unbind(cli: string): void;
  bindings(): CliBinding[];

  /** 只读，不动任何文件 */
  detect(spec: CliSpec, account: ModelAccount): Promise<InjectionStatus>;
  /** detect 说 matching 就直接返回，一个字节都不写 */
  ensure(spec: CliSpec, account: ModelAccount, opts?: { resident?: boolean }): Promise<PreparedRun>;
  release(run: PreparedRun): Promise<void>;
  recoverOrphans(): Promise<OrphanReport>;   // 上次异常退出的残留
}

interface CliBinding { cli: string; accountId: string; boundBy: BindingSource }
type BindingSource = "user" | "package";

interface PreparedRun {
  runId: string;
  /** isolated=A 隔离目录 / patched=B 改真配置 / passthrough=不注入，走它自己的 */
  mode: "isolated" | "patched" | "passthrough";
  /** 冻结。切换绑定不影响这次 —— 这就是「跑完为准」 */
  accountId: string;
  before: InjectionFingerprint;
  /** matching 时是 ["none"]。这一栏能证明「没动过」 */
  applied: Array<"none" | "created" | "overwritten" | "restored">;
  env?: EnvPatch;
  receipt?: PatchReceipt;
}

interface EnvPatch {
  /** 凭据用 burnInto 填，不直接持有明文 map */
  env: Record<string, SecretHandle>;
  disposable: boolean;             // 隔离目录用完即弃，不并进用户真配置
}

interface PatchReceipt {
  cli: string;
  path: string;
  backupPath: string;              // 固定路径，不是临时目录 —— 系统清理会带走临时目录
  before: string;                  // sha256，恢复后比对
  acquiredAt: string;
}

interface ReclaimReport { reclaimed: string[]; kept: string[]; reason?: string }
interface OrphanReport {
  orphans: Array<{ cli: string; path: string; backupPath: string; since: string }>;
}
```

**CLI 会话把方式 B 排除了**：`patched` 会让用户真配置在整个 CLI 会话期间停在别的站上，用户在这期间打开官方 CLI 看到的就是中转 —— 这是这类工具最难查的故障，而 CLI 会话把暴露窗口从一个任务拉长到一个 CLI 会话。所以：

```text
ensure(resident: true) + mode 想选 patched  → 降为 isolated 或 passthrough，并明确说明原因
```

降级必须说出口，不悄悄换。

#### 1.3.3 中转站（[relay-hub](../插件/适配/relay-hub/开发文档.md)）

> **「中转」在库里有两处，含义不同，不要混。**
> ① `SuiteEndpoint.route: "direct" | "relay"` —— **平台自己的转发节点**，见 [05 §6](05-云端套件双体结构.md)。
> ② 这里的 `ModelAccount.kind: "relay"` —— **第三方中转站**，见 [10 §2.3](10-工作台与项目管理.md)。
> 两者没有继承、没有交叉引用，第 ② 种是本节和 `cli-profile` 关心的那一种。

```ts
interface RelayStation {
  id: string;
  label: string;
  baseUrl: string;
  /** 站上暴露的模型名 → 平台侧模型名。中转站改过名的在这里对齐 */
  models: Array<{ alias: string; upstream: string; contextLimit?: number }>;
  credentialRefs: string[];       // 可能有几把 key
  health: "unknown" | "healthy" | "degraded" | "dead";
  lastProbedAt?: string;
}

/** 替代 ccswitch 的「名录」那一半。账号管「我现在用哪个」，站管「外面有哪些」 */
interface RelayHub {
  list(): RelayStation[];
  register(station: RelayStation): void;
  /** 探活 + 逐模型可调用性。**连通不等于能用** */
  probe(id: string): Promise<StationReport>;
  /** 站 + 模型 → 平台账号。账号是核心的，这里只做物化 */
  materialize(stationId: string, alias: string): ModelAccount;
  /** 反查：这个账号是从哪个站来的 */
  originOf(accountId: string): { stationId: string; alias: string } | null;
}

interface StationReport {
  stationId: string;
  reachable: boolean;
  latencyMs?: number;
  models: Array<{ alias: string; ok: boolean; reason?: string }>;
}
```

**为什么 `materialize` 而不是让站直接当账号用**：账号是核心能力，每个模式都要用；站是外部世界的信息，核心不该知道市面上有哪些。站可以全灭，账号机制照常。

#### 1.3.4 走 API 还是走 CLI

**核心只决定「哪个账号、哪个模型」**；走哪条通道由插件解析。

```ts
/** 优先级链：显式 > profile > 粘性 > 自动匹配 > API 兜底。
 *  显式永不被覆盖；自动匹配到 CLI 必须每次告知；
 *  covers() 匹配上但 accepts() 为 false 时回落 API —— 键名对得上不代表认证兼容。 */
interface TargetResolver {
  resolve(selection: ModelSelection): Target;
  explicit(modelId: string, transport: Transport): void;
  forget(modelId: string): void;                       // 粘性可清
  explain(target: Target): Reason;
}

type Transport = "api" | "cli";

interface Target {
  transport: Transport;
  accountId: string;
  modelId: string;
  cli?: { id: string; args: string[] };
  /** 决策链上谁做的决定 */
  decidedBy: "explicit" | "profile" | "sticky" | "auto-match" | "fallback";
  reason: string;                                       // 人可读
  /** 与目标并行执行（API 探索 + CLI 收敛），非二选一 */
  parallel?: Target[];
}

interface ModelSelection { accountId: string; modelId: string; taskKind: string }

interface Reason {
  step: string;                    // 停在哪一级
  considered: string[];            // 上游各级为什么没选中
  alternatives: Target[];          // 别的走法
}
```

**外部 CLI 也在模型路径上** —— 每次调用必须记入埋点，否则「模型路径可追踪」会漏掉这一段。

### 1.4 账号与凭据链

```ts
/** 账号是核心能力：所有模式都要用，所以不能做成插件 */
interface ModelAccount {
  id: string;
  label: string;                    // "主力-便宜" / "主力-强推理"
  provider: string;                 // 供应商或中转站 id
  /** 决定的是「怎么到达」，不是「按什么格式调」 */
  kind: "official" | "relay";
  /** 中转必须显式填。缺失时该账号不可用于 CLI 通道 */
  relayBaseUrl?: string;
  /** 结构化引用，不是裸 string。裸 string 会被误当路径或 URL 处理 */
  credentialRef?: CredentialRef;
  /** 凭据可能不止一把（主用 + 备用）。有备份时优先用第一把 */
  credentialRefs?: CredentialRef[];
  models: string[];
  capabilities?: SuiteCapability[]; // 部分账号兼作套件
  priority: number;
  limits?: { rpm?: number; tpm?: number; concurrency?: number };
  health: "unknown" | "healthy" | "degraded" | "disabled";
}
```

**`kind` 决定的是到达方式，不是适配策略** —— 官方直连 API 就到得了；中转要先把配置注入到外部 CLI，那份 CLI 才读得到（[§1.3.2](#132-注入检测cli-profile)）。它**不**决定套不套家族适配器，理由见 [§1.2](#12-模型侧-ir-与家族适配) 的「为什么适配器不看账号」。

`credentialRef` 是结构化引用，不是裸字符串。裸 `string` 会被下游误当路径或 URL 处理，且没有任何东西能证明它指向什么。

```ts
type CredentialRef = string & { readonly __brand: "CredentialRef" };

/** 不可字符串化。唯一出口是 use() 或 burnInto()，两者都可 grep */
interface SecretHandle {
  use<T>(fn: (secret: string) => T): T;
  /** 供 child_process.env、EnvPatch 这类必须拿到 Record<string,string> 的窄口 */
  burnInto(map: Record<string, string>, key: string): void;
  toString(): "[redacted]";
  toJSON(): undefined;             // JSON.stringify 打不进日志
}

/** 注入要用凭据，detect 要比对是哪一把 key —— 所以凭据链是注入链的底座，不是可选项 */
interface CredentialStore {
  put(secret: string, meta: { label: string; accountId: string }): Promise<CredentialRef>;
  resolve(ref: CredentialRef): Promise<SecretHandle>;   // 取不到 → 抛
  health(): Promise<{ available: boolean; backend: string; reason?: string }>;
  rotate(ref: CredentialRef): Promise<void>;
  revoke(ref: CredentialRef): Promise<void>;
  list(): Promise<Array<{ ref: CredentialRef; label: string; accountId: string }>>;
}
```

**TypeScript 拦不住字符串被复制**，目标是让泄漏路径**窄且可 grep**，不是绝对禁止：`toJSON` 返回 `undefined`（序列化打不进日志）、`toString` 返回 `[redacted]`（模板字符串打不进日志）、只有两条显式出口。

**取不到凭据不拒启整个系统**。底座的验收是「空配置能起来」，硬拒启会打破它。准确说法是**拒绝进入可运行（模型可用）状态**：

```text
resolve() 抛
  → AccountManager 把该账号标 degraded
  → ModelSelectionPolicy 按 fallback 换号
  → 全部不可用时该次调用失败，但不需要模型的工具照常跑
```

**`revoke` 对已经拉起的进程完全无效** —— 凭据早在 spawn 时就烧进环境变量了。这是 CLI 会话的一个真实缺口：`revoke` 之后，老 CLI 会话还能继续用那把 key 直到它自己退出。处置是明确的，不是假装能撤：**告诉人这把 key 要等 CLI 会话退干净才真正失效**，必要时由人手动终止 CLI 会话。

## 2. 底座 · agent 层装载器（[03](03-装载器与a2a规范.md)）

```ts
type LoaderKind = "skill" | "tool" | "mcp" | "workflow" | "file" | "input" | "a2a";

interface Loader<TManifest> {
  readonly kind: LoaderKind;
  load(manifest: TManifest, ctx: LoadContext): Promise<LoadResult>;
  healthCheck(handleId: string): Promise<HealthReport>;
  unload(handleId: string): Promise<void>;
}

interface LoadContext {
  runtimeVersion: string;
  packageId?: string;      // 当前整合包。装载器不解释它
  permissions: Set<Permission>;
  resolve(loaderKind: LoaderKind, ref: string): Promise<string>;   // 依赖解析
  registerDegradation(id: string, spec: DegradationSpec): void;
}

interface LoadResult {
  ok: boolean;
  handleId?: string;
  degraded?: boolean;
  error?: { code: string; message: string; retryable: boolean };
}

// 2.1 skill
interface SkillManifest {
  id: string; version: string; description: string;
  triggers: { packageIds?: string[]; states?: string[]; intentMatch?: string[] };
  requires: { tools?: string[]; mcpServers?: string[]; skills?: string[] };
  promptFragments?: string[];
  artifacts?: ArtifactRef[];
}

// 2.2 工具
type ToolCategory = "write" | "compute" | "model" | "draw" | "pentest";
type RiskLevel = "safe" | "elevated" | "privileged";

interface ToolManifest {
  id: string; version: string; description: string;
  category: ToolCategory;
  riskLevel: RiskLevel;
  entry: { kind: "inproc" | "process"; command: string; args?: string[] };
  ioSchema: { input: SchemaRef; output: SchemaRef };
  permissions: Permission[];
  requires?: string[];
  healthCheck?: HealthCheckSpec;
  office?: { app: "word" | "excel" | "ppt" | "outlook"; mode: "com" | "cli" };
}

// 2.3 MCP
interface McpServerManifest {
  id: string;
  transport: "stdio" | "sse" | "http";
  command?: string; args?: string[]; url?: string; env?: Record<string, string>;
  retrieval: { targetRecall: number; topK: number; rerank?: string };
  threading: { maxConcurrent: number; perSession?: boolean; idleTimeoutMs: number };
}

/** MCP server 线程。sessionId 指线程归属的会话（用户对话线程），不是 CLI 会话 */
interface McpThread { threadId: string; serverId: string; sessionId: string; acquiredAt: string; }
interface McpObservation { serverId: string; query: string; recalled: string[]; hit: boolean; at: string; }

// 2.4 工作流
interface WorkflowManifest {
  id: string; version: string;
  autoTrigger: { intents: string[]; conditions?: ConditionExpr[]; cooldownMs?: number };
  states: string[];
  transitions: Array<{ from: string; on: string; to: string }>;
  hooks: Array<{
    at: { state: string; phase: "before" | "after" };
    validator: string;
    selfImprove?: boolean;
  }>;
  onFailure: "abort" | "retry" | "degrade" | "handoff";
}

// 2.5 文件管理
type VcsStrategy = "none-to-git" | "existing-git" | "worktree";
interface FileManager {
  read(path: string, encoding?: string): Promise<FileContent>;
  write(path: string, content: string, opts?: WriteOptions): Promise<WriteResult>;
  pack(members: string[], opts?: PackOptions): Promise<PackageRef>;
  unpack(ref: PackageRef, dest: string): Promise<void>;
  toVCS(strategy: VcsStrategy): Promise<VcsBinding>;
}

// 2.6 输入转化
type InputKind = "text" | "voice" | "image" | "file" | "clipboard" | "screenshot" | "keypad";

interface InputTransformer {
  id: string;
  from: InputKind;
  to: StructuredInput;
  transform(raw: unknown, ctx: TransformContext): Promise<StructuredInput>;
}

interface StructuredInput {
  kind: string;
  text?: string;
  raw?: string;                     // 清洗前保留
  normalized?: string;              // 清洗后
  appliedRules?: string[];
  ambiguous?: Array<{ term: string; candidates: string[] }>;
  attachments: ArtifactRef[];
  viewport?: { screen: { w: number; h: number }; region?: { x: number; y: number; w: number; h: number } };
  confidence?: number;
}

// 2.7 A2A
interface AgentRef { kind: "agent" | "package" | "plugin" | "suite"; id: string; version: string; }

interface A2AEnvelope {
  protocolVersion: string;
  messageId: string;
  threadId: string;
  from: AgentRef;
  to: AgentRef;
  intent: string;
  payload: unknown;
  handoff?: HandoffEnvelope;
  signature?: string;
}
```

## 3. 三大核心对象（[04](04-三大核心对象.md)）

```ts
// 3.1 套件
interface SuiteCapability {
  name: string;                     // "kb.search" | "code.generate" | "build.remote"
  version: string;
  cost?: { unit: string; hint: string };
  sla?: { p95Ms: number; availability: number };
  inputSchema: SchemaRef;
  outputSchema: SchemaRef;
}

interface SuiteEndpoint {
  id: string;
  baseUrl: string;
  route: "direct" | "relay" | "auto";
  protocol: "https" | "http-internal";
  healthPath: string;
  timeoutMs: number;
}

interface DegradationSpec {
  offline: {
    available: string[];
    unavailable: string[];
    localFallback?: string;
    userNotice: string;             // 绝不静默失败
  };
}

interface SuiteManifest {
  id: string; name: string; version: string;
  capabilities: SuiteCapability[];
  localPlugin: PluginRef;           // 本地侧：可视化嵌入
  endpoints: SuiteEndpoint[];       // 云端侧：提供数据
  auth: { scheme: "bearer" | "mtls" | "local" | "none"; ref: string };
  degradation: DegradationSpec;
}

// 3.2 插件
interface PluginCapability {
  name: string;
  version: string;
  inputSchema: SchemaRef;
  outputSchema: SchemaRef;
}

interface PluginManifest {
  id: string; version: string; description: string;
  capabilities: PluginCapability[];
  permissions: Permission[];
  requires?: string[];
  healthCheck?: HealthCheckSpec;
  ownedBySuite?: string;           // 套件可视化插件时填写
}

// 3.3 整合包
type HandoffTrigger = "manual" | "完成条件" | "建议交接";
type PrepareKind = "language" | "framework" | "promptRules" | "tools" | "reportFormat" | "taskSplit" | "validation";

interface PackageTransition {
  targetPackageId: string;
  trigger: HandoffTrigger;
  requiredFacts: string[];
  handoffMapping: Record<string, string>;
  prepare?: Array<{ kind: PrepareKind; value: unknown }>;
}

interface IntegrationPackageManifest {
  id: string; name: string; version: string;
  profile: {
    language?: string; framework?: string; platform?: string;
    promptPolicy: string; reportFormat: string;
  };
  plugins: string[];
  suites: string[];
  states: string[];
  transitions: PackageTransition[];
}

// 套件可视化插件
interface SuiteVisualPluginManifest extends PluginManifest {
  ownedBySuite: string;
  view: {
    surface: "panel" | "inline" | "modal" | "tab" | "overlay" | "none";
    exposes: string[];
    mode: "read" | "read-write";
    offlineView?: string;
  };
  client: {
    endpointId: string;
    retry: { max: number; backoffMs: number };
    cache: { enabled: boolean; ttlMs: number };
  };
}
```

## 4. 交接协议（[06](06-模式衔接与交接协议.md)）

```ts
interface HandoffEnvelope {
  schemaVersion: SchemaVersion;
  handoffId: string;
  createdAt: string;

  source: { packageId: string; sessionId: string; state: string };
  target: { packageId: string; entryState?: string };

  project: { projectId: string; workspacePath?: string; branch?: string; checkpoint?: string };

  task: {
    objective: string;
    constraints: string[];
    decisions: string[];
    openQuestions: string[];
    nextActions: string[];
  };

  history: { conversationRef?: string; eventRef?: string; artifactRefs: string[] };

  runtime: {
    /** 开发语言。指编程语言，不是 i18n 的人读语言 —— 那个叫 locale */
    language?: string; framework?: string; platform?: string;
    pluginIds: string[]; suiteIds: string[];
    /** 会话级语言与账号。交接不改变它们，所以必须一起传，
     *  否则会话内切整合包会把这会话的语言和账号掉回全局 */
    locale?: LocaleRef;
    accountId?: string;
  };

  output: { requestedFormat?: string; reportFormat?: string };

  overrides: HandoffOverride[];
  prepared: PreparedSpec[];
  completeness: {
    required: string[]; satisfied: string[]; missing: string[];
    acceptedWithGaps: boolean;
  };
}

interface HandoffOverride {
  field: string;
  from: unknown;
  to: unknown;
  reason: "project" | "task" | "user" | "detected";
  by: string;
  at: string;
}

interface PreparedSpec { kind: PrepareKind; value: unknown; appliedBy: string; }

interface HandoffEvent {
  eventId: string; at: string;
  envelope: HandoffEnvelope;
  outcome: "success" | "degraded" | "failed";
  rolledBackTo?: string;
}
```

## 5. 知识库（[07](07-前端UI知识库.md) / [08](08-后端工程知识库.md)）

```ts
// 5.1 前端
interface TokenRef { token: string; fallback?: string; }
interface TokenTable { [name: string]: string | TokenRef; }

interface UiProp { name: string; tsType: string; required: boolean; default?: unknown; oneOf?: unknown[]; description?: string; }

interface UiComponent {
  id: string; name: string;
  category: "layout" | "form" | "display" | "feedback" | "navigation" | "overlay" | "data";
  version: string;
  props: UiProp[];
  style: {
    color?: TokenRef; fill?: TokenRef; spacing?: TokenRef; typography?: TokenRef;
    border?: { width?: TokenRef; color?: TokenRef; style?: "solid" | "dashed" | "dotted" | "none"; radius?: TokenRef };
  };
  composition?: { children?: string[]; slots?: Array<{ name: string; accepts: string[]; required: boolean }> };
  platforms: Array<{
    platform: "web" | "desktop" | "mobile";
    framework: "react";
    implementation: ArtifactRef;
    props?: Partial<UiProp>;
  }>;
  interactions: Array<{ trigger: string; state: string; action: string; nextState: string }>;
  mock?: SchemaRef;
  assets?: ArtifactRef[];
  tags: string[];
  source?: { repo?: string; path?: string; license?: string };
  updatedAt: string;
}

interface NodeSpec {
  component: string;
  props?: Record<string, unknown>;
  style?: Record<string, TokenRef | string>;
  bindings?: Array<{ prop: string; to: string }>;
  interactions?: Array<{ trigger: string; action: string }>;
  children?: NodeSpec[];
}

interface PageSpec {
  id: string;
  platform: "web" | "desktop" | "mobile";
  framework: "react";
  root: NodeSpec;
  dataSources: SchemaRef[];
  tokens: { primitive: TokenTable; semantic: TokenTable };
}

interface ComponentQuery {
  text?: string; tags?: string[];
  category?: UiComponent["category"];
  platform?: PageSpec["platform"];
  visualHints?: ArtifactRef[];
  extracted?: { color?: string; layout?: string; density?: string };
  limit?: number;
}

// 5.2 后端
/** H1-H5 = 知识库等级标注，不是交付阶段 */
type KnowledgeLevel = "H1" | "H2" | "H3" | "H4" | "H5";

interface ApiTemplate {
  method: "GET" | "POST" | "PUT" | "PATCH" | "DELETE";
  path: string;
  auth: "none" | "session" | "token" | "mtls";
  request?: SchemaRef;
  response: SchemaRef;
  errors: Array<{ code: string; httpStatus: number; when: string }>;
  idempotent: boolean;
  timeoutMs?: number;
}

interface MutabilitySpec {
  fields: Array<{ path: string; mutable: boolean; after?: "create" | "update" | "never" }>;
  versioning: "none" | "optimistic" | "append-only" | "sourceless";
  freeze?: Array<{ when: string; frozen: string[] }>;
}

interface ServiceTemplate {
  id: string; name: string;
  level: KnowledgeLevel;                     // 等级标注
  version: string;
  language: "java" | "go" | "rust" | "python" | "typescript";
  service: {
    port: number; dependencies: string[];
    healthCheck: { path: string; expect: number };
    config: Array<{ key: string; required: boolean; secret: boolean; default?: string }>;
  };
  apis: ApiTemplate[];
  models: DataModel[];
  seeds: SeedObject[];
  hooks: HookSpec[];
  mocks: MockSpec[];
  deployment: DeploymentSpec;
  observability: ObservabilitySpec;
  constraints: MutabilitySpec;
  tags: string[];
  source?: { repo?: string; path?: string; license?: string };
  updatedAt: string;
}

interface TemplateQuery {
  language?: string;
  serviceType?: string;
  level: KnowledgeLevel;
  allowDowngrade?: boolean;      // 向下取用需显式开启。向上永远禁止
  tags?: string[];
}

interface LanguageAdapter {
  id: string;
  language: ServiceTemplate["language"];
  materialize(template: ServiceTemplate, ctx: ProjectContext): ArtifactSet;
  detect(workspace: string): Promise<{ language: string; framework?: string; confidence: number }>;
}

interface ArtifactSet {
  files: Array<{ path: string; content: string; language: string }>;
  commands: Array<{ label: string; command: string; description: string }>;
  validation: Array<{ step: string; command: string; expect: "exit0" | RegExp }>;
}
```

## 6. AI 能力（[09](09-AI能力蓝图.md)）

```ts
interface Task {
  id: string;
  objective: string;
  constraints?: string[];
  dependsOn?: string[];
  priority?: number;
  inputs?: ArtifactRef[];
}

interface DispatchDecision {
  strategy: "single" | "parallel" | "debate" | "hierarchical";
  units: DispatchUnit[];
  rationale: string;
}

interface DispatchUnit {
  id: string;
  objective: string;
  dependsOn: string[];
  agentCount: number;                    // 1-3~5 动态决策
  mode: "execute" | "debate-then-decide";  // 多智能体取一测试
  workspace: string;                     // 新建空间
}

interface DecisionPoint { id: string; at: string; summary: string; }
interface TimelineNode { id: string; at: string; kind: "step" | "branch" | "merge" | "revert"; ref: string; }
interface Branch { branchId: string; forkedFrom: string; isolatedWorkspace: string; }

interface CollaborationRun {
  runId: string;
  checkpoints: DecisionPoint[];          // 时间回溯点
  timeline: TimelineNode[];
  rewindTo(pointId: string, reason: string): Branch;
}

interface ContextEntry { kind: "input" | "retrieval" | "decision" | "output" | "error" | "edit"; body: string; ref?: string; }
interface ContextNode { id: string; parentId?: string; entry: ContextEntry; at: string; }

interface ContextStore {
  append(entry: ContextEntry): ContextNode;              // 链式
  share(nodeIds: string[], groupId: string): void;         // 编组共享
  search(query: ContextQuery): ContextNode[];             // 动态搜索
  ask(nodeIds: string[], question: string): Promise<string>;  // 不打断
}

interface LivingDocument {
  id: string;
  type: "plan" | "spec" | "task-log" | "goal-table" | "decision-log";
  content: string;
  version: number; updatedAt: string;
  askable: true;
  revisions: Array<{ by: "human" | "agent"; kind: "edit" | "comment" | "accept" | "reject"; ref: string; at: string }>;
  sources: Array<{ agentId: string; ref: string }>;
}

interface GoalTable {
  goals: Array<{
    id: string; statement: string;
    status: "pending" | "in-progress" | "blocked" | "done";
    evidence?: string;                 // Agent 自查必须有证据
    checkedBy: "agent" | "human"; checkedAt?: string;
  }>;
}

interface TaskExecution {
  method: "TDD" | "IDD";
  workspace: { mode: "git-worktree" | "git-branch" | "sandbox" | "in-place"; path: string; onComplete: "commit" | "merge" | "discard" | "checkpoint" };
  apis?: string[];
  outputFormat?: string;
  deliveryCheck: DeliveryCheck;
}

interface DeliveryCheck {
  typeCheck: boolean; unitTest: boolean; build: boolean;
  auditTrail: boolean;      // 无条件要求
  rollbackReady: boolean;   // 无条件要求
}

interface CursorSprite {
  bound: ArtifactRef | ContextNode | string;
  onDrag(target: DropTarget): unknown;
  render: "ghost" | "preview" | "live";
}

interface HumanInteraction {
  pick(region: { x: number; y: number; w: number; h: number }): ArtifactRef;
  preview(ref: ArtifactRef): unknown;
  drop(ref: ArtifactRef, target: DropTarget): void;
  explain(ref: ArtifactRef): string;
}
```

## 7. 工作台与项目管理（[10](10-工作台与项目管理.md)）

```ts
interface TaskRecord {
  id: string; objective: string;
  dependsOn: string[]; priority: number;
  status: "waiting" | "ready" | "running" | "done" | "failed" | "blocked";
  independent: boolean;
  workspace: string;
  startedAt?: string; finishedAt?: string;
  result?: TaskResult;
}

interface TaskPool {
  tasks: TaskRecord[];
  admit(task: Task): void;
}
```

**三段式汇报**（[11 §2](11-设计理念与硬性约束.md)）：

```ts
interface ReportFormat {
  id: string;
  sections: Array<{
    key: "completed" | "issues" | "decisions" | "nextSteps" | "evidence";
    title: string;
    required: boolean;
  }>;
  requireEvidence?: boolean;
  outputFormat: "markdown" | "json" | "html" | "cli" | "voice";
}

interface Report {
  completed: string[];
  issues: string[];
  decisions: Array<{ question: string; options?: string[]; recommendation?: string; blocking: boolean }>;
  nextSteps?: string[];
  evidence?: Array<{ claim: string; ref: string }>;
  formatId: string;
  at: string;
}
```

**方案勾选 → 自动新建子智能体**（[11 §3.1](11-设计理念与硬性约束.md)）：

```ts
interface PlanApproval {
  checks: Array<{ id: string; label: string; checked: boolean }>;
  onApproved: "spawn-subagent";
}
```

**流式恢复**（[11 §3.2](11-设计理念与硬性约束.md)）：

```ts
interface StreamRecovery {
  mode: "discard-and-restate";   // 唯一允许。禁止 resend-partial / append-continue
}
```

## 8. 会话与多语言（[插件 session-hub](../插件/协作/session-hub/开发文档.md) / [插件 i18n](../插件/适配/i18n/开发文档.md)）

> 这一节的两块能力**都是插件**。核心只留 `SessionRef` 和 `LocaleRef` 两个类型（见 [§0](#0-公共基础)），
> 容器本身在 `core/`，跨会话调度和多语言都不在核心里。拿掉任一插件，系统照常跑：
> 拿掉 `session-hub` 退回单会话；拿掉 `i18n` 全库回退源文（＝不翻译）。

### 8.1 会话：容器与切换

`SessionRef` 的定义在 [§0](#0-公共基础)。这里只说三件容易搞错的事。

**一、整合包挂在会话上，不在应用上。** 同一个会话可以切换着挂不同的整合包，**切换走 06 的交接协议**
（时空衔接），不是热替换。所以「换模式」和「换会话」是两个动作：换模式 → 同一个会话内交接；
换会话 → 另一个 `sessionId`，各自挂各自的整合包。

**二、会话级语言与账号可以覆盖全局。** `locale` 和 `accountOverride` 不填就继承应用级。
覆盖是**会话级**的，不是全局的 —— A 会话切到英文账号，不影响 B 会话还在用哪个账号。

**三、「会话」只有一个意思。** 用户的对话线程。外部 agent CLI 那个进程级的常驻会话是 `CliSession`、
中文叫「CLI 会话」，两者不能混。浏览器会话、MCP 线程、网络出口保持各自带限定词。

```ts
interface SessionStatus {
  sessionId: string;
  packageId: string;
  state: "idle" | "running" | "waiting" | "closed";
  locale: LocaleRef;
  workspaceId: string;
  /** 派往本会话、未完成的工作单 */
  inbound: DispatchTicket[];
  /** 本会话派出去、未回传的工作单 */
  outbound: DispatchTicket[];
}

/** 核心唯一需要知道"当前是哪个会话"的地方。
 *  session-hub 被禁用时返回 null —— 核心照常跑，这是「拿掉它还能启动」的落点。 */
interface RuntimeContext {
  currentSession(): SessionRef | null;
  currentTrace(): string | null;
}
```

### 8.2 跨会话派活

```ts
// —— 派活 ——
parse(intent: string, from: string): Promise<DispatchProposal>;   // 模型解析成结构化，不直接执行
dispatch(proposal: DispatchProposal): Promise<DispatchTicket>;
chain(ticketId: string): Promise<DispatchTicket[]>;               // 链式
cancel(ticketId: string): Promise<void>;
tickets(): DispatchTicket[];

// —— 读：完全互通 ——
history(sessionId: string, range?: { from?: number; to?: number }): Promise<Message[]>;
artifacts(sessionId: string): Promise<ArtifactRef[]>;
status(sessionId: string): Promise<SessionStatus>;

// —— 回传 ——
setReturnPolicy(sessionId: string, policy: "on-completion" | "never"): void;
onReturn(cb: (from: string, result: DispatchResult) => void): void;
```

```ts
interface DispatchProposal {
  toSessionId?: string;      // 不给 = 按 spawn 新建
  spawn?: { title: string; packageId?: string; workspaceId?: string; locale?: LocaleRef };
  task: string;
  readOnly: boolean;         // 跨工作区时：true 读免确认，false 走授权闸门
  maxDepth: number;          // 还能往下派几层，默认 0
  /** 本次派活已经经过的会话链，防环用。maxDepth 挡不住 A→B→A→B→A */
  ancestorSessions: string[];
  rationale: string;         // 必填
}

interface DispatchTicket {
  ticketId: string;
  fromSessionId: string;
  toSessionId: string;
  parentTicketId?: string;   // 链式
  depth: number;             // 上限 3
  /** 本次派活已经经过的会话链，防环。两条都要查：会话级 + trace 级 */
  ancestorSessions: string[];
  ancestorTraceIds: string[];
  traceId: string;           // 派活方 A 侧的 trace
  parentTraceId: string;     // 承接方 B 侧的 trace
  status: "pending" | "running" | "done" | "failed" | "cancelled" | "rejected";
  returnPolicy: "on-completion" | "never";
  result?: DispatchResult;
}

interface DispatchResult {
  conclusion: string;        // 结论
  artifacts: ArtifactRef[];  // 产出引用，不是全文
  sessionId: string;         // 想看过程自己 read
  /** 回传没送出去时必须说出口。关着回传不等于静默失败 */
  undelivered?: { reason: "return-off" | "session-gone" | "declined"; note: string };
}
```

**`parse` 和 `dispatch` 分开不是冗余，是防模型被诱导。** 模型只能产出 `DispatchProposal`，
中间过一次 `SchemaRef` 校验和用户确认；模型**不能**直接调 `dispatch`。

**防环查两个维度，只查一个会被另一种绕过。** `ancestorSessions` 查会话级（A→B→A），
`ancestorTraceIds` 查 trace 级。只查深度不够 —— `maxDepth: 3` 之内 A→B→A→B→A 完全合法，
只会把预算用光。另外 `dispatch()` 校验 `to ≠ from`，且**只有链的末端会话能继续派**。

**回传关了要说出口。** `returnPolicy: "never"` 时 B 照常干活，但 `DispatchResult.undelivered`
必须填 —— 「B 干完了但 A 不知道」是静默失败，直接违反 [11 §4](11-设计理念与硬性约束.md)「降级可用 > 一步到位」里的"降级要可见"。

**跨会话不串 span。** `Span.parentTraceId` 指向 B 的 trace，但 B 的 span 不进 A 的 trace ——
不然 A 的上下文会被 B 的过程污染。B 侧的模型调用**照样全量记录**，「模型路径可追踪」这条不让，
不让的是「把两个会话的 span 合成一棵树」这个做法，不是记录本身。见 [10 §2.4.4](10-工作台与项目管理.md)。

**「完全互通」是读权限，不是回传内容。** 回传只带结论 + 产出引用；要过程 A 自己 `history()` 拉。
互通是**对话层**的，不是文件系统层 —— A 想读 B 工作区里的文件，走 A 自己的 `fs:read` 权限。
凭据本来就不在对话内容里（`SecretHandle` 只有 `use()` / `burnInto()` 两条出口），
所以「凭据不可跨会话读」是既有事实，不是新增拦截。

**`Permission` 不加新成员。** 跨会话读是用户选定的完全互通，不做成整合包能拒绝的权限；
跨工作区的**写**走 [action-authorize](../插件/操作/action-authorize/开发文档.md) 的 `policy("cross-workspace-write")`
这个 domain，不新开一套闸门。

### 8.3 多语言

```ts
// 词条：两个来源，统一落到 i18n 目录下
register(source: string, entries: Record<string, Record<LocaleRef, string>>): void;
t(key: string, locale: LocaleRef, fallbackChain?: LocaleRef[]): string;
declareFallback(pluginId: string, chain: LocaleRef[]): void;
coverage(): LocaleCoverage[];

// 提示词：每个 locale 一份人工翻译，运行时不做机器翻译
prompt(baseKey: string, locale: LocaleRef): string;
missing(baseKey: string, locale: LocaleRef): string[];
assertTranslated(baseKey: string, locale: LocaleRef): void;
```

**提示词不做运行时机器翻译。** 每个 locale 一份人工译文，思维链语言靠**译文里写死的指令**
保证，不是靠运行时拼一句「请用 X 语言思考」。

代价是漏译的 locale 思维链会退回源语言，处置是 `assertTranslated()` 报出缺口 —— **报缺口，不假装**。

**思维链语言只能验「指令发没发」，不能验「思维链是什么语言」。** 验收写第二条会得到一条永远
过不了的检查项。模型的推理内容（`ContentBlock.reasoning`）**必须原样回传、不得改写或摘要**
（[§1.2](#12-模型侧-ir-与家族适配)），所以 i18n 只能产出提示词指令，**模型是否照做不可保证**。观测到的语言
记进埋点，**不作保证**。

**回退链由插件自定。** 缺译文时走 `declareFallback()` 声明的顺序，最后回源文；
没声明就走「locale → 基础语言 → 源文」。`coverage()` 给出全库缺口。

**界面语言跟会话走会出事，所以它不跟。** 三层语言分得清清楚楚：

| | 管什么 | 归谁 | 初始值 |
|---|---|---|---|
| `SessionRef.locale` | **模型输出、思维链指令、插件提示、汇报标题** —— 用户看的是 Agent 生成的东西 | 会话级 | 创建会话时继承 `uiLocale`，之后各自独立 |
| 应用 `uiLocale` | **界面外壳**：按钮、菜单、设置面板 | 应用级 | 浏览器第一语言 / 系统 locale / 用户设置 |
| 词条 key | 两者共用一套表 | `i18n` 插件 | — |

理由：工作台支持**多个会话并排**（见 [10 §6](10-工作台与项目管理.md)）。如果界面语言跟会话走，
同一屏会出现一个中文标题配一个英文按钮，而且**用户在会话内改不了界面语言** —— 那是死路。

改会话语言**只改这会话的模型输出**，不动 `uiLocale`。两者不一致时不告警（用户可能就想要英文报告配中文界面），但在 `coverage()` 里记一笔。

**三处不要混**（混了会改错东西）：

| | 管什么 | 归谁 |
|---|---|---|
| i18n | **人读文本**的语言 | 本插件 |
| `LanguageAdapter`（[§5](#5-知识库07-08)） | **代码**的语言：把服务模板物化成 Go / Rust / Java | 知识库，不归 i18n |
| `runtime.language`（[§4](#4-交接协议06)） | **开发语言**（TypeScript / Go） | 整合包 profile，不是 i18n |

`07` / `08` 已声明「这是知识库，不是本 Agent 系统的组成部分」，且知识库只被云端套件读取
（见 [04 §5](04-三大核心对象.md)）—— 所以 i18n 对它们只能**在装载期给内容附加译文**，
不接管它们的组织与分级，运行时读知识库仍走云端套件。

公文式三段式汇报的 `ReportFormat.sections[].title` 写死了「允许本地化」
（[11 §2.2](11-设计理念与硬性约束.md)），是现成的落点。**三段存不存在看 `key` 不看 `title`**，
`i18n` 缺失时 `title` 回落源语言字面量，不空掉。

## 9. 建议的源码布局

```text
src/
  types/          ← 本文件的类型，按 §1-§8 拆分
  core/           运行时、事件总线、会话容器、项目状态
  loaders/        7 个装载器：skill / tool / mcp / workflow / file / input / a2a
  net/            网络层：gateway / proxy / ipPool / cloud / transport
  transform/      伪装层：转化器注册表与多转化口
  registry/       套件 / 插件 / 整合包 注册表
  packages/       整合包注册表、状态机、热加载
  handoff/        交接协议、交接事件、上下文整理
  context/        上下文链式存储、编组、动态搜索
  dispatch/       任务分发、协作引擎、时间回溯
  knowledge/      知识库检索客户端、页面 DSL、模板物化
  workbench/      项目制 / 对话式 / 助手式三种形态
  accounts/       账号（多模型，替代 ccswitch）
  tracing/        全程日志埋点。Trace 树，跨会话靠 Span.parentTraceId 串
  reporting/      公文式三段式汇报
  adapters/       模型、存储、网络、语言适配器
  i18n/           词条与 locale 目录。插件自带与集中持有解包后都归到这里
```

`core/` 只持有会话**容器**（对话循环、消息、工作区绑定、`RuntimeContext.currentSession()`）；跨会话调度（`SessionStatus` 里的 `inbound` / `outbound`、`DispatchTicket` 的流转）不写在这里，它属于 `session-hub` 插件。

`currentSession()` 返回 `SessionRef | null` —— **`session-hub` 被禁用时它是 `null`，核心照常跑**。这是「拿掉 session-hub 系统还能启动」在类型层面的落点。

`i18n/` 目录由插件在装载时填，核心不预置任何词条。

装载依赖顺序见 [03 §8](03-装载器与a2a规范.md)。
