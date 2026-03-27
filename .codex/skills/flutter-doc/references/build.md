# 编译与调试

## `build.bat` 约束

工程目录下需要有一个纯英文的 `build.bat`：

- 文件中不包含中文字符
- 避免 Windows 双击运行时的编码问题
- 生成的最终可执行文件名为 `flutter_doc.exe`
- 不能把所有文件都封进单一 `exe`
- 必要资源要随目录一起分发

脚本还应体现这些行为：

- 使用 `if %ERRORLEVEL% neq 0 (...)` 处理失败分支
- 在出错处 `pause`，便于看到错误信息
- 结束前 `pause >nul`，便于看到完成信息

## VS Code 调试配置

`.vscode/launch.json` 需要至少覆盖：

1. 启动 GUI
2. 初始化项目
3. 启动预览服务
4. 自定义命令调试
5. 作为模块运行 `src.main`

优先使用项目根目录下的 `.venv`；如果仓库未提供 `.venv`，先以工作区配置的 Python 解释器为准，并确认依赖已安装。

在直接使用 `src.main`、`build.bat` 或调试配置前，先确认这些入口文件在当前工作区实际存在。

## 交付物

- 完整源码
- 可执行 CLI
- 中文 GUI
- Windows `exe`
- README 和实现说明

## 推荐联读

- 项目边界：见 [constraints](constraints.md)
- 测试验收：见 [testing](testing.md)
- GUI / CLI 能力：见 [gui-cli](gui-cli.md)
