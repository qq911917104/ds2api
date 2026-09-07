# ds2api 框架剥离方案（GLM 网页反代接入）

> 目标：从 `ds2api` 里剥离出一套**与 DeepSeek 无关、可复用的网页大模型反代框架**，
> 让 GLM 网页反代项目（`D:\IDEA Project\QwenHub`）能直接基于这套框架接入智谱清言的网页版上游。
>
> 本文只描述「剥什么 / 留什么 / 哪些缝要抽象」，**不动任何代码**。

---

## 1. 术语约定

- **框架（framework）**：与具体上游无关，任何「网页版大模型 → 标准 HTTP API」网关都能复用的部分。
  包括：协议适配（OpenAI/Claude/Gemini/Ollama）、prompt 归一化、SSE 流消费、工具调用解析、
  tool sieve 防泄漏、账号池与并发调度、鉴权、配置、Admin/WebUI、对话记录。
- **上游专属（provider-specific）**：只对 DeepSeek 网页版有效的部分。
  包括：DeepSeek 登录/会话/completion/文件上传 API、PoW 求解、DeepSeek SSE 报文结构、
  DeepSeek 请求头与指纹、模型 ID 映射。

剥离的判断标准只有一条：**把「DeepSeek」相关的字面量、URL、报文结构、模型名从包里拿掉后，
这个包的代码是否仍语义完整。** 是 → 框架；否 → 上游专属或需抽象。

---

## 2. 现状：框架与 DeepSeek 的耦合点（核心结论）

`ds2api` 的框架和 DeepSeek 专属代码**不是按目录干净分开的**，而是通过 6 个「抽象缝」互相穿透。
剥离框架的工作量 90% 在这 6 个缝上，而不是简单的目录搬移。

### 缝 1：`DeepSeekCaller` 接口泄漏 `dsclient` 类型

`DeepSeekCaller` 接口被**重复定义在 4 处**，且每个都直接引用 `internal/deepseek/client` 的具体类型：

| 定义位置 | 泄漏的 dsclient 类型 |
|---|---|
| `internal/completionruntime/nonstream.go` | `UploadFileRequest/Result`、`DeleteSessionResult`、`SessionStats` |
| `internal/httpapi/openai/shared/deps.go` | 同上 + `DeleteSessionResult`、`SessionStats` |
| `internal/httpapi/claude/deps.go` | `UploadFileRequest/Result` |
| `internal/httpapi/admin/shared/deps.go` | `SessionStats` |

**改造方向**：提一个框架级的 `Provider` 接口（provider 中立类型），
`dsclient.Client` 只作为 DeepSeek 对这个接口的实现，从框架包里彻底消失。

### 缝 2：`internal/sse` 依赖 `deepseek/protocol`

`internal/sse/consumer.go` 与 `parser.go` 都 `import "ds2api/internal/deepseek/protocol"`，
使用了：

- `protocol.ScanSSELines`（SSE 行扫描器——其实通用）
- `protocol.SkipExactPathSet` / `protocol.SkipContainsPatterns`（DeepSeek 特有的 `response/fragments` 路径过滤）
- `protocol.ChatSessionReferer`（DeepSeek Referer 拼装）

也就是说，**「SSE 行解析」是通用的，但「哪些路径是 noise 要跳过」是 DeepSeek 专属的**。
`parser.go` 里大量 `response/thinking_content`、`response/fragments/-1/status` 这类路径判断
也是 DeepSeek SSE 报文结构特有的。

**改造方向**（方向 X）：SSE 行扫描器留在框架（通用）；「path → text/thinking 的分类规则」是
provider 内部的事——DeepSeek 在 Go、GLM 在 Node，不是框架接口。

### 缝 3：`internal/config/models.go` 硬编码 DeepSeek 模型

`GetModelConfig` / `GetModelType` / `IsSupportedDeepSeekModel` / `DefaultModelAliases` / `ResolveModel`
全部把模型写死成 `deepseek-v4-flash/pro/vision/search`，且 `GetModelType` 返回
`"expert"/"default"/"vision"` 这种 DeepSeek 上游才有的 `model_type`。

`DefaultModelAliases` 把 gpt/claude/gemini/qwen/llama 全映射到 `deepseek-v4-*`。

