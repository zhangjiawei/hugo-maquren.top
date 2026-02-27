---
title: "收录与监控"
description: "maquren.top 的 SEO 数据闭环清单：验证站点、提交 sitemap、跟踪收录与查询词、按周期更新。"
---

## 推荐先读

- [站长平台提交 SOP：Google / Bing / 百度 一次性配置与周月维护](/posts/webmaster-platform-sop/)

## 一次性配置

- Google Search Console：添加 `https://maquren.top`，提交 `/sitemap.xml`
- Bing Webmaster Tools：添加站点并提交 `/sitemap.xml`
- 百度搜索资源平台：添加站点并提交 `/sitemap.xml`
- 在 `hugo.toml` 填写站长验证码：
  - `params.webmaster.google`
  - `params.webmaster.bing`
  - `params.webmaster.baidu`

## 每周动作

- 检查抓取与索引覆盖（是否有新增未收录 URL）
- 检查搜索查询词，调整文章标题和描述
- 更新 1 篇旧文的 `lastmod` 与 FAQ（保持内容新鲜度）

## 每月动作

- 统计 Top 10 自然流量文章，补站内内链
- 为每篇核心文章新增 1-2 个长尾关键词段落
- 清理死链与 404，确认 `robots.txt` 与 `sitemap.xml` 正常

## 快速入口

- [站点地图 sitemap.xml](/sitemap.xml)
- [robots.txt](/robots.txt)
- [专题导航](/topics/)
