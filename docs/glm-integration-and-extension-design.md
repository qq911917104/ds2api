# GLM 集成与框架扩展设计（二改文档 · 方案 A）

> 本文档回答：GLM（chat.z.ai）网页反代如何以**方案 A（Node 侧车 = 完整 chat 管线）**接入 ds2api 框架。
> 方案 A 已拍板：当前 GLM 功能最大复用 QwenHub 已验证的 TS 代码，Go 侧只做「薄壳 + 侧车桥」；
> 后续新增 GLM 能力时再按情况选原生 Go（方案 B 混用）还是继续 Node。
> GLM 逆向成果来源：`D:\IDEA Project\QwenHub\docs\GLM_PROTOCOL.md`、`specs/glm-provider/`、`.qwen/glm-capture/`。

---

## 1. 方案 A 总览：Go 薄壳 + Node 侧车

```
客户端 ──▶ 框架（Go：路由/鉴权/账号池/后台/历史记录）
              │
              │  框架 Provider 接口（对 GLM 是「侧车适配器」实现）
              ▼
        glm 侧车适配器（Go 薄壳）
              │
              │  JSON-Lines over stdin/stdout（常驻子进程）
              ▼
        node glm-sidecar.mjs（常驻 Node 进程，跑 QwenHub 的 handleChat）
          收票 + 签名 + 发送 + SSE 解析 + 工具标签 + 换号重试
```

- **Go 侧**：GLM 的 `Provider` 是一个薄壳，每个方法把请求编码成 JSON 发给常驻 Node 子进程，再解码返回。
- **Node 侧**：`index.ts` 的 `handleChat` 管线（`captcha.ts` 票据工厂 + `signature.ts` 签名 +
  `transport.ts` 发送 + `sse.ts` 解析 + `toolTagParser.ts`/`toolStream.ts` 工具标签 + 换号重试）几乎原样复用。
- **框架职责不变**：路由、鉴权、账号池、后台 slot、聊天历史、响应历史仍是 Go 框架提供。

## 2. Provider 接口要粗粒度化（方案 A 带来的关键简化）

DeepSeek 现状的 `DeepSeekCaller` 是**DeepSeek 专属的细粒度**接口
（`CreateSession` / `GetPow` / `CallCompletion` / `StopStream` / `FireCompletionAndStop`），
GLM 侧车根本对不上这套 session/PoW 概念。

方案 A 下，框架的 `Provider` 接口应定义为**粗粒度、上游中立**的形态：

```go
package provider

type Provider interface {
    Name() string
    // Login 建立账号登录态（阻塞；GLM 侧车内部跑半自动登录 + 人工滑块）。
    Login(ctx context.Context, acc config.Account) (token string, err error)

    // Chat 发起一次对话，把 OpenAI 兼容的标准化请求交给上游实现，
    // 通过事件流回传。流式与非流式统一由实现方决定（框架按需聚合）。
    Chat(ctx context.Context, req ChatRequest) (*ChatStream, error)

    // ListModels 返回该上游对外暴露的模型列表。
    ListModels(ctx context.Context) ([]ModelInfo, error)

    // ClassifyError 把错误归类为处置动作（换号重试/冷却/直接失败），
    // 供框架的切号重试编排使用。
    ClassifyError(err error) ErrorAction
}

// ChatRequest 是框架中立的对话请求（由 promptcompat 归一化产出）。
type ChatRequest struct {
    Model    string
    Messages []Message
    Stream   bool
    Tools    any
    // ... thinking / search / 等中立字段
}

// ChatStream 是标准化的事件流（reasoning / content / tool_call / usage / done / error）。
type ChatStream interface {
    Next(ctx context.Context) (Event, error)
    Close() error
}
```

- **DeepSeek 原生实现**：`Chat` 内部在 Go 里跑 session/PoW/completion/SSE 解析/重试（现有 `completionruntime` 逻辑收进 DeepSeek 插件）。
- **GLM 侧车实现**：`Chat` 把 `ChatRequest` 序列化发给 Node 侧车，把 Node 回传的 OpenAI chunk 流解码成 `ChatStream`。

框架的切号重试、空输出重试、tool sieve、usage 统计全部消费这个粗粒度 `ChatStream`，
对 DeepSeek 和 GLM 一视同仁——这就是「一套统一运行时」。

## 3. Node 侧车桥协议（新增，方案 A 核心）

### 3.1 进程与通信

- Go 启动常驻子进程：`node glm-sidecar.mjs`（由 `provider/glm/sidecar/` 打包）。
- 通信：**JSON-Lines over stdin/stdout**，一行一个请求/响应，避免引入 HTTP 端口。
- 生命周期：Go 随服务启动拉起，优雅关机时发 `shutdown` 后关闭。

### 3.2 请求类型

| 方法 | 入参 | 出参 | 说明 |
|---|---|---|---|
| `login` | `{email, password, onStatus}` | `{ok, token, error}` | 半自动登录，`onStatus` 走进度回调回传「凭证已填入/检测到滑块」 |
| `chat` | `{requestId, model, messages, stream, tools}` | 流式：逐行 `{type:"chunk", delta}`；结束 `{type:"done"}` | 完整 chat 管线，Node 内部收票+签名+发送+解析+换号 |
| `models` | `{token}` | `{models:[...]}` | 拉 `/api/models` |
| `verify` | `{token}` | `{ok}` | token 有效性探针 |
| `shutdown` | — | `{ok}` | 关掉票据页与浏览器 |

### 3.3 协议骨架（示意）

