---
name: flutter_doc_errors
description: "Flutter/Dart 文档本地化工具错误经验。包含SQLite事务、跨线程信号等常见错误及解决方案。开发调试排错时调用。"
---

# Flutter/Dart 文档本地化工具 - 错误经验

## 错误1：SQLite 事务提交问题

当 INSERT OR IGNORE 没有实际插入数据时（例如 URL 已存在），SQLite 没有活动事务，但代码仍然尝试 commit，需要避免这种情况：

```python
# 错误
try:
    yield cursor
    conn.commit()
except Exception as e:
    conn.rollback()
    raise e

# 正确
try:
    yield cursor
    if conn.in_transaction:
        conn.commit()
except Exception as e:
    if conn.in_transaction:
        conn.rollback()
    raise e
```

只有在有活动事务时才会尝试 commit 或 rollback，避免 "cannot commit - no transaction is active" 错误。

## 错误2：无意义的事务提交

只有在实际插入数据时才 commit，避免无意义的事务。

## 错误3：跨线程信号问题

跨线程信号需要使用 Qt.QueuedConnection 来确保正确传递到主线程。

```python
# 错误
self.worker.progress.connect(self._on_progress)
self.worker.finished.connect(self._on_finished)
self.worker.log.connect(self._log)

# 正确
self.worker.progress.connect(self._on_progress, Qt.QueuedConnection)
self.worker.finished.connect(self._on_finished, Qt.QueuedConnection)
self.worker.log.connect(self._log, Qt.QueuedConnection)
```

Qt.QueuedConnection 确保信号在接收者所在的线程（主线程）中处理，而不是在发送者所在的线程（工作线程）中直接调用。这样可以避免跨线程 UI 更新问题。

## 错误4：数据库锁定问题

多线程同时访问 SQLite 数据库时，可能出现 "database is locked" 错误。

**解决方案**：
- 使用线程锁（threading.Lock）保护数据库操作
- 设置合适的超时时间：`sqlite3.connect(db_path, timeout=30)`
- 使用 WAL 模式提高并发性能：`PRAGMA journal_mode=WAL`

```python
import threading

class Database:
    def __init__(self, db_path: str):
        self._lock = threading.Lock()
        self._conn = sqlite3.connect(db_path, timeout=30)
        self._conn.execute("PRAGMA journal_mode=WAL")
    
    def execute(self, sql: str, params: tuple = ()):
        with self._lock:
            cursor = self._conn.cursor()
            cursor.execute(sql, params)
            return cursor
```

## 错误5：HTTP 请求超时未处理

网络请求可能因超时导致程序卡死或崩溃。

