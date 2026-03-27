# 翻译服务

## 总体流程

翻译链路由四步构成：

1. 文本抽取
2. 词库生成
3. 翻译包导出 / 导入
4. 回填到 `localized mirror`

核心是“文本单元 + hash 去重 + 翻译记忆复用”。

## 文本抽取规则

### 应抽取

- 说明性文本
- 文档正文
- 导航文案
- 按钮文案
- 表头
- `ARIA` / `placeholder` / `title`
- API 页面中的说明区与章节标题

### 明确不翻译

- 函数名、类名、变量名、参数名
- 包名、库名、签名
- `pre` / `code` 代码块
- URL
- 文件名
- 操作符
- `translate="no"` 节点
- API 符号标识符与 `.signature`

## 翻译记忆与词库

- 以 `text_hash` 作为唯一键去重
- 统计词频、来源页面、术语候选
- 支持术语锁定与禁止翻译词
- 词库管理页按分页展示记录

## 翻译包

### 导出

- 主格式：`JSONL`
- 辅助格式：`CSV`

建议包含：

- 文本单元
- 上下文
- 占位符
- 术语提示
- 来源页面

### 导入与校验

至少校验：

- `hash` 匹配
- 占位符完整
- 禁止翻译词未被改坏

不合格条目进入 `fail_log`，支持重试，不直接回填。

## 回填

- 把翻译写回 `translation_memory`
- 回填到 `localized mirror` HTML
- 重建 FTS5 搜索索引

## 翻译方式

文档允许三类翻译来源：

- 外部 API 翻译
- 人工翻译
- AI 翻译

无论来源是什么，导入与回填阶段都遵守同一套校验与日志规则。

## 日志与任务控制

- 翻译过程需要实时日志
- 错误、限流、失败条目要清晰可见
- 用户可暂停、恢复、继续未完成任务
- 开始翻译时，如果检测到未完成旧任务，应提示继续还是重开

## 核心接口草图

```python
class ExtractService:
    def extract_all(self, site: str) -> None: ...
    def generate_vocabulary(self) -> None: ...

class TranslationService:
    def export_pack(self, output_path: Path, format: str = "jsonl") -> Path: ...
    def import_pack(self, file_path: Path) -> None: ...
    def backfill_to_pages(self, site: str) -> None: ...

class VocabularyService:
    pass
```

## 推荐联读

- 翻译记忆与失败日志表：见 [database](database.md)
- GUI 入口与日志展示：见 [gui-cli](gui-cli.md)
- 搜索索引重建：见 [preview](preview.md)
- 验证回填质量：见 [testing](testing.md)

