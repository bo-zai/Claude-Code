# Claude Code 上下文管理实现深度分析报告

## 一、核心概念与配置参数

### 1.1 上下文窗口定义

Claude Code的上下文窗口管理以Token为核心计量单位，默认配置如下：

```
src/utils/context.ts:9
export const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000
```

系统支持多种模型的上下文窗口配置，通过`getContextWindowForModel`函数动态获取：

```
src/utils/context.ts:51-98
export function getContextWindowForModel(model: string, betas?: string[]): number {
  // 环境变量覆盖（ant-only）
  if (process.env.USER_TYPE === 'ant' && process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS) {
    const override = parseInt(process.env.CLAUDE_CODE_MAX_CONTEXT_TOKENS, 10)
    if (!isNaN(override) && override > 0) return override
  }

  // [1m]后缀显式启用1M上下文
  if (has1mContext(model)) return 1_000_000

  // 模型能力配置
  const cap = getModelCapability(model)
  if (cap?.max_input_tokens && cap.max_input_tokens >= 100_000) {
    if (cap.max_input_tokens > MODEL_CONTEXT_WINDOW_DEFAULT && is1mContextDisabled()) {
      return MODEL_CONTEXT_WINDOW_DEFAULT
    }
    return cap.max_input_tokens
  }

  // Beta特性检测
  if (betas?.includes(CONTEXT_1M_BETA_HEADER) && modelSupports1M(model)) {
    return 1_000_000
  }

  return MODEL_CONTEXT_WINDOW_DEFAULT
}
```

**配置参数表：**

| 参数名 | 默认值 | 用途 |
|--------|--------|------|
| `MODEL_CONTEXT_WINDOW_DEFAULT` | 200,000 | 默认上下文窗口大小 |
| `COMPACT_MAX_OUTPUT_TOKENS` | 20,000 | 压缩操作最大输出Token |
| `MAX_OUTPUT_TOKENS_DEFAULT` | 32,000 | 默认最大输出Token |
| `MAX_OUTPUT_TOKENS_UPPER_LIMIT` | 64,000 | 最大输出Token上限 |
| `CAPPED_DEFAULT_MAX_TOKENS` | 8,000 | 槽位预留优化的默认上限 |
| `ESCALATED_MAX_TOKENS` | 64,000 | 升级后的最大Token |

### 1.2 自动压缩阈值计算

```
src/services/compact/autoCompact.ts:62-65
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
export const MANUAL_COMPACT_BUFFER_TOKENS = 3_000
```

自动压缩阈值通过以下公式计算：

```
src/services/compact/autoCompact.ts:72-91
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  const autocompactThreshold = effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
  return autocompactThreshold
}
```

对于默认200k上下文窗口：
- 有效窗口 = 200,000 - 20,000 = 180,000 tokens
- 自动压缩阈值 = 180,000 - 13,000 = **167,000 tokens**

---

## 二、全局状态管理架构

### 2.1 状态定义结构

```
src/bootstrap/state.ts:45-257
type State = {
  originalCwd: string
  projectRoot: string
  totalCostUSD: number
  totalAPIDuration: number
  sessionId: SessionId
  parentSessionId: SessionId | undefined
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  lastAPIRequest: Omit<BetaMessageStreamParams, 'messages'> | null
  lastAPIRequestMessages: BetaMessageStreamParams['messages'] | null
  cachedClaudeMdContent: string | null
  inMemoryErrorLog: Array<{ error: string; timestamp: string }>
  invokedSkills: Map<string, InvokedSkillInfo>
  systemPromptSectionCache: Map<string, string | null>
  promptCache1hAllowlist: string[] | null
  pendingPostCompaction: boolean
  // ... 更多字段
}
```

### 2.2 会话状态流转

```
src/utils/sessionState.ts:1
export type SessionState = 'idle' | 'running' | 'requires_action'
```

状态转换通过监听器模式实现：

