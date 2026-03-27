---
name: flutter_doc_architecture
description: "Flutter/Dart 文档本地化工具系统架构。包含分层架构、目录结构、代码说明、枚举定义。开发架构相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 系统架构设计

## 分层架构

遵循 **GUI 层 → 业务逻辑层 → 核心服务层 → 数据访问层** 四层结构，职责清晰，易扩展维护。

```
┌─────────────────────────────────────────────────────────────┐
│                      GUI Layer (PySide6)                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 项目设置  │ │ 抓取/更新 │ │ 词库管理  │ │ 本地预览  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├─────────────────────────────────────────────────────────────┤
│                    Business Logic Layer                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ 站点镜像  │ │ 资源本地化│ │ 文本抽取  │ │ 翻译回填  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├─────────────────────────────────────────────────────────────┤
│                    Core Services Layer                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ HTTP客户端│ │ HTML解析 │ │ 任务调度  │ │ 词库管理  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
├─────────────────────────────────────────────────────────────┤
│                    Data Access Layer                         │
│  ┌──────────────────────────────────────────────────┐       │
│  │              SQLite Database                      │       │
│  │  (URL队列、资源缓存、翻译记忆、任务状态、本地化队列)  │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

## 项目目录结构

```
flutter_doc/
├── src/
│   ├── __init__.py
│   ├── main.py                    # 程序入口（GUI/CLI 统一调度）
│   ├── config.py                  # 配置管理（站点、线程数、端口等）
│   │
│   ├── core/                      # 核心服务层
│   │   ├── __init__.py
│   │   ├── database.py            # SQLite 数据库管理（含 FTS5 索引）
│   │   ├── http_client.py         # HTTP 客户端（断点续传、Range 请求）
│   │   ├── html_parser.py         # HTML/CSS 解析、链接改写、文本抽取
│   │   ├── task_scheduler.py      # 任务调度器（双线程池、优雅暂停）
│   │   └── url_manager.py         # URL 规范化、去重、队列管理
│   │
│   ├── services/                  # 业务服务层
│   │   ├── __init__.py
│   │   ├── mirror_service.py      # 站点镜像服务（双阶段：原站下载→本地化重建）
│   │   ├── localize_service.py    # 资源本地化服务（HTML/CSS 改写、硬链接同步）
│   │   ├── extract_service.py     # 文本抽取服务（去重、词库生成）
│   │   ├── translation_service.py # 翻译包导入导出（校验、回填、翻译记忆复用）
│   │   ├── vocabulary_service.py  # 词库管理服务（术语锁定、禁止翻译词）
│   │   └── preview_service.py     # 本地预览服务（内置 HTTP 服务、离线搜索）
│   │
│   ├── gui/                       # GUI 层
│   │   ├── __init__.py
│   │   ├── main_window.py         # 主窗口（5 标签页容器）
│   │   ├── widgets/               # 自定义控件
│   │   │   ├── __init__.py
│   │   │   ├── project_tab.py     # 项目设置标签页
│   │   │   ├── mirror_tab.py      # 抓取/更新标签页（双阶段状态显示）
│   │   │   ├── vocabulary_tab.py  # 词库标签页
│   │   │   ├── translation_tab.py # 翻译队列标签页
│   │   │   └── preview_tab.py     # 本地预览标签页
│   │   ├── dialogs/               # 对话框
│   │   │   ├── __init__.py
│   │   │   └── progress_dialog.py # 进度对话框
│   │   └── models/                # 数据模型
│   │       ├── __init__.py
│   │       └── task_model.py      # 任务状态模型（双阶段绑定）
│   │
│   ├── cli/                       # CLI 接口
│   │   ├── __init__.py
│   │   └── commands.py            # CLI 命令实现（与 GUI 共用服务层）
│   │
│   └── utils/                     # 工具函数
│       ├── __init__.py
│       ├── hash_utils.py          # Hash 计算（文本单元去重）
│       ├── file_utils.py          # 文件操作（硬链接、临时文件管理）
│       └── text_utils.py          # 文本处理（占位符保护、规范化）
│
├── data/                          # 数据目录（运行时生成）
│   ├── projects/                  # 项目数据（raw mirror、localized mirror）
│   └── cache/                     # 缓存数据（临时文件、资源哈希）
│
├── tests/                         # 测试目录
│   ├── __init__.py
│   ├── test_database.py
│   ├── test_mirror.py
│   ├── test_extract.py
│   └── test_translation.py
│
├── requirements.txt               # Python 依赖
├── pyproject.toml                 # 项目配置
├── setup.py                       # 打包配置
└── README.md                      # 项目说明
```

## 代码说明

- 目标：提高项目规范，提高项目功能健壮性。
- 为项目代码添加注释，说明每个模块的功能和实现细节。
- 严格遵守本章节代码说明。

### 枚举说明

阶段任务枚举 TaskPhase 

   - idle → 空闲
   - download → 下载
   - localize → 本地化
   - extract → 提取
   - translate → 翻译

任务状态枚举 TaskStatus 

   - idle → 空闲
   - running → 运行中
   - paused → 已暂停
   - completed → 已完成
   - failed → 失败

### 阶段状态枚举

   - discovering → 发现种子URL
   - crawling → 爬取页面
   - queueing → 加入队列
   - download → 下载
   - localize → 本地化
   - checking → 检查更新

### 代码功能详细补充

#### MirrorWorker 添加停止事件 ( mirror_tab.py )

- 使用 threading.Event 替代简单的布尔变量
- 添加 set_stop_event 方法传递停止事件给服务
- 在发射信号前检查是否已停止

#### MirrorService 添加停止支持

- 显示总共多少个种子URL
- 显示当前正在获取哪个URL
- 显示解析sitemap/JSON后获得了多少个URL
- 显示最终去重后的唯一URL数量
- 添加 _stop_event 和 _is_stopped() 方法
- 在所有关键位置检查停止状态
- 停止时不再发射进度信号

#### 停止按钮不再阻塞 GUI ( mirror_tab.py )

- 使用 terminate() 替代 wait() ，避免阻塞
- 断开信号连接防止内存泄漏

#### 移除完成时的弹窗 ( mirror_tab.py )

- QMessageBox 会阻塞 GUI，改为只记录日志

#### 进度匹配改为中文 ( mirror_tab.py )

- "download" → "下载"
- "localize" → "本地化"

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_overview](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_overview/SKILL.md) - 项目概述
- [flutter_doc_database](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_database/SKILL.md) - 数据库设计
- [flutter_doc_mirror](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_localize](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_localize/SKILL.md) - 本地化服务
