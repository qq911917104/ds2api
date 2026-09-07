# 框架 Hook / 插件架构（Provider 扩展点）

> 补充说明：上一份 `framework-extraction-plan.md` 是「剥什么 / 留什么」的清单。
> 本文定义**框架长什么样**——即「hook 钩子」：框架是一台**引擎**，
> 每个网页反代上游（DeepSeek / GLM / 未来的）都是一块**插件**。
>
> **关键定调（方向 X，已拍板）**：工具解析、SSE 解析、消息拍平、payload 构造、
> 签名/票据/会话这些**不是钩子，是 provider 内部的事**。DeepSeek 在 Go 里做、
> GLM 在 Node 侧车里做，各用各的，零共享。框架只透传标准化事件流，不感知这些细节。

---

## 1. 设计目标

- **框架只认接口，不认上游**：框架代码里不出现 `deepseek` / `glm` 字面量。
- **一个上游 = 一个 Plugin**：DeepSeek 是第一块插件，GLM 是第二块。
- **可无缝重集成**：`ds2api` = 框架 + deepseek 插件；`QwenHub` = 框架 + glm 插件。

依赖方向严格单向：

```
框架(framework) ──定义──▶ 扩展点接口(provider)
      ▲                          ▲
      │ 实现                      │ 实现
deepseek 插件                glm 插件
```

框架绝不 import 任何具体插件；插件 import 框架接口。

---

## 2. 真正的钩子（框架接口，就这几个）

方向 X 下，钩子面大幅收缩。**只有 4 个**（另有框架侧路由机制 `ModelRouter`，见 2.4，它不是 provider 钩子）：

| 钩子 | 职责 | 是否必实现 |
|---|---|---|
| `Provider` | 登录 / 对话（返回标准化事件流）/ 模型列表 / 错误分类 | ✅ 核心 |
| `AdminExtension` | 后台 slot 扩展（账号卡片、导航、设置面板、整页） | 可选 |
| `BrowserDriver` | 浏览器自动化（登录弹窗 + 无头票据页） | 可选，方案 B 才启用 |
| `FileUploader` | 上传文件拿引用注入上下文 | 可选，**后续规划（暂未实现）** |

> 文件上传：不是「不做」，是「目前还没做」。将来做的时候作为 `Provider` 的
> 可选能力钩子加进来，不是从框架里删掉。

### 2.1 `Provider`（核心钩子，粗粒度）

```go
package provider

type Provider interface {
    Name() string

    // Login 建立账号登录态（阻塞；GLM 侧车内部跑半自动登录 + 人工滑块，
    // 进度通过事件回传）。
    Login(ctx context.Context, acc config.Account) (token string, err error)

    // Chat 发起一次对话，把标准化请求交给上游实现，返回标准化事件流。
    // 流式与非流式统一由实现方决定，框架按需聚合。
    Chat(ctx context.Context, req ChatRequest) (*ChatStream, error)

    // ListModels 返回该上游对外暴露的模型列表。
    ListModels(ctx context.Context) ([]ModelInfo, error)

    // ClassifyError 把错误归类为处置动作（换号重试/冷却/直接失败）。
    ClassifyError(err error) ErrorAction
}

// ChatStream 是标准化事件流：reasoning / content / tool_call / usage / done / error。
// tool_call 事件由 provider 内部解析完成后回传，框架只透传，不再二次解析。
type ChatStream interface {
    Next(ctx context.Context) (Event, error)
    Close() error
}
```

### 2.2 `AdminExtension`（后台 slot 模板化）

```go
type AdminExtension interface {
    UIManifest() UIManifest   // 声明往哪些槽塞什么组件
    Assets() http.FileSystem  // 插件 JS bundle，挂 /admin/extensions/<id>/
}
```

前端 shell 暴露命名槽位（`account-card` / `nav-item` / `settings-section` / `admin-page`），
插件往槽里注册组件。账号管理页的 DeepSeek 卡片是框架自带，GLM 往 `account-card`
再注册一张 → 页面变成可切换双卡片，框架页面本体零改动。
完整机制（插槽 + 注册 + 动态装配）见独立文档 `admin-template-mechanism.md`。

### 2.3 `BrowserDriver`（可选，方案 B 才启用）

方案 A 下 GLM 的浏览器自动化全在 Node 侧车内部，Go 侧**不需要**这个钩子。
等将来要「去 Node 依赖、改用 Go 原生 chromedp」时（方案 B 混用）再启用：

```go
type BrowserDriver interface {
    StartPersistent(ctx context.Context, profileDir string, opts BrowserOptions) (BrowserSession, error)
}
```

### 2.4 `ModelRouter`（跨 provider 模型名路由，框架侧机制）