```
src/utils/sessionState.ts:92-134
export function notifySessionStateChanged(
  state: SessionState,
  details?: RequiresActionDetails,
): void {
  currentState = state
  stateListener?.(state, details)

  // 同步到external_metadata
  if (state === 'requires_action' && details) {
    hasPendingAction = true
    metadataListener?.({ pending_action: details })
  } else if (hasPendingAction) {
    hasPendingAction = false
    metadataListener?.({ pending_action: null })
  }

  if (state === 'idle') {
    metadataListener?.({ task_summary: null })
  }
}
```

### 2.3 状态存储模式

```
src/state/store.ts:4-34
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => void {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => () => void {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

---

## 三、Token计算策略

### 3.1 API精确计数

系统优先使用API返回的Token统计：

```
src/utils/tokens.ts:46-53
export function getTokenCountFromUsage(usage: Usage): number {
  return (
    usage.input_tokens +
    (usage.cache_creation_input_tokens ?? 0) +
    (usage.cache_read_input_tokens ?? 0) +
    usage.output_tokens
  )
}
```

### 3.2 粗略估算算法

当API计数不可用时，使用字符长度估算：

```
src/services/tokenEstimation.ts:203-208
export function roughTokenCountEstimation(
  content: string,
  bytesPerToken: number = 4,
): number {
  return Math.round(content.length / bytesPerToken)
}
```

不同文件类型使用不同的估算比率：

```
src/services/tokenEstimation.ts:215-224
export function bytesPerTokenForFileType(fileExtension: string): number {
  switch (fileExtension) {
    case 'json':
    case 'jsonl':
    case 'jsonc':
      return 2  // JSON密度更高，每Token约2字节
    default:
      return 4  // 默认每Token约4字节
  }
}
```

### 3.3 混合计算策略

核心函数`tokenCountWithEstimation`实现了API精确计数与粗略估算的结合：

```
src/utils/tokens.ts:226-261
export function tokenCountWithEstimation(messages: readonly Message[]): number {
  let i = messages.length - 1
  while (i >= 0) {
    const message = messages[i]
    const usage = message ? getTokenUsage(message) : undefined
    if (message && usage) {
      // 回溯到同一API响应的第一条记录
      const responseId = getAssistantMessageId(message)
      if (responseId) {
        let j = i - 1
        while (j >= 0) {
          const prior = messages[j]
          const priorId = prior ? getAssistantMessageId(prior) : undefined
          if (priorId === responseId) {
            i = j  // 继续向前追溯
          } else if (priorId !== undefined) {
            break
          }
          j--
        }
      }
      return (
        getTokenCountFromUsage(usage) +
        roughTokenCountEstimationForMessages(messages.slice(i + 1))
      )
    }
    i--
  }
  return roughTokenCountEstimationForMessages(messages)
}
```

---

## 四、多级压缩策略

### 4.1 压缩触发条件

```
src/services/compact/autoCompact.ts:160-239
export async function shouldAutoCompact(
  messages: Message[],
  model: string,
  querySource?: QuerySource,
  snipTokensFreed = 0,
): Promise<boolean> {
  // 递归保护
  if (querySource === 'session_memory' || querySource === 'compact') {
    return false
  }

  // 检查是否启用自动压缩
  if (!isAutoCompactEnabled()) return false

  // 上下文折叠模式抑制
  if (feature('CONTEXT_COLLAPSE') && isContextCollapseEnabled()) {
    return false
  }

  const tokenCount = tokenCountWithEstimation(messages) - snipTokensFreed
  const threshold = getAutoCompactThreshold(model)

  return tokenCount >= threshold
}
```

### 4.2 压缩执行流程

```
src/services/compact/autoCompact.ts:241-351
export async function autoCompactIfNeeded(
  messages: Message[],
  toolUseContext: ToolUseContext,
  cacheSafeParams: CacheSafeParams,
  querySource?: QuerySource,
  tracking?: AutoCompactTrackingState,
  snipTokensFreed?: number,
): Promise<{
  wasCompacted: boolean
  compactionResult?: CompactionResult
  consecutiveFailures?: number
}> {
  // 断路器：连续失败保护
  if (tracking?.consecutiveFailures !== undefined &&
      tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
    return { wasCompacted: false }
  }

  // 首先尝试会话内存压缩
  const sessionMemoryResult = await trySessionMemoryCompaction(
    messages,
    toolUseContext.agentId,
    recompactionInfo.autoCompactThreshold,
  )
  if (sessionMemoryResult) {
    runPostCompactCleanup(querySource)
    markPostCompaction()
    return { wasCompacted: true, compactionResult: sessionMemoryResult }
  }

  // 回退到传统LLM压缩
  try {
    const compactionResult = await compactConversation(...)
    return { wasCompacted: true, compactionResult, consecutiveFailures: 0 }
  } catch (error) {
    const nextFailures = (tracking?.consecutiveFailures ?? 0) + 1
    return { wasCompacted: false, consecutiveFailures: nextFailures }
  }
}
```

### 4.3 压缩结果构建

```
src/commands/compact/compact.ts:40-137
export const call: LocalCommandCall = async (args, context) => {
  let { messages } = context

  // 投影到压缩边界后的消息
  messages = getMessagesAfterCompactBoundary(messages)

  // 尝试会话内存压缩
  if (!customInstructions) {
    const sessionMemoryResult = await trySessionMemoryCompact(messages, context.agentId)
    if (sessionMemoryResult) {
      getUserContext.cache.clear?.()
      runPostCompactCleanup()
      markPostCompaction()
      suppressCompactWarning()
      return { type: 'compact', compactionResult: sessionMemoryResult, displayText }
    }
  }

  // 微压缩预处理
  const microcompactResult = await microcompactMessages(messages, context)
  const messagesForCompact = microcompactResult.messages

  // 传统压缩
  const result = await compactConversation(
    messagesForCompact,
    context,
    await getCacheSharingParams(context, messagesForCompact),
    false,
    customInstructions,
    false,
  )

  return { type: 'compact', compactionResult: result, displayText }
}
```

---

## 五、查询引擎核心逻辑

### 5.1 查询循环状态机

```
src/query.ts:203-217
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

