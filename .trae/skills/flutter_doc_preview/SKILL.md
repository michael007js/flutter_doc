---
name: flutter_doc_preview
description: "Flutter/Dart 文档本地化工具预览服务。包含内置HTTP服务、离线搜索、中英文切换。开发预览相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 预览服务

## 本地预览与离线搜索

### 内置 HTTP 服务

- 预览服务优先读取 `localized mirror`
- 支持中英文站点切换预览
- 默认端口：8080（可配置）
- 支持 MIME 类型自动识别

### 服务架构

```
┌─────────────────────────────────────────────────────────────┐
│                    PreviewService                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ HTTP Server  │  │ File Handler │  │ Search API   │      │
│  │ (QThread)    │  │ (localized)  │  │ (FTS5)       │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
├─────────────────────────────────────────────────────────────┤
│                    Data Sources                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ localized    │  │ raw mirror   │  │ SQLite FTS5  │      │
│  │ mirror       │  │ (fallback)   │  │ search_index │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 离线搜索替换

- `docs.flutter.dev/search` / `dart.dev/search` 替换为内置离线搜索页
- `api.flutter.dev` / `api.dart.dev` 支持英文符号名和中文说明检索（基于 FTS5）

### 搜索功能

- **符号搜索**：支持英文函数名、类名、方法名搜索
- **内容搜索**：支持中文说明文档搜索
- **混合搜索**：同时搜索符号和内容
- **搜索结果排序**：按相关度排序，支持分页

### 文件服务规则

1. 优先从 `localized mirror` 读取文件
2. 如果 `localized mirror` 中不存在，从 `raw mirror` 读取
3. 如果文件正在本地化中，返回占位页面
4. 支持范围请求（Range Request）用于大文件断点续传

### 线程安全

- HTTP 服务运行在独立线程中，不阻塞主线程
- 使用 Qt 信号槽机制进行跨线程通信
- 服务启动/停止通过信号通知主线程

## 核心类接口

```python
class PreviewService:
    """本地预览服务（内置 HTTP 服务、离线搜索）"""
    def __init__(self, db: Database, project_path: Path):
        pass

    def start_server(self, port: int = 8080) -> None:
        """启动本地 HTTP 预览服务"""
        pass

    def stop_server(self) -> None:
        """停止预览服务"""
        pass

    def is_running(self) -> bool:
        """检查服务是否正在运行"""
        pass

    def get_url(self, site: str, path: str = "") -> str:
        """获取指定站点的预览 URL"""
        pass

    def search(self, query: str, lang: str = "zh") -> List[Dict]:
        """离线搜索（支持英文符号名、中文说明检索）"""
        pass

    def replace_search_page(self, site: str) -> None:
        """替换 docs.flutter.dev/search 和 dart.dev/search 为内置离线搜索页"""
        pass

    def rebuild_search_index(self, site: str = None) -> None:
        """重建搜索索引"""
        pass
```

## 搜索索引结构

```sql
-- FTS5 全文搜索索引
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    url,           -- 页面 URL
    title,         -- 页面标题
    content_en,    -- 英文内容
    content_zh,    -- 中文内容（翻译后）
    site,          -- 所属站点
    type,          -- 页面类型（doc/api）
    tokenize = 'porter unicode61'
);
```

## 预览页面模板

### 搜索页面

- 提供搜索输入框
- 显示搜索结果列表
- 支持点击跳转到对应页面

### 占位页面

- 显示"正在本地化，请稍后..."
- 显示当前本地化进度
- 提供刷新按钮

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_database](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_database/SKILL.md) - 数据库设计（搜索索引）
- [flutter_doc_localize](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_localize/SKILL.md) - 本地化服务
- [flutter_doc_gui](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_gui/SKILL.md) - GUI 界面
