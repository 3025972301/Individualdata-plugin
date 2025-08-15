# Individualdata-plugin – 插件开发指南

个体数据插件库 · 插件开发文档

## 目录

* [1. 插件系统概览](#1-插件系统概览)
* [2. 插件元数据](#2-插件元数据)
* [3. 插件开发流程](#3-插件开发流程)
* [4. 实战示例](#4-实战示例)

  * [4.1 全局外观插件（Theme）](#41-全局外观插件theme)
  * [4.2 单页面外观插件（Component 样式定制）](#42-单页面外观插件component-样式定制)
  * [4.3 新页面 + API 插件（动态路由与数据写入）](#43-新页面--api-插件动态路由与数据写入)
* [5. 安全与注意事项](#5-安全与注意事项)
* [6. API 约定一览](#6-api-约定一览)

---

## 1. 插件系统概览

* 所有插件数据存放于 **MongoDB** 的 `plugins` 集合，并通过 **`/api/plugins`** 统一管理**安装 / 启用 / 停用**。
* 应用启动时，**`PluginManager`** 会从 **`/api/plugin-gateway`** 拉取所有**已激活插件**并按顺序执行：

  * 将 `code.css` 注入到 **`<head>`**，改变全局样式；
  * 在受限沙箱中执行 `code.js`（沙箱中提供 `app`、`axios` 等对象）。

---

## 2. 插件元数据

下表为核心字段（完整结构可参考 `sample-plugin.json`）：

| 字段            | 类型         |  必填 | 说明                                          |
| ------------- | ---------- | :-: | ------------------------------------------- |
| `id`          | `string`   |  ✔️ | 插件唯一标识（需全局唯一）                               |
| `name`        | `string`   |  ✔️ | 插件名称                                        |
| `version`     | `string`   |     | 版本号，语义化推荐                                   |
| `type`        | `string`   |  ✔️ | 插件类型（如 `theme` / `component` / `page` 等）    |
| `code.js`     | `string`   |     | 将要在沙箱内执行的脚本                                 |
| `code.css`    | `string`   |     | 将注入 `<head>` 的样式                            |
| `permissions` | `string[]` |     | 需要的权限（如 `database.insert`、`database.query`） |
| `status`      | `string`   |     | 插件状态（如 `active` / `inactive`）               |

---

## 3. 插件开发流程

1. **编写插件 JSON**：遵循上述字段结构，确保 `id` 唯一。
2. **上传**：管理员在 **系统 → 插件管理** 弹窗中选择 JSON，点击“上传插件”完成存储。
3. **启用/停用**：在同一列表中切换开关；启用后 `PluginManager` 会**重新加载**已激活插件。
4. **运行**：

   * 激活后，CSS/JS **立即生效**；
   * 涉及数据库等敏感操作时，通过 **`/api/plugin-gateway`** 调用，需**携带相应权限**。

> 💡 提示：样式覆盖建议**尽量使用更精准的选择器**，以免影响到非目标页面或组件。

---

## 4. 实战示例

### 4.1 全局外观插件（Theme）

```json
{
  "id": "glass-theme",
  "name": "液态玻璃主题",
  "version": "1.0.0",
  "type": "theme",
  "code": {
    "css": "body { backdrop-filter: blur(10px); background: rgba(255,255,255,0.2); }"
  },
  "permissions": []
}
```

启用后，`PluginManager` 会将 `code.css` 注入页面，实现全局“玻璃化”效果。

---

### 4.2 单页面外观插件（Component 样式定制）

```json
{
  "id": "home-dark-card",
  "name": "首页卡片暗化",
  "type": "component",
  "code": {
    "css": "#app .home-page .v-card { background-color: #1e1e1e !important; }"
  }
}
```

只需针对**特定页面根节点**添加选择器，即可将影响范围限制在该页面。

---

### 4.3 新页面 + API 插件（动态路由与数据写入）

**可读版（演示模板字符串，便于理解）：**

```json
{
  "id": "notes-page",
  "name": "笔记页面",
  "type": "page",
  "permissions": ["database.insert", "database.query"],
  "code": {
    "js": "`const Notes = {\
      \n  template: '<div><h1>笔记</h1><input v-model=\\\"text\\\"/><button @click=\\\"save\\\">保存</button></div>',\
      \n  data(){ return { text: '' } },\
      \n  methods:{\
      \n    async save(){\
      \n      await axios.post('/api/plugin-gateway', {\
      \n        pluginId: 'notes-page',\
      \n        operation: 'database.insert',\
      \n        data: { collection: 'notes', document: { text: this.text } }\
      \n      })\
      \n    }\
      \n  }\
      \n};\
      \napp.component('Notes', Notes);\
      \napp.config.globalProperties.$router.addRoute({ path: '/notes', component: Notes });`"
  }
}
```

**严格 JSON 版（可直接复制使用）：**

```json
{
  "id": "notes-page",
  "name": "笔记页面",
  "type": "page",
  "permissions": ["database.insert", "database.query"],
  "code": {
    "js": "const Notes = {\\n  template: '<div><h1>笔记</h1><input v-model=\\\"text\\\"/><button @click=\\\"save\\\">保存</button></div>',\\n  data(){ return { text: '' } },\\n  methods:{\\n    async save(){\\n      await axios.post('/api/plugin-gateway', {\\n        pluginId: 'notes-page',\\n        operation: 'database.insert',\\n        data: { collection: 'notes', document: { text: this.text } }\\n      })\\n    }\\n  }\\n};\\napp.component('Notes', Notes);\\napp.config.globalProperties.$router.addRoute({ path: '/notes', component: Notes });"
  }
}
```

> ✅ 效果：插件加载时**动态注册组件与路由**，新增 `/notes` 页面；通过 `pluginId + operation` 调用 `/api/plugin-gateway` 写入或查询数据，后端会根据 `permissions` 进行权限校验。

---

## 5. 安全与注意事项

* 当前沙箱使用 `new Function` + `with` 的方式，**隔离能力有限**；请避免引入危险调用与全局副作用。
* 需要访问数据库或其他敏感能力的插件，务必在 `permissions` 中**显式声明**，并通过后台审核再激活。
* 插件上传与启停均需**管理员登录**并携带 **JWT Token**。
* 更复杂的扩展（主题 / 功能库 / UI 组件等）可在 `code.js` 中使用 `app` 与全局 Router，或在 `code.css` 中进行样式覆盖。

> 🔒 最佳实践：
>
> * 默认最小权限（**Least Privilege**）；
> * 关键操作经由后端网关统一执行业务校验；
> * 对第三方依赖进行来源与完整性核验。

---

## 6. API 约定一览

* **`/api/plugins`**：插件的**创建 / 查询 / 启停**管理接口（需管理员）。
* **`/api/plugin-gateway`**：插件侧发起的**受控能力调用**与**数据访问**统一网关（后端按 `pluginId` + `operation` 与 `permissions` 进行授权与审计）。
