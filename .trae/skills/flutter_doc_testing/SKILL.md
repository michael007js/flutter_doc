---
name: flutter_doc_testing
description: "Flutter/Dart 文档本地化工具测试计划。包含单元测试、集成测试、任务控制测试、增量测试、打包测试。开发测试相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 测试计划

## 单元测试

- URL 规范化、四站互链改写、外域静态资源镜像规则
- CSS `url()` 改写、文本保护占位符、hash 去重
- 导出包结构、导入包校验、翻译记忆复用

## 抽取规则测试

- 覆盖 4 类样本页：Flutter 文档页、Dart 文档页、Flutter API 页、Dart API 页
- 断言：说明文本被抽出，signature/pre/code/identifier 不被抽出

## 集成测试

- 四站互链本地化、外域静态依赖本地可加载、外部导航保留外链
- 双阶段流程完整执行、预览服务正常访问

## 任务控制测试

- 开始后暂停、暂停后继续、程序退出后恢复
- 失败任务单独重试、多线程下不重复领取同一任务

## 增量测试

- 修改单页 hash 后执行 update，确认仅重抓该页及其依赖、仅新增导出变更文本、仅重建受影响索引

## 导出导入测试

- 导出待翻译包 → 人工模拟翻译 → 导入回填 → 页面中文可见且占位符未损坏

## 打包测试

- 生成 exe 后在 Windows 上双击启动、创建项目、执行镜像、暂停恢复、导出导入、预览本地站点

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_build](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_build/SKILL.md) - 编译与调试
- [flutter_doc_mirror](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_translation](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_translation/SKILL.md) - 翻译服务
- [flutter_doc_localize](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_localize/SKILL.md) - 本地化服务
