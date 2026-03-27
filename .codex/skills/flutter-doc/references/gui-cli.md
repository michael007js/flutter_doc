# GUI 与 CLI

## GUI 5 个主标签页

项目界面固定为 5 个主标签页：

| 标签页 | 核心职责 |
| --- | --- |
| 项目设置 | 站点配置、并发、超时、端口、导出格式 |
| 抓取/更新 | 双阶段状态、开始/暂停/继续/更新、失败重试、线程池状态 |
| 词库管理 | 术语锁定、禁止翻译词、词频浏览 |
| 翻译队列 | 导出包、导入包、回填进度、失败日志 |
| 本地预览 | 服务启停、内置浏览器、中英文切换、离线搜索 |

GUI 文案必须是中文。

## CLI 命令

既有命令集包括：

- `init`
- `mirror --site <site>`
- `resume --site <site>`
- `update --site <site>`
- `extract --site <site>`
- `export-translation-pack --output <path>`
- `import-translation-pack --file <path>`
- `serve --port <port>`
- `build-exe`

设计新功能时，优先复用这组命令，不随意新增不必要命令。

## 并发与任务控制

### 双线程池

- 下载线程池：负责网络下载与原始落盘
- 本地化线程池：负责 HTML/CSS 改写

两池隔离，避免互相阻塞。

### 暂停与恢复

- 设置暂停标记后停止领取新任务
- 等待当前批次安全完成后落库
- 支持从数据库状态与临时文件恢复

### 状态持久化

- 所有权威状态依赖 SQLite
- 支持旧项目迁移到新队列结构

## GUI 实现注意点

- 后台线程与 GUI 通信优先使用 Qt 信号槽
- 停止按钮不能靠阻塞等待实现
- 完成态优先写日志，不依赖阻塞式弹窗
- 进度阶段在界面中使用中文词汇

## 已落地经验

### 项目设置页

- 保存配置后应回显实际写入路径，避免用户误以为“保存成功”但写到了错误位置
- 配置默认路径应统一通过单一解析函数获取
- 源码运行时写入工程根目录的 `flutter_doc_config.json`
- PyInstaller 打包后写入 `sys.executable` 同目录，而不是 `_internal` 下的源码相对路径

### 抓取/更新页

- “开始镜像”和“一键更新”不能只连接占位日志，必须真正启动后台任务
- GUI 按钮应创建 worker + `QThread`，在后台复用共享服务层执行镜像
- “暂停/继续”应作用于同一调度器实例，不能只写日志或切换按钮状态
- 双阶段状态优先从 SQLite 刷新，避免只依赖内存态导致界面和真实任务脱节
- 进度文案中的“总数”必须来自 `url_queue` / `localize_queue` 的真实汇总，不能写成 `completed + failed`
- 下载阶段和本地化阶段应分别维护各自的 `task_status`，不要混用同一口径
- 阶段结束后必须把 `task_status.status` 切到 `completed` / `paused` / `failed`，避免界面长期停留在“运行中”
- 主窗口关闭时应停止状态轮询，并向后台任务发出协作式停止信号

## 推荐联读

- 线程与停止细节：见 [architecture](architecture.md)
- 镜像任务的运行方式：见 [mirror](mirror.md)
- 预览服务能力：见 [preview](preview.md)
- 已知 GUI / 线程错误：见 [errors](errors.md)