### 5.2 查询主循环

```
src/query.ts:241-1729
async function* queryLoop(params: QueryParams, consumedCommandUuids: string[]) {
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    autoCompactTracking: undefined,
    // ...
  }

  while (true) {
    // 应用工具结果预算
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...)

    // 应用历史截断 (Snip Compact)
    if (feature('HISTORY_SNIP')) {
      const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
      messagesForQuery = snipResult.messages
      snipTokensFreed = snipResult.tokensFreed
    }

    // 应用微压缩 (Micro Compact)
    const microcompactResult = await deps.microcompact(messagesForQuery, ...)
    messagesForQuery = microcompactResult.messages

    // 应用上下文折叠 (Context Collapse)
    if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
      const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
      messagesForQuery = collapseResult.messages
    }

    // 自动压缩
    const { compactionResult } = await deps.autocompact(...)

    // 调用模型API
    for await (const message of deps.callModel(...)) {
      yield message
      // 处理工具调用...
    }

    // 更新状态并继续循环
    state = next
  }
}
```

### 5.3 查询守卫状态机

```
src/utils/QueryGuard.ts:29-121
export class QueryGuard {
  private _status: 'idle' | 'dispatching' | 'running' = 'idle'
  private _generation = 0

  reserve(): boolean {
    if (this._status !== 'idle') return false
    this._status = 'dispatching'
    return true
  }

  tryStart(): number | null {
    if (this._status === 'running') return null
    this._status = 'running'
    ++this._generation
    return this._generation
  }

  end(generation: number): boolean {
    if (this._generation !== generation) return false
    if (this._status !== 'running') return false
    this._status = 'idle'
    return true
  }

  get isActive(): boolean {
    return this._status !== 'idle'
  }
}
```

---

## 六、会话持久化机制

### 6.1 JSONL存储格式

会话存储采用JSONL（JSON Lines）格式，每行一条记录：

```
src/utils/sessionStorage.ts
export class Project {
  private insertMessageChain(
    messages: Message[],
    startingParentUuid: string | null,
    sessionId: SessionId,
  ): void {
    for (const message of messages) {
      const entry: SerializedMessage = {
        type: message.type,
        uuid: message.uuid,
        parentUuid: currentParentUuid,
        sessionId,
        timestamp: message.timestamp,
        ...message
      }
      // 写入JSONL文件
      this.writeQueue.enqueue(entry)
    }
  }
}
```

