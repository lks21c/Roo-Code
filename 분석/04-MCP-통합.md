# 04. MCP (Model Context Protocol) 통합

## 개요

MCP(Model Context Protocol)는 AI 에이전트가 외부 도구, 리소스, 서비스와 표준화된 방식으로 통신할 수 있게 해주는 프로토콜입니다. Roo-Code는 MCP를 통해 확장 가능한 도구 생태계를 구현합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Roo-Code                                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                        McpHub                               ││
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ ││
│  │  │ Connection  │  │ Connection  │  │     Connection      │ ││
│  │  │  Manager    │  │   Watcher   │  │      Registry       │ ││
│  │  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ ││
│  └─────────┼────────────────┼────────────────────┼────────────┘│
│            │                │                    │              │
│  ┌─────────┴────────────────┴────────────────────┴────────────┐│
│  │                     Transport Layer                         ││
│  │  ┌──────────┐    ┌──────────┐    ┌─────────────────────┐   ││
│  │  │  Stdio   │    │   SSE    │    │  Streamable HTTP    │   ││
│  │  │Transport │    │Transport │    │     Transport       │   ││
│  │  └────┬─────┘    └────┬─────┘    └──────────┬──────────┘   ││
│  └───────┼───────────────┼─────────────────────┼──────────────┘│
└──────────┼───────────────┼─────────────────────┼───────────────┘
           │               │                     │
           ▼               ▼                     ▼
    ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐
    │ Local Server │ │ SSE Server   │ │ HTTP Server        │
    │ (stdio)      │ │ (EventSource)│ │ (streamable-http)  │
    │ npx, node    │ │ Remote API   │ │ REST-like          │
    └──────────────┘ └──────────────┘ └────────────────────┘
```

---

## McpHub 서비스 아키텍처

### 핵심 클래스 구조

```typescript
// src/services/mcp/McpHub.ts

// 연결 상태를 나타내는 판별 합집합 (Discriminated Union)
export type ConnectedMcpConnection = {
  type: "connected"
  server: McpServer
  client: Client
  transport: StdioClientTransport | SSEClientTransport | StreamableHTTPClientTransport
}

export type DisconnectedMcpConnection = {
  type: "disconnected"
  server: McpServer
  client: null
  transport: null
}

export type McpConnection = ConnectedMcpConnection | DisconnectedMcpConnection

// 비활성화 사유 열거형
export enum DisableReason {
  MCP_DISABLED = "mcpDisabled",
  SERVER_DISABLED = "serverDisabled",
}

export class McpHub {
  private providerRef: WeakRef<ClineProvider>
  private disposables: vscode.Disposable[] = []
  private settingsWatcher?: vscode.FileSystemWatcher
  private fileWatchers: Map<string, FSWatcher[]> = new Map()
  connections: McpConnection[] = []
  isConnecting: boolean = false
  private refCount: number = 0  // 참조 카운팅
  private sanitizedNameRegistry: Map<string, string> = new Map()

  constructor(provider: ClineProvider) {
    this.providerRef = new WeakRef(provider)
    this.watchMcpSettingsFile()
    this.watchProjectMcpFile()
    this.setupWorkspaceFoldersWatcher()
    this.initializeGlobalMcpServers()
    this.initializeProjectMcpServers()
  }

  // 클라이언트 참조 관리
  public registerClient(): void {
    this.refCount++
  }

  public async unregisterClient(): Promise<void> {
    this.refCount--
    if (this.refCount <= 0) {
      await this.dispose()
    }
  }
}
```

### 서버 설정 스키마 (Zod 기반 검증)

```typescript
// 기본 설정 스키마
const BaseConfigSchema = z.object({
  disabled: z.boolean().optional(),
  timeout: z.number().min(1).max(3600).optional().default(60),
  alwaysAllow: z.array(z.string()).default([]),
  watchPaths: z.array(z.string()).optional(),
  disabledTools: z.array(z.string()).default([]),
})

// 서버 타입별 스키마 (Union)
export const ServerConfigSchema = z.union([
  // Stdio 설정
  BaseConfigSchema.extend({
    type: z.enum(["stdio"]).optional(),
    command: z.string().min(1),
    args: z.array(z.string()).optional(),
    cwd: z.string().default(() => process.cwd()),
    env: z.record(z.string()).optional(),
  }).transform(data => ({ ...data, type: "stdio" as const })),

  // SSE 설정
  BaseConfigSchema.extend({
    type: z.enum(["sse"]),
    url: z.string().url(),
    headers: z.record(z.string()).optional(),
  }),

  // Streamable HTTP 설정
  BaseConfigSchema.extend({
    type: z.enum(["streamable-http"]),
    url: z.string().url(),
    headers: z.record(z.string()).optional(),
  }),
])