```jsonc
// Go → Node
{"id":"req-1","method":"chat","params":{"model":"glm-5.3","messages":[...],"stream":true}}
// Node → Go（流式，逐行）
{"id":"req-1","type":"chunk","delta":{"role":"assistant"}}
{"id":"req-1","type":"chunk","delta":{"content":"..."}}
{"id":"req-1","type":"chunk","delta":{"reasoning_content":"..."}}
{"id":"req-1","type":"usage","usage":{"prompt_tokens":10,"completion_tokens":20}}
{"id":"req-1","type":"done","finish_reason":"stop"}
```

> 侧车桥只做「把 OpenAI 兼容 chunk 透明转发」，不做任何 GLM 语义判断——
> 语义（相位映射、工具标签、错误分类）都在 Node 侧 `handleChat` 里，复用 QwenHub。

## 4. 复用地图（方案 A：现在复用 vs 以后按需移植）

### 4.1 现在直接复用（Node 侧车，零移植）

| QwenHub 模块 | 落在哪 | 说明 |
|---|---|---|
| `index.ts` 的 `handleChat` 管线 | `glm-sidecar.mjs` | 完整 chat 管线，含换号重试 |
| `captcha.ts` 票据工厂 | `glm-sidecar.mjs` | 每账号常驻无头页产票 |
| `account.ts` 半自动登录 | `glm-sidecar.mjs` | Playwright 弹窗 + 人工滑块 + 截获 token |
| `signature.ts` | `glm-sidecar.mjs` | 双层 HMAC，6/6 已验证 |
| `transport.ts` | `glm-sidecar.mjs` | URL/请求体/签名组装 + wreq-js 指纹传输 |
| `sse.ts` | `glm-sidecar.mjs` | phase→GatewayEvent 映射 |
| `toolTagParser.ts` / `toolStream.ts` | `glm-sidecar.mjs` | 工具标签 7 变体 + 流式状态机 |
| `context.ts` | `glm-sidecar.mjs` | 多轮拍平 |
| `constants.ts` | `glm-sidecar.mjs` | 端点/场景 ID/设备 ID |

**GLM 的一切上游逻辑现在都留在 Node，一条都不移植 Go。**

### 4.2 框架替代（QwenHub 对应代码废弃）

| QwenHub 模块 | 结论 |
|---|---|
| `accountManager.ts` 账号池 | 框架 `internal/account` 替代（需加 provider 分组） |
| `services/auth.ts` 鉴权 | 框架替代 |
| `routes/chat.ts` / `anthropic.ts` | 框架协议外壳替代 |
| `logStore` / `configService` | 框架替代 |
| `dashboard/*` | 框架 Admin + AdminExtension 替代 |

### 4.3 以后按需移植（方案 B，暂不做）

以下等后续要「去 Node 依赖 / 性能敏感 / 想让框架重试逻辑对 GLM 生效」时，再翻译成
Go 放进 `provider/glm/`（各自独立目录，与 DeepSeek 零共享，见 `framework-hook-architecture.md` 第 7 节）：

- `signature.ts` → `provider/glm/signature.go`（纯函数，最好移植）
- `sse.ts` → `provider/glm/sse.go`（实现框架 `SSEAdapter`）
- `toolTagParser.ts` → `provider/glm/toolcall.go`（实现框架 `ToolCallParser`）
- `toolStream.ts` → `provider/glm/sieve.go`（实现框架 `ToolStreamSieve`）
- `context.ts` → `provider/glm/normalize.go`（实现框架 `MessageNormalizer`）

## 5. 框架要改的点（方案 A 下收敛为两点）

1. **账号池多 provider 分组**（唯一稳定的硬改）：`internal/account` 加 `provider` 维度，
   `Pool` 按 provider 分组调度，否则 GLM 和 DeepSeek 账号混池轮询。
2. **粗粒度 Provider 接口**：把 DeepSeek 专属的 `DeepSeekCaller` 换成第 2 节的
   上游中立接口；DeepSeek 的 `completionruntime` 逻辑收进 `provider/deepseek` 作为原生实现，
   GLM 用侧车适配器实现同一接口。

> 原方案 B 里的另外两个硬点（`Prepare` 有状态后台产票、登录进度回调）在方案 A 下
> **不再是框架要改的事**——它们都在 Node 侧车内部处理，`login` 的进度回调走侧车桥的
> `onStatus` 通道回传，由 AdminExtension 展示。

## 6. AdminExtension（后台 slot 模板化，方案 A/B 通用）

后台做成「命名槽位 + 插件注册」，GLM 账号卡片、登录进度面板都是插进去的，框架页面本体零改动：

```go
type AdminExtension interface {
    UIManifest() UIManifest   // 声明往哪些槽塞什么组件
    Assets() http.FileSystem  // 插件 JS bundle，挂 /admin/extensions/<id>/
}
```

账号管理页的 DeepSeek 卡片是框架自带，GLM 往 `account-card` 槽再注册一张 → 页面变成可切换双卡片。

## 7. 落地路线

1. **立粗粒度 Provider 接口 + 账号池 provider 分组**，跑通 ds2api 现有全部测试（DeepSeek 行为不变）。
2. **把 DeepSeek 现有 `completionruntime` 收进 `provider/deepseek` 原生实现**，验证与剥离前一致。
3. **搭侧车桥**：Go 薄壳 `provider/glm` + `glm-sidecar.mjs`（复用 QwenHub handleChat），
   先跑通 `chat`（非流式）→ `models` → `login`。
4. **流式 + 工具标签**：侧车桥透传 OpenAI chunk，验证流式 SSE 与 tool_calls。
5. **AdminExtension**：GLM 账号卡片（可切换第二卡片）+ 登录进度面板。
6. **收口**：GLM 作为独立插件目录 `provider/glm/`，引用框架接口；ds2api 功能不变。

每步之间跑 ds2api 的 lint + 单测门禁（Windows 下 Git Bash + autocrlf + GOSUMDB 注意点见 memory）。
