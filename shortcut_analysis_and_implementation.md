# 快捷键系统分析与Ctrl+R话题重命名功能实现

## 项目现有快捷键机制分析

### 1. 架构概览
该项目是一个基于Electron的应用，快捷键系统采用了分层架构：
- **主进程（Main Process）**：处理全局系统级快捷键
- **渲染进程（Renderer Process）**：处理UI级别的快捷键

### 2. 快捷键存储结构
快捷键配置存储在Redux store中，位于 `src/renderer/src/store/shortcuts.ts`：

```typescript
interface Shortcut {
  key: string           // 快捷键标识符
  shortcut: string[]    // 快捷键组合，如 ['CommandOrControl', 'R']
  editable: boolean     // 是否可编辑
  enabled: boolean      // 是否启用
  system: boolean       // 是否为系统级快捷键
}
```

### 3. 现有快捷键实现机制

#### 3.1 主进程快捷键处理
- **文件**: `src/main/services/ShortcutService.ts`
- **功能**: 处理全局系统级快捷键，如缩放、显示应用等
- **实现**: 使用Electron的`globalShortcut`API

#### 3.2 渲染进程快捷键处理
- **文件**: `src/renderer/src/hooks/useShortcuts.ts`
- **功能**: 处理UI级别的快捷键
- **实现**: 使用`react-hotkeys-hook`库
- **核心Hook**: `useShortcut(shortcutKey, callback, options)`

### 4. 现有快捷键列表
- `Ctrl+,`: 显示设置
- `Ctrl+N`: 新建话题
- `Ctrl+[`: 切换显示助手
- `Ctrl+]`: 切换显示话题
- `Ctrl+F`: 在聊天中搜索消息
- `Ctrl+Shift+F`: 搜索消息
- `Ctrl+L`: 清除话题
- `Ctrl+K`: 切换新上下文
- `Ctrl+Shift+C`: 复制最后一条消息
- `Escape`: 退出全屏

## Ctrl+R话题重命名功能实现

### 1. 实现步骤

#### 1.1 添加快捷键定义
在 `src/renderer/src/store/shortcuts.ts` 中添加新的快捷键定义：

```typescript
{
  key: 'rename_topic',
  shortcut: ['CommandOrControl', 'R'],
  editable: true,
  enabled: true,
  system: false
}
```

#### 1.2 实现快捷键处理逻辑
在 `src/renderer/src/pages/home/HomePage.tsx` 中添加快捷键处理：

```typescript
useShortcut('rename_topic', async () => {
  if (!activeTopic || !activeAssistant) return
  
  const PromptPopup = await import('@renderer/components/Popups/PromptPopup')
  
  try {
    const name = await PromptPopup.default.show({
      title: t('chat.topics.edit.title'),
      message: '',
      defaultValue: activeTopic?.name || ''
    })
    
    if (name && activeTopic?.name !== name) {
      const updatedTopic = { ...activeTopic, name, isNameManuallyEdited: true }
      dispatch(updateTopic({ assistantId: activeAssistant.id, topic: updatedTopic }))
      setActiveTopic(updatedTopic)
    }
  } catch (error) {
    console.error('Failed to rename topic:', error)
  }
})
```

### 2. 功能特点

#### 2.1 智能上下文感知
- 只有在存在活动话题时才响应快捷键
- 自动获取当前活动的话题和助手

#### 2.2 用户体验优化
- 使用现有的`PromptPopup`组件确保UI一致性
- 预填充当前话题名称
- 支持回车确认，Shift+回车换行

#### 2.3 状态管理
- 使用Redux store更新话题状态
- 标记话题为手动编辑（`isNameManuallyEdited: true`）
- 同步更新当前活动话题

### 3. 技术实现细节

#### 3.1 异步导入
```typescript
const PromptPopup = await import('@renderer/components/Popups/PromptPopup')
```
使用动态导入避免循环依赖问题。

#### 3.2 Redux状态更新
```typescript
dispatch(updateTopic({ assistantId: activeAssistant.id, topic: updatedTopic }))
```
通过Redux action更新话题状态，确保状态同步。

#### 3.3 本地状态同步
```typescript
setActiveTopic(updatedTopic)
```
同步更新本地活动话题状态，确保UI立即响应。

### 4. 现有话题重命名功能对比

#### 4.1 自动重命名
- **触发**: 话题创建后自动触发
- **实现**: `autoRenameTopic` 函数
- **特点**: 使用AI生成话题名称

#### 4.2 手动重命名（原有）
- **触发**: 右键菜单 → 编辑
- **实现**: 上下文菜单调用`PromptPopup.show()`
- **特点**: 用户手动编辑

#### 4.3 快捷键重命名（新增）
- **触发**: `Ctrl+R`
- **实现**: `useShortcut` + `PromptPopup.show()`
- **特点**: 快速访问，提高效率

### 5. 兼容性和扩展性

#### 5.1 平台兼容
- 使用`CommandOrControl`修饰符确保跨平台兼容
- macOS使用`Cmd+R`，Windows/Linux使用`Ctrl+R`

#### 5.2 可配置性
- 快捷键可在设置中编辑
- 可以启用/禁用该功能

#### 5.3 扩展性
- 遵循现有快捷键架构
- 易于添加更多话题相关快捷键

## 总结

本次实现成功为应用添加了`Ctrl+R`快捷键来激活话题重命名功能，完全集成到现有的快捷键系统中。该功能：

1. **遵循现有架构**: 使用相同的快捷键定义和处理机制
2. **用户体验一致**: 复用现有的UI组件和交互模式
3. **功能完整**: 包含错误处理、状态同步等完整功能
4. **可维护性强**: 代码结构清晰，易于维护和扩展

用户现在可以通过按`Ctrl+R`（或macOS上的`Cmd+R`）快速重命名当前活动的话题，大大提高了使用效率。

## 实现的文件修改

1. **`src/renderer/src/store/shortcuts.ts`**: 添加了新的快捷键定义
2. **`src/renderer/src/pages/home/HomePage.tsx`**: 添加了快捷键处理逻辑和相关导入

两个文件的修改都已通过TypeScript类型检查，确保代码质量。