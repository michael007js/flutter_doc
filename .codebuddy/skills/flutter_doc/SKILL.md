---
name: flutter_doc
description: Flutter/Dart 全站本地化桌面工具 - 完整实现文档
---

# Flutter/Dart 全站本地化桌面工具 - 完整实现文档


## 关键约束

**重要关键**：

- 严格按照本文档开发，不改变任何功能。

**约束**：

- 工程目录：C:\app\flutter doc，本文中提到的【工程目录】都是指C:\app\flutter doc目录，请勿改变。
- 保持 Python 桌面方案，不切换到 Flutter 桌面或 Electron。
- “资源本地化”指四站全部页面与其实际依赖的静态资源镜像到本地，且四站互相指向的同步镜像，外部导航页面不递归。
- 首次全量镜像以稳定可恢复优先，不追求单次跑满带宽
- 旧项目数据库支持安全迁移到新结构
- 为保证项目的顺利性，开始前需要与我确认项目计划，有不清楚或模糊的地方，不要自己假设，需要与我确认。

## 一、项目概述

### 1.1 项目目标

交付一个 Windows 可视化桌面程序，实现对 Flutter/Dart 官方 4 大站点的**整站递归本地化镜像、文本抽取与翻译包管理、翻译回填与离线预览**，支持多线程、断点续传、增量更新，最终产出可双击运行的 exe。

### 1.2 技术栈

- **开发语言**：Python 3.8+
- **GUI 框架**：PySide6 6.5.x + QWebEngineView
- **数据库**：SQLite (启用 FTS5 全文搜索)
- **打包工具**：PyInstaller
- **核心能力**：HTTP 断点续传、HTML/CSS 解析改写、多线程任务调度、离线搜索

### 1.3 目标站点

固定为以下 4 个站点，站内互链本地化，外域导航保留外链：

1. `api.flutter.dev`
2. `docs.flutter.dev`
3. `dart.dev`
4. `api.dart.dev`

### 1.4 核心原则

- **集成在线翻译 API**：负责导出待翻译包、导入已翻译包、校验回填，翻译由外部（Codex/人工/翻译 API）处理。
- **整站递归本地化**：覆盖全部子链接、页面资源及页面依赖的外域静态资源（字体、图片、脚本等）。
- **优雅暂停与断点续传**：停止时等待当前批次完成后落库，重启后从数据库和临时文件恢复。
- **增量更新**：仅抓取变更页面/资源，复用未变化文本的翻译记忆。

## 二、系统架构设计

### 2.1 分层架构

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
│  │  (URL队列、资源缓存、翻译记忆、任务状态、本地化队列)  │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 项目目录结构

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

## 三、数据库设计

### 3.1 核心表结构

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
### 3.2 数据库管理

- 创建一个专门的数据库管理界面模块，支持直接在程序Gui中管理数据库连接，无需外部工具。
- 可以对数据进行批量操作，如查看/插入/更新/删除数据。


### 3.3 数据库性能优化

- **索引优化**：为常用查询字段添加索引，如 `url_queue` 中的 `url` 字段。
- **批量操作**：批量插入/更新/删除数据，减少数据库操作次数。
- **单一连接**：使用单一数据库连接池，不再频繁开关数据库连接。
- **线程锁**：添加线程锁保证线程安全，避免并发操作导致的数据库锁问题。
- **复用**：所有操作复用同一个数据库连接，避免频繁创建和销毁数据库连接。

## 四、同步本地化需求

### 4.1 需求概述

为了确保四个站点 (`api.flutter.dev`、`docs.flutter.dev`、`dart.dev`、`api.dart.dev`) 在本地化过程中互链正常，我们需要对这些站点之间的链接进行同步本地化处理。具体来说，我们需要：

- 处理站点内的链接，确保它们指向本地化的页面或资源。
- 保留外部链接不变，确保外部资源（如外部网站的链接）仍然有效。
- 在本地化过程中，实现不同站点间的资源同步，以确保站点间的相互链接不受影响。

### 4.2 同步本地化处理步骤

#### 1. **提取站点内链接**

