# 预览服务

## 目标

提供内置 HTTP 服务，用于离线预览镜像站点，并支持离线搜索与中英文切换。

## 服务行为

- 优先读取 `localized mirror`
- 本地化结果缺失时可回退到 `raw mirror`
- 默认端口是 `8080`，但应可配置
- 运行在线程中，不能阻塞主线程

## 文件服务规则

1. 优先从 `localized mirror` 读取
2. 若本地化文件不存在，则从 `raw mirror` 回退
3. 若文件仍在本地化中，返回占位页
4. 对大文件支持 Range Request

## 离线搜索

需要覆盖两类搜索：

- 英文符号名搜索
- 中文说明内容搜索

并支持：

- 混合搜索
- 相关度排序
- 分页

`docs.flutter.dev/search` 和 `dart.dev/search` 可以替换为内置离线搜索页。

## 搜索索引

可以在 FTS5 索引中扩展这些字段：

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS search_index USING fts5(
    url,
    title,
    content_en,
    content_zh,
    site,
    type,
    tokenize = 'porter unicode61'
);
```

## 预览页能力

### 搜索页

- 输入查询
- 展示结果列表
- 点击跳转对应页面

### 占位页

- 显示“正在本地化，请稍后”
- 展示当前进度
- 提供刷新入口

## 核心接口草图

```python
class PreviewService:
    def start_server(self, port: int = 8080) -> None: ...
    def stop_server(self) -> None: ...
    def is_running(self) -> bool: ...
    def get_url(self, site: str, path: str = "") -> str: ...
    def search(self, query: str, lang: str = "zh") -> list[dict]: ...
    def replace_search_page(self, site: str) -> None: ...
    def rebuild_search_index(self, site: str | None = None) -> None: ...
```

## 推荐联读

- 搜索索引结构：见 [database](database.md)
- 页面改写与资源准备：见 [localize](localize.md)
- GUI 启停与线程约束：见 [gui-cli](gui-cli.md)