客户端发 `model: "glm-5.3"`，框架怎么知道交给 GLM、`"gpt-4o"` 交给 DeepSeek？
这就是模型名路由。它**不是 provider 钩子**，而是框架持有的调度机制，
消费每个 provider 在注册时提供的 `matchModel` 认领谓词：

- **框架**：持有 provider 注册表，拿 `model` 逐个问「你认吗」，第一个认领的接管。
- **provider**：只声明「我认哪些模型名」（`matchModel(model) bool`），不参与「谁来选」。

```go
// 框架侧
type Registry struct {
    entries []entry
}

type entry struct {
    p     Provider
    match func(model string) bool
}

func (r *Registry) Register(p Provider, match func(string) bool) {
    r.entries = append(r.entries, entry{p, match})
}

// 按注册顺序，第一个认领的接管（对齐 QwenHub registry 先注册先匹配）
func (r *Registry) Route(model string) (Provider, bool) {
    for _, e := range r.entries {
        if e.match(model) {
            return e.p, true
        }
    }
    return nil, false
}
```

装配示例（DeepSeek 认 deepseek* 及其别名，GLM 认 glm* / x-preview*）：

```go
reg := provider.NewRegistry()
reg.Register(deepseekPlugin, func(m string) bool {
    return isDeepSeekOrAlias(m) // deepseek-v4-* 及 gpt-4o / claude 等别名
})
reg.Register(glmPlugin, func(m string) bool {
    return strings.HasPrefix(m, "glm") || strings.HasPrefix(m, "x-preview") // ...
})
```

每个 chat 入口先 `reg.Route(model)` 拿到 provider 再 `provider.Chat(...)`；
`GET /v1/models` 遍历所有 provider 调 `ListModels` 聚合。框架零感知上游模型名，
又能正确分单、不互相抢名。

---

## 3. Provider 内部（各用各的，不是钩子）

这些**不进入框架接口**，DeepSeek 在 Go 插件目录做、GLM 在 Node 侧车做，零共享：

| 能力 | DeepSeek（Go 插件内部） | GLM（Node 侧车内部） |
|---|---|---|
| 会话 / PoW / 签名 | session + PoW | createGlmChat + 双层 HMAC |
| SSE 解析 | path 分类 | phase 分类（`sse.ts`） |
| 工具解析 | EPSE/XML/CDATA | `<tool>`/`<tool_call>` 7 变体（`toolTagParser.ts`） |
| 流式工具防泄漏 | sieve 状态机 | `toolStream.ts` |
| 消息拍平 | history transcript | `context.ts` 拍平单条 user |
| payload 构造 | chat_session_id | signature_prompt / chat_id |
| 票据 / captcha | captcha 检测 | 每账号无头票据工厂 |
| 错误分类映射 | muted/captcha 判定 | 401/429/CONCURRENCY_LIMIT 判定 |

**框架完全不感知上面这些**。框架唯一看到的是 `Provider` 接口返回的标准化 `ChatStream`。

---

## 4. 框架共享（非钩子，纯通用层）

这些代码不含任何上游字面量，是框架本体：

- `internal/account` —— 账号池（含 **provider 分组** 维度）
- `internal/auth` —— 鉴权 / API key / token 生命周期
- `internal/server` —— 路由装配、中间件
- `internal/httpapi/*` —— OpenAI / Claude / Gemini / Ollama 协议外壳
- `internal/config` —— 配置加载、热更新（模型表由 provider 提供）
- `internal/chathistory` / `internal/responsehistory` —— 持久化
- 切号重试 / 空输出重试编排 —— 只消费 `Provider` 接口，不认上游
- 后台骨架 + slot 系统

---

## 5. 与剥离清单的关系

`framework-extraction-plan.md` 里的 8 条抽象缝，在方向 X 下的归属：

| 抽象缝 | 方向 X 归属 |
|---|---|
| 重复的 `DeepSeekCaller` | → 粗粒度 `Provider` 钩子 |
| `sse` 依赖 protocol | → **provider 内部**（不是钩子） |
| 硬编码模型 | → 模型数据归 `Provider.ListModels`；跨 provider 路由归框架 `ModelRouter`（见 2.4） |
| payload 形状 | → **provider 内部**（不是钩子） |
| 装配焊死 | → `NewApp(plugin)` 依赖注入 |
| 文件能力 | → `FileUploader` 可选钩子（**后续规划**） |
| 账号生命周期 | → 框架账号池（provider 分组） |
| 传输层 | → **provider 内部**（不是钩子） |

---

## 6. 结论

> 钩子只有 4 个：`Provider`（核心）、`AdminExtension`、`BrowserDriver`（可选）、
> `FileUploader`（可选，后续规划）。工具解析、SSE 解析、拍平、payload、签名、
> 票据——全部是 provider 内部的事，各用各的，零共享，不是钩子。