在本地化每个站点的页面时，我们首先需要提取 HTML 页面中所有的站内链接，包括 `<a href="...">`、`<img src="...">`、`<script src="...">`、`<link href="...">` 等标签中的链接。这些链接可能是相对的或绝对的，我们需要根据它们的性质进行相应的处理。

```python
def extract_internal_links(html: str, base_url: str) -> List[str]:
    """提取 HTML 页面中的站内链接"""
    # 使用正则表达式提取 href、src、srcset 中的链接
    link_pattern = r'href="([^"]+)"|src="([^"]+)"|srcset="([^"]+)"'
    links = re.findall(link_pattern, html)
    internal_links = [l for link in links for l in link if l]
    return internal_links
```

#### 2. **同步站点间的互链**

在本地化过程中，针对站点之间的互链（例如，`api.flutter.dev` 中的某个页面链接到 `dart.dev` 的页面），我们需要确保链接的目标是本地化后的页面路径，而非外部网站。

- 当 `api.flutter.dev` 中的某个页面链接到 `dart.dev`，我们需要将其链接改写为指向本地存储的 `dart.dev` 页面。
- 对于其他站点（如 `docs.flutter.dev` 和 `api.dart.dev`）的互链，也需要按相同的逻辑进行替换。

**处理逻辑：**

- **站内链接改写：** 判断链接是否属于同一个站点，如果是，则需要改写为本地路径。
- **外部链接保持不变：** 任何指向外部站点的链接（如 `https://google.com`）保持不变。

```python
def localize_internal_links(html: str, resource_map: Dict[str, Path], site: str) -> str:
    """将站内链接改写为本地路径"""
    internal_links = extract_internal_links(html, site)
    for link in internal_links:
        # 判断链接是否属于当前站点，且是否已经本地化
        if site in link:
            # 从资源映射中获取本地路径
            local_path = resource_map.get(link)
            if local_path:
                html = html.replace(link, str(local_path))
        # 对外部链接保持原链接
    return html
```

#### 3. **资源同步**

除了同步站内链接外，还需要处理站点间的资源（如图片、JS 文件、CSS 文件等）同步。本地化过程中，所有站点间共享的资源需要从原站下载并存储到本地。这些资源包括：

- 图片：从 `api.flutter.dev` 中下载图片并存储在本地，更新 `src` 属性为本地路径。
- 脚本和样式文件：下载并更新相应的资源链接。

**同步逻辑：**

- 对每个站点的资源文件（如图片、脚本、样式）进行下载。
- 使用硬链接或复制将原始资源同步到本地资源目录，避免占用过多磁盘空间。

```python
def sync_resources(html: str, resource_map: Dict[str, Path]) -> str:
    """同步资源文件到本地，并更新链接"""
    resources = extract_internal_links(html, site)  # 提取资源链接
    for resource in resources:
        if resource in resource_map:
            local_resource_path = resource_map[resource]
            html = html.replace(resource, str(local_resource_path))
    return html
```

#### 4. **外部链接保留原样**

对于所有指向外部网站的链接，我们需要保持其原始形式。外部链接包括：

- 外部网站链接：如 `https://www.google.com`
- 第三方资源链接：如 `https://cdn.example.com/image.png`

这些外部链接不做任何修改，确保用户访问这些链接时能够正确跳转到原始网站。

#### 5. **完整本地化示例**

以下是处理一个页面的完整本地化的代码示例，包括了站内链接的同步本地化和资源同步。

```python
def localize_page(html: str, site: str, resource_map: Dict[str, Path]) -> str:
    """本地化整个页面"""
    # 第一步：同步站内链接
    localized_html = localize_internal_links(html, resource_map, site)
    
    # 第二步：同步资源
    localized_html = sync_resources(localized_html, resource_map)
    
    return localized_html
```

### 4.3 重要提示

