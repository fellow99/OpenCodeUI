# OpenCodeUI 项目结构

> 本文档描述 OpenCodeUI 的完整目录结构与模块关系。所有文件均基于实际源码树，不含推测内容。

## 目录

- [完整目录树](#完整目录树)
- [Pages & Routes（页面与路由）](#pages--routes页面与路由)
- [Dialogs & Overlays（对话框与浮层）](#dialogs--overlays对话框与浮层)
- [Layout Components（布局组件）](#layout-components布局组件)
- [模块说明](#模块说明)

---

## 完整目录树

```
OpenCodeUI/
├── src/                              # 前端主源码（React 19 + TypeScript）
│   ├── api/                          # API 请求封装层（24 个文件）
│   │   ├── index.ts                  #   统一导出入口
│   │   ├── client.ts                 #   SDK 客户端初始化
│   │   ├── sdk.ts                    #   SDK 客户端管理（异步初始化）
│   │   ├── agent.ts                  #   Agent 相关 API
│   │   ├── command.ts                #   命令执行 API
│   │   ├── command.test.ts           #   命令 API 测试
│   │   ├── config.ts                 #   配置相关 API
│   │   ├── events.ts                 #   SSE 事件订阅
│   │   ├── file.ts                   #   文件操作 API
│   │   ├── global.ts                 #   全局状态 API
│   │   ├── http.ts                   #   HTTP 请求封装
│   │   ├── http.test.ts              #   HTTP 封装测试
│   │   ├── lsp.ts                    #   LSP（语言服务器协议）API
│   │   ├── mcp.ts                    #   MCP（模型上下文协议）API
│   │   ├── message.ts                #   消息相关 API
│   │   ├── permission.ts             #   权限审批 API
│   │   ├── pty.ts                    #   PTY 终端 API
│   │   ├── session.ts                #   会话管理 API
│   │   ├── skill.ts                  #   Skill 相关 API
│   │   ├── todo.ts                   #   Todo 相关 API
│   │   ├── tool.ts                   #   工具调用 API
│   │   ├── types.ts                  #   API 类型定义
│   │   ├── vcs.ts                    #   版本控制 API
│   │   └── worktree.ts               #   Git Worktree API
│   │
│   ├── assets/                       # 静态资源
│   │   ├── icons/                    #   SVG 图标（60+ 个）
│   │   └── react.svg                 #   React Logo
│   │
│   ├── components/                   # 通用 UI 组件（38+ 个文件）
│   │   ├── index.ts                  #   统一导出入口
│   │   ├── ui/                       #   基础 UI 原语
│   │   │   ├── index.ts              #     导出入口
│   │   │   ├── AnimatedPresence.tsx  #     动画容器（View Transitions）
│   │   │   ├── Button.tsx            #     按钮组件
│   │   │   ├── ConfirmDialog.tsx     #     确认对话框
│   │   │   ├── CopyButton.tsx        #     复制按钮
│   │   │   ├── Dialog.tsx            #     对话框基座
│   │   │   ├── Dialog.test.tsx       #     对话框测试
│   │   │   ├── DropdownMenu.tsx      #     下拉菜单
│   │   │   ├── DropdownMenu.test.tsx #     下拉菜单测试
│   │   │   ├── IconButton.tsx        #     图标按钮
│   │   │   ├── MenuItem.tsx          #     菜单项
│   │   │   ├── ModalShell.tsx        #     模态框外壳
│   │   │   ├── ResizablePanel.tsx    #     可拖拽调整大小的面板
│   │   │   ├── ScrollArea.tsx        #     滚动区域
│   │   │   └── SmoothHeight.tsx      #     平滑高度动画
│   │   ├── BottomPanel.tsx           #   底部面板（终端 / 文件浏览器等）
│   │   ├── CircularProgress.tsx      #   环形进度条
│   │   ├── CloseServiceDialog.tsx    #   关闭服务确认对话框
│   │   ├── CodeBlock.tsx             #   代码块（Shiki 高亮）
│   │   ├── CodeBlock.test.tsx        #   代码块测试
│   │   ├── CodePreview.tsx           #   代码预览
│   │   ├── CodePreview.test.tsx      #   代码预览测试
│   │   ├── CommandPalette.tsx        #   命令面板（⌘K）
│   │   ├── CommandPalette.test.tsx   #   命令面板测试
│   │   ├── ContentBlock.tsx          #   内容块容器
│   │   ├── DiffModal.tsx             #   单文件 Diff 弹窗
│   │   ├── DiffModal.test.tsx        #   Diff 弹窗测试
│   │   ├── DiffView.tsx              #   Diff 视图组件
│   │   ├── DiffView.test.tsx         #   Diff 视图测试
│   │   ├── DiffViewer.tsx            #   Diff 查看器
│   │   ├── DiffViewer.test.tsx       #   Diff 查看器测试
│   │   ├── FileExplorer.tsx          #   文件浏览器
│   │   ├── FullscreenViewer.tsx      #   全屏查看器
│   │   ├── FullscreenViewer.test.tsx #   全屏查看器测试
│   │   ├── Icons.tsx                 #   图标组件集合
│   │   ├── MarkdownRenderer.tsx      #   Markdown 渲染器
│   │   ├── MarkdownRenderer.test.tsx #   Markdown 渲染器测试
│   │   ├── McpPanel.tsx              #   MCP 服务器面板
│   │   ├── MultiFileDiffModal.tsx    #   多文件 Diff 弹窗
│   │   ├── MultiFileDiffModal.test.tsx # 多文件 Diff 弹窗测试
│   │   ├── OutlineIndex.tsx          #   大纲索引（ChatPane 内使用）
│   │   ├── PanelContainer.tsx        #   面板容器（带标签栏）
│   │   ├── PreviewTabsBar.tsx        #   预览标签栏
│   │   ├── RightPanel.tsx            #   右侧面板（文件 / 终端 / MCP 等）
│   │   ├── SessionChangesPanel.tsx   #   会话变更面板
│   │   ├── SessionChangesPanel.test.tsx # 会话变更面板测试
│   │   ├── SkillPanel.tsx            #   Skill 面板
│   │   ├── Terminal.tsx              #   终端组件（xterm.js）
│   │   ├── TerminalPanel.tsx         #   终端面板
│   │   ├── ToastContainer.tsx        #   Toast 通知容器
│   │   └── WorktreePanel.tsx         #   Git Worktree 面板
│   │
│   ├── constants/                    # 常量定义
│   │   ├── index.ts                  #   统一导出
│   │   ├── animation.ts              #   动画时长常量
│   │   ├── api.ts                    #   API 相关常量
│   │   ├── pagination.ts             #   分页常量
│   │   ├── storage.ts                #   Storage Key 常量
│   │   ├── time.ts                   #   时间相关常量
│   │   └── ui.ts                     #   UI 常量（最小宽度等）
│   │
│   ├── contexts/                     # React Context 提供者
│   │   ├── index.ts                  #   统一导出
│   │   ├── DirectoryContext.tsx      #   目录上下文（Provider）
│   │   ├── DirectoryContext.shared.ts #  目录上下文共享逻辑
│   │   ├── SessionContext.tsx        #   会话上下文（Provider）
│   │   ├── SessionContext.shared.ts  #   会话上下文共享逻辑
│   │   ├── SessionNavigationContext.tsx # 会话导航上下文
│   │   ├── useDirectory.ts           #   目录 Context Hook
│   │   └── useSessionContext.ts      #   会话 Context Hook
│   │
│   ├── features/                     # 业务功能模块
│   │   │
│   │   ├── attachment/               # 附件管理（@ 提及文件 / 图片上传）
│   │   │   ├── index.ts              #   导出入口
│   │   │   ├── AttachmentDetailModal.tsx # 附件详情弹窗
│   │   │   ├── AttachmentItem.tsx    #   附件列表项
│   │   │   ├── AttachmentPreview.tsx #   附件预览
│   │   │   ├── types.ts              #   附件类型定义
│   │   │   └── utils.ts              #   附件工具函数
│   │   │
│   │   ├── chat/                     # 聊天界面核心
│   │   │   ├── index.ts              #   导出入口（Sidebar, ChatArea, InputBox 等）
│   │   │   ├── ChatPane.tsx          #   聊天窗格（单 pane / split pane 共用）
│   │   │   ├── ChatArea.tsx          #   聊天区域（消息列表 + 输入框）
│   │   │   ├── ChatArea.test.ts      #   聊天区域测试
│   │   │   ├── chatAreaVisibility.ts #   聊天区域可见性逻辑
│   │   │   ├── chatViewport.tsx      #   聊天视口控制器（响应式布局）
│   │   │   ├── EmptyState.tsx        #   空状态（新建会话引导）
│   │   │   ├── Header.tsx            #   应用级顶部栏
│   │   │   ├── PaneHeader.tsx        #   Split 模式下的窗格标题栏
│   │   │   ├── SplitContainer.tsx    #   递归分割容器（可拖拽分隔线）
│   │   │   ├── InputBox.tsx          #   输入框组件
│   │   │   ├── InputBox.test.tsx     #   输入框测试
│   │   │   ├── InputBoxAttachments.test.tsx # 输入框附件测试
│   │   │   ├── InlinePermission.tsx  #   内联权限请求
│   │   │   ├── InlineQuestion.tsx    #   内联问题
│   │   │   ├── InlineToolRequestContext.tsx # 内联工具请求上下文
│   │   │   ├── ModelSelector.tsx     #   模型选择器
│   │   │   ├── ModelSelector.test.tsx #  模型选择器测试
│   │   │   ├── PermissionDialog.tsx  #   权限审批对话框
│   │   │   ├── ProjectDialog.tsx     #   项目选择对话框
│   │   │   ├── ProjectDialog.test.tsx #  项目对话框测试
│   │   │   ├── QuestionDialog.tsx    #   问题对话框
│   │   │   ├── RetryStatusInline.tsx #   重试状态内联显示
│   │   │   ├── ShareDialog.tsx       #   会话分享对话框
│   │   │   ├── Sidebar.tsx           #   侧边栏（会话列表 + 导航）
│   │   │   │
│   │   │   ├── input/                #   输入框子模块
│   │   │   │   ├── InputActions.tsx  #     输入操作按钮
│   │   │   │   ├── InputFooter.tsx   #     输入框底部（模型选择等）
│   │   │   │   ├── InputToolbar.tsx  #     输入工具栏
│   │   │   │   ├── InputToolbar.test.tsx # 输入工具栏测试
│   │   │   │   ├── inputUtils.ts     #     输入工具函数
│   │   │   │   ├── UndoStatus.tsx    #     撤销状态
│   │   │   │   ├── useAttachmentRail.ts #  附件栏 Hook
│   │   │   │   ├── useInputHistory.ts #    输入历史 Hook
│   │   │   │   └── useMobileCollapse.ts #  移动端折叠 Hook
│   │   │   │
│   │   │   └── sidebar/              #   侧边栏子模块
│   │   │       ├── SidePanel.tsx     #     侧面板容器
│   │   │       ├── SidebarFooter.tsx #     侧边栏底部
│   │   │       ├── ActiveSessionItem.tsx # 活跃会话列表项
│   │   │       ├── NotificationItem.tsx #  通知列表项
│   │   │       ├── FolderRecentList.tsx #  最近文件夹列表
│   │   │       ├── SessionChildrenSlot.tsx # 子会话插槽
│   │   │       ├── activeSessionTree.ts #   活跃会话树构建
│   │   │       ├── activeSessionTree.test.ts # 活跃会话树测试
│   │   │       ├── sidebarUtils.ts   #     侧边栏工具函数
│   │   │       └── ContextDetailsDialog.tsx # 上下文详情对话框
│   │   │
│   │   ├── mention/                  # @ 提及功能（文件 / 文件夹 / 规则）
│   │   │   ├── index.ts              #   导出入口
│   │   │   ├── MentionMenu.tsx       #   提及菜单
│   │   │   ├── MentionMenu.test.tsx  #   提及菜单测试
│   │   │   ├── MentionTag.tsx        #   提及标签渲染
│   │   │   ├── createMentionElement.ts # 提及元素创建
│   │   │   ├── types.ts              #   提及类型
│   │   │   ├── useMention.ts         #   提及 Hook
│   │   │   └── utils.ts              #   提及工具函数
│   │   │       └── utils.test.ts     #   提及工具测试
│   │   │
│   │   ├── message/                  # 消息渲染引擎
│   │   │   ├── index.ts              #   导出入口
│   │   │   ├── MessageRenderer.tsx   #   消息渲染器
│   │   │   ├── MessageRenderer.test.tsx # 消息渲染器测试
│   │   │   │
│   │   │   ├── parts/                #   消息片段渲染
│   │   │   │   ├── index.ts          #     导出入口
│   │   │   │   ├── AttachmentPartViews.tsx # 附件片段
│   │   │   │   ├── MessageErrorView.tsx #  错误视图
│   │   │   │   ├── ReasoningPartView.tsx #  推理过程视图
│   │   │   │   ├── ReasoningPartView.test.tsx # 推理视图测试
│   │   │   │   ├── StepFinishPartView.tsx # 步骤完成视图
│   │   │   │   ├── SubtaskPartView.tsx #   子任务视图
│   │   │   │   ├── SystemPartViews.tsx #   系统消息视图
│   │   │   │   ├── TextPartView.tsx  #     文本片段视图
│   │   │   │   └── ToolPartView.tsx  #     工具调用视图
│   │   │   │
│   │   │   └── tools/                #   工具调用渲染
│   │   │       ├── index.ts          #     导出入口
│   │   │       ├── registry.tsx      #     工具渲染器注册表
│   │   │       ├── registry.test.ts  #     注册表测试
│   │   │       ├── types.ts          #     工具类型
│   │   │       ├── icons.tsx         #     工具图标
│   │   │       └── renderers/        #     具体渲染器
│   │   │           ├── index.ts      #       导出入口
│   │   │           ├── BashRenderer.tsx #    Bash 命令
│   │   │           ├── DefaultRenderer.tsx # 默认渲染
│   │   │           ├── QuestionRenderer.tsx # 问题工具
│   │   │           ├── TaskRenderer.tsx #    子任务工具
│   │   │           ├── TodoRenderer.tsx #    Todo 工具
│   │   │           └── todoUtils.ts  #       Todo 工具函数
│   │   │
│   │   ├── sessions/                 # 会话管理
│   │   │   ├── index.ts              #   导出入口
│   │   │   ├── ProjectSelector.tsx   #   项目选择器
│   │   │   └── SessionList.tsx       #   会话列表
│   │   │
│   │   ├── settings/                 # 设置面板
│   │   │   ├── SettingsDialog.tsx    #   设置对话框（6 个标签页）
│   │   │   ├── KeybindingsSection.tsx #  快捷键设置区域
│   │   │   └── components/           #   各设置页
│   │   │       ├── AppearanceSettings.tsx #  外观设置
│   │   │       ├── ChatSettings.tsx  #     聊天设置
│   │   │       ├── NotificationSettings.tsx # 通知设置
│   │   │       ├── ServersSettings.tsx #     服务器设置
│   │   │       ├── ServiceSettings.tsx #     服务设置
│   │   │       └── SettingsUI.tsx    #     设置 UI 工具
│   │   │
│   │   └── slash-command/            # / 斜杠命令
│   │       ├── index.ts              #   导出入口
│   │       ├── SlashCommandMenu.tsx  #   斜杠命令菜单
│   │       └── SlashCommandMenu.test.tsx # 菜单测试
│   │
│   ├── hooks/                        # 自定义 React Hooks（47 个文件）
│   │   ├── index.ts                  #   统一导出
│   │   ├── useRouter.ts              #   Hash 路由（自定义，无 react-router）
│   │   ├── useChatSession.ts         #   聊天会话 Hook
│   │   ├── useChatSession.test.tsx   #   聊天会话测试
│   │   ├── useGlobalEvents.ts        #   全局事件监听（SSE 分发）
│   │   ├── useGlobalEvents.test.tsx  #   全局事件测试
│   │   ├── useSessionManager.ts      #   会话管理 Hook
│   │   ├── useSessions.ts            #   会话列表 Hook
│   │   ├── useSessions.test.tsx      #   会话列表测试
│   │   ├── usePermissions.ts         #   权限管理 Hook
│   │   ├── usePermissionHandler.ts   #   权限处理 Hook
│   │   ├── useKeybindings.ts         #   快捷键 Hook
│   │   ├── useModelSelection.ts      #   模型选择 Hook
│   │   ├── useModelSelection.test.tsx #  模型选择测试
│   │   ├── useModels.ts              #   模型列表 Hook
│   │   ├── useTheme.ts               #   主题 Hook
│   │   ├── useNotification.ts        #   通知 Hook
│   │   ├── useProject.ts             #   项目 Hook
│   │   ├── useFileExplorer.ts        #   文件浏览器 Hook
│   │   ├── useFileExplorer.test.tsx  #   文件浏览器测试
│   │   ├── useGitWorkspaceCatalog.ts #   Git 工作区目录 Hook
│   │   ├── useVcsInfo.ts             #   VCS 信息 Hook
│   │   ├── usePathMode.ts            #   路径模式 Hook（Unix/Windows）
│   │   ├── usePresence.ts            #   在线状态 Hook
│   │   ├── useServerStore.ts         #   服务器存储 Hook
│   │   ├── useCloseServiceDialog.ts  #   关闭服务对话框 Hook
│   │   ├── useClickOutside.ts        #   点击外部检测 Hook
│   │   ├── useDelayedRender.ts       #   延迟渲染 Hook
│   │   ├── useDelayedRender.test.tsx #   延迟渲染测试
│   │   ├── useDropdown.ts            #   下拉菜单 Hook
│   │   ├── useDynamicVirtualScroll.ts #  动态虚拟滚动 Hook
│   │   ├── useImageCompressor.ts     #   图片压缩 Hook
│   │   ├── useInputCapabilities.ts   #   输入能力 Hook
│   │   ├── useInView.ts              #   可见性检测 Hook
│   │   ├── useInView.test.tsx        #   可见性测试
│   │   ├── useIsMobile.ts            #   移动端检测 Hook
│   │   ├── useMessageAnimation.ts    #   消息动画 Hook
│   │   ├── useModalAnimation.ts      #   模态框动画 Hook
│   │   ├── useResponsiveMaxHeight.ts #   响应式最大高度 Hook
│   │   ├── useRevertState.ts         #   状态回退 Hook
│   │   ├── useSessionStats.ts        #   会话统计 Hook
│   │   ├── useSessionStats.test.ts   #   会话统计测试
│   │   ├── useSyntaxHighlight.ts     #   语法高亮 Hook
│   │   ├── useVerticalSplitResize.ts #   垂直分割调整大小 Hook
│   │   ├── useViewportHeight.ts      #   视口高度 Hook
│   │   ├── useCancelHint.ts          #   取消提示 Hook
│   │   └── sessionStatsUtils.ts      #   会话统计工具函数
│   │
│   ├── store/                        # 状态管理（Zustand，22 个文件）
│   │   ├── index.ts                  #   统一导出
│   │   ├── messageStore.ts           #   消息存储（核心）
│   │   ├── messageStore.test.ts      #   消息存储测试
│   │   ├── messageStoreHooks.ts      #   消息存储 Hooks
│   │   ├── messageStoreHooks.test.tsx #  消息存储 Hooks 测试
│   │   ├── messageStoreTypes.ts      #   消息存储类型
│   │   ├── activeSessionStore.ts     #   活跃会话存储
│   │   ├── childSessionStore.ts      #   子会话存储
│   │   ├── followupQueueStore.ts     #   后续队列存储
│   │   ├── layoutStore.ts            #   布局存储（底部面板 / 终端标签）
│   │   ├── layoutStore.test.ts       #   布局存储测试
│   │   ├── paneLayoutStore.ts        #   窗格布局存储（Split 模式）
│   │   ├── paneControllerStore.ts    #   窗格控制器存储
│   │   ├── themeStore.ts             #   主题存储
│   │   ├── serverStore.ts            #   服务器存储
│   │   ├── serviceStore.ts           #   服务存储
│   │   ├── notificationStore.ts      #   通知存储
│   │   ├── keybindingStore.ts        #   快捷键存储
│   │   ├── autoApproveStore.ts       #   自动审批存储
│   │   ├── changeScopeStore.ts       #   变更范围存储
│   │   ├── soundStore.ts             #   声音存储
│   │   └── todoStore.ts              #   Todo 存储
│   │
│   ├── themes/                       # 主题预设
│   │   └── index.ts                  #   主题入口（Eucalyptus / Claude / Breeze）
│   │
│   ├── types/                        # TypeScript 类型定义
│   │   ├── index.ts                  #   统一导出
│   │   ├── chat.ts                   #   聊天类型
│   │   ├── message.ts                #   消息类型
│   │   ├── ui.ts                     #   UI 类型
│   │   ├── view-transitions.d.ts     #   View Transitions 类型声明
│   │   └── api/                      #   API 类型
│   │       ├── index.ts              #     导出入口
│   │       ├── agent.ts              #     Agent 类型
│   │       ├── common.ts             #     通用类型
│   │       ├── config.ts             #     配置类型
│   │       ├── event.ts              #     事件类型
│   │       ├── file.ts               #     文件类型
│   │       ├── mcp.ts                #     MCP 类型
│   │       ├── message.ts            #     消息类型
│   │       ├── model.ts              #     模型类型
│   │       ├── permission.ts         #     权限类型
│   │       ├── project.ts            #     项目类型
│   │       ├── pty.ts                #     PTY 类型
│   │       ├── session.ts            #     会话类型
│   │       ├── skill.ts              #     Skill 类型
│   │       ├── tool.ts               #     工具类型
│   │       ├── vcs.ts                #     VCS 类型
│   │       └── worktree.ts           #     Worktree 类型
│   │
│   ├── utils/                        # 工具函数
│   │   ├── index.ts                  #   统一导出
│   │   ├── activeScope.ts            #   活跃范围工具
│   │   ├── activeScope.test.ts       #   活跃范围测试
│   │   ├── ansiUtils.ts              #   ANSI 转义序列处理
│   │   ├── clipboard.ts              #   剪贴板工具
│   │   ├── clipboard.test.ts         #   剪贴板测试
│   │   ├── dateUtils.ts              #   日期工具
│   │   ├── diffUtils.ts              #   Diff 工具
│   │   ├── directoryUtils.ts         #   目录工具
│   │   ├── downloadUtils.ts          #   下载工具
│   │   ├── errorHandling.ts          #   错误处理
│   │   ├── formatUtils.ts            #   格式化工具
│   │   ├── formatUtils.test.ts       #   格式化测试
│   │   ├── languageUtils.ts          #   语言工具
│   │   ├── logger.ts                 #   日志工具
│   │   ├── materialIcons.ts          #   Material Icons 映射
│   │   ├── messageConversion.ts      #   消息转换
│   │   ├── mimeUtils.ts              #   MIME 工具
│   │   ├── modelUtils.ts             #   模型工具
│   │   ├── notificationSoundBridge.ts #  通知声音桥接
│   │   ├── perServerStorage.ts       #   按服务器存储
│   │   ├── sessionHelpers.ts         #   会话辅助
│   │   ├── soundPlayer.ts            #   声音播放器
│   │   ├── stringUtils.ts            #   字符串工具
│   │   ├── tauri.ts                  #   Tauri 工具
│   │   └── tauri.test.ts             #   Tauri 测试
│   │
│   ├── workers/                      # Web Workers
│   │   └── imageCompressor.worker.ts #   图片压缩 Worker
│   │
│   ├── lib/                          # 第三方库封装
│   │   └── overlayScrollbar.ts       #   Overlay Scrollbars 初始化
│   │
│   ├── locales/                      # 国际化（i18n）
│   │   ├── en/                       #   英文
│   │   │   ├── chat.json             #     聊天
│   │   │   ├── commands.json         #     命令
│   │   │   ├── common.json           #     通用
│   │   │   ├── components.json       #     组件
│   │   │   ├── message.json          #     消息
│   │   │   └── settings.json         #     设置
│   │   └── zh-CN/                    #   简体中文
│   │       ├── chat.json             #     聊天
│   │       ├── commands.json         #     命令
│   │       ├── common.json           #     通用
│   │       ├── components.json       #     组件
│   │       ├── message.json          #     消息
│   │       └── settings.json         #     设置
│   │
│   ├── test/                         # 测试配置
│   │   └── setup.ts                  #   Vitest 测试环境设置
│   │
│   ├── App.tsx                       # 根组件（布局 + 路由 + 对话框编排）
│   ├── main.tsx                      # 应用入口（初始化 + 渲染）
│   ├── i18n.ts                       # i18n 配置
│   ├── index.css                     # 全局样式（Tailwind + 自定义变量）
│   └── vite-env.d.ts                 #   Vite 环境类型声明
│
├── src-tauri/                        # Tauri 2 桌面应用（Rust）
│   ├── src/                          #   Rust 源码
│   │   ├── main.rs                   #     入口
│   │   ├── lib.rs                    #     库入口
│   │   └── app/                      #     应用模块
│   │       ├── mod.rs                #       模块入口
│   │       ├── service.rs            #       服务管理（opencode serve）
│   │       ├── dir_state.rs          #       目录状态
│   │       ├── commands/             #       Tauri 命令
│   │       │   ├── mod.rs            #         命令入口
│   │       │   ├── opencode.rs       #         OpenCode 命令
│   │       │   ├── sse.rs            #         SSE 命令
│   │       │   └── utils.rs          #         工具命令
│   │       └── sse/                  #       SSE 模块
│   │           ├── mod.rs            #         SSE 入口
│   │           ├── connect_args.rs   #         连接参数
│   │           ├── event.rs          #         事件定义
│   │           └── state.rs          #         状态管理
│   │
│   ├── tauri.conf.json               #   Tauri 配置
│   ├── Cargo.toml                    #   Rust 依赖
│   ├── Cargo.lock                    #   依赖锁定文件
│   ├── build.rs                      #   构建脚本
│   ├── icons/                        #   应用图标（多尺寸）
│   ├── capabilities/                 #   Tauri 权限配置
│   │   └── default.json              #     默认权限
│   ├── gen/android/                  #   Android 生成文件（Gradle 项目）
│   └── windows/                      #   Windows 安装脚本
│       └── hooks.nsi                 #     NSIS 钩子
│
├── src-router/                       # 动态端口路由器（Rust）
│   ├── src/
│   │   ├── main.rs                   #   入口
│   │   ├── router.rs                 #   路由逻辑
│   │   ├── scanner.rs                #   端口扫描器
│   │   ├── config.rs                 #   配置
│   │   ├── state.rs                  #   状态管理
│   │   ├── caddy.rs                  #   Caddy 集成
│   │   └── router.html               #   路由管理页面
│   ├── Cargo.toml                    #   Rust 依赖
│   └── Cargo.lock                    #   依赖锁定文件
│
├── docker/                           # Docker 部署配置
│   ├── Dockerfile.frontend           #   前端镜像
│   ├── Dockerfile.backend            #   后端镜像
│   ├── Dockerfile.gateway            #   Gateway 镜像
│   ├── Caddyfile                     #   Caddy 主配置
│   ├── Caddyfile.gateway             #   Gateway Caddy 配置
│   ├── Caddyfile.standalone          #   独立模式 Caddy 配置
│   ├── entrypoint-gateway.sh         #   Gateway 入口脚本
│   ├── backend-entrypoint.sh         #   后端入口脚本
│   └── nginx.host.conf.example       #   Nginx 反向代理示例
│
├── package.json                      # 项目依赖与脚本
├── vite.config.ts                    # Vite 构建配置
├── tsconfig.json                     # TypeScript 配置
├── tailwind.config.ts                # Tailwind CSS 配置
├── index.html                        # HTML 入口
├── .env.example                      # 环境变量示例
├── docker-compose.yml                # Docker Compose（完整部署）
├── docker-compose.standalone.yml     # Docker Compose（纯前端）
└── README.md                         # 项目说明
```

---

## Pages & Routes（页面与路由）

本项目使用**自定义 Hash 路由**（`src/hooks/useRouter.ts`），不依赖 react-router。路由状态通过 `useSyncExternalStore` 管理，采用模块级单例 store，确保 App、DirectoryProvider、Settings 等所有消费方看到同一份路由状态。

| 路由格式                | 说明                             | 对应视图                               |
| ----------------------- | -------------------------------- | -------------------------------------- |
| `#/`                    | 首页（新建会话视图）             | `ChatPane` 渲染 `EmptyState`           |
| `#/session/{sessionId}` | 聊天会话视图                     | `ChatPane` 渲染对应 session 的消息列表 |
| `?dir={path}`           | 目录参数（可附加到以上两种路由） | 指定工作目录，持久化到 `serverStorage` |

**路由格式示例：**

```
#/                                    → 首页，无 session
#/?dir=/path/to/project               → 首页，带目录
#/session/abc-123                     → 打开指定 session
#/session/abc-123?dir=/path/to/project → session + 目录
```

**核心行为：**

- `navigateToSession(sessionId, directory)` — 导航到指定 session，可选带目录
- `navigateHome()` — 返回首页，保留当前目录
- `replaceSession(sessionId, directory)` — 替换 URL（不产生历史记录），用于 focused pane 变化时同步
- `setDirectory(directory)` — 设置目录，同时写入 `STORAGE_KEY_LAST_DIRECTORY` 到持久化存储
- `replaceDirectory(directory)` — 替换目录（不产生历史记录）
- 目录参数缺失时，自动从 `serverStorage` 恢复上次使用的目录
- 路径统一使用正斜杠（`/`），通过 `normalizeToForwardSlash` 转换

---

## Dialogs & Overlays（对话框与浮层）

所有对话框均基于 `components/ui/Dialog.tsx` 基座组件，通过 React `useState` 在 `App.tsx` 中管理开关状态。

| 对话框                    | 位置                                             | 触发方式                           | 说明                                                                                                   |
| ------------------------- | ------------------------------------------------ | ---------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **SettingsDialog**        | `features/settings/SettingsDialog.tsx`           | 侧边栏设置按钮 / 命令面板 / 快捷键 | 6 个标签页：Servers、Chat、Appearance、Notifications、Service、Keybindings。使用 `lazy()` 懒加载       |
| **CommandPalette**        | `components/CommandPalette.tsx`                  | 快捷键 `⌘K` / 命令面板命令         | 全局命令搜索与执行，支持分类（General、Session、Terminal、Model、Message、Pane）。使用 `lazy()` 懒加载 |
| **CloseServiceDialog**    | `components/CloseServiceDialog.tsx`              | 关闭 Tauri 应用窗口时              | 询问是否同时关闭 opencode serve 服务。使用 `lazy()` 懒加载                                             |
| **ProjectDialog**         | `features/chat/ProjectDialog.tsx`                | 侧边栏"打开项目"按钮 / 命令面板    | 文件系统浏览器，用于选择工作目录                                                                       |
| **ShareDialog**           | `features/chat/ShareDialog.tsx`                  | ChatPane 内的分享按钮              | 分享当前会话，生成公开链接                                                                             |
| **PermissionDialog**      | `features/chat/PermissionDialog.tsx`             | Agent 请求权限时（内联）           | 工具调用权限审批                                                                                       |
| **QuestionDialog**        | `features/chat/QuestionDialog.tsx`               | Agent 提问时（内联）               | 回答 Agent 的问题                                                                                      |
| **AttachmentDetailModal** | `features/attachment/AttachmentDetailModal.tsx`  | 点击附件详情                       | 查看附件详细信息                                                                                       |
| **DiffModal**             | `components/DiffModal.tsx`                       | 查看文件变更                       | 单文件 diff 对比                                                                                       |
| **MultiFileDiffModal**    | `components/MultiFileDiffModal.tsx`              | 查看多文件变更                     | 多文件 diff 对比                                                                                       |
| **FullscreenViewer**      | `components/FullscreenViewer.tsx`                | 全屏查看文件                       | 全屏查看代码 / 图片                                                                                    |
| **ContextDetailsDialog**  | `features/chat/sidebar/ContextDetailsDialog.tsx` | 侧边栏上下文详情按钮               | 查看会话上下文详情                                                                                     |
| **ToastContainer**        | `components/ToastContainer.tsx`                  | 全局通知                           | Toast 通知容器，始终渲染                                                                               |
| **ConfirmDialog**         | `components/ui/ConfirmDialog.tsx`                | 通用确认场景                       | 基础确认对话框原语                                                                                     |

---

## Layout Components（布局组件）

应用采用 **Sidebar + 多 Pane 聊天 + 右侧面板 + 底部面板** 的四区域布局。

```
┌─────────────────────────────────────────────────────────────────┐
│  App (h-[var(--app-height)])                                    │
│  ┌──────────┬──────────────────────────────────┬───────────────┐│
│  │          │                                  │               ││
│  │ Sidebar  │        Chat Surface              │  RightPanel   ││
│  │ (可折叠)  │  ┌────────────────────────────┐ │  (可折叠)     ││
│  │          │  │ Header / PaneHeader        │ │               ││
│  │ 会话列表  │  ├────────────────────────────┤ │  文件浏览器    ││
│  │ 最近文件  │  │ SplitContainer             │ │  终端          ││
│  │ 新建按钮  │  │ ┌──────────┬────────────┐  │ │  MCP 服务器    ││
│  │ 设置按钮  │  │ │ ChatPane │ ChatPane   │  │ │  Skill 面板    ││
│  │          │  │ │          │            │  │ │  Worktree     ││
│  │          │  │ └──────────┴────────────┘  │ │  会话变更      ││
│  │          │  │   (SplitContainer 递归树)   │ │               ││
│  │          │  ├────────────────────────────┤ │               ││
│  │          │  │ BottomPanel                │ │               ││
│  │          │  │ (终端标签 / 文件浏览器)      │ │               ││
│  │          │  └────────────────────────────┘ │               ││
│  └──────────┴──────────────────────────────────┴───────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 核心布局组件

| 组件               | 位置                               | 说明                                                                                                                                                                                              |
| ------------------ | ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sidebar**        | `features/chat/Sidebar.tsx`        | 左侧边栏，包含会话列表、最近文件夹、新建会话、设置入口。可折叠，支持拖拽调整宽度                                                                                                                  |
| **ChatPane**       | `features/chat/ChatPane.tsx`       | 聊天窗格，是单 pane 和 split pane 共用的最小聊天表面。包含 Header/PaneHeader、ChatArea、InputBox、ModelSelector、PermissionDialog、QuestionDialog                                                 |
| **SplitContainer** | `features/chat/SplitContainer.tsx` | 递归分割容器，渲染 `PaneNode` 树。叶子节点通过 `renderLeaf` 回调渲染为 `ChatPane`。支持拖拽分隔线调整大小，CSS Grid 实现，全屏模式使用 `content-visibility: hidden` 避免卸载开销                  |
| **RightPanel**     | `components/RightPanel.tsx`        | 右侧面板，可折叠，支持多标签切换。包含：SessionChanges（会话变更）、FileExplorer（文件浏览器）、Terminal（终端）、McpPanel（MCP 服务器）、SkillPanel（Skill 列表）、WorktreePanel（Git Worktree） |
| **BottomPanel**    | `components/BottomPanel.tsx`       | 底部面板，默认收起。包含：Terminal（终端）、SessionChanges（会话变更）、FileExplorer（文件浏览器）、McpPanel（MCP 服务器）、SkillPanel（Skill 列表）、WorktreePanel（Git Worktree）               |
| **PanelContainer** | `components/PanelContainer.tsx`    | 面板容器，为 RightPanel 和 BottomPanel 提供标签栏 + 内容区的通用外壳                                                                                                                              |
| **ResizablePanel** | `components/ui/ResizablePanel.tsx` | 可拖拽调整大小的面板原语，用于 RightPanel 宽度和 BottomPanel 高度调整                                                                                                                             |

### 布局 Store

| Store                 | 说明                                                                         |
| --------------------- | ---------------------------------------------------------------------------- |
| `layoutStore`         | 管理底部面板开关、终端标签页、右侧面板开关与宽度                             |
| `paneLayoutStore`     | 管理 Split 模式：pane 树结构、focused pane、fullscreen pane、pane 间分割比例 |
| `paneControllerStore` | 管理每个 pane 的控制器（用于导航、新建会话等操作）                           |

### 响应式视口

`chatViewport.tsx` 定义了 `ChatViewportController`，根据视口宽度、Sidebar 状态、RightPanel 状态计算布局策略：

- `sidebarBehavior`: `'push'`（推挤内容区）或 `'overlay'`（覆盖在内容区上）
- `rightPanelBehavior`: 同上
- `CHAT_SURFACE_MIN_WIDTH`: 内容区最小宽度常量
- `canUseSplitPane(viewport)`: 判断当前视口是否满足 split pane 条件

---

## 模块说明

### `src/api/` — API 请求层

封装所有与 OpenCode 后端的通信。基于 `src/api/client.ts` 初始化的 SDK 客户端，提供类型安全的 API 调用。`src/api/events.ts` 负责 SSE 事件订阅，是全局实时消息的唯一入口。

### `src/features/` — 业务功能模块

按功能域组织的业务代码，每个子模块包含组件、Hook、类型和测试。

- **chat/** — 聊天界面核心，包含 Sidebar、ChatPane、ChatArea、InputBox、SplitContainer 等
- **message/** — 消息渲染引擎，将 API 返回的消息 parts 渲染为可视化的 UI（文本、代码、工具调用、推理过程等）
- **mention/** — `@` 提及功能，支持在输入框中 @ 文件、文件夹、规则等
- **slash-command/** — `/` 斜杠命令功能
- **sessions/** — 会话管理（项目选择器、会话列表）
- **settings/** — 设置面板（6 个标签页）
- **attachment/** — 附件管理（图片上传、文件附件）

### `src/hooks/` — 自定义 React Hooks

47 个 Hook，覆盖路由、会话管理、权限、模型选择、主题、快捷键、虚拟滚动等。其中 `useRouter.ts` 是自定义 Hash 路由的核心，`useGlobalEvents.ts` 负责 SSE 事件的全局分发。

### `src/store/` — 状态管理

使用 Zustand 管理全局状态。核心 store 包括：

- `messageStore` — 消息存储（最核心，管理所有 session 的消息）
- `paneLayoutStore` — 窗格布局（Split 模式）
- `layoutStore` — 底部面板和右侧面板状态
- `serverStore` — 多服务器管理
- `themeStore` — 主题系统
- `keybindingStore` — 快捷键配置

### `src-tauri/` — Tauri 桌面应用

Rust 编写的桌面应用外壳，通过 Tauri 2 框架打包为 macOS / Linux / Windows 原生应用。核心功能：

- 自动启动 `opencode serve` 服务
- 管理 SSE 连接
- 提供 Tauri 命令（`start_opencode_service` 等）
- 支持 Android 平台（生成的 Gradle 项目）

### `src-router/` — 动态端口路由器

独立的 Rust 服务，用于 Docker 部署场景。自动扫描容器内 3000-9999 端口的 HTTP 服务，通过 Caddy 动态生成反向代理配置，提供 `/p/{token}/` 路径访问开发服务预览。

### `docker/` — Docker 部署配置

包含三个 Dockerfile（frontend / backend / gateway）、Caddy 配置、Nginx 示例配置和入口脚本。支持两种部署模式：

- **完整部署**：`docker-compose.yml`，包含 Gateway、Frontend、Backend、Router
- **纯前端部署**：`docker-compose.standalone.yml`，仅前端容器，连接已有后端