**改造方向**（方向 X）：模型数据归 `Provider.ListModels`，由 provider 内部提供，框架不硬编码任何模型字面量；
跨 provider 的模型名路由归框架 `ModelRouter`（见框架钩子文档 2.4 节）。

### 缝 4：`promptcompat` 的 payload 形状是 DeepSeek 专属

`StandardRequest.CompletionPayload()`（`internal/promptcompat/standard_request.go`）生成的
map 字段全是 DeepSeek completion 接口的形状：

- `chat_session_id`、`parent_message_id`
- `model_type`（`expert/default/vision`）
- `ref_file_ids`（DeepSeek 文件上传概念）
- `thinking_enabled`、`search_enabled`、`action`、`preempt`

而 `StandardRequest` 本身携带 `RefFileIDs`、`CurrentInputFileID`、`CurrentToolsFileID`
这些 DeepSeek「文件作为上下文」的概念。

**改造方向**（方向 X）：payload 渲染是 provider 内部的事——DeepSeek 在 Go、GLM 在 Node，不是框架接口。
文件上传/引用从核心字段里移出，变成 provider 的可选能力（`FileUploader`，后续规划）。

### 缝 5：`internal/server/router.go` 装配 DeepSeek client 与 login

`NewApp()` 里直接 `dsclient.NewClient(...)`，并把 `dsClient.Login` 作为回调注入 `auth.Resolver`。
DeepSeek 的 client、session、PoW 都在这里被焊死。

**改造方向**：`NewApp()` 接收一个「provider 工厂」（GLM 项目提供 GLM 实现），
路由、账号池、鉴权、各协议 handler 都只依赖框架级接口。

### 缝 6：文件上传 / 文件上下文是 DeepSeek 概念

`internal/httpapi/openai/files/`、`internal/httpapi/openai/history/current_input_file.go`、
`internal/promptcompat/file_refs.go` 处理的「文件上传到上游、把 file_id 注入 prompt」是
DeepSeek 网页版的上传 API 语义。GLM 的文件能力（如有）会完全不同。

**改造方向**：files 能力做成 provider 可选能力；框架只保留「本地上传→内容缓存」的通用部分，
「上传到上游拿 file_id」的部分进 provider。

---

## 3. 目录级划分清单

### 3.1 框架（保留，provider 无关，基本不用改）

| 目录 | 说明 |
|---|---|
| `internal/toolcall/` | 工具调用解析（EPSE→XML canonical、CDATA 修复、markdown fence 剥离）——纯文本处理，通用 |
| `internal/toolstream/` | 流式 tool sieve 防泄漏状态机——通用 |
| `internal/prompt/` | prompt 组装（messages → 纯文本）——通用 |
| `internal/util/` | JSON 写入、类型转换、token 计数、thinking 解析——通用 |
| `internal/textclean/` | 文本清洗（`[reference: N]` 标记）——通用 |
| `internal/chathistory/` | 服务器端对话记录持久化——通用 |
| `internal/responsehistory/` | 上游响应归档——通用 |
| `internal/stream/` | 统一流式消费引擎——通用 |
| `internal/format/` | OpenAI/Claude 输出渲染——通用（输入是 `assistantturn.Turn`） |
| `internal/version/` | 版本查询——通用 |
| `internal/webui/` | Admin WebUI 静态托管——通用 |
| `internal/account/` | 账号池、并发槽位、等待队列、弹性号池——通用（见缝 7 说明） |
| `internal/httpapi/requestbody/` | 请求体 UTF-8/JSON 校验——通用 |
| `internal/httpapi/ollama/` | Ollama 模型/能力查询——通用 |

### 3.2 框架（保留，但需按缝抽象改造）

