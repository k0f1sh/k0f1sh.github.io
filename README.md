# k0f1sh.github.io

Zola で生成する個人の技術ブログ

## ローカルで確認する

[Zola](https://www.getzola.org/documentation/getting-started/installation/) をインストールして、次を実行します。

```sh
zola serve
```

表示された URL（通常は <http://127.0.0.1:1111>）をブラウザで開きます。公開用ファイルだけを生成する場合は `zola build` を実行します。

## 記事を追加する

`content/posts/` に Markdown ファイルを追加します。

```toml
+++
title = "記事のタイトル"
description = "一覧に表示する短い説明"
date = 2026-09-14
[taxonomies]
tags = ["Zola", "Rust"]
+++
```

`main` ブランチへの push 後は GitHub Actions がビルドし、GitHub Pages へ公開します。