### 6.2 父UUID链结构

```
┌─────────────┐
│  Message 1  │ parentUuid: null
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Message 2  │ parentUuid: uuid1
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Message 3  │ parentUuid: uuid2
└─────────────┘
```

### 6.3 文件历史快照

```
src/utils/fileHistory.ts:53
const MAX_SNAPSHOTS = 100

export type FileHistorySnapshot = {
  messageId: UUID
  trackedFileBackups: Record<string, FileHistoryBackup>
  timestamp: Date
}

export type FileHistoryState = {
  snapshots: FileHistorySnapshot[]
  trackedFiles: Set<string>
  snapshotSequence: number
}
```

---

## 七、上下文注入系统

### 7.1 系统上下文获取

```
src/context.ts:116-149
export const getSystemContext = memoize(async (): Promise<{[k: string]: string}> => {
  const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
    !shouldIncludeGitInstructions() ? null : await getGitStatus()

  return {
    ...(gitStatus && { gitStatus }),
    ...(feature('BREAK_CACHE_COMMAND') && injection && { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }),
  }
})
```

### 7.2 用户上下文获取

```
src/context.ts:155-189
export const getUserContext = memoize(async (): Promise<{[k: string]: string}> => {
  const shouldDisableClaudeMd =
    isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
    (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

  const claudeMd = shouldDisableClaudeMd
    ? null
    : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

  setCachedClaudeMdContent(claudeMd || null)

  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

### 7.3 Git状态获取

```
src/context.ts:36-111
export const getGitStatus = memoize(async (): Promise<string | null> => {
  if (process.env.NODE_ENV === 'test') return null

  if (!await getIsGit()) return null

  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short'], ...),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5'], ...),
    execFileNoThrow(gitExe(), ['config', 'user.name'], ...),
  ])

  const truncatedStatus = status.length > MAX_STATUS_CHARS
    ? status.substring(0, MAX_STATUS_CHARS) + '\n... (truncated)'
    : status

  return [
    `This is the git status at the start of the conversation...`,
    `Current branch: ${branch}`,
    `Main branch (you will usually use this for PRs): ${mainBranch}`,
    ...(userName ? [`Git user: ${userName}`] : []),
    `Status:\n${truncatedStatus || '(clean)'}`,
    `Recent commits:\n${log}`,
  ].join('\n\n')
})
```

---

## 八、文件状态缓存

### 8.1 LRU缓存实现

```
src/utils/fileStateCache.ts:30-93
export class FileStateCache {
  private cache: LRUCache<string, FileState>