- **4站同步** 在进行本地化时，需要对4个站点一起进行同步处理，确保所有站点的链接都能正确指向本地资源路径，这一点非常重要。
- **增量更新：** 在进行本地化时，我们仅对有变化的页面或资源进行重新处理，避免重新抓取所有页面。
- **外部链接：** 外部链接始终保留原样，保证不会改变外部资源的指向。
- **本地资源：** 本地资源如图片、CSS 文件、JS 文件等，必须从站点抓取并存储到本地文件系统，以确保离线预览时能够正常加载。

### 4.4 同步本地化的实现效果

#### 4.4.1 站内链接同步本地化

在本地化完成后，站内的所有链接（如从 `api.flutter.dev` 到 `docs.flutter.dev` 的链接）都会指向本地存储的资源路径，而不是外部的 URL。

#### 4.4.2 资源同步

图片、脚本、样式表等外部资源都会被下载并存储到本地，并在 HTML 页面中改写为本地路径。这确保了站点能够完整地在离线环境下工作。

#### 4.4.3 外部链接保留原样

对于指向外部网站的链接（如 `https://google.com`），它们会保持原样，不会被本地化替换。

## 五、核心类与接口设计

### 5.1 核心服务类

```python
from typing import List, Dict, Connection
from pathlib import Path

class Database:
    """SQLite 数据库管理（含连接池、FTS5 索引维护）"""
    def init_db(self) -> None:
        """初始化数据库表结构与索引"""
        pass

    def get_connection(self) -> Connection:
        """获取数据库连接"""
        pass

    def migrate_old_project(self) -> None:
        """迁移旧项目数据库到新结构（url_queue → localize_queue 拆分）"""
        pass

class HTTPClient:
    """HTTP 客户端（支持断点续传、Range 请求、并发控制）"""
    async def download(self, url: str, save_path: Path, part_suffix: str = ".part") -> bool:
        """下载文件，支持断点续传（使用 .part 临时文件）"""
        pass

    async def fetch(self, url: str) -> str:
        """获取 HTML/JSON 文本内容"""
        pass

class HTMLParser:
    """HTML/CSS 解析与改写、文本抽取"""
    def extract_links(self, html: str, base_url: str) -> List[str]:
        """从 HTML 中提取站内链接（href/src/srcset）"""
        pass

    def extract_css_links(self, css_content: str, base_url: str) -> List[str]:
        """从 CSS url() 中提取资源链接"""
        pass

    def localize_resources(self, html: str, resource_map: Dict[str, Path]) -> str:
        """改写 HTML 中的资源链接为本地路径"""
        pass

    def localize_css(self, css_content: str, resource_map: Dict[str, Path]) -> str:
        """改写 CSS url() 为本地路径"""
        pass

    def extract_text_units(self, html: str, url: str) -> List[Dict]:
        """
        抽取待翻译文本单元（跳过代码块、函数名等）
        返回: [{"text": "...", "context": "...", "hash": "...", "is_protected": False}, ...]
        """
        pass

    def protect_code_blocks(self, html: str) -> tuple[str, Dict]:
        """保护代码块、签名等不被翻译，返回带占位符的 HTML 和占位符映射"""
        pass

    def restore_code_blocks(self, html: str, placeholder_map: Dict) -> str:
        """恢复代码块占位符"""
        pass

class TaskScheduler:
    """任务调度器（双线程池、优雅暂停、状态持久化）"""
    def __init__(self, download_concurrency: int, localize_concurrency: int):
        self.download_pool = ThreadPoolExecutor(max_workers=download_concurrency)
        self.localize_pool = ThreadPoolExecutor(max_workers=localize_concurrency)
        self.is_paused = False

    def start_download_phase(self, task_queue: List) -> None:
        """启动下载阶段任务"""
        pass

    def start_localize_phase(self, task_queue: List) -> None:
        """启动本地化重建阶段任务"""
        pass

    def pause(self) -> None:
        """优雅暂停：停止领取新任务，等待当前批次完成后落库"""
        pass

    def resume(self) -> None:
        """从暂停状态恢复"""
        pass

    def get_status(self) -> Dict:
        """获取双阶段任务状态（下载/本地化计数、线程池状态）"""
        pass

class MirrorService:
    """站点镜像服务（双阶段：原站下载→本地化重建）"""
    def __init__(self, db: Database, http_client: HTTPClient, parser: HTMLParser, scheduler: TaskScheduler):
        pass

    def mirror_site(self, site: str, mode: str = "full") -> None:
        """
        全量镜像站点
        内部流程：
        1. 阶段1：下载原站资源（sitemap/index.json 优先 → 递归发现链接 → 下载到 raw mirror）
        2. 阶段2：本地化重建（读取 raw mirror → 改写 HTML/CSS → 同步资源到 localized mirror）
        """
        pass

    def update_incremental(self, site: str) -> None:
        """
        增量更新站点
        流程：刷新种子清单 → 比较 lastmod/etag/content hash → 仅抓取变更项 → 仅重建变更项
        """
        pass

    def resume_task(self, site: str) -> None:
        """续跑上次任务（自动判断当前阶段：下载未完成则续下载，否则续本地化）"""
        pass

class LocalizeService:
    """资源本地化服务（HTML/CSS 改写、资源同步）"""
    def __init__(self, db: Database, parser: HTMLParser):
        pass

    def rebuild_page(self, url: str) -> None:
        """重建单个页面的本地化 HTML/CSS"""
        pass

    def rebuild_site(self, site: str) -> None:
        """批量重建站点本地化文件"""
        pass

    def sync_resources_to_localized(self, site: str, use_hardlink: bool = True) -> None:
        """
        同步资源到 localized mirror
        默认使用硬链接（避免重复占用磁盘），不支持硬链接时回退到复制
        """
        pass

class ExtractService:
    """文本抽取服务（去重、词库生成）"""
    def __init__(self, db: Database, parser: HTMLParser):
        pass

    def extract_all(self, site: str) -> None:
        """抽取全站待翻译文本单元，去重后写入 translation_memory"""
        pass

    def generate_vocabulary(self) -> None:
        """基于翻译记忆生成词库（统计词频、术语候选）"""
        pass

class TranslationService:
    """翻译包导入导出服务（校验、回填、翻译记忆复用）"""
    def __init__(self, db: Database):
        pass

    def export_pack(self, output_path: Path, format: str = "jsonl") -> Path:
        """
        导出待翻译包
        格式：JSONL（默认）/ JSON 批次包 / CSV（辅助）
        包含：文本单元、上下文、占位符、术语提示、来源页
        """
        pass

    def import_pack(self, file_path: Path) -> None:
        """
        导入已翻译包并回填
        校验项：hash 匹配、占位符完整、禁止翻译词未被误改
        不合格条目进入 fail_log，拒绝回填
        """
        pass

    def backfill_to_pages(self, site: str) -> None:
        """将翻译回填到本地化 HTML，重建搜索索引"""
        pass

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

    def search(self, query: str, lang: str = "zh") -> List[Dict]:
        """离线搜索（支持英文符号名、中文说明检索）"""
        pass

    def replace_search_page(self, site: str) -> None:
        """替换 docs.flutter.dev/search 和 dart.dev/search 为内置离线搜索页"""
        pass
```

