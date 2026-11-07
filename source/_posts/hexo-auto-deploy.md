---
title: 用 GitHub Actions 自动构建与部署 Hexo 博客
date: 2026-11-07 21:46:43
tags:
  - Hexo
  - GitHub Actions
  - GitHub Pages
categories:
  - 折腾笔记
---

每次写完文章都要在本地敲一遍 `hexo clean && hexo g -d`，时间久了总会忘。把构建搬到 GitHub Actions 之后，本地只需要写 Markdown 然后 `git push`，剩下的交给流水线。

<!-- more -->

## 先想清楚：源码放在哪

Hexo 仓库常见两种布局：

- **单分支 + Actions（推荐）**：`main` 存放源码（`_config.yml`、`package.json`、`source/`、`themes/`），Actions 生成 `public/` 后直接部署到 Pages，仓库里不保存产物。
- **双分支**：`main` 或 `gh-pages` 保存生成后的静态页面，源码放在另一个分支，本地用 `hexo-deployer-git` 推送。这种方式不需要 Actions，但源码与产物容易不同步。

本文按第一种布局写。要注意：如果仓库里目前只有 `2022/`、`archives/`、`css/`、`index.html` 这类目录，说明它是**产物分支**，需要先补回源码结构，否则 Actions 没有东西可以构建。

## 一个最小可用工作流

在 `.github/workflows/pages.yml` 中：

```yaml
name: Deploy Hexo site to Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive   # 主题是 git submodule 时必须有
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm
      - run: npm ci
      - run: npx hexo clean && npx hexo generate
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: public
      - uses: actions/deploy-pages@v5
```

几个容易漏掉的点：

- `permissions` 必须同时给 `pages: write` 和 `id-token: write`，否则部署接口会返回 403。
- `concurrency` 不要开 `cancel-in-progress: true`，生产部署被中途取消容易留下半截站点。
- `upload-pages-artifact` 的 `path` 指向 Hexo 的输出目录 `public`，而不是仓库根目录。
- 仓库 **Settings → Pages → Build and deployment → Source** 要选 **GitHub Actions**，不是 "Deploy from a branch"。

> 上面用的是官方 starter workflow 里的稳定版本；正式使用前建议到 `actions/` 官方仓库核对各个 action 的最新主版本。

## 时区与永久链接

`front-matter` 里的 `date` 按站点 `_config.yml` 的 `timezone` 解析。中文站点建议显式配置：

```yaml
timezone: Asia/Shanghai
permalink: :year/:month/:day/:title/
```

少了 timezone，Actions 的 runner 默认按 UTC 处理，文章日期可能整体差 8 小时，归档路径也会跟着偏移。

## 常见的坑

- **没有 `package-lock.json`**：`npm ci` 会直接失败，改用 `npm install`，或者把 lock 文件提交进仓库。
- **主题目录是空的**：主题以 submodule 引入时，checkout 要带 `submodules: recursive`；以 npm 包引入时，确认 `package.json` 里有对应依赖。
- **`hexo` 命令找不到**：不要全局装 CLI，用 `npx hexo` 或本地 `devDependency`，构建环境更可复现。
- **push 了却没反应**：先去 Actions 页看有没有触发记录；如果一条都没有，检查 workflow 文件是不是放到了别的分支，或者 `on.push.branches` 写错了分支名。

## 小结

流水线跑通之后，写博客的流程就只剩三步：`hexo new`、写正文、`git push`。源码和产物各归其位，换电脑也不用重新配一遍本地部署环境——这大概是把博客搬上 Actions 最实在的收益。
