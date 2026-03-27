# 项目概述

## 项目目标

交付一个 Windows 可视化桌面程序，对 Flutter/Dart 官方四个站点执行：

- 整站递归本地化镜像
- 文本抽取与翻译包管理
- 翻译回填
- 离线预览与离线搜索

最终产物是可双击运行的 `exe`，并支持多线程、断点续传与增量更新。

## 技术栈

- 开发语言：Python 3.8+
- GUI：PySide6 6.5.x + QWebEngineView
- 数据库：SQLite，启用 FTS5
- 打包：PyInstaller
- 关键能力：HTTP 断点续传、HTML/CSS 解析改写、多线程任务调度、离线搜索

## 目标站点

固定支持以下四站：

1. `docs.flutter.dev`
2. `api.flutter.dev`
3. `dart.dev`
4. `api.dart.dev`

约束是：站内互链本地化，外域导航保留外链。

## 核心原则

- **翻译能力集成**：负责导出待翻译包、导入已翻译包、校验与回填；具体翻译可由人工、外部 API 或 AI 处理。
- **整站递归本地化**：覆盖页面、页面依赖资源，以及必要的外域静态资源。
- **优雅暂停与恢复**：停止时等当前批次落库，重启后能从数据库和临时文件恢复。
- **增量更新**：只重新抓取变化页面与资源，尽量复用翻译记忆。

## 典型实现心智模型

- 先做“原站下载”，再做“本地化重建”。
- 文本翻译以“文本单元 + hash 去重 + 翻译记忆复用”为核心。
- 预览优先读取 `localized mirror`，必要时回退到 `raw mirror`。

## 推荐联读

- 不可突破的边界：见 [constraints](constraints.md)
- 结构设计：见 [architecture](architecture.md)
- 镜像链路：见 [mirror](mirror.md)
- 翻译链路：见 [translation](translation.md)

