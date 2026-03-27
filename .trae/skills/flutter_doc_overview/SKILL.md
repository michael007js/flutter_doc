---
name: flutter_doc_overview
description: "Flutter/Dart 文档本地化工具项目概述。包含项目目标、技术栈、目标站点、核心原则。开发前必读。"
---

# Flutter/Dart 文档本地化工具 - 项目概述

**重要**：开发前请先阅读 [flutter_doc_constraints](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_constraints/SKILL.md) 了解开发约束。

## 项目目标

交付一个 Windows 可视化桌面程序，实现对 Flutter/Dart 官方 4 大站点的**整站递归本地化镜像、文本抽取与翻译包管理、翻译回填与离线预览**，支持多线程、断点续传、增量更新，最终产出可双击运行的 exe。

## 技术栈

- **开发语言**：Python 3.8+
- **GUI 框架**：PySide6 6.5.x + QWebEngineView
- **数据库**：SQLite (启用 FTS5 全文搜索)
- **打包工具**：PyInstaller
- **核心能力**：HTTP 断点续传、HTML/CSS 解析改写、多线程任务调度、离线搜索

## 目标站点

固定为以下 4 个站点，站内互链本地化，外域导航保留外链：

1. `docs.flutter.dev`
2. `api.flutter.dev`
3. `dart.dev`
4. `api.dart.dev`

## 核心原则

- **集成在线翻译 API**：负责导出待翻译包、导入已翻译包、校验回填，翻译由外部（Codex/人工/翻译 API）处理。
- **整站递归本地化**：覆盖全部子链接、页面资源及页面依赖的外域静态资源（字体、图片、脚本等）。
- **优雅暂停与断点续传**：停止时等待当前批次完成后落库，重启后从数据库和临时文件恢复。
- **增量更新**：仅抓取变更页面/资源，复用未变化文本的翻译记忆。

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_architecture](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_architecture/SKILL.md) - 系统架构设计
- [flutter_doc_mirror](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_localize](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_localize/SKILL.md) - 本地化服务