// 전체 설정 스키마
const McpSettingsSchema = z.object({
  mcpServers: z.record(ServerConfigSchema),
})
```

---

## 연결 유형별 구현

### 1. Stdio Transport (로컬 프로세스)

```typescript
// 로컬 MCP 서버 연결 (가장 일반적)
if (configInjected.type === "stdio") {
  // Windows에서 PowerShell 스크립트 처리
  const isWindows = process.platform === "win32"
  const isAlreadyWrapped = configInjected.command.toLowerCase() === "cmd.exe"

  const command = isWindows && !isAlreadyWrapped ? "cmd.exe" : configInjected.command
  const args = isWindows && !isAlreadyWrapped
    ? ["/c", configInjected.command, ...(configInjected.args || [])]
    : configInjected.args

  transport = new StdioClientTransport({
    command,
    args,
    cwd: configInjected.cwd,
    env: {
      ...getDefaultEnvironment(),
      ...(configInjected.env || {}),
    },
    stderr: "pipe",
  })

  // stderr 스트림 모니터링
  transport.stderr.on("data", async (data: Buffer) => {
    const output = data.toString()
    const isInfoLog = /INFO/i.test(output)

    if (isInfoLog) {
      console.log(`Server "${name}" info:`, output)
    } else {
      console.error(`Server "${name}" stderr:`, output)
      this.appendErrorMessage(connection, output)
    }
  })
}
```

### 2. SSE Transport (Server-Sent Events)

```typescript
// 원격 SSE 서버 연결
if (configInjected.type === "sse") {
  const reconnectingOptions = {
    max_retry_time: 5000,
    withCredentials: !!configInjected.headers?.["Authorization"],
    fetch: (url: string, init: RequestInit) => {
      const headers = new Headers({
        ...(init?.headers || {}),
        ...(configInjected.headers || {})
      })
      return fetch(url, { ...init, headers })
    },
  }

  // 자동 재연결 지원
  global.EventSource = ReconnectingEventSource

  transport = new SSEClientTransport(new URL(configInjected.url), {
    requestInit: { headers: configInjected.headers },
    eventSourceInit: reconnectingOptions,
  })
}
```

### 3. Streamable HTTP Transport

```typescript
// HTTP 기반 스트리밍 연결
if (configInjected.type === "streamable-http") {
  transport = new StreamableHTTPClientTransport(
    new URL(configInjected.url),
    {
      requestInit: {
        headers: configInjected.headers,
      },
    }
  )
}
```

---

## 서버 생명주기 관리

### 초기화 흐름

```
┌──────────────────────────────────────────────────────────────────┐
│                     Server Initialization Flow                    │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ McpHub Created  │────▶│ Watch Settings  │────▶│ Parse Config    │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                         │
                        ┌────────────────────────────────┼────────────────────────────────┐
                        │                                │                                │
                        ▼                                ▼                                ▼
            ┌───────────────────┐         ┌───────────────────┐         ┌───────────────────┐
            │ Global Servers    │         │ Project Servers   │         │ Disabled Servers  │
            │ ~/.roo-code/      │         │ .roo/mcp.json     │         │ (Placeholder)     │
            └─────────┬─────────┘         └─────────┬─────────┘         └─────────┬─────────┘
                      │                             │                             │
                      ▼                             ▼                             ▼
            ┌───────────────────┐         ┌───────────────────┐         ┌───────────────────┐
            │ Create Transport  │         │ Create Transport  │         │ Create Placeholder│
            │ Connect Client    │         │ Connect Client    │         │ Connection Object │
            └─────────┬─────────┘         └─────────┬─────────┘         └─────────┬─────────┘
                      │                             │                             │
                      └─────────────────────────────┼─────────────────────────────┘
                                                    │
                                                    ▼
                                        ┌───────────────────┐
                                        │ Fetch Tools &     │
                                        │ Resources         │
                                        └─────────┬─────────┘
                                                  │
                                                  ▼
                                        ┌───────────────────┐
                                        │ Notify Webview    │
                                        └───────────────────┘