## 六、核心工作流程

### 6.1 双阶段镜像流程（单按钮触发）

**目标**：
   - 先提升下载吞吐，再批量本地化，避免“下载被改写拖死”。
   - 所有数据目录需要在【工程目录】目录下data目录下，详细要求见【配置持久化】中的说明。

#### 阶段 1：原站镜像下载
1. **镜像下载**：
   - 镜像下载的时候，需要根据网站的原始情况严格执行相对路径，不能使用域名，不能使用 IP。
2. **种子 URL 发现**：
   - `docs.flutter.dev` / `dart.dev`：读取 `sitemap.xml`
   - `api.flutter.dev`：读取 `index.html` + `flutter/index.json` + 同域递归
   - `api.dart.dev`：读取 `index.json` + 同域递归
3. **递归发现链接**：
   - 解析 HTML `href`/`src`/`srcset`、CSS `url()`、JS/JSON 中可识别的站内资源
   - 覆盖 manifest/opensearch/xml/json/pdf/font/image/js/css 等所有资源类型
4. **下载执行**：
   - 线程池并发下载（`download_concurrency` 控制）
   - 大文件使用 `.part` 临时文件 + Range 续传
   - 资源写入 `raw mirror` 目录，元数据写入 `resource_cache`
   - URL 状态流转：`queued` → `downloading` → `downloaded`/`failed`
