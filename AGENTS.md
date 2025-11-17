# AGENTS.md

本文档为 AI 编码代理提供项目构建、测试和代码风格等详细信息。

## 项目概述

这是一个基于 [Hugo](https://gohugo.io/) 静态网站生成器和 [Congo](https://github.com/jpanther/congo) 主题构建的个人技术博客网站。

**技术栈：**
- 静态网站生成器：Hugo (Go 1.23.0)
- 主题：Congo v2.12.2
- 部署：GitHub Pages
- 模块管理：Go Modules

**项目结构：**
```
.
├── archetypes/          # Hugo 内容模板
├── assets/              # 资源文件（图片、CSS、JS 等）
│   └── img/            # 图片资源
├── config/             # Hugo 配置文件
│   └── _default/      # 默认配置
├── content/            # 网站内容
│   ├── _index.md      # 首页
│   ├── about.md       # 关于页面
│   ├── projects/      # 项目展示
│   ├── writings/      # 技术文章
│   └── tags/          # 标签页面
├── static/             # 静态文件（favicon 等）
├── .github/workflows/  # GitHub Actions 工作流
├── go.mod              # Go 模块定义
└── go.sum              # Go 模块校验和
```

## 构建和测试命令

### 安装依赖

```bash
# 获取 Hugo 模块依赖（主题等）
hugo mod get -u

# 整理和验证模块
hugo mod tidy
hugo mod verify
```

### 本地开发

```bash
# 启动开发服务器（包含草稿）
hugo server -D

# 指定端口和绑定地址
hugo server -D --bind 0.0.0.0 --port 1313
```

访问 http://localhost:1313 查看网站。

### 构建静态文件

```bash
# 构建网站
hugo

# 构建后的文件在 public/ 目录
```

### 验证构建

```bash
# 构建并检查错误
hugo --quiet --minify

# 验证配置文件
hugo config
```

## 代码风格

### Markdown 文件

- 使用 2 个空格缩进
- 文件末尾保留空行
- Front Matter 使用 YAML 格式（`---` 分隔符）
- 标题使用 `#` 标记，层级清晰
- 代码块指定语言类型

### Front Matter 格式

```yaml
---
title: "文章标题"
date: 2024-12-XXTXX:XX:XX+08:00
draft: false
tags:
- 标签1
- 标签2
---
```

### 文件命名规范

- 文章目录：使用小写字母和连字符，如 `my-article/`
- 文章文件：目录内使用 `index.md`
- 图片文件：使用小写字母和连字符，如 `feature-image.png`

### Go 文件

- 使用 Go 1.23.0 或更高版本
- 遵循 Go 官方代码风格指南
- 使用 `gofmt` 格式化代码

## 内容管理

### 创建新文章

```bash
# 使用 Hugo 命令创建
hugo new writings/my-new-post/index.md

# 或手动创建目录和文件
mkdir -p content/writings/my-new-post
touch content/writings/my-new-post/index.md
```

### 文章结构

每篇文章应包含：
- 清晰的 Front Matter（标题、日期、标签等）
- 引言段落
- 分章节的内容
- 代码示例（如适用）
- 总结部分

### 标签使用规范

- **技术栈标签**：使用小写英文，如 `docker`, `spring-boot`
- **主题标签**：可以使用中文，如 `性能优化`, `项目实践`
- 每篇文章建议 3-5 个标签
- 优先使用已有标签，保持一致性

## 测试说明

### 本地测试

1. **启动开发服务器**：`hugo server -D`
2. **检查页面渲染**：访问各个页面确保正常显示
3. **验证链接**：检查内部链接和外部链接
4. **检查图片**：确保所有图片路径正确

### 构建测试

```bash
# 构建并检查错误
hugo --quiet --minify

# 检查构建输出
ls -la public/
```

### 内容检查清单

在提交前检查：
- [ ] 无错别字和术语错误
- [ ] 文章结构完整（引言、正文、总结）
- [ ] 代码示例格式正确
- [ ] 图片路径正确且存在
- [ ] Front Matter 格式正确
- [ ] 标签使用规范

## 配置文件说明

### 主要配置文件

- `config/_default/config.toml` - 主配置文件
- `config/_default/params.toml` - 主题参数配置
- `config/_default/menus.en.toml` - 菜单配置
- `config/_default/languages.en.toml` - 语言配置

### 重要配置项

- `baseURL` - 网站基础 URL
- `taxonomies` - 标签和分类配置
- `theme` - 使用的主题
- `showTaxonomies` - 是否显示标签

## 部署流程

### GitHub Pages 自动部署

项目使用 GitHub Actions 自动部署：

1. 推送代码到 `main` 分支
2. GitHub Actions 自动触发构建
3. 构建完成后自动部署到 GitHub Pages

工作流文件：`.github/workflows/hugo.yml`

### 手动部署

```bash
# 构建网站
hugo

# 推送到 GitHub Pages 分支（如果需要）
cd public
git add .
git commit -m "Update site"
git push
```

## 常见任务

### 添加新文章

1. 创建文章目录：`mkdir -p content/writings/article-name`
2. 创建文章文件：`touch content/writings/article-name/index.md`
3. 添加 Front Matter 和内容
4. 添加合适的标签
5. 本地测试：`hugo server -D`

### 修改主题配置

编辑 `config/_default/params.toml` 文件，参考 [Congo 主题文档](https://jpanther.github.io/congo/docs/configuration/)。

### 添加新标签

直接在文章的 Front Matter 中添加 `tags` 字段，Hugo 会自动生成标签页面。

## 注意事项

1. **草稿文章**：`draft: true` 的文章在开发服务器中需要 `-D` 参数才能显示
2. **模块更新**：主题更新后需要运行 `hugo mod get -u`
3. **图片路径**：文章中的图片使用相对路径，相对于文章目录
4. **中文支持**：项目已启用 CJK 语言支持

## 参考文档

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [Congo 主题文档](https://jpanther.github.io/congo/docs/)
- [项目 README](./README.md)
- [写作风格指南](./WRITING_STYLE_GUIDE.md)
- [标签实施指南](./TAG_IMPLEMENTATION_SUMMARY.md)