```

### 서버 연결 함수

```typescript
private async connectToServer(
  name: string,
  config: z.infer<typeof ServerConfigSchema>,
  source: "global" | "project" = "global",
): Promise<void> {
  // 1. 기존 연결 제거
  await this.deleteConnection(name, source)

  // 2. 이름 정규화 등록 (API 호환성)
  const sanitizedName = sanitizeMcpName(name)
  this.sanitizedNameRegistry.set(sanitizedName, name)

  // 3. MCP 전역 활성화 확인
  const mcpEnabled = await this.isMcpEnabled()
  if (!mcpEnabled) {
    const connection = this.createPlaceholderConnection(name, config, source, DisableReason.MCP_DISABLED)
    this.connections.push(connection)
    return
  }

  // 4. 개별 서버 비활성화 확인
  if (config.disabled) {
    const connection = this.createPlaceholderConnection(name, config, source, DisableReason.SERVER_DISABLED)
    this.connections.push(connection)
    return
  }

  // 5. 파일 감시자 설정
  this.setupFileWatcher(name, config, source)

  try {
    // 6. MCP 클라이언트 생성
    const client = new Client(
      { name: "Roo Code", version: "..." },
      { capabilities: {} },
    )

    // 7. Transport 생성 (타입별 분기)
    let transport: StdioClientTransport | SSEClientTransport | StreamableHTTPClientTransport
    // ... transport 생성 로직

    // 8. 연결 객체 생성 및 등록
    const connection: ConnectedMcpConnection = {
      type: "connected",
      server: { name, config: JSON.stringify(config), status: "connecting", ... },
      client,
      transport,
    }
    this.connections.push(connection)

    // 9. 연결 수립
    await client.connect(transport)
    connection.server.status = "connected"

    // 10. 도구/리소스 목록 가져오기
    connection.server.tools = await this.fetchToolsList(name, source)
    connection.server.resources = await this.fetchResourcesList(name, source)
    connection.server.resourceTemplates = await this.fetchResourceTemplatesList(name, source)

  } catch (error) {
    // 에러 처리
    const connection = this.findConnection(name, source)
    if (connection) {
      connection.server.status = "disconnected"
      this.appendErrorMessage(connection, error.message)
    }
    throw error
  }
}
```

---

## 설정 파일 감시 및 자동 재연결

### 파일 감시 시스템

```typescript
// 글로벌 설정 파일 감시
private async watchMcpSettingsFile(): Promise<void> {
  const settingsPath = await this.getMcpSettingsFilePath()
  const settingsPattern = new vscode.RelativePattern(
    path.dirname(settingsPath),
    path.basename(settingsPath)
  )

  this.settingsWatcher = vscode.workspace.createFileSystemWatcher(settingsPattern)

  // 변경 이벤트 (디바운스 적용)
  this.settingsWatcher.onDidChange((uri) => {
    this.debounceConfigChange(settingsPath, "global")
  })

  // 생성 이벤트
  this.settingsWatcher.onDidCreate((uri) => {
    this.debounceConfigChange(settingsPath, "global")
  })
}

// 프로젝트별 설정 파일 감시
private async watchProjectMcpFile(): Promise<void> {
  const workspaceFolder = this.providerRef.deref()?.cwd ?? getWorkspacePath()
  const projectMcpPattern = new vscode.RelativePattern(workspaceFolder, ".roo/mcp.json")

  this.projectMcpWatcher = vscode.workspace.createFileSystemWatcher(projectMcpPattern)

  this.projectMcpWatcher.onDidChange((uri) => {
    this.debounceConfigChange(uri.fsPath, "project")
  })

  this.projectMcpWatcher.onDidDelete(async () => {
    await this.cleanupProjectMcpServers()
    await this.notifyWebviewOfServerChanges()
  })
}

// 디바운스된 설정 변경 처리
private debounceConfigChange(filePath: string, source: "global" | "project"): void {
  // 프로그래밍적 업데이트 시 무시
  if (this.isProgrammaticUpdate) {
    return
  }

  const key = `${source}-${filePath}`

  // 기존 타이머 취소
  const existingTimer = this.configChangeDebounceTimers.get(key)
  if (existingTimer) {
    clearTimeout(existingTimer)
  }

  // 새 타이머 설정 (500ms 디바운스)
  const timer = setTimeout(async () => {
    this.configChangeDebounceTimers.delete(key)
    await this.handleConfigFileChange(filePath, source)
  }, 500)

  this.configChangeDebounceTimers.set(key, timer)
}
```

### 서버 빌드 파일 감시 (핫 리로드)

```typescript
private setupFileWatcher(
  name: string,
  config: z.infer<typeof ServerConfigSchema>,
  source: "global" | "project" = "global",
) {
  if (config.type === "stdio") {
    // 1. 커스텀 watchPaths 감시
    if (config.watchPaths && config.watchPaths.length > 0) {
      const watchPathsWatcher = chokidar.watch(config.watchPaths)

      watchPathsWatcher.on("change", async (changedPath) => {
        await this.restartConnection(name, source)
      })

      this.fileWatchers.get(name)?.push(watchPathsWatcher)
    }

    // 2. build/index.js 자동 감시 (개발 편의)
    const filePath = config.args?.find(arg => arg.includes("build/index.js"))
    if (filePath) {
      const indexJsWatcher = chokidar.watch(filePath)

      indexJsWatcher.on("change", async () => {
        await this.restartConnection(name, source)
      })

      this.fileWatchers.get(name)?.push(indexJsWatcher)
    }
  }
}
```

---

## MCP 도구 호출 흐름

### UseMcpToolTool 구현

```typescript
// src/core/tools/UseMcpToolTool.ts

interface UseMcpToolParams {
  server_name: string
  tool_name: string
  arguments?: Record<string, unknown>
}

export class UseMcpToolTool extends BaseTool<"use_mcp_tool"> {
  readonly name = "use_mcp_tool" as const

  async execute(params: UseMcpToolParams, task: Task, callbacks: ToolCallbacks): Promise<void> {
    const { askApproval, handleError, pushToolResult } = callbacks

    try {
      // 1. 파라미터 검증
      const validation = await this.validateParams(task, params, pushToolResult)
      if (!validation.isValid) return

      const { serverName, toolName, parsedArguments } = validation

      // 2. 도구 존재 여부 확인
      const toolValidation = await this.validateToolExists(task, serverName, toolName, pushToolResult)
      if (!toolValidation.isValid) return

      // 3. 사용자 승인 요청
      const completeMessage = JSON.stringify({
        type: "use_mcp_tool",
        serverName,
        toolName,
        arguments: params.arguments ? JSON.stringify(params.arguments) : undefined,
      })

      const didApprove = await askApproval("use_mcp_server", completeMessage)
      if (!didApprove) return

      // 4. 도구 실행
      await this.executeToolAndProcessResult(
        task, serverName, toolName, parsedArguments, executionId, pushToolResult
      )

    } catch (error) {
      await handleError("executing MCP tool", error)
    }
  }