| 目录 | 需改造点 |
|---|---|
| `internal/sse/` | 拆「SSE 行扫描器（通用，留框架）」与「path→content 分类规则（provider 内部：DeepSeek 在 Go、GLM 在 Node）」（缝 2） |
| `internal/promptcompat/` | `CompletionPayload` 迁到 provider 内部（不是框架接口）；文件字段下放为可选能力（缝 4） |
| `internal/assistantturn/` | 基本通用，但因依赖 `sse` 与 `promptcompat` 需跟随改造；`UpstreamEmptyOutputDetail` 里的 `upstream_unavailable`/`account_muted` 错误语义是通用模式，保留 |
| `internal/httpapi/openai/` | 本地 `DeepSeekCaller` 接口改名/收口；`files`/`history` 的文件能力下放 provider（缝 1、6） |
| `internal/httpapi/claude/` | 同上 |
| `internal/httpapi/gemini/` | 同上 |
| `internal/httpapi/admin/` | `admin/shared` 的 `DeepSeekCaller` 收口；admin 里「账号测试」调用 DeepSeek session 的逻辑下放 |
| `internal/config/` | 模型数据迁到 `Provider.ListModels`（provider 内部）；跨 provider 路由归框架 `ModelRouter`（缝 3）；其余配置加载/热更新是通用 |
| `internal/auth/` | 本身通用（login 由外部注入返回 string token），但 `SetAccountMutedUntil`/`SetAccountBanned` 的「muted/banned」语义要确认 GLM 是否有对应概念，否则变成可选 hook |
| `internal/server/` | `NewApp()` 改为接收 provider 工厂（缝 5） |

### 3.3 上游专属（从框架中剔除，留在 ds2api / 移到 provider 目录）

| 目录 | 说明 |
|---|---|
| `internal/deepseek/client/` | DeepSeek 登录、会话、completion、continue、上传、删除会话、PoW、captcha、user settings |
| `internal/deepseek/protocol/` | DeepSeek URL、请求头、skip pattern、locale、Referer |
| `internal/deepseek/transport/` | chrome/httpcloak TLS 指纹传输（**注**：httpcloak 的「浏览器指纹传输」本身可复用，需把 DeepSeek 头常量剥出去，见缝 8） |
| `pow/` | DeepSeek PoW 求解（纯上游专属） |
| `internal/completionruntime/` | **骨架可复用**：空输出重试、切号重试是通用模式；但当前实现里 session/PoW/completion 编排是 DeepSeek 的，需把「重试编排」与「上游调用」拆开 |
| `internal/translatorcliproxy/` | Claude/Gemini↔OpenAI 结构互转桥（Vercel fallback 用），半专属 |
| `internal/devcapture/` | 调试抓包（通用，但当前 capture 类型偏 DeepSeek） |
| `internal/rawsample/` | 上游原始响应采集（通用，偏 DeepSeek） |

---

## 4. 抽象缝汇总表（改造清单）

| # | 缝 | 现状 | 目标 |
|---|---|---|---|
| 1 | Provider 接口 | `DeepSeekCaller` 重复 4 处且泄漏 `dsclient` 类型 | 单一框架级 `Provider` 接口，provider 中立类型 |
| 2 | SSE 分类策略 | `sse` 直接 import `deepseek/protocol` 的 skip pattern 和 path 规则 | 行扫描留框架；内容分类是 provider 内部，不是框架接口 |
| 3 | 模型注册表 | `config/models.go` 硬编码 deepseek 模型 + alias | 模型数据归 `Provider.ListModels`；跨 provider 路由归框架 `ModelRouter` |
| 4 | Payload 渲染 | `StandardRequest.CompletionPayload` 生成 DeepSeek 形状 | payload 渲染是 provider 内部，不是框架接口 |
| 5 | 装配 | `router.go` 焊死 `dsclient.NewClient` + DeepSeek login | `NewApp(providerFactory)`，依赖注入 |
| 6 | 文件能力 | files/history 直接走 DeepSeek 上传 API | 文件能力变成 provider 可选能力 |
| 7 | 账号生命周期 | `account` 与 `auth` 里 muted/banned/elastic 语义 | 保留为框架通用（账号生命周期本就是通用），GLM 有就复用、没有就不触发 |
| 8 | 传输层 | `transport/httpcloak` 与 `protocol/constants` 混着 DeepSeek 头 | 拆「浏览器指纹传输（通用）」与「DeepSeek 头常量（专属）」 |

---

## 5. 目标框架目录结构（草案）

