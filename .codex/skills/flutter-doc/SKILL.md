---
name: flutter-doc
description: 用于 Flutter/Dart 文档本地化桌面工具项目的开发、设计、排错、测试、打包、数据库与翻译相关工作。遇到该项目的架构、镜像、本地化、预览、GUI/CLI 或交付问题时使用此技能。
---

# Flutter/Dart 文档本地化桌面工具

## 概览

这个技能服务于 `C:\app\flutter_doc` 中的 Flutter/Dart 文档本地化桌面工具项目。
它采用“主技能导航 + 按需加载 references”的结构：主 `SKILL.md` 只负责路由，具体规则和细节放在 `references/` 中。

## 何时使用

当任务涉及以下内容时使用此技能：

- 设计或实现整站镜像、本地化、翻译、预览相关功能
- 调整 Python / PySide6 / SQLite / PyInstaller 方案下的项目结构
- 排查 GUI 卡死、线程、数据库、HTTP、文件路径等问题
- 为该项目补测试、验证打包、梳理数据库迁移或交付要求

如果只是做与本项目无关的通用 Python 问题，不需要使用这个技能。

## 使用流程

1. 先检查当前工作区是否存在项目源码与交付文件，例如 `src/`、`build.bat`、`tests/`、`requirements*.txt`、`pyproject.toml`。
2. 如果当前工作区只有技能文档或 IDE 配置，不要假设源码已存在；先向用户确认真实工程路径或让用户同步源码后再继续。
3. 再读 [constraints](references/constraints.md)，确认不能突破的项目边界。
4. 按任务类型读取对应主题文档，不要一次性加载全部 reference。
5. 需要交付或验收时，补读 [testing](references/testing.md) 和 [build](references/build.md)。
6. 遇到异常、历史坑或线程/数据库类问题时，补读 [errors](references/errors.md)。

## 任务路由

### 先读

- [constraints](references/constraints.md): 工程目录、技术选型、GUI/CLI、性能、安全、测试与打包约束

### 常见入口

- [overview](references/overview.md): 项目目标、技术栈、目标站点、核心原则
- [architecture](references/architecture.md): 分层架构、目录结构、枚举、关键实现约束
- [database](references/database.md): SQLite 表结构、索引、迁移与管理界面
- [mirror](references/mirror.md): 双阶段镜像、种子 URL、下载与资源覆盖
- [localize](references/localize.md): 四站互链、CSS/图片/字体改写、视觉一致性
- [translation](references/translation.md): 文本抽取、词库、翻译包、回填、校验与日志
- [preview](references/preview.md): 内置 HTTP 服务、离线搜索、预览回退规则
- [gui-cli](references/gui-cli.md): 5 个主标签页、CLI 命令、任务控制
- [testing](references/testing.md): 单元、集成、任务控制、增量、打包测试
- [build](references/build.md): `build.bat`、VS Code 调试、交付物
- [errors](references/errors.md): 常见错误与修复策略

## 场景示例

- 开发镜像功能：读 `constraints` + `mirror` + `testing`
- 开发翻译回填：读 `constraints` + `translation` + `database`
- 排查 GUI 卡死或停止按钮：读 `constraints` + `gui-cli` + `architecture` + `errors`
- 排查配置保存失效或“开始镜像”无响应：读 `constraints` + `gui-cli` + `mirror` + `errors`
- 设计预览与离线搜索：读 `constraints` + `preview` + `database`
- 梳理打包交付：读 `constraints` + `build` + `testing`

## 默认工作方式

- 保持 Python 桌面方案，不切换到 Flutter Desktop 或 Electron
- 先以工作区真实文件结构为准，文档与仓库不一致时先指出差异，再实施修改
- 默认目标始终是四个固定站点：`docs.flutter.dev`、`api.flutter.dev`、`dart.dev`、`api.dart.dev`
- GUI 面向中文用户，CLI 与代码标识符保持英文
- 优先做稳定、可恢复、可验证的实现，再追求吞吐
