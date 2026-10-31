# sakuratetsu1004.github.io

GitHub Pages で動く作業メモブログ。テーマは [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)(スターター: [chirpy-starter](https://github.com/cotes2020/chirpy-starter))。
ビルドは GitHub Actions (`.github/workflows/pages-deploy.yml`) が行うのでローカルにツールは不要。

## 記事の追加

`_posts/YYYY-MM-DD-slug.md` を作って `main` に push するだけ。先頭に front matter を付ける:

```markdown
---
title: "タイトル"
date: 2026-11-01 12:00:00 +0900
categories: [Linux, GNOME]   # 大分類, 小分類 (最大2階層)
tags: [ubuntu, gnome]        # 小文字
description: 一覧・SEO用の短い説明 (任意)
---
```

- 目次 (TOC) は H2/H3 から自動生成
- 画像は `assets/img/posts/` に置いて `![alt](/assets/img/posts/xxx.png)`
- 固定ページ (About 等) は `_tabs/`
