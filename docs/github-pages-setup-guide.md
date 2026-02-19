# Jekyll + GitHub Pages デプロイガイド

本ドキュメントは、京大エスペラント語研究会のウェブサイトを GitHub Pages で公開するまでの過程で得た知見をまとめたものです。同様のプロジェクトを立ち上げる際の参考にしてください。

---

## 目次

1. [前提知識：GitHub Pages の種類](#1-前提知識github-pages-の種類)
2. [リポジトリの命名規則（最重要）](#2-リポジトリの命名規則最重要)
3. [Jekyll プロジェクトの構成](#3-jekyll-プロジェクトの構成)
4. [GitHub Actions ワークフローの設定](#4-github-actions-ワークフローの設定)
5. [_config.yml の設定](#5-_configyml-の設定)
6. [ローカル開発環境のセットアップ](#6-ローカル開発環境のセットアップ)
7. [GitHub への初回プッシュ](#7-github-への初回プッシュ)
8. [GitHub Pages の有効化](#8-github-pages-の有効化)
9. [遭遇した問題と解決策](#9-遭遇した問題と解決策)
10. [チェックリスト](#10-チェックリスト)

---

## 1. 前提知識：GitHub Pages の種類

GitHub Pages には **2種類** あり、挙動が大きく異なります。

| 種類 | リポジトリ名 | 公開URL | baseurl |
|------|-------------|---------|---------|
| **Organization/User Pages** | `<org名>.github.io` | `https://<org名>.github.io/` | 不要（空） |
| **Project Pages** | それ以外の任意の名前 | `https://<org名>.github.io/<repo名>/` | `/<repo名>` が必要 |

### 教訓：リポジトリ名は最初から正しく決める

当プロジェクトでは、最初 `es-kiotauniv.github.io` というリポジトリ名で作成しました。これは Organization Pages ではなく **Project Pages** として扱われ、`https://...github.io/es-kiotauniv.github.io/` というサブパスで配信されました。

その結果、CSS・画像などの全アセットのパスに `/es-kiotauniv.github.io/` というプレフィックスが必要になり、設定が複雑化しました。

**最終的にリポジトリ名を `esperanto-societo-de-kioto-universitato.github.io`（Organization 名と一致）に変更して解決しました。**

> **推奨：** 特別な理由がない限り、Organization/User Pages（`<org名>.github.io`）を使ってください。`baseurl` が不要になり、設定がシンプルになります。

---

## 2. リポジトリの命名規則（最重要）

```
✅ 推奨：<org名>.github.io
   → baseurl 不要、ルートで配信

❌ 避ける：任意の名前.github.io（org名と不一致）
   → Project Pages 扱い、baseurl 必須、パス設定が複雑
```

Organization 名が長い場合でも、リポジトリ名を合わせることを強く推奨します。

---

## 3. Jekyll プロジェクトの構成

```
.
├── .github/
│   └── workflows/
│       └── jekyll.yml          # GitHub Actions ワークフロー
├── _includes/                  # ヘッダー、フッターなどの部品
├── _layouts/                   # ページレイアウト
├── _plugins/                   # カスタムプラグイン（※GitHub の組み込み Jekyll では無効）
├── _sass/                      # SCSS スタイルシート
├── assets/
│   ├── img/                    # 画像ファイル
│   ├── js/                     # JavaScript
│   ├── main.scss               # メインCSS（SCSS→CSSにコンパイルされる）
│   └── social-icons.svg        # SNSアイコン
├── ja/                         # 日本語ページ
├── eo/                         # エスペラント語ページ
├── _config.yml                 # Jekyll 設定ファイル
├── Gemfile                     # Ruby 依存関係
├── Gemfile.lock                # 依存関係のロックファイル
└── .gitignore
```

### .gitignore

```
_site/
_image/
```

`_site/` はビルド出力なので、Git に含めません。

---

## 4. GitHub Actions ワークフローの設定

### jekyll.yml（実際に動作した設定）

```yaml
name: Deploy Jekyll site to Pages

on:
  push:
    branches: ["main"]
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
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'       # ← Gemfile.lock の Ruby バージョンと合わせる！
          bundler-cache: true

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Jekyll
        run: bundle exec jekyll build  # ← --baseurl は付けない（_config.yml に任せる）
        env:
          JEKYLL_ENV: production

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 重要なポイント

#### Ruby バージョンを合わせる

```yaml
# ❌ NG：Gemfile.lock が Ruby 3.2 で生成されているのに 3.1 を指定
ruby-version: '3.1'

# ✅ OK：ローカル環境と同じバージョンを指定
ruby-version: '3.2'
```

**確認方法：**
```bash
ruby --version          # ローカルの Ruby バージョン
```

バージョン不一致だと `gem install` が失敗します（`bigdecimal 4.0.1` 等が Ruby 3.2+ を要求）。GitHub Actions 上ではエラーメッセージが分かりにくく、ワークフローが数秒で「失敗」するだけなので、原因特定が困難です。

#### `--baseurl` オプションは使わない

GitHub のテンプレートには以下の行がありますが、削除を推奨します：

```yaml
# ❌ configure-pages の出力が空になる場合があり、_config.yml の設定を上書きしてしまう
run: bundle exec jekyll build --baseurl "${{ steps.pages.outputs.base_path }}"

# ✅ _config.yml の url/baseurl 設定に任せる
run: bundle exec jekyll build
```

#### ワークフローファイルは1つだけにする

複数のワークフローファイル（例：`deploy.yml` と `jekyll.yml`）が同じトリガーで存在すると、両方が実行されて混乱の原因になります。

---

## 5. _config.yml の設定

### Organization Pages の場合（推奨）

```yaml
url: "https://<org名>.github.io"
# baseurl は設定しない（不要）
```

### Project Pages の場合（非推奨）

```yaml
url: "https://<org名>.github.io"
baseurl: "/<repo名>"
```

### テンプレートでのパス指定

```html
<!-- CSS -->
<link rel="stylesheet" href="{{ '/assets/main.css' | relative_url }}">

<!-- 画像 -->
<img src="{{ '/assets/img/logo.jpg' | relative_url }}" alt="ロゴ">

<!-- リンク -->
<a href="{{ '/' | relative_url }}">ホーム</a>
```

`relative_url` フィルターは `baseurl` を自動的に付加します。**絶対パスをハードコードしない**でください：

```html
<!-- ❌ NG：baseurl が無視される -->
<img src="/assets/img/logo.jpg">

<!-- ✅ OK：baseurl が自動付加される -->
<img src="{{ '/assets/img/logo.jpg' | relative_url }}">
```

---

## 6. ローカル開発環境のセットアップ

### 必要なパッケージ（Ubuntu/Debian）

```bash
sudo apt-get install -y ruby-dev build-essential
```

### Gem のインストール（ユーザーローカル）

```bash
export GEM_HOME="$HOME/.gem"
export PATH="$HOME/.gem/bin:$PATH"

# .bashrc に追加しておくと便利
echo 'export GEM_HOME="$HOME/.gem"' >> ~/.bashrc
echo 'export PATH="$HOME/.gem/bin:$PATH"' >> ~/.bashrc

gem install bundler
bundle install
```

### ローカルプレビュー

```bash
bundle exec jekyll serve
# → http://localhost:4000/ でプレビュー
```

---

## 7. GitHub への初回プッシュ

### SSH 設定（複数アカウント対応）

`~/.ssh/config` に Host エイリアスを設定している場合：

```bash
# g1 は ~/.ssh/config で定義した Host エイリアス
git remote add origin g1:OrgName/repo-name.git
git push -u origin main
```

### 権限エラーの場合

リポジトリの作成者と push するアカウントが異なる場合：
1. リポジトリの **Settings → Collaborators** で push するアカウントを追加
2. 招待を承認後、再度 push

---

## 8. GitHub Pages の有効化

1. リポジトリの **Settings** → **Pages** を開く
2. **Build and deployment** の **Source** を **「GitHub Actions」** に設定
3. ワークフローが正常に実行されるとサイトが公開される

> **注意：** 「Deploy from a branch」ではなく「GitHub Actions」を選択してください。
> 「Deploy from a branch」は GitHub の組み込み Jekyll を使い、`_plugins/` のカスタムプラグインが無効になります。

---

## 9. 遭遇した問題と解決策

### 問題1：ワークフローが全て失敗していた（最大の問題）

**症状：** GitHub Actions のワークフロー実行が数秒～十数秒で完了し、赤い × マーク。WebFetch で確認した際「Success」と誤認してしまった。

**原因：** `jekyll.yml` の Ruby バージョン（3.1）と `Gemfile.lock` で要求される Ruby バージョン（3.2+）の不一致。`bigdecimal 4.0.1` や `sass-embedded 1.97.3` が Ruby 3.2 以上を要求。

**解決：** `jekyll.yml` の `ruby-version` を `'3.2'` に変更。

**教訓：** ローカルで `ruby --version` を確認し、CI の Ruby バージョンを合わせること。

---

### 問題2：CSS・画像が表示されない（404）

**症状：** HTML は表示されるが、CSS なし・画像なしの素のページ。

**原因：** Project Pages（`/repo-name/` サブパスで配信）なのに `baseurl` が設定されていなかった。CSS は `/repo-name/assets/main.css` にあるのに、HTML は `/assets/main.css` を参照していた。

**解決：** リポジトリ名を Organization 名と一致させ（`<org>.github.io`）、Organization Pages として配信。`baseurl` が不要になった。

---

### 問題3：`--baseurl` が `_config.yml` の設定を上書き

**症状：** `_config.yml` に `baseurl` を設定したのに反映されない。

**原因：** `jekyll.yml` の `--baseurl "${{ steps.pages.outputs.base_path }}"` が空文字を返し、`_config.yml` の `baseurl` を空で上書きしていた。

**解決：** `--baseurl` オプションを削除し、`_config.yml` の設定に任せた。

---

### 問題4：ワークフローファイルが2つ存在

**症状：** 同じコミットに対してワークフローが2回実行される。

**原因：** `deploy.yml` と `jekyll.yml` の両方が存在し、同じトリガー（`push: branches: ["main"]`）を持っていた。

**解決：** `deploy.yml` を削除し、`jekyll.yml` に一本化。

---

### 問題5：Git push の権限エラー

**症状：** `git push` で `Permission denied` エラー。

**原因：** SSH キーのアカウントがリポジトリの Collaborator に追加されていなかった。

**解決：** リポジトリの Settings → Collaborators でアカウントを追加。

---

### 問題6：リポジトリ名変更後の 404

**症状：** リポジトリ名変更後、サイトが 404 になる。

**原因：** 名変更後、GitHub Pages の DNS 反映に時間がかかる。また、ワークフローの再実行が必要。

**解決：** Actions タブから手動でワークフローを再実行（Run workflow）。数分で反映された。

---

## 10. チェックリスト

### 初回セットアップ時

- [ ] リポジトリ名が `<org名>.github.io` と一致しているか
- [ ] `_config.yml` の `url` が正しいか
- [ ] Organization Pages なら `baseurl` が設定されていないか
- [ ] `.github/workflows/` にワークフローファイルが **1つだけ** あるか
- [ ] ワークフローの `ruby-version` がローカルと一致しているか
- [ ] `Gemfile.lock` がコミットされているか
- [ ] テンプレート内の全パスが `relative_url` フィルターを使っているか
- [ ] GitHub Pages の Source が「GitHub Actions」になっているか
- [ ] push するアカウントが Collaborator に追加されているか

### デプロイ後の確認

- [ ] サイトにアクセスしてページが表示されるか
- [ ] CSS が適用されているか（スタイル崩れがないか）
- [ ] 画像が表示されているか
- [ ] ナビゲーションリンクが正しく動作するか
- [ ] 言語切り替え（ja/eo）が動作するか
- [ ] OGP 画像が正しい URL を指しているか

### トラブルシューティング

- [ ] Actions タブでワークフローの実行結果を確認（赤 × なら失敗）
- [ ] 失敗した場合、ビルドログで具体的なエラーメッセージを確認
- [ ] `curl -sI <サイトURL>` で HTTP ステータスコードを確認
- [ ] ブラウザの開発者ツール（F12）→ Console/Network タブで 404 を確認

---

## 参考リンク

- [GitHub Pages ドキュメント](https://docs.github.com/ja/pages)
- [Jekyll 公式サイト](https://jekyllrb.com/)
- [GitHub Actions for GitHub Pages](https://github.com/actions/deploy-pages)

---

*本ドキュメントは 2026年2月20日に作成されました。*
*京大エスペラント語研究会 (La Esperanto-Societo de la Universitato de Kioto)*
