# 引継ぎ資料 — sakuratetsu1004.github.io

作成日: 2026-11-01
対象: このブログを別セッション/別マシン/別の AI から保守する人

> AI (Claude Code) 向けの要約版はリポジトリ直下の `CLAUDE.md`。本書はその背景説明と判断理由を含む完全版。

---

## 1. 概要

| 項目 | 内容 |
|---|---|
| 公開 URL | https://sakuratetsu1004.github.io/ |
| リポジトリ | https://github.com/sakuratetsu1004/sakuratetsu1004.github.io (**public**) |
| 用途 | ネットワーク/インフラの作業メモ。手順より「なぜ」「何が駄目だったか」を残す |
| 静的サイト生成 | Jekyll + テーマ [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) v7.6 系(`Gemfile`) |
| ビルド/デプロイ | GitHub Actions (`.github/workflows/pages-deploy.yml`)。Pages の build_type は `workflow` |
| ローカル環境 | 不要。Ruby/Jekyll をローカルに入れずに運用する前提 |
| ローカルクローン | `/home/hqnet/work/sakuratetsu1004.github.io`(remote は SSH) |

### 経緯(なぜこの構成か)

1. 「GitHub に気楽に公開できるブログ」が要件。候補は Zenn(GitHub 連携)/ GitHub Pages + Jekyll / Hugo
2. Zenn は zenn.dev 側でのアカウント連携操作が必要、Hugo は Actions の自作が必要。**Pages + Jekyll** は `gh` だけで完結するので採用
3. 最初は minima(GitHub 標準ビルド)で公開 → 見た目の要望で **Chirpy** に移行。Chirpy は Actions ビルドが前提なので Pages を workflow 方式に切替
4. 配色・フォント・レイアウトはユーザーの好みで大きく上書きしている(§4)

---

## 2. リポジトリ構成

```
.
├── CLAUDE.md                      # AI 向け運用メモ(公開物からは exclude)
├── docs/HANDOVER.md               # 本書(exclude)
├── README.md                      # 記事追加の最小手順(exclude)
├── _config.yml                    # サイト設定(§3)
├── _posts/                        # 記事。YYYY-MM-DD-slug.md
├── _tabs/                         # サイドバーの固定ページ(about/archives/categories/tags)
├── _data/
│   ├── origin/cors.yml            # CDN/Web フォント URL。テーマ同名ファイルの丸ごと上書き(§4.3)
│   ├── contact.yml                # サイドバー下部の連絡先アイコン
│   └── share.yml                  # 記事下の共有ボタン
├── _plugins/posts-lastmod-hook.rb # 最終更新日を git log から取る(スターター同梱)
├── assets/css/jekyll-theme-chirpy.scss  # 見た目の上書きはすべてここ(§4)
├── index.html                     # トップ(スターター同梱、触らない)
├── .github/workflows/pages-deploy.yml   # ビルド & デプロイ
├── Gemfile                        # jekyll-theme-chirpy ~> 7.6
├── tools/                         # ローカルビルド用スクリプト(未使用)
└── LICENSE-chirpy-starter         # スターターの MIT ライセンス表記
```