  constructor(maxEntries: number, maxSizeBytes: number) {
    this.cache = new LRUCache<string, FileState>({
      max: maxEntries,
      maxSize: maxSizeBytes,
      sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content)),
    })
  }

  get(key: string): FileState | undefined {
    return this.cache.get(normalize(key))
  }

  set(key: string, value: FileState): this {
    this.cache.set(normalize(key), value)
    return this
  }
}
```

### 8.2 缓存配置

```
src/utils/fileStateCache.ts:17-22
export const READ_FILE_STATE_CACHE_SIZE = 100
const DEFAULT_MAX_CACHE_SIZE_BYTES = 25 * 1024 * 1024  // 25MB
```

---

## 九、工具结果存储与预算

### 9.1 工具结果持久化

工具结果存储系统负责将大型工具输出持久化到磁盘，避免内存溢出：

```
src/utils/toolResultStorage.ts
export async function persistToolResult(
  toolUseId: string,
  content: string,
  agentId?: string,
): Promise<ContentReplacementRecord>
```

### 9.2 预算强制机制

```
src/utils/toolResultStorage.ts
export async function applyToolResultBudget(
  messages: Message[],
  contentReplacementState: ContentReplacementState | undefined,
  persistReplacements?: (records: ContentReplacementRecord[]) => void,
  unlimitedTools?: Set<string>,
): Promise<Message[]>
```

---

## 十、内存类型系统

### 10.1 四种内存类型

```
src/memdir/memoryTypes.ts
- user: 用户角色、偏好、职责信息
- feedback: 用户关于如何工作的指导
- project: 项目工作信息
- reference: 外部系统资源的指针
```

### 10.2 内存文件加载优先级

```
Managed → User → Project → Local
```

---

## 十一、系统架构图

```
                              ┌─────────────────────────────────────────────────────────────┐
                              │                      用户输入                               │
                              └─────────────────────────────────────────────────────────────┘
                                                           │
                                                           ▼
                              ┌─────────────────────────────────────────────────────────────┐
                              │                    QueryEngine.submitMessage()              │
                              │  ┌─────────────────────────────────────────────────────┐    │
                              │  │  1. 处理用户输入 (processUserInput)                   │    │
                              │  │  2. 构建系统提示 (fetchSystemPromptParts)            │    │
                              │  │  3. 初始化查询上下文 (toolUseContext)              │    │
                              │  └─────────────────────────────────────────────────────┘    │
                              └─────────────────────────────────────────────────────────────┘
                                                           │
                                                           ▼
                              ┌─────────────────────────────────────────────────────────────┐
                              │                         query() 主循环                       │
                              │  ┌─────────────────────────────────────────────────────┐    │
                              │  │  while (true) {                                       │    │
                              │  │    // 1. 应用工具结果预算                             │    │
                              │  │    messages = applyToolResultBudget(messages)        │    │
                              │  │                                                       │    │
                              │  │    // 2. 应用历史截断 (Snip Compact)                  │    │
                              │  │    messages = snipCompactIfNeeded(messages)          │    │
                              │  │                                                       │    │
                              │  │    // 3. 应用微压缩 (Micro Compact)                   │    │
                              │  │    messages = microcompactMessages(messages)         │    │
                              │  │                                                       │    │
                              │  │    // 4. 应用上下文折叠 (Context Collapse)            │    │
                              │  │    messages = applyCollapsesIfNeeded(messages)       │    │
                              │  │                                                       │    │
                              │  │    // 5. 检查并执行自动压缩                           │    │
                              │  │    if (shouldAutoCompact(messages)) {                │    │
                              │  │      result = autoCompactIfNeeded(messages)          │    │
                              │  │    }                                                  │    │
                              │  │                                                       │    │
                              │  │    // 6. 调用模型API                                  │    │
                              │  │    for await (msg of callModel(messages)) {          │    │
                              │  │      yield msg                                        │    │
                              │  │    }                                                  │    │
                              │  │                                                       │    │
                              │  │    // 7. 执行工具调用                                 │    │
                              │  │    toolResults = runTools(toolUseBlocks)             │    │
                              │  │                                                       │    │
                              │  │    // 8. 更新状态继续循环                             │    │
                              │  │    state = { messages: [...messages, ...results] }   │    │
                              │  │  }                                                    │    │
                              │  └─────────────────────────────────────────────────────┘    │
                              └─────────────────────────────────────────────────────────────┘
                                                           │
                               ┌───────────────────────────┼───────────────────────────┐
                               │                           │                           │
                               ▼                           ▼                           ▼
               ┌───────────────────────────┐ ┌───────────────────────────┐ ┌───────────────────────────┐
               │     Token 计算            │ │      压缩策略             │ │     状态管理             │
               │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │
               │  │ tokenCountWith      │  │ │  │ Session Memory      │  │ │  │ AppStateStore       │  │
               │  │ Estimation()        │  │ │  │ Compact             │  │ │  │                     │  │
               │  └─────────────────────┘  │ │  └─────────────────────┘  │ │  └─────────────────────┘  │
               │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │
               │  │ getTokenCount       │  │ │  │ Traditional LLM     │  │ │  │ bootstrap/state.ts  │  │
               │  │ FromUsage()         │  │ │  │ Compact             │  │ │  │                     │  │
               │  └─────────────────────┘  │ │  └─────────────────────┘  │ │  └─────────────────────┘  │
               │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │ │  ┌─────────────────────┐  │
               │  │ roughTokenCount     │  │ │  │ Reactive Compact    │  │ │  │ sessionState.ts     │  │
               │  │ Estimation()        │  │ │  │                     │  │ │  │                     │  │
               │  └─────────────────────┘  │ │  └─────────────────────┘  │ │  └─────────────────────┘  │
               └───────────────────────────┘ └───────────────────────────┘ └───────────────────────────┘
