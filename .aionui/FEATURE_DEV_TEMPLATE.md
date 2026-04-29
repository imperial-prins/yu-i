# AionUi Feature Development Specification Template

> This template standardizes how feature development requirements are described to AI, ensuring the AI can accurately understand the task and follow project conventions.

---

## 1. Feature Overview

### 1.1 Basic Information

- **Feature Name**: [concise name]
- **Module**: [ ] Agent Layer [ ] Conversation System [ ] Preview System [ ] Settings System [ ] Workspace [ ] Other
- **Processes Involved**: [ ] Main Process (process) [ ] Renderer Process (renderer) [ ] WebServer [ ] Worker

### 1.2 Feature Description

[Describe the core purpose and value of the feature in 1–3 sentences]

### 1.3 User Scenario

```
Trigger: [how the user triggers this feature]
Process: [how the system responds]
Result:  [state after the feature completes]
```

### 1.4 Data Flow

| Direction | Data Type | Description |
| --------- | --------- | ----------- |
| Input     |           |             |
| Output    |           |             |

---

## 2. Development Standards

### 2.1 Tech Stack Constraints

- **Framework**: Electron 37 + React 19 + TypeScript 5.8
- **UI Library**: Arco Design (@arco-design/web-react)
- **Icons**: Icon Park (@icon-park/react)
- **CSS**: UnoCSS atomic styles
- **State Management**: React Context (AuthContext / ConversationContext / ThemeContext / LayoutContext)
- **IPC Communication**: @office-ai/platform bridge system
- **Internationalization**: i18next + react-i18next
- **Database**: better-sqlite3

### 2.2 Naming Conventions

| Type          | Convention            | Example                                         |
| ------------- | --------------------- | ----------------------------------------------- |
| React Component | PascalCase          | `MessageList.tsx`, `FilePreview.tsx`            |
| Hooks         | `use` prefix + PascalCase | `useAutoScroll.ts`, `useColorScheme.ts`    |
| Bridge files  | featureName + Bridge  | `conversationBridge.ts`, `databaseBridge.ts`    |
| Service files | featureName + Service | `WebuiService.ts`                               |
| Interface types | `I` prefix          | `ICreateConversationParams`, `IResponseMessage` |
| Type aliases  | `T` prefix or direct name | `TChatConversation`, `PresetAgentType`      |
| Constants     | UPPER_SNAKE_CASE      | `MAX_RETRY_COUNT`                               |
| Utility functions | camelCase         | `formatMessage`, `parseResponse`                |

### 2.3 File Placement Rules

```
New files should be placed in the corresponding directory:

src/
├── agent/                        # AI agent implementations
│   ├── acp/                      # ACP protocol agent
│   ├── codex/                    # Codex agent
│   └── gemini/                   # Gemini agent
│
├── common/                       # Cross-process shared modules
│   ├── adapters/                 # API adapters
│   ├── types/                    # Shared type definitions
│   └── utils/                    # Shared utility functions
│
├── process/                      # Electron main process
│   ├── bridge/                   # IPC bridge definitions (24+)
│   ├── database/                 # SQLite database operations
│   ├── services/                 # Business logic services
│   └── task/                     # Task management
│
├── renderer/                     # React renderer process
│   ├── components/               # Reusable UI components
│   │   └── base/                 # Base components
│   ├── context/                  # React Context state
│   ├── hooks/                    # Custom hooks (31+)
│   ├── pages/                    # Page components
│   │   ├── conversation/         # Conversation page
│   │   │   ├── preview/          # Preview panel
│   │   │   └── workspace/        # Workspace
│   │   ├── settings/             # Settings pages (12+)
│   │   └── login/                # Login page
│   ├── messages/                 # Message rendering components
│   ├── i18n/locales/             # Internationalization text
│   ├── services/                 # Frontend services
│   └── utils/                    # Frontend utility functions
│
├── webserver/                    # Web server (WebUI mode)
│   ├── routes/                   # API routes
│   └── middleware/               # Middleware
│
├── worker/                       # Web Worker
│
└── types/                        # Global type definitions
```

