---
name: flutter_doc_localize
description: "Flutter/Dart 文档本地化工具本地化服务。包含同步本地化、四站互链、资源同步、外部链接保留。开发本地化相关功能时调用。"
---

# Flutter/Dart 文档本地化工具 - 本地化服务

## 核心目标

- **四站互链本地化**：确保四个站点之间的链接正确指向本地化页面
- **视觉一致性**：确保本地化后的站点与原站在视觉上完全一致
- **外部链接保留**：外域导航保留外链，不做本地化处理

---

## 视觉一致性保证

### 概述

本地化重建必须确保镜像站点与原站在视觉上完全一致，包括：
- 页面布局
- 字体样式
- 颜色方案
- 图标显示
- 图片加载
- 动画效果
- 响应式设计

### 视觉关键资源

| 资源类型 | 视觉影响 | 处理要求 |
|---------|---------|---------|
| **CSS 样式表** | 决定页面布局、颜色、间距 | 必须完整同步，路径正确改写 |
| **Web 字体** | 决定文字显示效果 | 必须下载并同步，CSS 引用正确 |
| **图标字体** | 决定图标显示 | 必须下载并同步 |
| **图片资源** | 决定图片显示 | 必须下载并同步 |
| **JavaScript** | 决定页面交互和动态效果 | 必须完整同步 |

### CSS 处理规则

#### 1. CSS 文件同步

