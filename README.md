# 个人技术博客

基于 [Hugo](https://gohugo.io/) 和 [Congo](https://github.com/jpanther/congo) 主题构建的个人技术博客网站。

## 项目简介

这是一个用于分享技术见解、源码阅读心得以及技术深入理解的个人博客平台。内容包括：

- 📝 **技术文章**：Elasticsearch、Git、性能优化、Docker、Spring Boot 等技术主题
- 🚀 **项目展示**：个人项目介绍和展示
- 📚 **学习笔记**：技术学习和实践过程中的心得体会

## 技术栈

- **静态网站生成器**：Hugo (Go 1.23.0)
- **主题**：Congo v2.12.2
- **部署**：GitHub Pages

## 本地开发

### 前置要求

- Go 1.23.0 或更高版本
- Hugo Extended 版本（推荐使用最新版本）

### 安装 Hugo

```bash
# macOS
brew install hugo

# 或使用 Go 安装
go install -tags extended github.com/gohugoio/hugo@latest
```

### 安装依赖

```bash
# 获取 Hugo 模块依赖（主题等）
hugo mod get -u

# 整理和验证模块
hugo mod tidy
hugo mod verify
```

### 运行本地服务器

```bash
# 启动开发服务器
hugo server -D

# 或指定端口和绑定地址
hugo server -D --bind 0.0.0.0 --port 1313
```

访问 http://localhost:1313 查看网站。

**注意**：`-D` 参数表示包含草稿（draft）内容。

### 构建静态文件

```bash
# 构建网站
hugo

# 构建后的文件在 public/ 目录
```

## 项目结构

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
│   └── writings/      # 技术文章
├── static/             # 静态文件（favicon 等）
├── go.mod              # Go 模块定义
└── README.md           # 项目说明
```

## 添加新内容

### 创建新文章

```bash
# 使用 Hugo 命令创建新文章
hugo new writings/my-new-post/index.md

# 或手动创建目录和文件
mkdir -p content/writings/my-new-post
touch content/writings/my-new-post/index.md
```

### 创建新项目

```bash
mkdir -p content/projects/my-project
touch content/projects/my-project/index.md
```

## 部署

网站通过 GitHub Actions 自动部署到 GitHub Pages。推送代码到 `main` 分支后会自动触发构建和部署流程。

## 许可证

MIT License

## 联系方式

如有问题或建议，欢迎通过 GitHub Issues 联系。
