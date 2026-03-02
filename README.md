# AI Coding 示例插件

一个 VS Code 扩展示例，演示如何实现 IDE / SOLO 模式一键切换（类似 Trae）。

## 功能特性

- **IDE 模式**：在右侧辅助栏显示聊天界面，不影响正常编码
- **SOLO 模式**：全屏聊天界面，隐藏所有干扰元素
- **状态栏按钮**：快速切换两种模式
- **活动栏状态恢复**：切换回 IDE 模式时恢复活动栏显示（如果之前被隐藏）

## 目录结构

```
ai-coding-sample/
├── .vscode/
│   ├── launch.json      # 调试配置
│   └── tasks.json       # 任务配置（编译）
├── src/
│   └── extension.ts     # 扩展主入口
├── package.json         # 扩展配置和命令注册
├── tsconfig.json        # TypeScript 配置
└── README.md            # 本文档
```

## 核心原理

### 1. 视图容器注册（package.json）

```json
{
  "viewsContainers": {
    "activitybar": [{
      "id": "ai-coding",
      "title": "AI Coding",
      "icon": "$(robot)"
    }]
  },
  "views": {
    "ai-coding": [{
      "type": "webview",
      "id": "aiCoding.chatView",
      "name": "Chat"
    }]
  }
}
```

- `viewsContainers.activitybar`：在左侧活动栏注册一个图标入口
- `views`：在视图容器中注册一个 Webview 类型的聊天视图

### 2. WebviewViewProvider（IDE 模式）

```typescript
class ChatViewProvider implements vscode.WebviewViewProvider {
  public static readonly viewType = 'aiCoding.chatView';
  
  resolveWebviewView(webviewView: vscode.WebviewView) {
    webviewView.webview.options = { enableScripts: true };
    webviewView.webview.html = getChatSidebarHtml();
  }
}
```

- `WebviewViewProvider` 用于在侧边栏中嵌入自定义 HTML 内容
- `resolveWebviewView` 在视图首次显示时调用，设置 HTML 内容

### 3. WebviewPanel（SOLO 模式）

```typescript
soloPanel = vscode.window.createWebviewPanel(
  'aiCoding.soloView',
  'AI Coding',
  vscode.ViewColumn.One,
  { enableScripts: true, retainContextWhenHidden: true }
);
soloPanel.webview.html = getSoloHtml();
```

- `createWebviewPanel` 创建一个独立的编辑器面板
- `retainContextWhenHidden` 保持面板状态，避免切换时重新加载

### 4. 模式切换命令

#### IDE 模式命令

```typescript
vscode.commands.registerCommand('aiCoding.ideMode', async () => {
  // 1. 关闭 SOLO 面板
  soloPanel?.dispose();
  
  // 2. 恢复活动栏显示（如果之前被隐藏）
  if (activityBarHiddenByUs) {
    await vscode.commands.executeCommand('workbench.action.toggleActivityBarVisibility');
  }
  
  // 3. 聚焦辅助栏聊天视图
  await vscode.commands.executeCommand('workbench.action.focusAuxiliaryBar');
  await vscode.commands.executeCommand('aiCoding.chatView.focus');
});
```

#### SOLO 模式命令

```typescript
vscode.commands.registerCommand('aiCoding.soloMode', async () => {
  // 如果已在 SOLO 模式，只聚焦面板
  if (soloPanel) {
    soloPanel.reveal(vscode.ViewColumn.One, true);
    return;
  }
  
  // 1. 隐藏活动栏（记录原始状态，用于恢复）
  // 新版本使用 workbench.activityBar.location，旧版本使用 workbench.activityBar.visible
  const config = vscode.workspace.getConfiguration('workbench');
  originalActivityBarLocation = config.get<string>('activityBar.location');
  originalActivityBarVisible = config.get<boolean>('activityBar.visible');
  
  // 只有当活动栏可见时才隐藏（避免反向打开）
  if (originalActivityBarLocation !== 'hidden' && originalActivityBarVisible !== false) {
    await vscode.commands.executeCommand('workbench.action.toggleActivityBarVisibility');
  }
  
  // 2. 关闭侧边栏（使用 close 而非 toggle，因为没有API检测状态）
  await vscode.commands.executeCommand('workbench.action.closeSidebar');
  
  // 3. 关闭底部面板和辅助栏（同样没有API检测状态）
  await vscode.commands.executeCommand('workbench.action.closePanel');
  await vscode.commands.executeCommand('workbench.action.closeAuxiliaryBar');
  
  // 4. 创建全屏 Webview 面板
  soloPanel = vscode.window.createWebviewPanel(...);
});
```