**解决方案**：
- 设置合理的超时时间
- 使用重试机制
- 捕获并处理超时异常

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def create_http_client():
    session = requests.Session()
    retry = Retry(
        total=3,
        backoff_factor=0.5,
        status_forcelist=[500, 502, 503, 504]
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    return session

# 使用时设置超时
response = session.get(url, timeout=30)
```

## 错误6：文件路径编码问题

Windows 上文件路径包含中文或空格时，可能导致编码错误。

**解决方案**：
- 使用 Path 对象处理路径
- 确保文件编码为 UTF-8
- 使用原始字符串处理 Windows 路径

```python
from pathlib import Path

# 正确：使用 Path 对象
project_path = Path(r"C:\app\flutter_doc")
data_path = project_path / "data"

# 错误：直接拼接字符串
# data_path = "C:\app\flutter_doc\data"  # \f 会被转义
```

## 错误7：内存泄漏 - 信号未断开

Worker 线程完成后，如果信号未断开连接，可能导致对象无法被垃圾回收。

**解决方案**：
- 在线程完成后断开信号连接
- 使用弱引用或 lambda 避免强引用

```python
# 错误：信号连接未断开
self.worker = MirrorWorker()
self.worker.progress.connect(self._on_progress)
self.worker.start()

# 正确：完成后断开连接
self.worker = MirrorWorker()
self.worker.progress.connect(self._on_progress, Qt.QueuedConnection)
self.worker.finished.connect(self._on_worker_finished)

def _on_worker_finished(self):
    self.worker.progress.disconnect(self._on_progress)
    self.worker.finished.disconnect(self._on_worker_finished)
    self.worker.deleteLater()
```

## 错误8：大文件下载内存溢出

下载大文件时一次性加载到内存，可能导致内存溢出。

**解决方案**：
- 使用流式下载
- 分块写入文件

```python
# 错误：一次性加载到内存
response = requests.get(url)
with open(path, 'wb') as f:
    f.write(response.content)

# 正确：流式下载
response = requests.get(url, stream=True)
with open(path, 'wb') as f:
    for chunk in response.iter_content(chunk_size=8192):
        f.write(chunk)
```

## 错误9：HTML 解析编码问题

HTML 页面编码不一致，可能导致解析错误或乱码。

**解决方案**：
- 从 HTTP 响应头或 meta 标签获取编码
- 使用 BeautifulSoup 自动检测编码

```python
from bs4 import BeautifulSoup

# 正确：让 BeautifulSoup 自动检测编码
response = requests.get(url)
soup = BeautifulSoup(response.content, 'html.parser', from_encoding=response.encoding)

# 或明确指定编码
response.encoding = response.apparent_encoding
html = response.text
```

## 错误10：线程池未正确关闭

程序退出时线程池未关闭，导致程序无法正常退出。

**解决方案**：
- 使用上下文管理器或显式关闭
- 设置合理的等待时间

```python
from concurrent.futures import ThreadPoolExecutor

# 正确：使用上下文管理器
with ThreadPoolExecutor(max_workers=5) as executor:
    futures = [executor.submit(task, url) for url in urls]
    for future in futures:
        result = future.result()

# 或显式关闭
executor = ThreadPoolExecutor(max_workers=5)
try:
    executor.submit(task)
finally:
    executor.shutdown(wait=True)
```

## 错误11：项目配置保存后看似成功，但重新打开仍然失效

典型表现是 GUI 提示“配置已保存”，但下次启动又回到了旧值。常见根因是默认配置路径绑定到了源码目录，打包后实际写到了 `_internal` 或临时目录。

**解决方案**：
- 将应用根目录解析收敛到统一函数，例如 `resolve_app_root()`
- 源码运行时使用工程根目录
- 打包运行时使用 `Path(sys.executable).parent`
- `load()` 与 `save()` 必须共用同一套默认路径解析逻辑
- 保存后返回实际落盘路径，并在界面中回显

```python
def resolve_app_root() -> Path:
    if getattr(sys, "frozen", False):
        return Path(sys.executable).resolve().parent
    return Path(__file__).resolve().parents[1]
```

## 错误12：点击“开始镜像”无响应，只输出占位日志

典型表现是日志里只有“开始镜像功能已准备好。”，但没有任何下载、本地化或状态变化。根因通常是按钮只连接了日志函数，没有接到真实服务。

**解决方案**：
- 不要把核心按钮只绑定到 `append_log()`
- 通过 worker + `QThread` 在后台执行镜像任务
- GUI 只负责信号槽更新日志和状态
- “暂停/继续”必须作用于同一 `TaskScheduler`
- 双阶段状态从 SQLite `task_status` 刷新

```python
self.start_button.clicked.connect(self.start_mirror)

def start_mirror(self):
    self.worker = MirrorWorker(config, site)
    self.thread = QThread(self)
    self.worker.moveToThread(self.thread)
    self.thread.started.connect(self.worker.run)
    self.worker.log_message.connect(self.append_log)
    self.thread.start()
```

## 错误13：镜像进度显示为 `1/1 -> 2/2 -> 5/5`，总数和完成数一起增长

典型表现是下载阶段看起来一直是“满进度推进”，例如先显示 `1/1`，接着 `2/2`、`5/5`。根因通常是把 `task_status.total_count` 写成了 `completed + failed`，而不是队列真实总量。

**解决方案**：
- 从 `url_queue` 聚合下载阶段总数、完成数、失败数、进行中数
- 从 `localize_queue` 聚合本地化阶段总数、完成数、失败数、进行中数
- 每次队列状态变化后刷新对应阶段的 `task_status`
- 阶段结束后根据是否还有剩余任务写入 `completed` / `paused` / `failed`

```python
summary = db.summarize_url_queue(site)
db.upsert_task_status(
    "mirror",
    site,
    "download",
    summary["total_count"],
    summary["completed_count"],
    summary["failed_count"],
    status,
)
```

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_database](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_database/SKILL.md) - 数据库设计
- [flutter_doc_mirror](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_gui](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_gui/SKILL.md) - GUI 与 CLI 功能
- [flutter_doc_build](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_build/SKILL.md) - 编译与调试