### 2.4 Code Style (Prettier Configuration)

```json
{
  "semi": true, // use semicolons
  "singleQuote": true, // use single quotes
  "jsxSingleQuote": true, // JSX uses single quotes
  "trailingComma": "es5", // ES5-compatible trailing commas
  "tabWidth": 2, // 2-space indentation
  "useTabs": false, // do not use tabs
  "bracketSpacing": true, // spaces inside brackets
  "arrowParens": "always", // always parenthesize arrow function params
  "endOfLine": "lf" // Unix line endings
}
```

### 2.5 Quality Requirements

- [ ] TypeScript types are complete — avoid using `any`
- [ ] Use the bridge system for IPC communication
- [ ] Implement error boundary handling
- [ ] Support internationalization (use i18next `t()` function)
- [ ] Compatible with dark / light themes
- [ ] Responsive layout support

### 2.6 Prohibited Practices

- ❌ Direct use of `ipcMain` / `ipcRenderer` — must go through the bridge system
- ❌ Accessing Node.js APIs directly in the renderer process
- ❌ Hardcoding Chinese/English text — use i18n keys
- ❌ Using inline styles — use UnoCSS class names
- ❌ Directly manipulating the DOM in components — use React ref
- ❌ Suppressing TypeScript errors (`@ts-ignore`)

---

## 3. 实现架构

### 3.1 分层架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户界面 (UI)                         │
│  React 组件 / Hooks / Context                           │
└─────────────────────┬───────────────────────────────────┘
                      │ IPC Bridge
┌─────────────────────▼───────────────────────────────────┐
│                   主进程 (Main)                          │
│  Bridge → Service → Database / External API             │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              数据层 (Data)                               │
│  SQLite / LocalStorage / External Services              │
└─────────────────────────────────────────────────────────┘
```

### 3.2 需要修改/新增的文件

**主进程 (src/process/)**

| 文件路径 | 操作                | 说明 |
| -------- | ------------------- | ---- |
|          | [ ] 新增 / [ ] 修改 |      |

**渲染进程 (src/renderer/)**

| 文件路径 | 操作                | 说明 |
| -------- | ------------------- | ---- |
|          | [ ] 新增 / [ ] 修改 |      |

**共享模块 (src/common/)**

| 文件路径 | 操作                | 说明 |
| -------- | ------------------- | ---- |
|          | [ ] 新增 / [ ] 修改 |      |

**类型定义 (src/types/)**

| 文件路径 | 操作                | 说明 |
| -------- | ------------------- | ---- |
|          | [ ] 新增 / [ ] 修改 |      |

### 3.3 IPC 通信设计

如需新增 IPC 通道，遵循以下模式：

```typescript
// src/process/bridge/[功能]Bridge.ts
import { bridge } from '@anthropic/platform';

export const [功能名] = {
  // Provider 模式: 请求-响应 (类似 HTTP 请求)
  [方法名]: bridge.buildProvider<TResponse, TParams>('[通道名]'),

  // Emitter 模式: 事件流 (用于流式数据)
  [事件名]: bridge.buildEmitter<TData>('[通道名].stream'),
};

// 使用示例:
// 渲染进程调用: const result = await [功能名].[方法名].request(params);
// 渲染进程监听: [功能名].[事件名].on((data) => { ... });
```

### 3.4 状态管理设计

- [ ] 使用现有 Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] 需要新增 Context: **\*\*\*\***\_\_\_\_**\*\*\*\***
- [ ] 仅组件内部状态 (useState/useReducer)
- [ ] 需要持久化存储

### 3.5 国际化 Key 设计

```json
// 添加到 src/renderer/i18n/locales/[lang].json
// Key 命名规范: [模块].[功能].[描述]

