---
name: flutter_doc_gui
description: "Flutter/Dart 文档本地化工具GUI与CLI功能。包含5个主标签页、CLI命令列表、并发与任务控制。开发界面相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - GUI 与 CLI 功能

## GUI 5 个主标签页

| 标签页       | 核心功能                                                           |
| --------- | -------------------------------------------------------------- |
| **项目设置**  | 配置站点信息、下载并发数、本地化并发数、超时重试、本地预览端口、导出包格式                          |
| **抓取/更新** | 双阶段状态显示（下载队列/已下载/待本地化/本地化中/已本地化）、开始/暂停/继续/一键更新按钮、失败项单独重试、线程池状态、视觉一致性验证 |
| **词库管理**  | 词库浏览、术语锁定/解锁、禁止翻译词标记、词频统计                                      |
| **翻译队列**  | 导出待翻译包、导入已翻译包、回填进度显示、失败日志查看                                    |
| **本地预览**  | 启动/停止预览服务、内置浏览器预览、中英文切换、离线搜索、视觉对比验证                             |

## CLI 命令列表

| 命令                                        | 功能说明                  |
| ----------------------------------------- | --------------------- |
| `init`                                    | 初始化项目                 |
| `mirror --site <site>`                    | 全量镜像站点（双阶段自动执行）       |
| `resume --site <site>`                    | 续跑上次任务（自动判断阶段）        |
| `update --site <site>`                    | 增量更新站点                |
| `extract --site <site>`                   | 抽取待翻译文本               |
| `export-translation-pack --output <path>` | 导出待翻译包（JSONL/CSV）     |
| `import-translation-pack --file <path>`   | 导入已翻译包并回填             |
| `serve --port <port>`                     | 启动本地预览服务              |
| `build-exe`                               | 调用 PyInstaller 打包 exe |

## 并发与任务控制

### 双线程池分离

- 下载线程池：仅负责网络 IO 和原始文件落盘，由 `download_concurrency` 控制
- 本地化重建线程池：仅负责 HTML/CSS 改写，由 `concurrency` 控制
- 两池互不干扰，避免下载被改写拖死

### 优雅暂停

- 设置 `is_paused` 标志，停止领取新任务
- 等待当前下载/改写批次完成后，将状态落库
- 支持程序重启后从数据库和 `.part` 临时文件恢复

### 状态持久化

- 所有任务状态唯一依赖 SQLite，避免内存状态丢失
- 支持旧项目数据库迁移（拆分 `url_queue` → `url_queue` + `localize_queue`）

## 已落地经验

### 项目设置页

- 保存配置后必须展示实际写入路径，避免“提示保存成功但用户不知道写到哪里”
- 默认配置路径必须通过统一解析函数计算
- 源码运行时写入工程根目录的 `flutter_doc_config.json`
- 打包运行时写入 `sys.executable` 同目录，避免把配置误写到 `_internal` 或临时目录

### 抓取/更新页

- “开始镜像”和“一键更新”必须启动真实后台任务，不能只输出“功能已准备好”之类的占位日志
- GUI 入口应创建 worker 和 `QThread`，在后台复用服务层执行镜像
- “暂停/继续”必须作用于同一调度器实例，而不是只修改界面状态
- 双阶段状态应从 SQLite `task_status` 刷新，不能只依赖内存变量
- “完成/总数”里的总数必须来自 `url_queue` / `localize_queue` 的真实统计，不能用 `completed + failed` 伪造
- 下载阶段和本地化阶段必须分别写各自的 `task_status`
- 阶段结束后必须把状态切到 `completed` / `paused` / `failed`，不能一直停在“运行中”
- 主窗口关闭时应停止轮询并向 worker 发出协作式停止信号

## 相关技能

- [flutter_doc_architecture](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_architecture/SKILL.md) - 系统架构设计
- [flutter_doc_mirror](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_translation](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_translation/SKILL.md) - 翻译服务
- [flutter_doc_preview](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_preview/SKILL.md) - 预览服务