  private async executeToolAndProcessResult(...): Promise<void> {
    await task.say("mcp_server_request_started")

    // 실행 상태 전송
    await this.sendExecutionStatus(task, {
      executionId,
      status: "started",
      serverName,
      toolName,
    })

    // MCP Hub를 통한 도구 호출
    const toolResult = await task.providerRef.deref()
      ?.getMcpHub()
      ?.callTool(serverName, toolName, parsedArguments)

    // 결과 처리 및 응답
    const outputText = this.processToolContent(toolResult)

    await this.sendExecutionStatus(task, {
      executionId,
      status: toolResult.isError ? "error" : "completed",
      response: outputText,
    })

    await task.say("mcp_server_response", outputText)
    pushToolResult(formatResponse.toolResult(outputText))
  }
}
```

### 도구 호출 시퀀스

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MCP Tool Call Sequence                               │
└─────────────────────────────────────────────────────────────────────────────┘

    AI Model           Task           UseMcpToolTool      McpHub        MCP Server
       │                 │                  │               │               │
       │ use_mcp_tool    │                  │               │               │
       │────────────────▶│                  │               │               │
       │                 │ execute()        │               │               │
       │                 │─────────────────▶│               │               │
       │                 │                  │               │               │
       │                 │                  │ validateParams│               │
       │                 │                  │◀─────────────▶│               │
       │                 │                  │               │               │
       │                 │                  │ validateTool  │               │
       │                 │                  │───────────────▶ findConnection│
       │                 │                  │               │───────────────▶
       │                 │                  │◀──────────────│◀──────────────│
       │                 │                  │               │               │
       │                 │ askApproval      │               │               │
       │                 │◀─────────────────│               │               │
       │                 │                  │               │               │
       │ [User Approves] │                  │               │               │
       │                 │                  │               │               │
       │                 │                  │ callTool      │               │
       │                 │                  │──────────────▶│ tools/call    │
       │                 │                  │               │──────────────▶│
       │                 │                  │               │               │
       │                 │                  │               │◀──────────────│
       │                 │                  │◀──────────────│  Result       │
       │                 │                  │               │               │
       │                 │ pushToolResult   │               │               │
       │◀────────────────│◀─────────────────│               │               │
       │                 │                  │               │               │
```

---

## 리소스 접근 패턴

### McpServer 타입 정의

```typescript
// src/shared/mcp.ts

export type McpServer = {
  name: string
  config: string
  status: "connected" | "connecting" | "disconnected"
  error?: string
  errorHistory?: McpErrorEntry[]
  tools?: McpTool[]
  resources?: McpResource[]
  resourceTemplates?: McpResourceTemplate[]
  disabled?: boolean
  timeout?: number
  source?: "global" | "project"  // 설정 소스
  projectPath?: string
  instructions?: string  // 서버 지시사항
}

export type McpTool = {
  name: string
  description?: string
  inputSchema?: object
  alwaysAllow?: boolean       // 자동 승인 여부
  enabledForPrompt?: boolean  // 프롬프트에 포함 여부
}

export type McpResource = {
  uri: string
  name: string
  mimeType?: string
  description?: string
}

export type McpResourceTemplate = {
  uriTemplate: string
  name: string
  description?: string
  mimeType?: string
}

export type McpToolCallResponse = {
  _meta?: Record<string, any>
  content: Array<
    | { type: "text"; text: string }
    | { type: "image"; data: string; mimeType: string }
    | { type: "audio"; data: string; mimeType: string }
    | { type: "resource"; resource: { uri: string; mimeType?: string; text?: string; blob?: string } }
  >
  isError?: boolean
}
```

### 리소스 읽기/도구 호출 API

```typescript
// McpHub 메서드

async readResource(
  serverName: string,
  uri: string,
  source?: "global" | "project"
): Promise<McpResourceResponse> {
  const connection = this.findConnection(serverName, source)

  if (!connection || connection.type !== "connected") {
    throw new Error(`No connection found for server: ${serverName}`)
  }

  if (connection.server.disabled) {
    throw new Error(`Server "${serverName}" is disabled`)
  }

  return await connection.client.request(
    { method: "resources/read", params: { uri } },
    ReadResourceResultSchema,
  )
}

async callTool(
  serverName: string,
  toolName: string,
  toolArguments?: Record<string, unknown>,
  source?: "global" | "project",
): Promise<McpToolCallResponse> {
  const connection = this.findConnection(serverName, source)

  if (!connection || connection.type !== "connected") {
    throw new Error(`No connection found for server: ${serverName}`)
  }

  // 설정에서 타임아웃 값 가져오기
  const parsedConfig = ServerConfigSchema.parse(JSON.parse(connection.server.config))
  const timeout = (parsedConfig.timeout ?? 60) * 1000

  return await connection.client.request(
    { method: "tools/call", params: { name: toolName, arguments: toolArguments } },
    CallToolResultSchema,
    { timeout },
  )
}
```