```

---

## 十二、Token计算数据流

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                                  API Response                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  usage: {                                                                           │  │
│  │    input_tokens: 50000,                                                             │  │
│  │    output_tokens: 2000,                                                             │  │
│  │    cache_creation_input_tokens: 10000,                                              │  │
│  │    cache_read_input_tokens: 5000                                                    │  │
│  │  }                                                                                  │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                           getTokenCountFromUsage()                                        │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  return input_tokens + cache_creation + cache_read + output_tokens                 │  │
│  │       50000    + 10000            + 5000        + 2000                              │  │
│  │       = 67,000 tokens                                                              │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                      tokenCountWithEstimation()                                           │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  // 查找最近一条有usage的assistant消息                                              │  │
│  │  // 处理并行tool_call情况（相同message.id的多条记录）                                │  │
│  │  // 对新增消息进行粗略估算                                                          │  │
│  │                                                                                     │  │
│  │  return getTokenCountFromUsage(usage) +                                            │  │
│  │         roughTokenCountEstimationForMessages(messages.slice(i + 1))                │  │
│  │         67000                               + ~500 (估算)                            │  │
│  │         ≈ 67,500 tokens                                                            │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                          阈值比较与决策                                                   │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  autoCompactThreshold = 167,000                                                    │  │
│  │  currentTokenCount = 67,500                                                        │  │
│  │                                                                                     │  │
│  │  if (currentTokenCount >= autoCompactThreshold) {                                  │  │
│  │    // 触发自动压缩                                                                  │  │
│  │  } else {                                                                           │  │
│  │    // 继续正常处理                                                                  │  │
│  │  }                                                                                  │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十三、压缩策略层级

```
压缩触发优先级（从低到高）：

┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. Snip Compact (HISTORY_SNIP feature)                                                  │
│    - 时间边界截断                                                                       │
│    - 移除过时的消息块                                                                   │
│    - 最轻量级，不调用LLM                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 2. Micro Compact (CACHED_MICROCOMPACT feature)                                          │
│    - 基于时间的缓存编辑                                                                 │
│    - 移除重复的工具调用结果                                                             │
│    - 不调用LLM，使用缓存策略                                                            │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 3. Session Memory Compact                                                               │
│    - 使用预提取的会话内存摘要                                                           │
│    - 快速路径，无需LLM调用                                                              │
│    - 保留关键上下文信息                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 4. Context Collapse (CONTEXT_COLLAPSE feature)                                          │
│    - 渐进式上下文折叠                                                                   │
│    - 保留结构化摘要                                                                     │
│    - 按需提交折叠操作                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 5. Traditional LLM Compact                                                              │
│    - 使用LLM生成摘要                                                                   │
│    - 最完整的压缩策略                                                                   │
│    - 成本最高但效果最好                                                                 │
│    - 保留关键决策和操作历史                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 6. Reactive Compact (REACTIVE_COMPACT feature)                                          │
│    - API 413错误后的响应式压缩                                                          │
│    - 尝试移除媒体文件后重试                                                             │
│    - 最后的恢复手段                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十四、会话恢复机制

### 14.1 恢复流程

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                              --resume 启动                                                │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    读取 JSONL 会话文件                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. 解析每行JSON记录                                                                │  │
│  │  2. 通过parentUuid重建消息链                                                        │  │
│  │  3. 验证链完整性                                                                    │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    状态恢复                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  1. 恢复文件历史快照                                                                │  │
│  │  2. 恢复工具结果内容替换                                                            │  │
│  │  3. 恢复成本追踪状态                                                                │  │
│  │  4. 恢复会话ID和父会话ID                                                            │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    中断检测                                                               │
│  ┌────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  if (lastMessage.type === 'assistant' && lastMessage.stop_reason === null) {       │  │
│  │    // 检测到中断的响应                                                              │  │
│  │    // 生成中断恢复消息                                                              │  │
│  │  }                                                                                  │  │
│  └────────────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                    继续对话                                                               │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

