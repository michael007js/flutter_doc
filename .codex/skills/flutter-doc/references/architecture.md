# 系统架构

## 分层架构

遵循四层结构：

1. GUI 层
2. 业务逻辑层
3. 核心服务层
4. 数据访问层

这个拆分的目标是让界面、流程编排、底层能力和持久化职责清晰分离。

## 目录结构

建议的核心目录如下：

```text
src/
  main.py
  config.py
  core/
    database.py
    http_client.py
    html_parser.py
    task_scheduler.py
    url_manager.py
  services/
    mirror_service.py
    localize_service.py
    extract_service.py
    translation_service.py
    vocabulary_service.py
    preview_service.py
  gui/
    main_window.py
    widgets/
    dialogs/
    models/
  cli/
    commands.py
  utils/
    hash_utils.py
    file_utils.py
    text_utils.py
data/
tests/
```

## 关键职责分配

- `core/database.py`: SQLite、索引、事务、连接与查询封装
- `core/http_client.py`: HTTP 请求、重试、断点续传、流式下载
- `core/html_parser.py`: HTML/CSS 解析、链接改写、文本抽取
- `core/task_scheduler.py`: 下载/本地化双线程池与暂停恢复
- `services/mirror_service.py`: 原站镜像下载与阶段编排
- `services/localize_service.py`: 资源同步与 HTML/CSS 重建
- `services/translation_service.py`: 导出、导入、校验、回填
- `services/preview_service.py`: 本地预览、离线搜索、服务启停

## 关键枚举

- `TaskPhase`: `idle` / `download` / `localize` / `extract` / `translate`
- `TaskStatus`: `idle` / `running` / `paused` / `completed` / `failed`
- 阶段状态：`discovering` / `crawling` / `queueing` / `download` / `localize` / `checking`

## 与实现强相关的架构细节

- 下载与本地化必须分线程池，避免相互拖慢。
- 停止任务优先使用显式停止事件，不依赖裸布尔值。
- 停止按钮不能靠阻塞式等待来完成 GUI 同步。
- 进度与阶段信息在界面中使用中文呈现。
- 完成提示优先落日志，避免阻塞式弹窗影响流程。

## 推荐联读

- 数据库细节：见 [database](database.md)
- 任务控制与界面：见 [gui-cli](gui-cli.md)
- 已知线程/数据库坑：见 [errors](errors.md)