---

## 프롬프트 통합

### MCP 서버 섹션 생성

```typescript
// src/core/prompts/sections/mcp-servers.ts

export async function getMcpServersSection(
  mcpHub?: McpHub,
  diffStrategy?: DiffStrategy,
  enableMcpServerCreation?: boolean,
  includeToolDescriptions: boolean = true,
): Promise<string> {
  if (!mcpHub) return ""

  const connectedServers = mcpHub.getServers()
    .filter(server => server.status === "connected")
    .map(server => {
      // 활성화된 도구만 포함
      const tools = includeToolDescriptions
        ? server.tools
            ?.filter(tool => tool.enabledForPrompt !== false)
            ?.map(tool => {
              const schemaStr = tool.inputSchema
                ? `    Input Schema:\n    ${JSON.stringify(tool.inputSchema, null, 2)}`
                : ""
              return `- ${tool.name}: ${tool.description}\n${schemaStr}`
            })
            .join("\n\n")
        : undefined

      const resources = server.resources
        ?.map(r => `- ${r.uri} (${r.name}): ${r.description}`)
        .join("\n")

      return (
        `## ${server.name}` +
        (server.instructions ? `\n\n### Instructions\n${server.instructions}` : "") +
        (tools ? `\n\n### Available Tools\n${tools}` : "") +
        (resources ? `\n\n### Direct Resources\n${resources}` : "")
      )
    })
    .join("\n\n")

  // 프로토콜에 따른 도구 접근 방법 안내
  const toolAccessInstructions = includeToolDescriptions
    ? `When a server is connected, you can use the server's tools via the \`use_mcp_tool\` tool...`
    : `Each server's tools are available as native tools with pattern \`mcp_{server_name}_{tool_name}\`...`

  return `MCP SERVERS

The Model Context Protocol (MCP) enables communication between the system and MCP servers...

# Connected MCP Servers

${toolAccessInstructions}

${connectedServers}`
}
```

### 이중 프로토콜 지원 (XML vs Native)

```
┌─────────────────────────────────────────────────────────────────┐
│                   MCP Tool Protocol Support                      │
└─────────────────────────────────────────────────────────────────┘

                      ┌──────────────────┐
                      │   AI Model       │
                      │   Tool Call      │
                      └────────┬─────────┘
                               │
               ┌───────────────┴───────────────┐
               │                               │
               ▼                               ▼
    ┌─────────────────────┐       ┌─────────────────────┐
    │   XML Protocol      │       │   Native Protocol   │
    │                     │       │                     │
    │ <use_mcp_tool>      │       │ mcp_server_tool()   │
    │   <server_name>     │       │                     │
    │   <tool_name>       │       │ Tool name format:   │
    │   <arguments>       │       │ mcp_{server}_{tool} │
    │ </use_mcp_tool>     │       │                     │
    └─────────┬───────────┘       └─────────┬───────────┘
              │                             │
              └─────────────┬───────────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   UseMcpToolTool  │
                  │   execute()       │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │     McpHub        │
                  │   callTool()      │
                  └───────────────────┘
```

---

## 에러 처리 및 복구

### 에러 히스토리 관리

```typescript
private appendErrorMessage(
  connection: McpConnection,
  error: string,
  level: "error" | "warn" | "info" = "error"
) {
  const MAX_ERROR_LENGTH = 1000
  const truncatedError = error.length > MAX_ERROR_LENGTH
    ? `${error.substring(0, MAX_ERROR_LENGTH)}...(truncated)`
    : error

  if (!connection.server.errorHistory) {
    connection.server.errorHistory = []
  }

  connection.server.errorHistory.push({
    message: truncatedError,
    timestamp: Date.now(),
    level,
  })

  // 최근 100개만 유지
  if (connection.server.errorHistory.length > 100) {
    connection.server.errorHistory = connection.server.errorHistory.slice(-100)
  }

  connection.server.error = truncatedError
}
```

### 자동 재연결

```typescript
// SSE Transport 자동 재연결 설정
const reconnectingEventSourceOptions = {
  max_retry_time: 5000,  // 최대 재시도 간격
  withCredentials: !!configInjected.headers?.["Authorization"],
}

// Transport 에러/종료 핸들러
transport.onerror = async (error) => {
  console.error(`Transport error for "${name}":`, error)
  const connection = this.findConnection(name, source)
  if (connection) {
    connection.server.status = "disconnected"
    this.appendErrorMessage(connection, error.message)
  }
  await this.notifyWebviewOfServerChanges()
}