### 14.2 断路器机制

```
src/services/compact/autoCompact.ts:70
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 在autoCompactIfNeeded中检查
if (tracking?.consecutiveFailures !== undefined &&
    tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES) {
  return { wasCompacted: false }
}
```

---

## 十五、文件历史追踪系统

### 15.1 快照机制

```
src/utils/fileHistory.ts:198-342
export async function fileHistoryMakeSnapshot(
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void,
  messageId: UUID,
): Promise<void> {
  // 阶段1：捕获当前状态
  let captured: FileHistoryState | undefined
  updateFileHistoryState(state => { captured = state; return state })

  // 阶段2：异步备份所有追踪文件
  const trackedFileBackups: Record<string, FileHistoryBackup> = {}
  await Promise.all(Array.from(captured.trackedFiles, async trackingPath => {
    const backup = await createBackup(filePath, nextVersion)
    trackedFileBackups[trackingPath] = backup
  }))

  // 阶段3：提交新快照
  updateFileHistoryState((state: FileHistoryState) => {
    const newSnapshot: FileHistorySnapshot = {
      messageId,
      trackedFileBackups,
      timestamp: new Date(),
    }

    const allSnapshots = [...state.snapshots, newSnapshot]
    return {
      ...state,
      snapshots: allSnapshots.length > MAX_SNAPSHOTS
        ? allSnapshots.slice(-MAX_SNAPSHOTS)
        : allSnapshots,
      snapshotSequence: state.snapshotSequence + 1,
    }
  })
}
```

### 15.2 回滚机制

```
src/utils/fileHistory.ts:347-397
export async function fileHistoryRewind(
  updateFileHistoryState: (updater: (prev: FileHistoryState) => FileHistoryState) => void,
  messageId: UUID,
): Promise<void> {
  const targetSnapshot = captured.snapshots.findLast(
    snapshot => snapshot.messageId === messageId
  )

  const filesChanged = await applySnapshot(captured, targetSnapshot)
}
```

---

## 十六、应用状态管理

### 16.1 AppState类型定义

```
src/state/AppStateStore.ts:89-452
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  statusLineText: string | undefined
  toolPermissionContext: ToolPermissionContext
  fileHistory: FileHistoryState
  attribution: AttributionState
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  plugins: { enabled: LoadedPlugin[]; disabled: LoadedPlugin[]; ... }
  todos: { [agentId: string]: TodoList }
  speculation: SpeculationState
  // ... 更多字段
}>
```

### 16.2 默认状态初始化

```
src/state/AppStateStore.ts:456-569
export function getDefaultAppState(): AppState {
  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    verbose: false,
    mainLoopModel: null,
    toolPermissionContext: {
      ...getEmptyToolPermissionContext(),
      mode: initialMode,
    },
    fileHistory: {
      snapshots: [],
      trackedFiles: new Set(),
      snapshotSequence: 0,
    },
    // ...
  }
}
```

---

## 十七、总结

Claude Code的上下文管理实现了一个完整的多层次系统：

1. **配置层**：定义上下文窗口大小、压缩阈值等核心参数
2. **状态层**：管理全局会话状态、应用状态和查询状态
3. **Token计算层**：结合API精确计数和粗略估算
4. **压缩策略层**：实现六级压缩策略，从轻量到重量递进
5. **持久化层**：JSONL格式的会话存储，支持恢复和断点续传
6. **缓存层**：LRU缓存的文件状态管理，优化内存使用
7. **历史追踪层**：文件快照和回滚机制

该系统设计遵循以下原则：
- **渐进式压缩**：优先使用轻量级策略，必要时才调用LLM
- **断路器保护**：防止连续失败导致的资源浪费
- **混合Token计算**：精确与估算结合，平衡性能和准确性
- **状态一致性**：通过parentUuid链保证消息顺序
- **可恢复性**：完整的会话持久化支持中断恢复