### 5. 关键 VS Code 命令

| 命令 | 作用 |
|------|------|
| `workbench.action.toggleActivityBarVisibility` | 切换活动栏可见性 |
| `workbench.action.closeSidebar` | 关闭侧边栏（单向操作，不会反向打开） |
| `workbench.action.closePanel` | 关闭底部面板 |
| `workbench.action.closeAuxiliaryBar` | 关闭右侧辅助栏 |
| `workbench.action.focusAuxiliaryBar` | 聚焦右侧辅助栏 |
| `workbench.view.extension.<id>` | 打开指定的扩展视图容器 |

> **注意**：VSCode 没有提供检查侧边栏、辅助栏、面板是否可见的 API，因此这些元素使用 `close` 命令而非 `toggle` 命令。活动栏可以通过 `workbench.activityBar.location`（新版本）或 `workbench.activityBar.visible`（旧版本）配置项来检测状态，因此可以记录原始状态并在退出 SOLO 模式时恢复。

### 6. UI 元素状态检测能力

| UI 元素 | 状态检测方式 | 处理策略 |
|---------|-------------|---------|
| **活动栏** | `workbench.activityBar.location` (新) / `activityBar.visible` (旧) | 记录状态，恢复状态 |
| **侧边栏** | 无 API | 使用 `closeSidebar` 关闭 |
| **辅助栏** | 无 API | 使用 `closeAuxiliaryBar` 关闭 |
| **面板** | 无 API | 使用 `closePanel` 关闭 |

### 7. 状态栏按钮

```typescript
// 优先级数字越大越靠左，使用较大的数字确保按钮在最左侧
const ideBtn = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Left, 1000);
ideBtn.text = '$(layout-sidebar-right) IDE';
ideBtn.command = 'aiCoding.ideMode';
ideBtn.show();

const soloBtn = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Left, 999);
soloBtn.text = '$(screen-full) SOLO';
soloBtn.command = 'aiCoding.soloMode';
soloBtn.show();
```

- `StatusBarAlignment.Left`：显示在状态栏左侧
- 第二个参数是优先级（数字越大越靠左）
- 使用较大的数字（1000/999）确保按钮在最左侧，且两个按钮紧挨着

### 8. 启动时自动打开视图

```typescript
setTimeout(async () => {
  // 打开视图容器
  await vscode.commands.executeCommand('workbench.view.extension.ai-coding');
  
  // 尝试移动到右侧辅助栏（可能不可用）
  try {
    await vscode.commands.executeCommand('workbench.action.moveViewContainerToSecondarySideBar', 'ai-coding');
  } catch (e) {
    // 备用方案：直接聚焦辅助栏
    await vscode.commands.executeCommand('workbench.action.focusAuxiliaryBar');
  }
  
  // 聚焦聊天视图
  await vscode.commands.executeCommand('aiCoding.chatView.focus');
}, 1500);
```

## 限制说明

### 无法在顶部标题栏添加按钮

VS Code 扩展 API 不支持在顶部标题栏（Command Center 那一行）添加自定义按钮。这是平台限制，只有 fork VS Code 的项目（如 Cursor、Trae）才能实现。

当前替代方案：
- 状态栏按钮（始终可见）
- 编辑器标题栏菜单（`editor/title`，需要打开文件才可见）

### 视图容器位置

VS Code 的 `viewsContainers` 只支持 `activitybar`（左侧活动栏），不支持直接配置到右侧辅助栏。需要通过命令移动。

## 安装与编译

```bash
# 安装依赖
npm install
# 或使用 pnpm
pnpm install

# 编译 TypeScript
npm run compile
# 或
npm run build
# 或使用 pnpm
pnpm build

# 监听模式编译（开发时使用）
npm run watch

# 打包成 VSIX 文件
npm run vsix
# 或使用 pnpm
pnpm vsix
```

## 调试方法

1. 在项目根目录运行 `npm install`
2. 运行 `npm run compile` 编译项目
3. 按 `F5` 启动调试
4. 在扩展开发主机中测试功能
5. 使用 `Help > Toggle Developer Tools` 查看控制台日志

## 扩展建议

如果需要实现真正的 AI 聊天功能，可以考虑：

1. 添加 LLM API 集成（OpenAI、Claude 等）
2. 使用流式响应（Server-Sent Events）
3. 添加对话历史持久化
4. 支持代码高亮和 Markdown 渲染
5. 添加上下文菜单和快捷键

## 许可证

MIT