transport.onclose = async () => {
  const connection = this.findConnection(name, source)
  if (connection) {
    connection.server.status = "disconnected"
  }
  await this.notifyWebviewOfServerChanges()
}
```

### 서버 재시작

```typescript
async restartConnection(serverName: string, source?: "global" | "project"): Promise<void> {
  this.isConnecting = true

  const mcpEnabled = await this.isMcpEnabled()
  if (!mcpEnabled) {
    this.isConnecting = false
    return
  }

  const connection = this.findConnection(serverName, source)
  const config = connection?.server.config

  if (config) {
    vscode.window.showInformationMessage(`Restarting MCP server ${serverName}...`)

    connection.server.status = "connecting"
    connection.server.error = ""
    await this.notifyWebviewOfServerChanges()

    await delay(500)  // UI 피드백용 지연

    try {
      await this.deleteConnection(serverName, connection.server.source)
      const validatedConfig = this.validateServerConfig(JSON.parse(config), serverName)
      await this.connectToServer(serverName, validatedConfig, connection.server.source || "global")

      vscode.window.showInformationMessage(`MCP server ${serverName} connected`)
    } catch (error) {
      console.error(`Failed to restart ${serverName}:`, error)
    }
  }

  await this.notifyWebviewOfServerChanges()
  this.isConnecting = false
}
```

---

## Jupyter 에이전트 적용 가이드

### Jupyter 커널을 MCP 서버로 래핑하기

```python
# jupyter_mcp_bridge.py - Jupyter 커널을 MCP 서버로 노출

from mcp.server import Server
from mcp.types import Tool, TextContent
import jupyter_client

class JupyterMcpBridge:
    """Jupyter 커널을 MCP 프로토콜로 래핑"""

    def __init__(self, kernel_name: str = "python3"):
        self.server = Server("jupyter-kernel")
        self.kernel_manager = jupyter_client.KernelManager(kernel_name=kernel_name)
        self.kernel_client = None

        self._setup_tools()

    def _setup_tools(self):
        """MCP 도구 정의"""

        @self.server.tool()
        async def execute_code(code: str) -> list[TextContent]:
            """Jupyter 셀 실행"""
            if not self.kernel_client:
                self.kernel_manager.start_kernel()
                self.kernel_client = self.kernel_manager.client()
                self.kernel_client.start_channels()

            msg_id = self.kernel_client.execute(code)

            # 결과 수집
            outputs = []
            while True:
                try:
                    msg = self.kernel_client.get_iopub_msg(timeout=30)
                    if msg['parent_header'].get('msg_id') != msg_id:
                        continue

                    msg_type = msg['msg_type']
                    content = msg['content']

                    if msg_type == 'stream':
                        outputs.append(TextContent(type="text", text=content['text']))
                    elif msg_type == 'execute_result':
                        outputs.append(TextContent(type="text", text=content['data'].get('text/plain', '')))
                    elif msg_type == 'error':
                        outputs.append(TextContent(type="text", text='\n'.join(content['traceback'])))
                    elif msg_type == 'status' and content['execution_state'] == 'idle':
                        break
                except:
                    break

            return outputs if outputs else [TextContent(type="text", text="(No output)")]

        @self.server.tool()
        async def get_kernel_info() -> list[TextContent]:
            """커널 정보 조회"""
            if not self.kernel_client:
                return [TextContent(type="text", text="Kernel not started")]

            info = self.kernel_client.kernel_info()
            return [TextContent(type="text", text=str(info))]

        @self.server.tool()
        async def interrupt_kernel() -> list[TextContent]:
            """실행 중단"""
            if self.kernel_manager:
                self.kernel_manager.interrupt_kernel()
                return [TextContent(type="text", text="Kernel interrupted")]
            return [TextContent(type="text", text="No kernel to interrupt")]

        @self.server.tool()
        async def restart_kernel() -> list[TextContent]:
            """커널 재시작"""
            if self.kernel_manager:
                self.kernel_manager.restart_kernel()
                return [TextContent(type="text", text="Kernel restarted")]
            return [TextContent(type="text", text="No kernel to restart")]

    async def run_stdio(self):
        """stdio 모드로 실행"""
        from mcp.server.stdio import stdio_server
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream, write_stream,
                self.server.create_initialization_options()
            )

# 실행
if __name__ == "__main__":
    import asyncio
    bridge = JupyterMcpBridge()
    asyncio.run(bridge.run_stdio())
```

### MCP 설정 예시

```json
// .roo/mcp.json 또는 ~/.roo-code/mcp_settings.json
{
  "mcpServers": {
    "jupyter-python": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "jupyter_mcp_bridge"],
      "timeout": 300,
      "alwaysAllow": ["execute_code"],
      "watchPaths": ["./jupyter_mcp_bridge.py"]
    },
    "jupyter-remote": {
      "type": "sse",
      "url": "http://localhost:8765/mcp",
      "headers": {
        "Authorization": "Bearer ${JUPYTER_TOKEN}"
      },
      "timeout": 120
    }
  }
}
```

### Python MCP Hub 구현

```python
# mcp_hub.py - Roo-Code McpHub의 Python 버전

import asyncio
import json
from dataclasses import dataclass, field
from enum import Enum
from typing import Dict, List, Optional, Any, Union
from pathlib import Path
import aiofiles
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class ConnectionStatus(Enum):
    CONNECTED = "connected"
    CONNECTING = "connecting"
    DISCONNECTED = "disconnected"

class TransportType(Enum):
    STDIO = "stdio"
    SSE = "sse"
    STREAMABLE_HTTP = "streamable-http"

