---
name: flutter_doc_translation
description: "Flutter/Dart 文档本地化工具翻译服务。包含文本抽取、词库管理、翻译包导入导出、回填校验。开发翻译相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 翻译服务

## 文本抽取与翻译包流程

### 文本抽取

- 仅抽取说明性文本：英文注释、文档、导航文案、按钮文案、表头、ARIA/placeholder/title。
- **明确不翻译**：函数名、类名、变量名、参数名、库名、包名、签名、代码块（`pre`/`code`）、URL、文件名、操作符、`translate="no"` 节点。
- API 页面特殊处理：翻译 `.desc.markdown`、摘要说明、章节标题；跳过 `.signature`、符号标识符。
- 以"文本单元 hash"去重，相同英文文本全站仅保留一份主记录，写入 `translation_memory`。

### 词库生成

- 统计词频、来源页面、术语候选。
- 支持锁定术语、标记禁止翻译词。
- 词库标签页添加分页功能，每页显示 100 条记录。

### 翻译包导出

- 主格式：JSONL（包含文本单元、上下文、占位符、术语提示、来源页）。
- 辅助格式：CSV（供人工检查）。

### 翻译包导入与回填

- **校验项**：hash 匹配、占位符完整、禁止翻译词未被误改。
- 不合格条目进入 `fail_log`，支持单独重试。
- 回填翻译到 `localized mirror` HTML，重建 FTS5 搜索索引。

### 翻译方式选择与切换

- API 翻译：
  - 在用户选择 API 翻译时，程序会自动将翻译请求发送到指定的翻译 API（如 Google Translate API、DeepL API 等）。
  - 翻译包中的每个文本单元将通过 API 发送并返回翻译结果。用户可以选择翻译的语言。
  - 翻译的结果将直接填充到 `translation_memory`，并回填到 `localized mirror` 中的相应 HTML 文件。
  - **翻译进度和日志**：API 翻译会显示实时进度，包括翻译成功、失败的条目，以及 API 的返回信息（如限流、翻译失败等）。
- 人工翻译：
  - 用户可手动翻译翻译包中的文本单元，并在翻译完成后上传已翻译的文件。
  - 程序会验证上传的翻译包（如验证占位符是否完整，禁止翻译的词是否未被修改），然后将翻译内容回填到 `translation_memory` 中。
  - 如果遇到无法回填的条目（如占位符未匹配），它们将被标记为"待人工确认"状态，方便人工修正。
  - **翻译进度和日志**：显示人工翻译的条目及当前进度。用户可以查看每个翻译条目的状态（例如：待翻译、已翻译、回填成功、回填失败）。
- AI 翻译：
  - AI 翻译采用本地 AI 模型（例如 GPT 或其他机器翻译模型）进行翻译。
  - 用户可以选择指定翻译语言以及特定的模型（如翻译领域相关的定制模型）。
  - 翻译包中的每个文本单元将被发送到 AI 模型进行处理，生成翻译结果。
  - 翻译的结果将直接填充到 `translation_memory`，并回填到 `localized mirror` 中的相应 HTML 文件。
  - **翻译进度和日志**：显示 AI 翻译的进度，包括模型调用、成功翻译、翻译失败等日志。

### 翻译包回填

- **回填校验**：在回填翻译包时，程序会先校验每个翻译条目的 hash 是否匹配，确认占位符是否完整，以及是否有禁止翻译的词被误改。
- 不符合要求的翻译条目会被记录到 `fail_log`，并且不会回填。
- 程序会回填翻译到 `localized mirror` 的 HTML 文件，确保站点的所有文本内容都已被正确翻译并且保持原文的结构。

### 翻译包导入后的实时反馈

- 当用户导入翻译包并开始回填时，系统将提供一个实时日志，展示当前翻译的进度、遇到的错误、翻译 API 的响应信息等。
- 如果 API 翻译时出现限流，用户会看到翻译暂停的提示，并且可以选择继续翻译或调整翻译策略。
- 人工翻译和 AI 翻译过程中，系统将实时显示翻译状态、可能的错误信息和当前进度，以确保用户能够跟踪和处理任何翻译问题。

### 翻译日志管理

- 翻译加一个实时日志，如果报错、或者API返回限流什么的都显示，不要写到编辑框，用标签，就显示当前的
- 在翻译过程中，系统将会保存详细的日志记录，帮助用户分析翻译进度、失败任务、翻译 API 错误等信息。
- 用户可以查看翻译日志、导出日志文件或查看翻译历史，方便后期审计与修复。

### 翻译任务管理

- 在翻译过程中，用户可以暂停、恢复翻译任务，并且可以选择是否跳过某些翻译单元或继续上次未完成的翻译任务。
- 任务状态（例如：待翻译、翻译中、已完成、翻译失败）会被实时更新，帮助用户了解翻译的当前状态。
  - **校验项**：hash 匹配、占位符完整、禁止翻译词未被误改
  - 不合格条目进入 `fail_log`，支持单独重试
  - 回填翻译到 `localized mirror` HTML，重建 FTS5 搜索索引

### 翻译开始

- 点击开始翻译选中包的时候如果匹配到上次有翻译未完成的任务时弹窗让用户是否继续上次任务还是重新翻译

## 核心类接口

```python
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

class VocabularyService:
    """词库管理服务（术语锁定、禁止翻译词）"""
    pass
```

## 相关技能

- [flutter_doc_database](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_database/SKILL.md) - 数据库设计（翻译记忆表）
- [flutter_doc_localize](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_localize/SKILL.md) - 本地化服务
- [flutter_doc_gui](file:///c:/app/flutter%20doc/.trae/skills/flutter_doc_gui/SKILL.md) - GUI 与 CLI 功能
