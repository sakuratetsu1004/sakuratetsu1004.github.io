# CLAUDE.md — sakuratetsu1004.github.io

公開ブログ(GitHub Pages / Jekyll / Chirpy テーマ)のリポジトリ。人間向けの詳細は `docs/HANDOVER.md`。

## これは何か

- 公開 URL: https://sakuratetsu1004.github.io/ 。**public リポジトリ**。push すれば誰でも読める
- ネットワーク/インフラの作業メモ。「なぜそうなったか」「何を試して駄目だったか」を残すのが趣旨
- ビルドは GitHub Actions (`.github/workflows/pages-deploy.yml`)。ローカルに Ruby/Jekyll は不要で、入れない

## 記事を書く

1. `_posts/YYYY-MM-DD-slug.md` を作る(slug は英小文字とハイフン。URL は `/posts/<slug>/`)
2. front matter:
   ```yaml
   ---
   title: "タイトル"
   date: 2026-11-01 12:00:00 +0900
   categories: [Linux, GNOME]   # [大分類, 小分類] 最大2階層。既存カテゴリ名を再利用する
   tags: [ubuntu, gnome]        # 小文字
   description: 一覧と SEO に出る1文(任意)
   ---
   ```
3. 本文は H2/H3 で構成(目次が自動生成される)。表・コードブロック可。画像は `assets/img/posts/`
4. commit → `git push origin main` → 1〜2分でビルド。確認手順は下記

### 公開前の必須チェック(public なので)

- ホスト名・IP・MAC・証明書フィンガープリント・メールアドレス・顧客名・社名を含めない。例示は `<RDP用ユーザー名>` や `192.0.2.x` のようなプレースホルダにする
- 鍵・トークン・パスワードは当然不可。`~/.config` 由来の設定を貼るときも中身を確認する
- 既存記事の文体に合わせる: 常体(〜だ/〜した)、表で選択肢比較、「ハマった点」「試して駄目だったこと」「最終的な対処」を見出しに

## 変更を確認する

```bash
git push origin main
sleep 10; RUN=$(gh run list --limit 1 --json databaseId -q '.[0].databaseId'); gh run watch "$RUN" --exit-status
curl -s -o /dev/null -w "%{http_code}\n" https://sakuratetsu1004.github.io/posts/<slug>/
```

CSS を触ったら配信中の CSS に値が入ったかを grep で確かめる(過去にセレクタ不一致で「入っているが効かない」事故あり):

```bash
curl -s https://sakuratetsu1004.github.io/assets/css/jekyll-theme-chirpy.css | grep -oE '<探したい値>' | head
```

ユーザーの画面で変わらない場合は Ctrl+Shift+R を案内する(旧 Service Worker のキャッシュ)。

## 見た目のカスタマイズはすべて1ファイル

`assets/css/jekyll-theme-chirpy.scss` 。先頭の `@use` 2行はテーマのエントリポイントで**変更禁止**、その下が自前の追記。

- 配色: `@mixin cream-light` / `@mixin cream-dark` の CSS 変数。クリーム背景 + テラコッタのアクセント(Claude 風)
- モード切替セレクタは **`:root[data-bs-theme=light|dark]` と `:root:not([data-bs-theme])`**(Chirpy 7.x)。`html[data-mode]` は効かない
- フォント: `$jp-sans` 。**Meiryo 最優先**(ユーザー指定)。Chirpy 既定の `"Microsoft Yahei"` は中華フォントに見えるので必ず外す
- サイズ: 本文 `.content` 0.92rem、h1 1.6 / h2 1.25 / h3 1.1rem。ユーザーは小さめを好む
- レイアウト: `#main-wrapper > .container` を左寄せ(`margin-left:0; padding-left:0; max-width:none`)。中央寄せに戻さない

Web フォントの追加は `_data/origin/cors.yml` の `webfonts:` 行(テーマの同名ファイルを丸ごと上書きしているので、テーマ更新時は差分を確認)。

## 設定の要点 (`_config.yml`)

- `lang: ja-JP`(`ja` だと UI 文言が英語のまま。ロケールファイル名に合わせる)
- `timezone: Asia/Tokyo`、`url: https://sakuratetsu1004.github.io`
- `pwa.cache.enabled: false`(古い CSS を掴み続ける事故を防ぐため)
- `avatar:` は GitHub のアイコン URL
- `exclude:` に `docs`, `CLAUDE.md` が入っている(公開物に混ぜない)

## やってはいけない / 注意

- `_config.yml` の `theme_mode` を固定しない(OS 追従 + 手動切替が現状の仕様)
- Chirpy のテーマ本体(gem)は触れない。上書きは `assets/css/...scss`、`_data/`、`_tabs/`、`_includes/` のコピーで行う
- `git push` は既存リポジトリの更新なので Claude から実行してよい。新規 public リポジトリ作成や Pages 設定変更のような「公開面の作成」は自動モードで拒否されるので、ユーザーに `!` 付きコマンドで依頼する
- コミットメッセージは英語1行要約 + 必要なら本文。セッションの attribution 行はそのまま付ける
- Gemfile のバージョン (`jekyll-theme-chirpy ~> 7.6`) を上げるときは、上書きしている `_data/origin/cors.yml` とセレクタ名が上流で変わっていないか確認してから

## 関連

- ユーザーの私的ナレッジ(非公開)は別リポジトリ `workspace/knowledge/`。公開に向かない内容はそちら
- 初回記事(2026-11-01)の題材になった環境: VMware 上の Ubuntu 26.04 / GNOME 50 / gnome-remote-desktop ヘッドレス / fcitx5+Mozc
