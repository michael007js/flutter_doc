---
name: flutter_doc_database
description: "Flutter/Dart 文档本地化工具数据库设计。包含表结构、索引、性能优化、管理界面。开发数据库相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 数据库设计

## 核心表结构

```sql
-- URL 队列表（仅管理下载阶段状态）
CREATE TABLE url_queue (
    id INTEGER PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    site TEXT NOT NULL,
    status TEXT DEFAULT 'queued', -- queued/downloading/downloaded/failed
    depth INTEGER DEFAULT 0,
    retry_count INTEGER DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 本地化重建队列表（管理本地化阶段状态）
CREATE TABLE localize_queue (
    id INTEGER PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    site TEXT NOT NULL,
    status TEXT DEFAULT 'queued', -- queued/rebuilding/completed/failed
    raw_html_path TEXT,
    localized_html_path TEXT,
    error_message TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 资源缓存表
CREATE TABLE resource_cache (
    id INTEGER PRIMARY KEY,
    original_url TEXT UNIQUE NOT NULL,
    local_path TEXT NOT NULL,
    content_hash TEXT,
    size INTEGER,
    mime_type TEXT,
    downloaded_at TIMESTAMP
);

-- 翻译记忆表（文本单元去重、翻译复用）
CREATE TABLE translation_memory (
    id INTEGER PRIMARY KEY,
    text_hash TEXT UNIQUE NOT NULL,
    original_text TEXT NOT NULL,
    translated_text TEXT,
    context TEXT,
    source_urls TEXT,
    occurrence_count INTEGER DEFAULT 1,
    status TEXT DEFAULT 'pending', -- pending/translated/verified
    is_locked INTEGER DEFAULT 0, -- 术语锁定
    is_protected INTEGER DEFAULT 0, -- 禁止翻译词
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 任务状态表（双阶段任务总控）
CREATE TABLE task_status (
    id INTEGER PRIMARY KEY,
    task_type TEXT NOT NULL,
    site TEXT,
    phase TEXT, -- download/localize/extract/translate
    total_count INTEGER DEFAULT 0,
    completed_count INTEGER DEFAULT 0,
    failed_count INTEGER DEFAULT 0,
    status TEXT DEFAULT 'idle', -- idle/running/paused/completed/failed
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- 页面内容表
CREATE TABLE page_content (
    id INTEGER PRIMARY KEY,
    url TEXT UNIQUE NOT NULL,
    site TEXT NOT NULL,
    raw_html_path TEXT,
    localized_html_path TEXT,
    is_localized BOOLEAN DEFAULT 0,
    content_hash TEXT, -- 用于增量更新检测
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 失败日志表
CREATE TABLE fail_log (
    id INTEGER PRIMARY KEY,
    task_type TEXT NOT NULL,
    url TEXT,
    error_message TEXT,
    stack_trace TEXT,
    created_at TIMESTAMP
);

-- SQLite FTS5 全文搜索索引（用于 API 文档离线搜索）
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    url,
    title,
    content_en,
    content_zh,
    tokenize = 'porter unicode61'
);
```

## 数据库管理

- 创建一个专门的数据库管理界面模块，支持直接在程序Gui中管理数据库连接，无需外部工具。
- 可以对数据进行批量操作，如查看/插入/更新/删除数据。

## 数据库性能优化

- **索引优化**：为常用查询字段添加索引，如 `url_queue` 中的 `url` 字段。
- **批量操作**：批量插入/更新/删除数据，减少数据库操作次数。
- **单一连接**：使用单一数据库连接池，不再频繁开关数据库连接。
- **线程锁**：添加线程锁保证线程安全，避免并发操作导致的数据库锁问题。
- **复用**：所有操作复用同一个数据库连接，避免频繁创建和销毁数据库连接。

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_architecture](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_architecture/SKILL.md) - 系统架构设计
- [flutter_doc_mirror](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_translation](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_translation/SKILL.md) - 翻译服务
- [flutter_doc_preview](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_preview/SKILL.md) - 预览服务（搜索索引）
- [flutter_doc_errors](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_errors/SKILL.md) - 错误经验