{
  "conversation.export.title": "导出对话",
  "conversation.export.success": "导出成功",
  "conversation.export.error": "导出失败"
}
```

**支持的语言文件:**

- `zh-CN.json` - 简体中文 (必须)
- `en-US.json` - English (必须)
- `zh-TW.json` - 繁體中文
- `ja-JP.json` - 日本語
- `ko-KR.json` - 한국어

---

## 4. 验收标准

### 4.1 功能验收

- [ ] [具体功能点 1]
- [ ] [具体功能点 2]
- [ ] [具体功能点 3]

### 4.2 边界情况

- [ ] [异常场景 1 的处理]
- [ ] [异常场景 2 的处理]

### 4.3 兼容性验收

- [ ] macOS 正常运行
- [ ] Windows 正常运行
- [ ] 深色模式显示正确
- [ ] 浅色模式显示正确
- [ ] 多语言切换正常

### 4.4 代码质量

- [ ] `npm run lint` 无错误
- [ ] `npm run build` 构建成功
- [ ] TypeScript 无类型错误
- [ ] 无 console.log 遗留

---

## 5. 参考资料

### 5.1 类似功能参考

[列出项目中可参考的类似实现]

| 功能 | 文件路径 | 说明 |
| ---- | -------- | ---- |
|      |          |      |

### 5.2 依赖的现有模块

[列出需要调用的现有接口/组件/Hook]

| 模块 | 路径 | 用途 |
| ---- | ---- | ---- |
|      |      |      |

### 5.3 外部依赖

[如需引入新依赖，列出并说明理由]

| 依赖包 | 版本 | 用途 | 必要性说明 |
| ------ | ---- | ---- | ---------- |
|        |      |      |            |

### 5.4 特殊注意事项

[列出实现过程中需要特别注意的事项]

---

## 使用示例

以下是一个完整的功能需求示例：

```markdown
## 1. 功能概述

### 1.1 基本信息

- **功能名称**: 对话导出 PDF
- **所属模块**: [x] 对话系统
- **涉及进程**: [x] 主进程(process) [x] 渲染进程(renderer)

### 1.2 功能描述

允许用户将当前对话导出为 PDF 文件，保留消息格式、代码高亮和图片。

### 1.3 用户场景

触发: 用户点击对话页面右上角的"导出"按钮，选择"导出为 PDF"
过程: 系统收集对话内容，渲染为 HTML，转换为 PDF
结果: 弹出保存对话框，用户选择保存位置后生成 PDF 文件

### 3.2 需要修改/新增的文件

**主进程 (src/process/)**
| 文件路径 | 操作 | 说明 |
|----------|------|------|
| src/process/bridge/exportBridge.ts | [x] 新增 | PDF 导出 IPC 通道定义 |
| src/process/services/ExportService.ts | [x] 新增 | PDF 生成逻辑 |

**渲染进程 (src/renderer/)**
| 文件路径 | 操作 | 说明 |
|----------|------|------|
| src/renderer/pages/conversation/components/ChatHeader.tsx | [x] 修改 | 添加导出下拉菜单 |
| src/renderer/hooks/useExportPdf.ts | [x] 新增 | 导出功能 Hook |

### 4.1 功能验收

- [ ] 点击导出按钮显示导出选项菜单
- [ ] 选择 PDF 后弹出保存对话框
- [ ] 生成的 PDF 包含完整对话内容
- [ ] 代码块保留语法高亮样式
- [ ] 图片正确嵌入 PDF

### 5.1 类似功能参考

| 功能          | 文件路径                                              | 说明            |
| ------------- | ----------------------------------------------------- | --------------- |
| Markdown 导出 | src/renderer/hooks/useExportMarkdown.ts               | 可参考导出流程  |
| PDF 预览      | src/renderer/pages/conversation/preview/PdfViewer.tsx | 可参考 PDF 处理 |
```

---

## 模板维护

- **创建日期**: 2025-01-27
- **适用版本**: AionUi v0.x+
- **维护者**: [项目团队]

如需更新模板，请同步修改本文件并通知团队成员。
