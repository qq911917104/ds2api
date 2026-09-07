# 后台模板化机制（Slot 扩展点系统）

> 独立设计文档，只讲「后台页面怎么被插件扩展」，不与框架剥离/钩子架构掺在一起。
> 本机制由 `framework-hook-architecture.md` 2.2 节的 `AdminExtension` 钩子承载，
> 本文是它的完整实现说明。

---

## 1. 定位

把后台从「写死的 React 单页应用」变成「**带命名插槽的模板 + 可插拔组件**」。
本质是 PHP 里 WordPress `add_action` / Discuz 插件钩子在 React SPA 上的等价物：

- 框架 = 模板 + 一批命名插槽（挖好的「洞」）
- 每个 provider 插件 = 往洞里塞组件的 bundle
- 加 provider 不重建框架 webui，删 provider 不影响框架默认视图

---

## 2. 四个组成

### 2.1 命名插槽（Slot）——框架在页面里预先挖的「洞」

框架页面模板在写的时候就留出扩展点。第一批预定义插槽：

| 槽名 | 位置 | 说明 |
|---|---|---|
| `account-card` | 账号管理页的账号卡片区 | 每个 provider 一张可切换卡片 |
| `account-action` | 账号卡片内的操作按钮区 | 账号操作 provider 相关（测试/重登/启用） |
| `account-extra` | 账号卡片的扩展状态/详情区 | provider 特有状态（muted/banned/票据页就绪/登录进度） |
| `nav-item` | 侧边导航栏 | 插件可加自己的菜单项 |
| `settings-section` | 设置页的配置分区 | 插件加自己的设置块 |
| `admin-page` | 全新独立页面/路由 | 插件加一个完整子页面 |
| `dashboard-widget` | 总览页统计卡片区 | provider 健康指标卡片 |
| `log-detail` | 请求日志/对话记录详情页的附加面板 | raw sample、抓包、tool trace 等调试数据 |

以账号管理页为例，槽位长这样：

```
账号管理页（框架模板）
├── 标题栏                  ← 固定，插件不可动
├── [account-card 插槽]     ← 允许插件塞组件
│     └── 框架默认渲染：DeepSeek 卡片（Tab 1）
└── 其他固定区域            ← 固定
```

### 2.2 插件注册（Register）——provider 的 JS bundle 声明「我要塞哪」

每个 provider 打一个 `glm-extension.js`，加载后调用全局注册函数：

```js
// provider/glm 的 extension bundle
window.__ds2api_ext.register('glm', {
  slots: {
    'account-card':      { component: 'GlmAccountCard' },
    'nav-item':          { component: 'GlmNavItem' },
    'settings-section':  { component: 'GlmSettings' },
  },
  routes: [
    { path: '/admin/glm', component: 'GlmAdminPage' },
  ],
});
```

`GlmAccountCard`、`GlmAdminPage` 这些组件由 GLM 插件自己实现，与框架组件零共享。

### 2.3 Go 侧提供（`AdminExtension` 钩子）——provider 声明「我有 UI 要挂」

```go
type AdminExtension interface {
    // UIManifest 声明要填哪些插槽 + 对应组件名。
    UIManifest() UIManifest
    // Assets 提供插件的 JS bundle 静态资源，框架挂到 /admin/extensions/<id>/。
    Assets() http.FileSystem
}

type UIManifest struct {
    ID     string
    Name   string
    Slots  []UISlot   // 要填充的插槽
    Bundle string     // bundle 入口路径（相对 /admin/extensions/<id>/）
}

type UISlot struct {
    Slot      string // 如 "account-card"
    Component string // bundle 内导出的组件名
}
```

### 2.4 Shell 动态装配——前端启动时把插件组件挂进插槽

框架 React shell 启动时：

1. 请求 `/admin/extensions` 拿到所有已注册 provider 的 manifest 聚合。
2. 对每个 provider，`import('/admin/extensions/<id>/<bundle>')` 运行时加载 bundle。
3. bundle 加载后调 `window.__ds2api_ext.register(...)` 注册组件。
4. shell 把各插槽的注册组件，与框架默认内容**增量合并**渲染。

渲染结果：

```
账号管理页
├── 标题栏
├── [account-card 插槽]
│     ├── Tab 1: DeepSeek 卡片（框架默认）
│     └── Tab 2: GLM 卡片（GLM 插件塞入）
└── 其他固定区域
```

框架页面本体一行不改；GLM 进来只多一个 bundle 注册，撤掉就删 bundle。

---

## 3. 端到端例子：GLM 账号卡片

**框架侧（已有，不动）**：账号管理页模板里，`account-card` 插槽默认渲染 DeepSeek 卡片。

**GLM 插件侧（新增）**：

1. Go 注册 `AdminExtension`，manifest 声明 `{slot: "account-card", component: "GlmAccountCard"}`，Assets 提供 `index.js`。
2. `index.js` 里 export 一个 `GlmAccountCard` 组件（表单 + 列表 + 登录进度）。
3. `index.js` 自执行时调 `register('glm', { slots: { 'account-card': { component: 'GlmAccountCard' } } })`。

**结果**：账号管理页自动出现「DeepSeek / GLM」两个 Tab，切换即可分别管理两类账号。DeepSeek 卡片无感知。

---

## 4. 落地关键点

1. **增量合并，不是覆盖**。框架默认组件永远渲染，插件组件 append 进去——保证框架自带可用默认视图，插件只是加东西。

2. **运行时动态 import**。bundle 用 `import('/admin/extensions/...')` 运行时加载，不进编译期打包。否则每加一个 provider 就要重构建整个 webui，失去「想拼哪拼哪」的意义。

3. **bundle 之间隔离**。每个 provider 的 bundle 是独立 ES module，互不 import；组件名冲突由「provider id + 组件名」双重命名空间规避。

4. **降级安全**。某个 provider 的 bundle 加载失败，只影响该 provider 的插槽为空，不拖垮整个后台。

---

## 5. 与框架文档的关系

- `framework-hook-architecture.md` 2.2 节：`AdminExtension` 钩子的接口定义 + 指向本文档。
- `framework-extraction-plan.md`：剥离清单里 `webui/` 标注「后台模板化见 admin-template-mechanism.md」。
- 本文档单独维护，独立实现，不与上述两份掺在一起。
