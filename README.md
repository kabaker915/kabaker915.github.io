# kabaker 的安全笔记

个人学习博客。基于 Jekyll + GitHub Pages，站点地址：<https://kabaker915.github.io>

## 写一篇新文章

1. 在 `_posts/` 下新建文件，**文件名必须是 `年-月-日-英文短标题.md`**
   例如：`2026-11-02-how-waf-detects-sqli.md`
2. 文件开头必须有这一段（front matter），注意三横线不能少：

```yaml
---
layout: post
title: "文章标题"
date: 2026-11-02 20:00:00 +0800
tags: [蓝队, 检测]
description: "一句话摘要，会显示在首页列表里。"
---
```

3. 下面直接用 Markdown 写正文。
4. 保存 → 提交 → 推送，GitHub 会自动构建，约 30 秒后线上生效。

> 文件名里的日期**不能晚于今天**，否则 Jekyll 默认不发布它（这叫"未来文章"）。
> 想要定时发布，在 `_config.yml` 里加 `future: true`。

## 本地预览（可选）

本地没装 Ruby，所以现在只能推上去看效果。想本地预览的话装一次 Ruby 即可：

```bash
gem install bundler jekyll
bundle exec jekyll serve
```

然后浏览器打开 <http://127.0.0.1:4000>，改文件会自动刷新。

## 发布流程

```bash
git add .
git commit -m "新增文章：xxx"
git push
```

## 站点设置（只需在网页上点一次）

仓库 Settings → Pages → Source 选 **Deploy from a branch**，
Branch 选 `main`，目录选 `/ (root)`，保存。

## 目录结构

```
_config.yml          站点配置（标题、描述、地址）
_layouts/            页面模板
_includes/head.html  <head> 里的内容
_posts/              文章，一篇一个文件
assets/css/style.css 全站样式
index.md             首页
about.md             关于页
```

## 改站点标题 / 简介

编辑 `_config.yml` 里的 `title` 和 `description`，全站会同步更新。

## 注意

- 不要往这个仓库放任何密钥、Token、密码。仓库是公开的。
- 不要放真实姓名 + 学号的组合。
