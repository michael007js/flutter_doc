---
name: flutter_doc_build
description: "Flutter/Dart 文档本地化工具编译与调试。包含编译脚本、DEBUG脚本、VS Code调试配置。开发编译调试相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 编译与调试

## 编译脚本

- 在项目里创建一个纯英文的编译脚本，用于编译项目
- 脚本运行为避免在 Windows 上双击运行时出错，需要注意编码问题，且脚本中不能包含中文字符
- 脚本目录为【工程目录】（定义见 [flutter_doc_constraints 名词约束](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_constraints/SKILL.md#名词约束)），脚本文件名为`build.bat`
- 脚本不能将所有文件全部打包到 exe 中，所有的必须文件需要存放到目录中，还有一个最终生成的 exe 文件为`flutter_doc.exe`
- 添加``if %ERRORLEVEL% neq 0 ()`` 语句,无人值守运行，不能依赖人工操作
- 在错误处添加了``pause`` 让用户能看到错误信息
- 最后添加了``pause >nul`` 让用户能看到完成信息

## DEBUG脚本

创建 .vscode/launch.json 文件，包含以下几种调试配置：

1. 启动 GUI - 启动图形用户界面（默认方式）
2. 初始化项目 - 运行 init 命令初始化项目
3. 启动预览服务 - 启动本地预览服务（端口 8080）
4. 自定义命令 - 可修改 args 参数添加自定义命令
5. 模块调试 - 作为 Python 模块运行 src.main

可以在 VS Code 的调试面板中选择这些配置来运行和调试项目。优先使用项目根目录的 `.venv` 虚拟环境；如果仓库未提供 `.venv`，先以工作区配置的 Python 解释器为准，并确认依赖已安装。

在直接使用 `src.main`、`build.bat` 或调试配置前，先确认这些入口文件在当前工作区实际存在。

## 交付物

1. **源码**：完整项目代码（含目录结构、核心类、测试用例）
2. **CLI 工具**：可通过命令行执行全流程
3. **GUI 程序**：PySide6 可视化界面，gui中的用户交互界面必须是中文
4. **Windows exe**：PyInstaller 打包的可双击运行程序
5. **文档**：README（使用说明）、本文档（实现细节）

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_testing](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_testing/SKILL.md) - 测试计划
- [flutter_doc_gui](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_gui/SKILL.md) - GUI 与 CLI 功能
- [flutter_doc_architecture](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_architecture/SKILL.md) - 系统架构设计
