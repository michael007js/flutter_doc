# 数据库设计

## 目标

SQLite 是任务状态、资源缓存、翻译记忆和搜索索引的唯一权威来源。设计时优先保证：

- 可恢复
- 可迁移
- 可查询
- 能支撑多线程任务状态持久化

## 核心表

### `url_queue`

用于下载阶段的 URL 队列与状态流转：

- `queued`
- `downloading`
- `downloaded`
- `failed`

建议字段包括：`url`、`site`、`depth`、`retry_count`、`error_message`、时间戳。

### `localize_queue`

用于本地化重建阶段：

- `queued`
- `rebuilding`
- `completed`
- `failed`

建议字段包括：`url`、`site`、`raw_html_path`、`localized_html_path`、错误信息、时间戳。

### `resource_cache`

记录原始资源与本地路径的映射，至少包含：

- `original_url`
- `local_path`
- `content_hash`
- `size`
- `mime_type`
- `downloaded_at`

### `translation_memory`

翻译记忆是翻译流程的核心：

- `text_hash`
- `original_text`
- `translated_text`
- `context`
- `source_urls`
- `occurrence_count`
- `status`
- `is_locked`
- `is_protected`

### `task_status`

记录全局任务进度与阶段统计，至少包含：

- `task_type`
- `site`
- `phase`
- `total_count`
- `completed_count`
- `failed_count`
- `status`

### `page_content`

记录页面路径、本地化状态与增量更新依据，至少包含：

- `url`
- `site`
- `raw_html_path`
- `localized_html_path`
- `is_localized`
- `content_hash`

### `fail_log`

集中记录下载、翻译、回填等失败项，保留：

- `task_type`
- `url`
- `error_message`
- `stack_trace`
- `created_at`

## 搜索索引

使用 FTS5 建立离线搜索表，例如：

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    url,
    title,
    content_en,
    content_zh,
    tokenize = 'porter unicode61'
);
```

预览能力可以在此基础上扩展 `site`、`type` 等字段。

## 设计要求

- 为高频查询字段建立索引，尤其是 `url`、`site`、`status`。
- 批量写入减少数据库往返次数。
- 尽量复用连接能力，避免频繁创建/关闭连接。
- 多线程访问时要有锁、超时和 WAL 方案。
- 需要支持旧数据库迁移到新队列拆分结构。

## 管理能力

项目希望具备一个数据库管理界面，允许在 GUI 中查看和批量操作数据，而不是依赖外部工具。

## 推荐联读

- 架构分层：见 [architecture](architecture.md)
- 翻译与搜索如何使用这些表：见 [translation](translation.md) 与 [preview](preview.md)
- SQLite 常见问题：见 [errors](errors.md)

