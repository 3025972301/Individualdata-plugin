# IndividualData-plugin – 插件开发指南 V2

个体数据插件库 · 插件开发文档

> 本指南整合了原有的增强插件、全局扩展、调试与 Vue 插件文档，面向需要编写、维护或让 AI 自动生成插件的开发者。阅读本文后，你应能够清晰地描述需求，并据此构建出完整的插件方案。

## 目录
1. [系统概览](#系统概览)
2. [插件包结构](#插件包结构)
3. [插件类型与适用场景](#插件类型与适用场景)
4. [Vue/页面型插件编写流程](#vue页面型插件编写流程)
5. [功能型与主题型插件编写](#功能型与主题型插件编写)
6. [插件 API 速查表](#插件-api-速查表)
7. [全局扩展能力](#全局扩展能力)
8. [数据管理与持久化](#数据管理与持久化)
9. [生命周期与钩子](#生命周期与钩子)
10. [安全策略与权限](#安全策略与权限)
11. [调试与排错清单](#调试与排错清单)
12. [插件配置与全局设置](#插件配置与全局设置)
13. [最佳实践与 AI 提示词模板](#最佳实践与-ai-提示词模板)

---

## 系统概览
- 插件以 JSON 配置形式存储和分发，支持运行时加载、验证与卸载。
- 支持 **组件 (component/vue-component)**、**页面 (page)**、**功能 (function)**、**主题 (theme)** 四大类型，均可附带脚本、样式、HTML、路由和菜单定义。
- 每个插件都可以调用统一的 `$pluginAPI`，访问 Vue/Vuetify 能力、系统扩展接口、数据存储与工具函数。
- 通过全局扩展系统，可在任何页面注入悬浮按钮、小部件、模态框、工具栏按钮、插槽内容以及任意 DOM 片段，无需宿主页面提前集成。
- V2插件系统支持V1时期的插件，V2插件系统在IndividualData 1.7版本中开始使用。

## 插件包结构
典型插件对象字段：

| 字段 | 说明 |
| --- | --- |
| `id` | 插件唯一 ID（建议全局唯一、语义化）。 |
| `name` / `description` / `author` | 基本信息，用于展示和管理。 |
| `version` | 语义化版本号，便于升级管理。 |
| `type` | `component` / `vue-component` / `page` / `function` / `theme`。 |
| `status` | `active` 时才会加载执行。 |
| `dependencies` / `permissions` | 记录依赖与权限声明（目前未强制校验，但建议准确填写）。 |
| `config` | 自定义配置对象，可在插件内部读取。 |
| `route` | 页面型插件的路由定义（`path`、`name`、`title`、`meta` 等）。 |
| `menu` | 页面型插件的菜单配置（`title`、`icon`、`order`）。 |
| `component` | Vue 组件定义（模板、数据、方法、计算属性、监听器、生命周期等）。 |
| `code` | 额外的 JS/CSS/HTML 代码块，用于功能增强或向后兼容旧插件。 |
| `theme` | 主题插件的主题配置（暗色开关、颜色表）。 |
| `hooks` | 生命周期钩子，目前支持 `onInit` 字段。字符串将作为 JS 代码沙箱执行。 |

> **快速生成模板：** 使用 `src/plugins/PluginTemplate.js` 中的模板函数可生成不同类型的初始结构。

## 插件类型与适用场景

| 类型 | 特点 | 常见场景 |
| --- | --- | --- |
| Vue 组件 (`component`/`vue-component`) | 编译为全局 Vue 组件，可注册路由、使用 Vuetify，支持完整响应式能力。 | 仪表板、数据录入表单、复杂交互组件。 |
| 页面 (`page`) | 在组件基础上额外注册路由与菜单。 | 独立业务页面、管理后台模块。 |
| 功能 (`function`) | 以 JS/CSS/HTML 注入方式运行，可调用全局扩展 API。 | 浮动工具、快捷操作、页面增强脚本。 |
| 主题 (`theme`) | 注入样式 + 可选主题配置/脚本，实现 UI 统一定制。 | 深浅色主题、品牌配色、全局样式覆盖。 |

## Vue/页面型插件编写流程
1. **准备组件定义**：`component` 字段必须是对象或 JSON 字符串，包含 `template`、`data`、`methods`、`computed` 等。编译时会将字符串函数转换为真实函数并注入 `$pluginAPI`。
2. **使用 Vuetify**：所有 Vuetify 组件已自动注册，可直接在模板中引用，也可以通过 `$pluginAPI.vuetify` 访问主题、显示与本地化信息。
3. **注册路由与菜单**：页面插件会自动调用 `registerPluginRoute` 并在必要时重新挂载通配符路由，同时可将菜单项派发给宿主应用。
4. **附加代码与样式**：如果需要额外逻辑，可在 `code.js` 中调用 `$pluginAPI` 或浏览器 API；`code.css` 会作为独立 `<style>` 注入；`code.html` 会经过净化后插入页面。
5. **访问插件上下文**：在组件 `setup` 中可通过注入获得 `pluginAPI` 与 `plugin` 元信息，也会在 `created` 和 `mounted` 生命周期中输出日志，便于调试。

### 示例：页面插件最小骨架
```json
{
  "id": "analytics-dashboard",
  "name": "数据分析仪表板",
  "type": "page",
  "status": "active",
  "route": { "path": "/plugins/analytics", "title": "数据分析" },
  "menu": { "title": "数据分析", "icon": "mdi-chart-line", "order": 120 },
  "component": {
    "template": "<v-container><v-card><v-card-title>{{ title }}</v-card-title></v-card></v-container>",
    "data": { "title": "欢迎使用数据分析" },
    "methods": {
      "notify": "function() { this.$pluginAPI.system.showNotification('加载完成', 'success'); }"
    },
    "mounted": "function() { this.notify(); }"
  }
}
```

## 功能型与主题型插件编写
- **功能插件**：重点在 `code.js` 中执行逻辑，可监听 DOM、发送事件或调用 `$pluginAPI.system` 以创建扩展按钮、模态框等。`code.css`/`code.html` 用于界面补充。
- **主题插件**：除 JS/CSS 外，可通过 `theme` 字段切换暗色模式并覆盖 Vuetify 颜色表，实现全局配色统一。
- **模板参考**：`PluginTemplate.js` 提供功能型模板示例（数据处理、API 集成等），可作为 AI 生成代码的提示。

## 插件 API 速查表

### Vue 能力
| 方法 | 说明 |
| --- | --- |
| `vue.createComponent(definition)` | 运行时注册组件。 |
| `vue.registerRoute(route)` | 手动注册额外路由。 |
| `vue.useRouter()` | 获取 Vue Router 实例。 |
| `vue.useVuetify()` | 获取 Vuetify 实例。 |
| `vue.reactive/ref/computed/watch/onMounted/onUnmounted` | 直接转发 Vue Composition API。 |

### Vuetify 能力
| 字段 | 说明 |
| --- | --- |
| `vuetify.components` | 可用的 Vuetify 组件名称清单。 |
| `vuetify.theme` / `vuetify.display` / `vuetify.locale` | 访问主题、响应式断点和国际化信息。 |

### 系统扩展
| 方法 | 作用 |
| --- | --- |
| `system.addMenuItem(item)` | 向宿主应用菜单派发新入口。 |
| `system.addToolbarButton(button)` | 添加工具栏按钮（若失败自动降级为悬浮按钮）。 |
| `system.addFloatButton(button)` | 创建悬浮按钮，可带徽章、颜色、点击事件。|
| `system.addWidget(widget)` | 右上角小部件，支持 HTML/Vue 组件。|
| `system.addModal(modal)` | 模态对话框，可配置宽度、关闭逻辑。|
| `system.injectToSlot(config)` | 向声明了 `data-plugin-slot` 的元素注入内容。|
| `system.injectToDOM(config)` | 按选择器在任意 DOM 位置插入 HTML/Vue 组件。|
| `system.registerHook(name, callback)` | 记录自定义钩子，便于宿主或其他插件调用。|
| `system.modifyLayout(mod)` | 向宿主广播布局修改请求。|
| `system.showNotification(message, type)` | 显示通知或退化为控制台输出。|
| `system.showDialog(options)` | 调用宿主对话框或浏览器 `alert`。|

### 数据与工具
| 方法 | 说明 |
| --- | --- |
| `data.fetch(endpoint, options)` | 通过 Axios 调用接口。 |
| `data.store(key, value)` / `data.retrieve(key)` | 使用 `localStorage` 做持久化。 |
| `data.subscribe(event, callback)` | 订阅浏览器事件并返回取消函数。 |
| `utils.generateId()` | 生成唯一 ID。 |
| `utils.formatDate(date, format)` | 常用日期格式化。 |
| `utils.debounce(fn, delay)` / `utils.throttle(fn, delay)` | 防抖、节流工具。 |

所有 API 实现在 `PluginManager` 中，可直接引用其行为定义。

## 全局扩展能力
全局扩展由 `GlobalPluginExtension` 单例负责初始化与事件派发，任何插件（尤其是功能型插件）都可以通过 `$pluginAPI.system.*` 将 UI 注入到宿主页面。

### 支持的注入方式
- **悬浮按钮**：固定在页面右下角，可设置图标、颜色、徽章。点击执行回调。
- **工具栏按钮**：智能定位常见工具栏，失败时降级为悬浮按钮。
- **小部件**：挂载在右上角容器，支持 HTML 或 Vue 组件渲染。
- **模态框**：弹出式对话框，支持自动显示、关闭回调及点击遮罩关闭。
- **插槽注入**：根据 `data-plugin-slot="slotName"` 定位元素并注入内容。若找不到会输出警告。
- **DOM 注入**：通过选择器在目标元素前后/内部插入 HTML 或 Vue 组件，可选择 `append/prepend/before/after/replace` 模式。
- **扩展移除**：调用 `window.$pluginExtension.removeExtension(id)` 可清理 DOM、组件实例与内部缓存。

系统启动时会自动创建悬浮按钮/模态框/小部件容器，并注入基础样式，确保视觉一致性。

## 数据管理与持久化
- `data.store` / `data.retrieve` 默认使用 `localStorage`，键名格式为 `plugin_data_${key}`，适合轻量配置缓存。
- `data.fetch` 基于 Axios，可与后端接口交互，错误会抛出异常需自行处理。
- `data.subscribe` 直接注册浏览器事件，返回取消订阅函数，请在插件卸载或组件销毁时调用以避免内存泄漏。

## 生命周期与钩子
- 插件加载流程：`loadPlugins` → `verify` → `execute` → 按类型分发到 `loadVueComponentPlugin` / `loadPagePlugin` / `loadFunctionPlugin` / `loadThemePlugin`。
- Vue 组件在 `created`、`mounted` 钩子中会自动打印日志，并保持对字符串函数的兼容性。你可以在这些钩子里访问 `this.$pluginAPI`。
- `hooks.onInit` 支持字符串或函数。字符串会进入沙箱执行，函数会在插件管理器上下文中直接调用，可用于初始化数据或注册全局监听。
- 需要在插件卸载时清理资源，可调用 `window.$pluginExtension.removeExtension(pluginId)` 或在 `data.subscribe` 返回的取消函数中处理。

## 安全策略与权限
- JS 代码在沙箱中执行，提供限定的 `console`、`window`、`document`、`setTimeout` 等对象，以防止污染宿主环境，同时保留向后兼容性。
- 注入 HTML 前会移除 `<script>`、`iframe>` 等危险元素，并删除所有以 `on` 开头的事件属性，降低 XSS 风险。
- CSS 直接插入 `<style>`，请自行避免全局覆盖或冲突；建议通过 BEM 或前缀约定隔离样式。
- `permissions` 字段可用于标记敏感能力需求（例如 `data.fetch`、`system.notification`），宿主可在安装审核环节参考该信息。

## 调试与排错清单
1. **确认插件被加载**：浏览器控制台会打印 `🔌 执行插件` 日志以及组件注册成功信息。
2. **检查全局扩展初始化**：控制台应看到 `🔌 全局插件扩展系统已初始化`，并能在 DOM 中找到 `plugin-float-buttons`/`plugin-widgets`/`plugin-modals` 容器。
3. **验证 `$pluginAPI`**：在控制台执行 `window.app?.config?.globalProperties?.$pluginManager?.pluginAPI`，应返回包含 `system/data/utils` 等字段的对象。
4. **手动注入测试按钮**：通过 `window.$pluginExtension.addFloatButton` 验证扩展系统是否可用。
5. **常见问题与解决方案**：参考调试文档中的"没有日志""悬浮按钮缺失""GlobalPluginExtension 未初始化""component 的 code.js 不执行"等条目，按步骤排查接口返回、导入顺序或插件配置。
6. **测试插件**：仓库自带 `simple-test` 等测试插件，可通过接口 `plugin_gateway` 检查加载状态。

## 插件配置与全局设置

### 全局配置存储
- 插件配置现在统一存储在数据库中，实现全局生效，不再使用本地缓存
- 所有用户访问网站时都会看到相同的插件配置，确保一致性
- 配置修改会立即同步到数据库，并通知所有活跃的插件实例

### 配置架构
```json
{
  "config": {
    "enabled": true,
    "intensity": 0.6,
    "transitionSpeed": "normal",
    "settingsSchema": {
      "enabled": {
        "type": "boolean",
        "label": "启用功能",
        "description": "开启或关闭此功能"
      },
      "intensity": {
        "type": "slider",
        "label": "强度",
        "min": 0.1,
        "max": 1,
        "step": 0.1
      }
    }
  }
}
```

### 插件代码访问配置
在插件 JavaScript 代码中，可以通过以下方式访问全局配置：

```javascript
// 方法1：通过 $plugin.config 访问（推荐）
if ($plugin && $plugin.config) {
  this.config = { ...this.config, ...$plugin.config };
}

// 方法2：监听配置更新事件
window.addEventListener('plugin-settings-updated', (e) => {
  if (e.detail.pluginId === 'your-plugin-id') {
    this.config = { ...this.config, ...e.detail.settings };
    // 重新应用配置
    this.applySettings();
  }
});
```

### 配置界面
- 通过 `PluginSettingsDialog` 组件提供用户友好的配置界面
- 支持多种配置字段类型：文本、数字、布尔值、下拉选择、滑块、颜色选择器、文本域、多选框组
- 配置修改实时保存到数据库，无需手动提交
- 提供高级选项：恢复默认设置、JSON 编辑器

### 配置字段类型
| 类型 | 用途 | 示例 |
|------|------|------|
| `boolean` | 开关选项 | 启用/禁用功能 |
| `text` | 文本输入 | 标题、描述 |
| `number` | 数字输入 | 数量、阈值 |
| `select` | 下拉选择 | 速度选项 |
| `slider` | 滑块控件 | 强度、透明度 |
| `color` | 颜色选择 | 主题颜色 |
| `textarea` | 多行文本 | 长描述、配置说明 |
| `checkbox-group` | 多选框组 | 功能模块选择 |

### 向后兼容性
- 现有插件无需修改即可继续使用
- 新插件建议使用全局配置存储
- 系统会自动处理配置的加载和更新

## 最佳实践与 AI 提示词模板
- **唯一命名**：`id`、DOM 容器、事件名称请保持唯一，避免与其他插件冲突。
- **延迟注入**：在页面结构尚未稳定前适当 `setTimeout`，或监听路由变化再注入元素，保证选择器能匹配到目标。
- **清理资源**：插件卸载或路由切换时移除事件监听、定时器与注入内容，维持性能与稳定性。
- **用户反馈**：使用 `system.showNotification` 或自定义 UI 给出操作结果，提升体验。
- **调试日志**：合理使用 `console.log`，并在生产版本中保留关键日志，便于诊断问题。

### 面向 AI 的需求描述模板
```
我需要一个插件，基本信息如下：
- 类型：页面 / 组件 / 功能 / 主题
- 目标：描述核心功能与场景
- 关键 API：列出需要调用的 $pluginAPI.system / data / utils 方法
- UI 要求：说明是否使用 Vuetify 组件、需要注入的 DOM 位置或全局扩展形式
- 数据交互：列出需要的接口、存储键名、事件订阅
- 安全/权限：标明需要的权限或潜在风险
- 调试方式：期望在控制台或界面上的提示
```
将上述信息提供给 AI，即可自动生成符合本指南的插件结构与代码。

---

> 如果你在阅读或交付插件时需要更多示例，请参考 `PluginTemplate.js` 中的模板或仓库内现有插件。祝开发顺利！
