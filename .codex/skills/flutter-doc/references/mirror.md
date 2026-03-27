# 站点镜像

## 总体目标

镜像链路采用“双阶段”：

1. 原站镜像下载
2. 本地化重建

这样做是为了把下载吞吐和改写工作解耦，避免“下载被本地化拖死”。

## 阶段 1：原站镜像下载

### 初始化

- 创建 `raw mirror` 目录
- 检查数据库里是否存在未完成任务
- 初始化下载线程池

### 种子 URL 发现

| 站点 | 种子来源 |
| --- | --- |
| `docs.flutter.dev` | `sitemap.xml` |
| `dart.dev` | `sitemap.xml` |
| `api.flutter.dev` | `index.html` + `flutter/index.json` + 同域递归 |
| `api.dart.dev` | `index.json` + 同域递归 |

### 递归发现

从已下载页面里继续提取站内资源与页面链接，覆盖：

- HTML 中的 `href` / `src` / `srcset`
- CSS 中的 `url()`
- JS / JSON 中可识别的同域 URL

### 下载执行

下载规则：

- 数据库无记录：发现并下载
- 有记录且文件存在：跳过
- 有记录但文件缺失：重新下载

下载特性：

- 使用 `download_concurrency` 控制并发
- 大文件用 `.part` + Range 续传
- 元数据写入 `resource_cache`
- 状态从 `queued -> downloading -> downloaded/failed`

## 阶段 2：本地化重建

- 为待处理页面建立 `localize_queue`
- 改写 HTML/CSS 中的本地路径
- 同步依赖资源到 `localized mirror`
- 更新本地化阶段状态与统计

## 必须覆盖的资源类型

- HTML
- CSS
- JS / MJS
- JSON / Manifest / XML / Opensearch
- 图片：`png` / `jpg` / `svg` / `webp` / `gif`
- 字体：`woff` / `woff2` / `ttf` / `otf` / `eot`
- 其他必要静态资源，如 `pdf`

目标是保证镜像后的视觉与交互尽可能与原站一致。

## 与任务控制的关系

- 支持开始、暂停、恢复、失败单项重试
- 停止时先停领新任务，再等待当前批次安全落库
- 恢复时从数据库状态和 `.part` 文件继续

## 推荐联读

- 四站互链与改写：见 [localize](localize.md)
- 任务控制与界面入口：见 [gui-cli](gui-cli.md)
- 测试场景：见 [testing](testing.md)