```
┌─────────────────────────────────────────────────────────────┐
│                    CSS 文件处理流程                          │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐ │
│  │ 读取    │ →  │ 解析    │ →  │ 改写    │ →  │ 写入    │ │
│  │ raw CSS │    │ url()   │    │ 路径    │    │ localized│ │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### 2. CSS url() 改写

| 原始引用 | 改写后 | 说明 |
|---------|-------|------|
| `url(/assets/font.woff2)` | `url(./assets/font.woff2)` | 绝对路径改相对路径 |
| `url(https://fonts.gstatic.com/...)` | `url(./fonts/external/...)` | 外域字体本地化 |
| `url(../images/icon.png)` | `url(../images/icon.png)` | 相对路径保持不变 |

#### 3. 字体声明处理

```css
/* 原始 CSS */
@font-face {
  font-family: 'Roboto';
  src: url(https://fonts.gstatic.com/roboto.woff2) format('woff2');
}

/* 本地化后 CSS */
@font-face {
  font-family: 'Roboto';
  src: url(./fonts/external/roboto.woff2) format('woff2');
}
```

### 字体处理规则

#### 1. 字体文件下载

| 字体来源 | 处理方式 |
|---------|---------|
| 站内字体 | 直接下载到本地 |
| Google Fonts | 下载 CSS 和字体文件到本地 |
| 外域 CDN 字体 | 下载到本地，改写 CSS 引用 |

#### 2. 字体路径映射

```
raw mirror/
├── fonts/
│   ├── roboto.woff2
│   └── material-icons.woff2

localized mirror/
├── fonts/
│   ├── roboto.woff2          # 硬链接或复制
│   └── material-icons.woff2  # 硬链接或复制
```

### 图片处理规则

#### 1. 图片引用改写

| HTML 标签 | 属性 | 改写规则 |
|----------|------|---------|
| `<img>` | `src` | 改写为本地路径 |
| `<img>` | `srcset` | 改写所有图片路径 |
| `<picture>` | `srcset` | 改写所有图片路径 |
| `<svg>` | `href` | 改写引用路径 |
| CSS | `background-image` | 改写 url() |
| CSS | `list-style-image` | 改写 url() |

#### 2. 响应式图片处理

```html
<!-- 原始 HTML -->
<img srcset="img-320.jpg 320w, img-640.jpg 640w, img-1280.jpg 1280w"
     sizes="(max-width: 640px) 100vw, 640px"
     src="img-640.jpg">

<!-- 本地化后 HTML -->
<img srcset="./images/img-320.jpg 320w, ./images/img-640.jpg 640w, ./images/img-1280.jpg 1280w"
     sizes="(max-width: 640px) 100vw, 640px"
     src="./images/img-640.jpg">
```

---

## 同步本地化处理

### 1. 提取站点内链接

提取 HTML 页面中所有的站内链接：

| 标签 | 属性 | 说明 |
|------|------|------|
| `<a>` | `href` | 页面链接 |
| `<img>` | `src`, `srcset` | 图片资源 |
| `<script>` | `src` | JavaScript 文件 |
| `<link>` | `href` | CSS、图标等 |
| `<source>` | `src`, `srcset` | 媒体资源 |
| `<video>` | `src`, `poster` | 视频资源 |
| `<audio>` | `src` | 音频资源 |
| `<iframe>` | `src` | 内嵌页面 |

### 2. 四站互链处理

```
┌─────────────────────────────────────────────────────────────┐
│                    四站互链本地化                            │
│                                                             │
│  ┌─────────────────┐         ┌─────────────────┐          │
│  │ docs.flutter.dev│ ←─────→ │ api.flutter.dev │          │
│  └─────────────────┘         └─────────────────┘          │
│          ↑                           ↑                      │
│          │                           │                      │
│          ↓                           ↓                      │
│  ┌─────────────────┐         ┌─────────────────┐          │
│  │    dart.dev     │ ←─────→ │  api.dart.dev   │          │
│  └─────────────────┘         └─────────────────┘          │
│                                                             │
│  所有互链改写为本地路径，确保离线可用                        │
└─────────────────────────────────────────────────────────────┘
```

### 3. 外部链接保留

| 链接类型 | 处理方式 | 示例 |
|---------|---------|------|
| 外部网站 | 保留原链接 | `https://github.com/...` |
| 外部 CDN | 下载并本地化 | `https://cdn.jsdelivr.net/...` |
| 社交媒体 | 保留原链接 | `https://twitter.com/...` |
| 第三方服务 | 保留原链接 | `https://www.google.com/...` |

---

## 资源同步

### 同步方式

| 方式 | 优先级 | 说明 |
|------|-------|------|
| 硬链接 | 优先 | 避免重复占用磁盘空间 |
| 复制 | 回退 | 不支持硬链接时使用 |

### 同步清单

```
┌─────────────────────────────────────────────────────────────┐
│                    资源同步清单                              │
├─────────────────────────────────────────────────────────────┤
│  □ CSS 样式文件 (.css)                                      │
│  □ Web 字体文件 (.woff, .woff2, .ttf, .otf)                 │
│  □ 图标字体文件                                             │
│  □ 图片资源 (.png, .jpg, .svg, .webp, .gif)                │
│  □ JavaScript 文件 (.js, .mjs)                              │
│  □ 配置文件 (.json, .webmanifest)                           │
│  □ 其他静态资源                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 视觉验证

### 验证清单

本地化完成后，需验证以下视觉要素：

| 验证项 | 检查内容 |
|-------|---------|
| **布局** | 页面布局与原站一致 |
| **字体** | 自定义字体正确加载 |
| **图标** | 图标字体和 SVG 图标正确显示 |
| **图片** | 所有图片正确加载 |
| **颜色** | 颜色方案与原站一致 |
| **动画** | CSS 动画和过渡效果正常 |
| **响应式** | 响应式布局正常工作 |
| **交互** | JavaScript 交互功能正常 |

### 常见视觉问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 字体显示异常 | 字体文件未下载或路径错误 | 检查字体文件和 CSS 引用 |
| 图标不显示 | 图标字体未同步 | 确保图标字体文件已同步 |
| 样式丢失 | CSS 文件未同步或路径错误 | 检查 CSS 文件和引用路径 |
| 图片加载失败 | 图片未下载或路径错误 | 检查图片文件和引用路径 |
| 布局错乱 | CSS 未正确加载 | 检查 CSS 文件完整性 |

---

## 核心类接口

```python
class LocalizeService:
    """资源本地化服务（HTML/CSS 改写、资源同步）"""
    
    def __init__(self, db: Database, parser: HTMLParser):
        pass

    def rebuild_page(self, url: str) -> None:
        """
        重建单个页面的本地化 HTML/CSS
        包括：
        - HTML 链接改写
        - CSS url() 改写
        - 资源路径修正
        """
        pass

    def rebuild_site(self, site: str) -> None:
        """
        批量重建站点本地化文件
        确保视觉与原站一致
        """
        pass

    def sync_resources_to_localized(self, site: str, use_hardlink: bool = True) -> None:
        """
        同步资源到 localized mirror
        包括：
        - CSS 文件
        - 字体文件
        - 图片资源
        - JavaScript 文件
        默认使用硬链接（避免重复占用磁盘），不支持硬链接时回退到复制
        """
        pass

    def rewrite_css_urls(self, css_content: str, base_path: str) -> str:
        """
        改写 CSS 中的 url() 引用
        确保字体、图片等资源路径正确
        """
        pass

    def validate_visual_consistency(self, url: str) -> Dict[str, bool]:
        """
        验证页面视觉一致性
        返回各项检查结果
        """
        pass
```

---

## 相关技能

- [flutter_doc_constraints](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_constraints/SKILL.md) - 开发约束
- [flutter_doc_mirror](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_mirror/SKILL.md) - 站点镜像服务
- [flutter_doc_translation](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_translation/SKILL.md) - 翻译服务
- [flutter_doc_preview](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_preview/SKILL.md) - 预览服务
- [flutter_doc_errors](file:///c:/app/flutter_doc/.trae/skills/flutter_doc_errors/SKILL.md) - 错误经验
