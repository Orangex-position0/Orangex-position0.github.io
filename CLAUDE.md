# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 Hugo 静态网站生成器的个人技术博客，使用 Stack 主题，通过 GitHub Actions 自动部署到 GitHub Pages。

- **Hugo 版本要求**: >= 0.146.0
- **主题**: Stack (通过 git submodule 引入)
- **部署目标**: https://Orangex-position0.github.io/
- **语言**: 中文 (zh-cn)

## 常用命令

### 本地开发
```bash
# 启动开发服务器（默认 http://localhost:1313）
hugo server

# 启动开发服务器并构建草稿文章
hugo server -D

# 启动开发服务器并显示未来日期的文章
hugo server -F
```

### 内容创建
```bash
# 创建新文章（使用 archetypes/default.md 模板）
hugo new posts/my-article.md

# 在分类目录下创建文章（推荐）
hugo new posts/category-name/article-name.md
```

### 构建与部署
```bash
# 构建网站到 public/ 目录
hugo

# 构建并优化（清理缓存、压缩）
hugo --gc --minify
```

### 主题管理
```bash
# 更新 Stack 主题（通过 git submodule）
git submodule update --remote --merge
```

## 项目结构

```
.
├── hugo.toml              # 主配置文件（baseURL、主题、菜单、参数等）
├── .gitmodules            # Git submodule 配置（Stack 主题）
├── .github/workflows/
│   └── hugo-deploy.yml    # GitHub Actions 自动部署配置
├── content/               # 内容目录（Markdown 文章）
│   ├── _index.md          # 内容索引页
│   ├── posts/             # 博客文章目录
│   │   ├── _index.md      # 文章列表页
│   │   └── **/           # 按分类组织的文章
│   ├── about.md           # 关于页面
│   └── categories/        # 分类页面（自动生成）
├── layouts/               # 自定义布局模板（可选覆盖主题模板）
├── static/                # 静态资源（直接复制到 public/）
│   └── img/               # 图片资源（如头像 avatar.png）
├── assets/                # 资源文件（通过 Hugo Pipes 处理）
├── archetypes/            # 内容模板
│   └── default.md         # 默认文章 front matter 模板
├── public/                # 生成的静态网站（已 gitignore）
└── themes/Stack/          # Stack 主题（git submodule）
```

## 内容组织

### Front Matter 模板
文章使用 TOML 格式的 front matter：

```toml
+++
title = '文章标题'
date = 2026-03-16T13:00:00+08:00
draft = false              # 是否为草稿
description = '文章描述（可选）'

categories = ['category-name']
tags = ['tag1', 'tag2']
series = ''                # 系列文章（可选）
seriesWeight = 0           # 系列内排序
+++
```

### 分类组织
文章建议按分类目录组织：
```
content/posts/
├── java/                  # Java 相关文章
├── rust/                  # Rust 相关文章
├── distributed-systems/   # 分布式系统
└── computer-fundamentals/ # 计算机基础
```

### 菜单配置
主菜单在 `hugo.toml` 中配置，包含：首页、文章、分类、标签、关于。

## 部署流程

1. 推送代码到 `main` 分支
2. GitHub Actions 自动触发 `.github/workflows/hugo-deploy.yml`
3. 执行 `hugo --gc --minify` 构建
4. 将 `public/` 目录部署到 `gh-pages` 分支
5. GitHub Pages 自动发布

**注意**: 主题需要通过 git submodule 引入，CI 配置中使用了 `submodules: recursive` 确保主题被正确拉取。

## 主题定制

- **Stack 主题位置**: `themes/Stack/`
- **主题配置**: `hugo.toml` 中的 `[params]` 部分
- **自定义布局**: 在项目根目录的 `layouts/` 中创建文件可覆盖主题模板
- **静态资源**: 图片等放在 `static/img/` 目录
- **代码高亮**: 使用 `github-dark` 样式，配置在 `hugo.toml`

## 注意事项

- `public/` 和 `resources/` 目录由 Hugo 生成，已加入 `.gitignore`
- 中文章务必设置 `hasCJKLanguage = true` 以正确统计字数
- Stack 主题需要 extended 版本的 Hugo
- 更新主题后需检查配置兼容性