5. **资源下载规则**：
   - 数据库中记录不存在，需要重新发现
   - 数据库中记录存在，实际文件存在，无需重新下载
   - 数据库中记录存在，实际文件不存在，需要重新下载
   - 数据库中记录存在，实际文件存在，无需重新下载
6. **阶段 1 完成判定**：`url_queue` 中所有 URL 状态为 `downloaded` 或 `failed`

#### 阶段 2：本地化重建

1. **路径处理**：本地化重建的时候，需要根据网站的原始情况严格执行相对路径，不能使用域名，不能使用 IP。
2. **本地化队列生成**：将 `url_queue` 中 `downloaded` 的 URL 写入 `localize_queue`，状态 `queued`
3. **HTML/CSS 改写**：
   - 读取 `raw mirror` 中的原始 HTML/CSS
   - 保护代码块/签名（生成占位符）
   - 改写站内链接为本地路径，四站互链本地化，外域导航保留外链
   - 恢复代码块占位符
   - 改写后的文件写入 `localized mirror` 目录
4. **资源同步**：
   - 使用硬链接（优先）或复制将 `raw mirror` 中的二进制资源同步到 `localized mirror`
5. **状态流转**：`localize_queue` 状态：`queued` → `rebuilding` → `completed`/`failed`
6. **阶段 2 完成判定**：`localize_queue` 中所有 URL 状态为 `completed` 或 `failed`

#### 预览适配

- 阶段 2 未完成时，访问 HTML 返回“已下载，等待本地化重建”占位页
- 阶段 2 完成后，预览服务优先读取 `localized mirror`

### 6.2 文本抽取与翻译包流程

**文本抽取**：

- 仅抽取说明性文本：英文注释、文档、导航文案、按钮文案、表头、ARIA/placeholder/title。
- **明确不翻译**：函数名、类名、变量名、参数名、库名、包名、签名、代码块（`pre`/`code`）、URL、文件名、操作符、`translate="no"` 节点。
- API 页面特殊处理：翻译 `.desc.markdown`、摘要说明、章节标题；跳过 `.signature`、符号标识符。
- 以“文本单元 hash”去重，相同英文文本全站仅保留一份主记录，写入 `translation_memory`。

**词库生成**：

- 统计词频、来源页面、术语候选。
- 支持锁定术语、标记禁止翻译词。
- 词库标签页添加分页功能，每页显示 100 条记录。

**翻译包导出**：

- 主格式：JSONL（包含文本单元、上下文、占位符、术语提示、来源页）。
- 辅助格式：CSV（供人工检查）。

**翻译包导入与回填**：

- **校验项**：hash 匹配、占位符完整、禁止翻译词未被误改。
- 不合格条目进入 `fail_log`，支持单独重试。
- 回填翻译到 `localized mirror` HTML，重建 FTS5 搜索索引。

**翻译方式选择与切换**：

- API 翻译：
  - 在用户选择 API 翻译时，程序会自动将翻译请求发送到指定的翻译 API（如 Google Translate API、DeepL API 等）。
  - 翻译包中的每个文本单元将通过 API 发送并返回翻译结果。用户可以选择翻译的语言。
  - 翻译的结果将直接填充到 `translation_memory`，并回填到 `localized mirror` 中的相应 HTML 文件。
  - **翻译进度和日志**：API 翻译会显示实时进度，包括翻译成功、失败的条目，以及 API 的返回信息（如限流、翻译失败等）。
- 人工翻译：
  - 用户可手动翻译翻译包中的文本单元，并在翻译完成后上传已翻译的文件。
  - 程序会验证上传的翻译包（如验证占位符是否完整，禁止翻译的词是否未被修改），然后将翻译内容回填到 `translation_memory` 中。
  - 如果遇到无法回填的条目（如占位符未匹配），它们将被标记为“待人工确认”状态，方便人工修正。
  - **翻译进度和日志**：显示人工翻译的条目及当前进度。用户可以查看每个翻译条目的状态（例如：待翻译、已翻译、回填成功、回填失败）。