```
D:\IDEA Project\webchat-gateway\          # 建议的框架根目录（可改名）
├── go.mod                                # module 名建议改为独立名，如 github.com/<owner>/webchat-gateway
├── cmd/
│   └── gateway/                          # 框架自带的最小启动入口（provider 通过配置/注册选择）
├── internal/
│   ├── account/                          # 账号池（框架）
│   ├── auth/                             # 鉴权（框架，login 注入）
│   ├── chathistory/                      # 对话记录
│   ├── completion/                       # 空输出重试 + 切号编排（从 completionruntime 抽出的通用骨架）
│   ├── config/                           # 配置（通用；模型数据由 provider 提供）
│   ├── format/                           # 输出渲染
│   ├── httpapi/                          # openai/claude/gemini/ollama/admin/requestbody
│   ├── modelrouter/                      # 跨 provider 模型路由（框架侧，消费 provider 的 matchModel 谓词）（缝 3）
│   ├── prompt/                           # prompt 组装
│   ├── promptcompat/                     # 归一化内核（payload 渲染归 provider 内部）
│   ├── provider/                         # ★ Provider 接口定义（缝 1）
│   ├── server/                           # 路由装配（NewApp(provider)）
│   ├── sse/                              # 行扫描（通用）；分类规则归 provider 内部（缝 2）
│   ├── stream/                           # 流消费
│   ├── toolcall/                         # 工具调用解析
│   ├── toolstream/                       # tool sieve
│   ├── transport/                        # 浏览器指纹传输（从 deepseek/transport 抽出，缝 8）
│   ├── util/                             # 工具函数
│   ├── version/                          # 版本
│   └── webui/                            # Admin UI
└── docs/
```

GLM 项目（`QwenHub`）里再放一个 `provider/glm/`（或独立模块），
实现 `internal/provider` 定义的接口，并提供自己的模型列表、SSE 分类、payload 渲染（均为 provider 内部实现，不共享）。

---

## 6. 分阶段剥离步骤（建议顺序）

1. **先立接口、不删代码**：在 ds2api 内新建 `internal/provider` 包，把 `DeepSeekCaller` 统一成
   `Provider` 接口（provider 中立类型），让 4 处重复定义都指向它；`dsclient.Client` 实现它。跑通全部测试。
2. **收口 SSE 缝**：SSE 行扫描器留框架；path→content 分类规则归 provider 内部（DeepSeek 在 Go、GLM 在 Node），`sse` 包不再 import `deepseek/protocol`。
3. **模型**：模型数据迁到 `Provider.ListModels`；跨 provider 路由归框架 `ModelRouter`。
4. **payload**：渲染归 provider 内部，`CompletionPayload` 从框架包里迁到 provider 目录。
5. **文件能力下放**：files/history 的文件上传路径进 provider 可选能力。
6. **装配解耦**：`NewApp` 接收 provider 工厂，删除对 `dsclient` 的直接引用。
7. **物理搬移**：把 3.1/3.2 的通用目录复制到新框架根目录，`module` 改名，补最小可编译入口。
8. **裁剪**：把 `internal/deepseek/**`、`pow/` 从框架中删除（留在 ds2api），
   DeepSeek 作为框架的第一个「provider 实现」独立成目录/模块。

每步之间跑 `./scripts/lint.sh` 与 `./tests/scripts/run-unit-all.sh`（注意本仓库 Windows 下的
Git Bash / autocrlf / GOSUMDB 坑，见 memory `ds2api-windows-lint-gates`）。

---

## 7. 需要注意的非功能性点

- **framework 与 provider 的依赖方向必须单向**：框架 → `internal/provider` 接口，
  DeepSeek/GLM 实现 → 框架接口，框架绝不能反向 import 任何具体 provider。
- **`completionruntime` 的「空输出重试 + 切号」是框架最值钱的骨架**，
  剥离时把它拆成「编排逻辑（框架）」+「上游调用（provider）」两层，别当成 DeepSeek 专属丢掉。
- **httpcloak 浏览器指纹传输**对 GLM 网页版同样关键，把它作为框架级 `internal/transport`
  保留（只是把 DeepSeek 的 Host/头常量留在 DeepSeek provider 里）。
- 文档 `docs/prompt-compatibility.md` 是「API→纯文本上下文」兼容流程的事实来源，
  剥离后要改写成 provider 中立的描述。
- **后台模板化**（后台页面如何被 provider 插件扩展）单独见
  `docs/admin-template-mechanism.md`，不与本剥离清单掺在一起。