@dataclass
class McpTool:
    name: str
    description: Optional[str] = None
    input_schema: Optional[dict] = None
    always_allow: bool = False
    enabled_for_prompt: bool = True

@dataclass
class McpServer:
    name: str
    config: str
    status: ConnectionStatus = ConnectionStatus.DISCONNECTED
    tools: List[McpTool] = field(default_factory=list)
    resources: List[dict] = field(default_factory=list)
    error: Optional[str] = None
    error_history: List[dict] = field(default_factory=list)
    disabled: bool = False
    source: str = "global"

@dataclass
class McpConnection:
    server: McpServer
    client: Optional[Any] = None
    transport: Optional[Any] = None

    @property
    def is_connected(self) -> bool:
        return self.client is not None and self.transport is not None

class McpHub:
    """Python 버전 MCP Hub"""

    def __init__(self, config_path: Path):
        self.config_path = config_path
        self.connections: List[McpConnection] = []
        self._is_connecting = False
        self._file_watchers: Dict[str, Observer] = {}
        self._debounce_timers: Dict[str, asyncio.Task] = {}

    async def initialize(self):
        """MCP 서버 초기화"""
        await self._load_config_and_connect()
        self._setup_file_watcher()

    async def _load_config_and_connect(self):
        """설정 파일 로드 및 연결"""
        if not self.config_path.exists():
            return

        async with aiofiles.open(self.config_path) as f:
            content = await f.read()
            config = json.loads(content)

        servers_config = config.get("mcpServers", {})

        for name, server_config in servers_config.items():
            await self._connect_to_server(name, server_config, "global")

    async def _connect_to_server(
        self,
        name: str,
        config: dict,
        source: str
    ):
        """서버 연결"""
        # 기존 연결 제거
        await self._delete_connection(name, source)

        # 비활성화된 서버 처리
        if config.get("disabled", False):
            connection = McpConnection(
                server=McpServer(
                    name=name,
                    config=json.dumps(config),
                    status=ConnectionStatus.DISCONNECTED,
                    disabled=True,
                    source=source
                )
            )
            self.connections.append(connection)
            return

        # 새 연결 생성
        server = McpServer(
            name=name,
            config=json.dumps(config),
            status=ConnectionStatus.CONNECTING,
            source=source
        )
        connection = McpConnection(server=server)
        self.connections.append(connection)

        try:
            transport_type = config.get("type", "stdio")

            if transport_type == "stdio":
                # Stdio transport 연결
                client, transport = await self._create_stdio_connection(config)
            elif transport_type == "sse":
                # SSE transport 연결
                client, transport = await self._create_sse_connection(config)
            else:
                raise ValueError(f"Unsupported transport type: {transport_type}")

            connection.client = client
            connection.transport = transport
            connection.server.status = ConnectionStatus.CONNECTED

            # 도구 목록 가져오기
            connection.server.tools = await self._fetch_tools(client)
            connection.server.resources = await self._fetch_resources(client)

        except Exception as e:
            connection.server.status = ConnectionStatus.DISCONNECTED
            connection.server.error = str(e)
            self._append_error(connection, str(e))

    async def _create_stdio_connection(self, config: dict):
        """Stdio transport 생성"""
        # MCP SDK를 사용한 stdio 연결
        from mcp.client.stdio import stdio_client

        command = config["command"]
        args = config.get("args", [])
        env = config.get("env", {})

        async with stdio_client(command, args, env) as (read, write):
            from mcp.client import Client
            client = Client()
            await client.initialize(read, write)
            return client, (read, write)

    async def _create_sse_connection(self, config: dict):
        """SSE transport 생성"""
        from mcp.client.sse import sse_client

        url = config["url"]
        headers = config.get("headers", {})

        async with sse_client(url, headers) as (read, write):
            from mcp.client import Client
            client = Client()
            await client.initialize(read, write)
            return client, (read, write)

    async def _fetch_tools(self, client) -> List[McpTool]:
        """도구 목록 가져오기"""
        try:
            result = await client.request("tools/list", {})
            return [
                McpTool(
                    name=tool["name"],
                    description=tool.get("description"),
                    input_schema=tool.get("inputSchema")
                )
                for tool in result.get("tools", [])
            ]
        except Exception:
            return []

    async def _fetch_resources(self, client) -> List[dict]:
        """리소스 목록 가져오기"""
        try:
            result = await client.request("resources/list", {})
            return result.get("resources", [])
        except Exception:
            return []

    async def call_tool(
        self,
        server_name: str,
        tool_name: str,
        arguments: Optional[dict] = None
    ) -> dict:
        """MCP 도구 호출"""
        connection = self._find_connection(server_name)

        if not connection or not connection.is_connected:
            raise ConnectionError(f"Server {server_name} not connected")

        if connection.server.disabled:
            raise RuntimeError(f"Server {server_name} is disabled")

        # 타임아웃 설정
        config = json.loads(connection.server.config)
        timeout = config.get("timeout", 60)

        return await asyncio.wait_for(
            connection.client.request("tools/call", {
                "name": tool_name,
                "arguments": arguments or {}
            }),
            timeout=timeout
        )

    def _find_connection(
        self,
        server_name: str,
        source: Optional[str] = None
    ) -> Optional[McpConnection]:
        """연결 찾기 (프로젝트 우선)"""
        if source:
            for conn in self.connections:
                if conn.server.name == server_name and conn.server.source == source:
                    return conn
            return None

        # 프로젝트 서버 우선
        for conn in self.connections:
            if conn.server.name == server_name and conn.server.source == "project":
                return conn

        # 글로벌 서버
        for conn in self.connections:
            if conn.server.name == server_name:
                return conn

        return None

    async def _delete_connection(self, name: str, source: Optional[str] = None):
        """연결 삭제"""
        connections_to_remove = [
            conn for conn in self.connections
            if conn.server.name == name and (source is None or conn.server.source == source)
        ]

        for conn in connections_to_remove:
            if conn.transport:
                # Transport 정리
                pass
            self.connections.remove(conn)

    def _append_error(self, connection: McpConnection, error: str):
        """에러 히스토리 추가"""
        import time

        if len(error) > 1000:
            error = error[:1000] + "...(truncated)"

        connection.server.error_history.append({
            "message": error,
            "timestamp": time.time(),
            "level": "error"
        })

        # 최근 100개만 유지
        if len(connection.server.error_history) > 100:
            connection.server.error_history = connection.server.error_history[-100:]

        connection.server.error = error

    def _setup_file_watcher(self):
        """설정 파일 감시"""
        class ConfigChangeHandler(FileSystemEventHandler):
            def __init__(self, hub: McpHub):
                self.hub = hub

            def on_modified(self, event):
                if event.src_path == str(self.hub.config_path):
                    asyncio.create_task(self.hub._on_config_change())

        observer = Observer()
        observer.schedule(
            ConfigChangeHandler(self),
            str(self.config_path.parent),
            recursive=False
        )
        observer.start()
        self._file_watchers["config"] = observer

    async def _on_config_change(self):
        """설정 변경 처리 (디바운스)"""
        key = "config_change"

        if key in self._debounce_timers:
            self._debounce_timers[key].cancel()

        async def debounced_reload():
            await asyncio.sleep(0.5)  # 500ms 디바운스
            await self._load_config_and_connect()

        self._debounce_timers[key] = asyncio.create_task(debounced_reload())

    def get_servers(self) -> List[McpServer]:
        """활성화된 서버 목록"""
        return [
            conn.server for conn in self.connections
            if not conn.server.disabled
        ]

    def get_all_servers(self) -> List[McpServer]:
        """모든 서버 목록"""
        return [conn.server for conn in self.connections]

    async def dispose(self):
        """리소스 정리"""
        for observer in self._file_watchers.values():
            observer.stop()

        for conn in self.connections:
            await self._delete_connection(conn.server.name)