- AI 翻译：
  - AI 翻译采用本地 AI 模型（例如 GPT 或其他机器翻译模型）进行翻译。
  - 用户可以选择指定翻译语言以及特定的模型（如翻译领域相关的定制模型）。
  - 翻译包中的每个文本单元将被发送到 AI 模型进行处理，生成翻译结果。
  - 翻译的结果将直接填充到 `translation_memory`，并回填到 `localized mirror` 中的相应 HTML 文件。
  - **翻译进度和日志**：显示 AI 翻译的进度，包括模型调用、成功翻译、翻译失败等日志。

**翻译包回填**：

- **回填校验**：在回填翻译包时，程序会先校验每个翻译条目的 hash 是否匹配，确认占位符是否完整，以及是否有禁止翻译的词被误改。
- 不符合要求的翻译条目会被记录到 `fail_log`，并且不会回填。
- 程序会回填翻译到 `localized mirror` 的 HTML 文件，确保站点的所有文本内容都已被正确翻译并且保持原文的结构。

**翻译包导入后的实时反馈**：

- 当用户导入翻译包并开始回填时，系统将提供一个实时日志，展示当前翻译的进度、遇到的错误、翻译 API 的响应信息等。
- 如果 API 翻译时出现限流，用户会看到翻译暂停的提示，并且可以选择继续翻译或调整翻译策略。
- 人工翻译和 AI 翻译过程中，系统将实时显示翻译状态、可能的错误信息和当前进度，以确保用户能够跟踪和处理任何翻译问题。

**翻译日志管理**：

- 翻译加一个实时日志，如果报错、或者API返回限流什么的都显示，不要写到编辑框，用标签，就显示当前的
- 在翻译过程中，系统将会保存详细的日志记录，帮助用户分析翻译进度、失败任务、翻译 API 错误等信息。
- 用户可以查看翻译日志、导出日志文件或查看翻译历史，方便后期审计与修复。

**翻译任务管理**：

- 在翻译过程中，用户可以暂停、恢复翻译任务，并且可以选择是否跳过某些翻译单元或继续上次未完成的翻译任务。
- 任务状态（例如：待翻译、翻译中、已完成、翻译失败）会被实时更新，帮助用户了解翻译的当前状态。
  - **校验项**：hash 匹配、占位符完整、禁止翻译词未被误改
  - 不合格条目进入 `fail_log`，支持单独重试
  - 回填翻译到 `localized mirror` HTML，重建 FTS5 搜索索引

**翻译开始**：
- 点击开始翻译选中包的时候如果匹配到上次有翻译未完成的任务时弹窗让用户是否继续上次任务还是重新翻译

### 6.3 增量更新流程

1. **刷新种子清单**：重新获取 sitemap/index.json
2. **变更检测**：比较 `lastmod`/`etag`/`content_hash`
3. **增量执行**：
   - 仅抓取新增或变更的页面/资源
   - 仅重抽取变更页面的文本单元
   - 仅导出新增待翻译文本
   - 导入新翻译后仅重建受影响页面与搜索索引
4. **翻译记忆复用**：未变化文本不重新导出、不重新回填

### 6.4 本地预览与离线搜索

1. **内置 HTTP 服务**：
   - 预览服务优先读取 `localized mirror`
   - 支持中英文站点切换预览
2. **离线搜索替换**：
   - `docs.flutter.dev/search` / `dart.dev/search` 替换为内置离线搜索页
   - `api.flutter.dev` / `api.dart.dev` 支持英文符号名和中文说明检索（基于 FTS5）

## 七、GUI 与 CLI 功能

### 7.1 GUI 5 个主标签页

