+++
title = "choにCSVの列名参照を追加した"
description = "CSVの列を名前で選ぶ%nameの記法と、ヘッダーを列番号に結び付ける実装の話"
date = 2026-09-26
+++

[cho](../cho01/) は、Lisp風の式で入力行を加工するコマンドラインツールです。通常は空白で区切られた行を列に分けて扱いますが、`--csv` を付けるとCSVも読み込めます。この記事では、CSVのヘッダー名で列を指定する方法と、その実装を紹介します。

choでは、入力の列を `$1`、`$2` のように番号で参照できます。小さいファイルならこれで十分です。でも、横に長いCSVのヘッダーを左から追って「`created_at` は何列目だっけ」と数えるのは、かなり面倒です。しかも途中に列が一つ増えたらズレてしまいます。

そこで、ヘッダー付きのCSVでは `%created_at` と名前で参照できるようにしました。

例えば、次のような `users.csv` があるとします。データは架空のものです。

```text
id,account_id,name,email,team,region,plan,status,owner,source,created_at,score
1,101,Alice,alice@example.com,dev,JP,free,active,bob,web,2026-09-01,90
```

メールアドレスと作成日を取り出す式は、列番号なら `$4` と `$11`、列名なら `%email` と `%created_at` です。`p` は指定した値を空白区切りで出力します。

```console
$ cat users.csv | cho --csv --skip-header '(p $4 $11)'
alice@example.com 2026-09-01

$ cat users.csv | cho --csv '(p %email %created_at)'
alice@example.com 2026-09-01
```

`%name` を使うと、choは先頭レコードをヘッダーとして読み、データ行から処理を始めます。`--skip-header` を付け忘れてヘッダー行を処理してしまう心配もありません。

[GNU awkでは5.3から `--csv` でCSVを読めます](https://www.gnu.org/software/gawk/manual/html_node/Feature-History.html)。これがあったのでchoにもCSV入力をサポートしたいと思いました。さらに、ヘッダー付きCSVを扱うなら、列名を式に直接書けると便利だと思い、`%name` という記法を作りました。

## `%name` という記法にした理由

列名の前に付ける記号は、Perlのハッシュ変数から連想して `%` にしました。CSVのヘッダー名から対応する列を引く、という操作がキーから値を引くハッシュに少し似ているためです。

```text
$1       1列目を参照する
%email   ヘッダーが email の列を参照する
```

列名に空白がある場合は、`%"display name"` と書けます。これは `display name` という名前の列を探します。

choでは、式の中で列番号を指定する `$1`、CSVのヘッダー名を指定する `%email`、[`-C` オプション](../cho04/) の引数で列番号を指定する `@1` という三つの書き方があります。`@1` はシェルに展開されないようにするための記法で、指す列は `$1` と同じです。

## ヘッダー名は最初に列番号へ解決する

choは先にプログラムを読み取って評価用の式に組み立て、それから入力を1レコードずつ処理します。式を組み立てる時点ではCSVのヘッダーをまだ読んでいないため、`%email` を直接「4列目」にはできません。そこで、式に現れたヘッダー名を `Program.header_fields` に集め、式にはその配列の添字を `Expr::HeaderField(0)` のように持たせます。この `0` はCSVの列番号ではありません。

入力の処理を始めると、最初のCSVレコードをヘッダーとして読みます。このレコードはデータ行として評価せず、参照した名前が何列目かを調べるために使います。名前と同じ順序で列番号を `header_field_indices: Vec<usize>` に保存してから、次のレコード以降を処理します。データ行では、式が持つ添字から列番号を引いてフィールドを読むだけです。

<figure class="header-resolve">
  <div class="header-resolve-step">
    <div class="header-resolve-heading"><span class="header-resolve-number">1</span><strong>式を組み立てる</strong><span class="header-resolve-timing">ヘッダーを読む前</span><a class="header-resolve-source" href="https://github.com/k0f1sh/cho/blob/32de98f7f3b65327b0078869260209d1ed1581e6/src/compiler.rs#L128-L138">compiler.rs ↗</a></div>
    <div class="header-resolve-flow"><span class="header-resolve-node"><code>%email</code><span class="header-resolve-detail">式では <code>Expr::HeaderField(0)</code></span></span><span class="header-resolve-arrow" aria-hidden="true">→</span><span class="header-resolve-node"><code>Program.header_fields: Vec&lt;String&gt;</code><span class="header-resolve-detail"><code>["email", "created_at"]</code></span></span></div>
  </div>
  <div class="header-resolve-step">
    <div class="header-resolve-heading"><span class="header-resolve-number">2</span><strong>先頭レコードで列番号を決める</strong><span class="header-resolve-timing">一度だけ</span><a class="header-resolve-source" href="https://github.com/k0f1sh/cho/blob/32de98f7f3b65327b0078869260209d1ed1581e6/src/runtime/runner.rs#L135-L158">runner.rs ↗</a></div>
    <div class="header-resolve-flow"><span class="header-resolve-node">ヘッダーでは <code>email</code> が4列目</span><span class="header-resolve-arrow" aria-hidden="true">→</span><span class="header-resolve-node"><code>header_field_indices: Vec&lt;usize&gt;</code><span class="header-resolve-detail"><code>[4, 11]</code></span></span></div>
  </div>
  <div class="header-resolve-step">
    <div class="header-resolve-heading"><span class="header-resolve-number">3</span><strong>データ行から値を読む</strong><span class="header-resolve-timing">各レコードで繰り返す</span><a class="header-resolve-source" href="https://github.com/k0f1sh/cho/blob/32de98f7f3b65327b0078869260209d1ed1581e6/src/runtime/eval.rs#L202-L207">eval.rs ↗</a></div>
    <div class="header-resolve-flow"><span class="header-resolve-node"><code>Expr::HeaderField(0)</code></span><span class="header-resolve-arrow" aria-hidden="true">→</span><span class="header-resolve-node"><code>header_field_indices[0] = 4</code></span><span class="header-resolve-arrow" aria-hidden="true">→</span><span class="header-resolve-node"><code>record.field(4)</code><span class="header-resolve-detail"><code>alice@example.com</code></span></span></div>
  </div>
  <figcaption><code>(p %email %created_at)</code> の例。配列の添字 <code>0</code> は <code>email</code> に対応し、列番号は先頭の一度だけ調べる。</figcaption>
</figure>

列名は完全一致で探します。式で参照した名前がヘッダーにない場合や、同じ名前の列が複数ある場合は、データ行を出力する前にエラーにします。例えば `email` が二列あれば、`%email` をどちらに結び付けるか決められません。一方、式に登場しない列名が重複していてもエラーにはしません。読まない列の名前が重なっているだけで、必要な列まで取り出せなくなるのは困るからです。

また、`%name` は `--csv` と一緒に使う記法です。通常の空白区切り入力ではヘッダー名による参照はできません。

## まとめ

choの仕様はできるだけ小さく保ちたかったので、最初はCSVでも `$1` のような番号指定で十分かと思っていました。しかし横に長いCSVの列を数えるのはやっぱり面倒で、フィールド名での参照記法を追加しました。

`$1` の番号指定は残しているので、番号だけで済むときはこれまでどおり書けます。横に長いCSVでは `%name` のフィールド名指定を選べるようになり、記法は増えましたが、choは前より使いやすくなったと思います。