```

---

## 아키텍처 비교: Roo-Code vs Jupyter 에이전트

| 구성 요소 | Roo-Code | Jupyter 에이전트 |
|-----------|----------|------------------|
| **Hub 클래스** | `McpHub` | `McpHub` (Python) |
| **연결 관리** | WeakRef + RefCount | asyncio + context manager |
| **Transport** | Stdio/SSE/HTTP | Stdio/SSE/WebSocket |
| **설정 위치** | `~/.roo-code/`, `.roo/` | `~/.jupyter/`, `.jupyter/` |
| **검증** | Zod 스키마 | Pydantic 모델 |
| **파일 감시** | chokidar + VSCode API | watchdog |
| **디바운스** | setTimeout | asyncio.Task |
| **커널 통합** | N/A | jupyter_client |

---

## 핵심 패턴 요약

### 1. 판별 합집합을 통한 연결 상태 관리
```typescript
type McpConnection = ConnectedMcpConnection | DisconnectedMcpConnection
```

### 2. 참조 카운팅을 통한 생명주기 관리
```typescript
registerClient() / unregisterClient()
```

### 3. 이중 설정 소스 (Global vs Project)
```typescript
source: "global" | "project"
```

### 4. 디바운스된 설정 변경 감지
```typescript
debounceConfigChange(filePath, source)
```

### 5. 자동 재연결 및 핫 리로드
```typescript
setupFileWatcher() + restartConnection()
```

### 6. 이중 프로토콜 지원
```typescript
// XML: use_mcp_tool
// Native: mcp_{server}_{tool}
```

---

## 참고 파일

| 파일 | 설명 |
|------|------|
| `src/services/mcp/McpHub.ts` | MCP Hub 핵심 구현 |
| `src/services/mcp/McpServerManager.ts` | 싱글톤 매니저 |
| `src/shared/mcp.ts` | MCP 타입 정의 |
| `src/core/tools/UseMcpToolTool.ts` | MCP 도구 호출 도구 |
| `src/core/tools/accessMcpResourceTool.ts` | MCP 리소스 접근 도구 |
| `src/core/prompts/sections/mcp-servers.ts` | 프롬프트 MCP 섹션 |
| `src/core/prompts/tools/use-mcp-tool.ts` | use_mcp_tool 프롬프트 |