| 标签页       | 核心功能                                                           |
| --------- | -------------------------------------------------------------- |
| **项目设置**  | 配置站点信息、下载并发数、本地化并发数、超时重试、本地预览端口、导出包格式                          |
| **抓取/更新** | 双阶段状态显示（下载队列/已下载/待本地化/本地化中/已本地化）、开始/暂停/继续/一键更新按钮、失败项单独重试、线程池状态 |
| **词库管理**  | 词库浏览、术语锁定/解锁、禁止翻译词标记、词频统计                                      |
| **翻译队列**  | 导出待翻译包、导入已翻译包、回填进度显示、失败日志查看                                    |
| **本地预览**  | 启动/停止预览服务、内置浏览器预览、中英文切换、离线搜索                                   |

### 7.2 CLI 命令列表

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

## 八、并发与任务控制

- **双线程池分离**：
  - 下载线程池：仅负责网络 IO 和原始文件落盘，由 `download_concurrency` 控制
  - 本地化重建线程池：仅负责 HTML/CSS 改写，由 `concurrency` 控制
  - 两池互不干扰，避免下载被改写拖死
- **优雅暂停**：
  - 设置 `is_paused` 标志，停止领取新任务
  - 等待当前下载/改写批次完成后，将状态落库
  - 支持程序重启后从数据库和 `.part` 临时文件恢复
- **状态持久化**：
  - 所有任务状态唯一依赖 SQLite，避免内存状态丢失
  - 支持旧项目数据库迁移（拆分 `url_queue` → `url_queue` + `localize_queue`）

## 九、测试计划

### 9.1 单元测试

- URL 规范化、四站互链改写、外域静态资源镜像规则
- CSS `url()` 改写、文本保护占位符、hash 去重
- 导出包结构、导入包校验、翻译记忆复用

### 9.2 抽取规则测试

- 覆盖 4 类样本页：Flutter 文档页、Dart 文档页、Flutter API 页、Dart API 页
- 断言：说明文本被抽出，signature/pre/code/identifier 不被抽出

### 9.3 集成测试

- 四站互链本地化、外域静态依赖本地可加载、外部导航保留外链
- 双阶段流程完整执行、预览服务正常访问

### 9.4 任务控制测试

- 开始后暂停、暂停后继续、程序退出后恢复
- 失败任务单独重试、多线程下不重复领取同一任务

### 9.5 增量测试

- 修改单页 hash 后执行 update，确认仅重抓该页及其依赖、仅新增导出变更文本、仅重建受影响索引

### 9.6 导出导入测试

- 导出待翻译包 → 人工模拟翻译 → 导入回填 → 页面中文可见且占位符未损坏

### 9.7 打包测试

- 生成 exe 后在 Windows 上双击启动、创建项目、执行镜像、暂停恢复、导出导入、预览本地站点

## 十、交付物

1. **源码**：完整项目代码（含目录结构、核心类、测试用例）
2. **CLI 工具**：可通过命令行执行全流程
3. **GUI 程序**：PySide6 可视化界面，gui中的用户交互界面必须是中文
4. **Windows exe**：PyInstaller 打包的可双击运行程序
5. **文档**：README（使用说明）、本文档（实现细节）


## 十一、编译脚本

- 在项目里创建一个纯英文的编译脚本，用于编译项目
- 脚本运行为避免在 Windows 上双击运行时出错，需要注意编码问题，且脚本中不能包含中文字符
- 脚本目录为【工程目录】，脚本文件名为`build.bat`
- 脚本不能将所有文件全部打包到 exe 中，所有的必须文件需要存放到目录中，还有一个最终生成的 exe 文件为`flutter_doc.exe`
- 添加``if %ERRORLEVEL% neq 0 ()`` 语句,无人值守运行，不能依赖人工操作
- 在错误处添加了``pause`` 让用户能看到错误信息
- 最后添加了``pause >nul`` 让用户能看到完成信息

## 十二、配置持久化
- 项目配置保存到 【工程目录】下的目录`flutter_doc_config.json` 文件，配置文件采用 JSON 格式，包含项目路径，重启后自动加载
- 项目数据持久化,项目数据保存到【工程目录】下的`data` 目录，包含项目的数据库、翻译记忆、任务状态、日志等文件、镜像数据、本地化数据等所有数据文件