スターター([chirpy-starter](https://github.com/cotes2020/chirpy-starter) 2026-06-20 時点)から `.devcontainer` `.vscode` `.gitmodules` を除いて取り込んだ。

---

## 3. `_config.yml` の要点

| キー | 値 | 理由 |
|---|---|---|
| `lang` | `ja-JP` | テーマの UI 文言ロケール。`ja` だと該当ファイルが無く**英語のまま**になる(上流の `_data/locales/ja-JP.yml` に合わせる) |
| `timezone` | `Asia/Tokyo` | 記事日時の解釈 |
| `title` / `tagline` / `description` | 日本語 | サイドバーと meta |
| `url` | `https://sakuratetsu1004.github.io` | 末尾スラッシュなし |
| `github.username` / `social.name` / `social.links` | `sakuratetsu1004` | メールは非公開(空) |
| `avatar` | `https://github.com/sakuratetsu1004.png` | GitHub アイコンを流用 |
| `theme_mode` | 空 | OS 設定に追従 + サイドバーのボタンで手動切替。固定しない |
| `pwa.cache.enabled` | `false` | Service Worker が古い CSS を配り続け「変更が反映されない」事故が起きたため |
| `exclude` | `docs`, `CLAUDE.md`, `README.md` など | 運用文書を公開物に混ぜない |
| `paginate` | 10 | 既定 |

---

## 4. 見た目のカスタマイズ(重要)

### 4.1 仕組み

Chirpy は色を CSS カスタムプロパティで定義している。サイト側に `assets/css/jekyll-theme-chirpy.scss` を置くと、テーマの同名ファイル(スタイルのエントリポイント)を**置き換える**ことになるため、先頭はテーマと同じ内容にして、その後ろに自前のスタイルを追記する。

```scss
---
---
@use 'abstracts/variables' with ($theme: '{{ site.theme_mode }}');
@use 'main{%- if jekyll.environment == 'production' -%}.bundle{%- endif -%}';
/* ↑ ここまでテーマと同一。変更禁止 ↓ ここから自前 */
```

**モード切替のセレクタ**は Chirpy 7.x では Bootstrap 5.3 流儀:

```scss
:root:not([data-bs-theme]), :root[data-bs-theme='light'] { ...ライト... }
:root[data-bs-theme='dark'] { ...ダーク... }
@media (prefers-color-scheme: dark) { :root:not([data-bs-theme]) { ...ダーク... } }
```

最初 `html[data-mode=...]` で書いて「CSS には入っているが何も変わらない」状態になった。配信 CSS を grep して `--main-bg` がどのセレクタで定義されているかを見て気づいた。**セレクタはテーマ更新で変わりうる**ので、テーマのバージョンを上げたら配信 CSS で再確認する。

### 4.2 配色(Claude 風クリーム & テラコッタ)

| 役割 | ライト | ダーク |
|---|---|---|
| 背景 `--main-bg` | `#F7F5EE` | `#262624` |
| サイドバー `--sidebar-bg` | `#EDE9DE` | `#1F1E1C` |
| 本文 `--text-color` | `#2B2924` | `#E6E3DA` |
| 見出し | `#1A1916` | `#F2F0E8` |
| リンク `--link-color` | `#A84E2B`(クリーム上で 5:1 確保) | `#E89B7A` |
| アクセント(チェック/フォーカス/記事タイトル上の罫線) | `#D97757` | `#D97757` |
| コードブロック `--highlight-bg-color` | `#EFECE3` | `#1F1E1C` |

`@mixin cream-light` / `@mixin cream-dark` にまとめてある。値を変えるだけなら mixin 内の数値を触る。

### 4.3 フォント

- Chirpy 既定は `"Source Sans Pro", "Microsoft Yahei", sans-serif`。**Microsoft Yahei は簡体字中国語フォント**で、Windows の日本語表示が中華風になる。これを外すのが最優先
- 現在は `$jp-sans: 'Meiryo', 'Noto Sans JP', 'Yu Gothic UI', 'Yu Gothic', 'Hiragino Sans', ...` で、**Meiryo 最優先はユーザー指定**。英数字も Meiryo になる(ユーザー了承済み)
- Noto Sans JP は Web フォントとして読み込む。読み込み URL は `_data/origin/cors.yml` の `webfonts:` 行。このファイルはテーマの同名ファイルを**丸ごとコピーして1行だけ変えた**もの。テーマ更新時は上流の cors.yml と差分を取る

### 4.4 サイズ

ユーザーは小さめを好む。Chirpy 既定 → 現在値:

| 要素 | 既定 | 現在 |
|---|---|---|
| 本文 `.content` | 1.08rem | **0.92rem**(行間 1.8) |
| h1 / h2 / h3 / h4 | 2 / 1.54 / 1.36 / 1.18rem | **1.6 / 1.25 / 1.1 / 1.0rem** |
| 記事一覧タイトル | 1.25rem | 1.1rem |
| サイトタイトル | 1.75rem | 1.5rem |
| 目次・メタ情報・右パネル | — | 0.85rem |

### 4.5 レイアウト

Chirpy 既定は `#main-wrapper > .container { max-width:1250px; margin:auto; padding-left:1.75rem }` で中央寄せ。ユーザーは**左寄せ・余白なし**を希望:

```scss
@media all and (min-width: 850px) {
  #main-wrapper > .container { max-width:none; margin-left:0; padding-left:0 !important; padding-right:1rem !important; }
}
```

残っている左の隙間は Bootstrap 列ガターの 0.75rem のみ。超ワイド(≥1650px)の右余白 4.5rem も 1.5rem に縮めている。

---

## 5. 記事の書き方

### 5.1 ファイルと front matter

`_posts/YYYY-MM-DD-slug.md`。slug は英小文字とハイフン。URL は `/posts/<slug>/`(日付は URL に入らない。初回は minima の `/2026/11/...` だったので旧 URL は 404)。

```yaml
---
title: "タイトル"
date: 2026-11-01 12:00:00 +0900
categories: [Linux, GNOME]   # [大分類, 小分類] 最大2階層。既存名を再利用
tags: [ubuntu, gnome, rdp]   # 小文字
description: 一覧カードと SEO に出る1文(任意)
---
```

`layout: post` は既定なので省略可。

### 5.2 文体・構成(既存記事に合わせる)

- 常体(〜だ / 〜した)。冒頭に「環境」表、以降は問題ごとに `## N. 見出し` → `### 症状` `### 原因` `### 試して駄目だったこと`(表) `### 最終的な対処` → 末尾に `## まとめ`
- 選択肢の比較は表。コマンドはコードブロック。注意書きは `>` 引用
- 目次は H2/H3 から自動生成されるので見出し階層を崩さない

### 5.3 公開前チェック(public リポジトリ)

- ホスト名・IP・MAC・証明書フィンガープリント・メールアドレス・顧客名/社名を書かない。`<RDP用ユーザー名>` のようなプレースホルダに置換
- 設定ファイルを貼るときは中身(トークン、個人パス)を確認
- 公開に向かない内容は非公開リポジトリ `workspace/knowledge/` へ

---

## 6. デプロイと確認

```bash
cd /home/hqnet/work/sakuratetsu1004.github.io
git add -A && git commit -m "..." && git push origin main

# ビルド待ち(1〜2分)
sleep 10; RUN=$(gh run list --limit 1 --json databaseId -q '.[0].databaseId'); gh run watch "$RUN" --exit-status

# 公開確認
curl -s -o /dev/null -w "%{http_code}\n" https://sakuratetsu1004.github.io/posts/<slug>/
# CSS を触った場合: 値が配信 CSS に入っているか
curl -s https://sakuratetsu1004.github.io/assets/css/jekyll-theme-chirpy.css | grep -oE '<値>' | head
```

ワークフローは `README.md` `.gitignore` `LICENSE` `CLAUDE.md` `docs/**` の変更ではビルドしない(`paths-ignore`)。

ブラウザで変化が見えないときは **Ctrl+Shift+R**。PWA キャッシュは無効化済みだが、無効化前に訪れたブラウザには旧 Service Worker が残っていることがある。

---

## 7. 権限まわり(Claude Code から操作する場合)

- `git push`(既存リポジトリの更新)、`gh run watch`、`gh api` の読み取りは自動モードで実行できる
- **public リポジトリの新規作成**(`gh repo create --public`)は「公開面の作成」として自動モードの分類器に拒否される。Pages 設定の変更も同系統の可能性がある。これらはユーザーに `!` 付きコマンドで実行してもらう
- `gh auth setup-git` 済みなので HTTPS でも認証は通る。現在の remote は SSH(`git@github.com:`)、`known_hosts` に github.com 登録済み
- コミットは `user.name=sakuratetsu1004` / `user.email=sakuratetsu1004@users.noreply.github.com`(実メール非公開)

---

## 8. 既知の課題・今後の候補

- コメント欄は未設定。付けるなら giscus(GitHub Discussions)が Chirpy 標準対応(`_config.yml` の `comments:`)
- Google Analytics / Search Console 未設定(`_config.yml` の `analytics:` / `webmaster_verifications:`)
- `_data/contact.yml` `_data/share.yml` はスターター既定のまま(Twitter 等のリンクが出る)。不要なものは `false` にして整理してよい
- Chirpy のメジャーアップデート(8.x)時は §4.1 のセレクタ、§4.3 の cors.yml、エントリポイント scss の `@use` 行を上流と突き合わせる
- 記事カテゴリ体系は現状 `[Linux, GNOME]` のみ。増えてきたら大分類の命名規則(Linux / Network / Windows / Tools など)を決める

---

## 9. 変更履歴(要約)

| 日付 | 内容 |
|---|---|
| 2026-11-01 | 初回公開(minima)→ Chirpy 移行、`ja-JP` ロケール、クリーム配色、PWA キャッシュ無効化、日本語フォント(Meiryo 優先)と縮小、左寄せレイアウト、本書と CLAUDE.md 作成 |
