+++
title = "awkをLispで書きたかったのでテキスト処理ツール『cho』を作った"
description = "awkのように列を扱い、日時やIPアドレスも比較・加工できる、S式のテキスト処理ツールの紹介"
date = 2026-09-20
+++

[`cho`](https://github.com/k0f1sh/cho) というコマンドラインツールを作っています。
choはawkを参考にした小さなテキスト処理ツールです。
標準入力を1行ずつ読み込み、行の絞り込みや列の取り出し、加工をS式で記述できます。

awkの列を手軽に扱えるところが好きですが、いつも書き方を忘れてしまいます。
EmacsやClojureが好きなので、awkもLispで書けたらなぁと思って作り始めました。

choでは、日時やIPアドレスなど、文字列の比較だけでは書きにくい絞り込みもできます。
次の例では、日時とIPアドレスが並ぶログから「2026年8月1日以降、かつIPアドレスが `10.0.0.0/8` に含まれる行」を取り出しています。

```console
$ cat events.log
2026-08-02T09:00:00Z 10.1.2.3 deploy
2026-07-31T23:00:00Z 10.2.3.4 old
2026-08-03T12:00:00Z 8.8.8.8 external

$ cat events.log | cho '(f (dt/>= $1 "2026-08-01T00:00:00Z")) (f (cidr/contains? "10.0.0.0/8" $2)) (p $1 $2 $3)'
2026-08-02T09:00:00Z 10.1.2.3 deploy
```

## cho の特徴

### 1. awkのように手軽に列を取り出せる

「Lispでテキスト処理をする」だけであれば、SchemeやBabashkaを使うほうがよいと思います。
しかし、1行ずつ読み込んで、列に分割して値を取り出すところまで書くとちょっと面倒です。
choはawkのように入力をデフォルトで空白区切りに分割し、各列を `$1`、`$2` で参照できるようにします。
列数の `NF`、レコード番号の `NR`、レコード全体の `$0` も使えます。

たとえば、サービス名、応答時間（ミリ秒）、リクエストのパスが並ぶログを処理してみます。
次の例では、2列目を数値として比較し、応答時間が20ミリ秒を超えた行のサービス名とパスを出力します。

```console
$ cat timings.log
api 18 /health
api 150 /users
api 240 /orders
worker 180 /jobs

$ cat timings.log | cho '(f (> $2 20)) (p $1 $3)'
api /users
api /orders
worker /jobs
```

`f` は `filter` の短縮形で、条件を満たしたレコードだけを通します。`p` は `print` の短縮形で、指定した値を出力します。

CSVやTSVにも対応しています。CSVモード（`--csv`）では、引用符で囲まれたフィールド内のカンマや改行も扱えます。
次の例では、`Tokyo, Japan` がひとつのフィールドになります。

```console
$ cat people.csv
name,city
Alice,"Tokyo, Japan"
Bob,Osaka

$ cat people.csv | cho --csv --skip-header '(p (s/join " -> " $1 $2))'
Alice -> Tokyo, Japan
Bob -> Osaka
```

### 2. 処理をS式で書ける

choでは、関数の呼び出しを `(関数名 引数...)` の形で書きます。

```lisp
(filter (> $2 20))
(print $1 (s/upper $3))
```

awkでいう `awk '$2 > 20 {print $1, toupper($3)}'` に相当します。

### 3. 集計やソートは既存コマンドと組み合わせる

choはスクリプトを書くほどではない処理を、ワンライナーで簡単に書けるようにするために作っています。
そのため、配列、ループ、ユーザー定義関数、代入、`BEGIN` / `END` ブロックは用意していません。こうした機能が必要になる処理には、awkやスクリプト言語を使うほうがよいと思っています。

出現頻度のカウントやソートなどは、素直に `sort` や `uniq` などの既存コマンドにお任せする方針にしています。

先ほどの `timings.log` から、応答時間が100ミリ秒を超えたリクエストをサービスごとに数えるなら、こう書きます。

```console
$ cat timings.log | cho '(f (> $2 100)) (p $1)' | sort | uniq -c
      2 api
      1 worker
```

### 4. 日時やIPアドレスなどを、型に応じて比較・加工できる

ワンライナーでコマンドを組み立てようとするとき、文字列や数値だけでなく、日時やIPアドレス、URL、単位付きのファイルサイズなどを扱いたい場面がよく出てきます。

日時の比較ではタイムゾーンを考慮する必要がありますし、IPアドレスがCIDRで指定したネットワークに含まれるかは、文字列の比較だけでは判定できません。

choでは、各フィールドを文字列として読み込み、関数が求める型に応じて自動的に変換します。使う側で型を指定しなくても、次のような処理を書けます。

- 日付や日時を比較し、日数の加算や日時の差分を計算する
- `MB` や `MiB` などの単位が付いたサイズを比較し、バイト数に変換する
- IPv4・IPv6アドレスを正規化し、CIDRで指定したネットワークに含まれるかを判定する
- URLからホスト名やパスを取り出す

このほか、SemVerの比較やUUID・ULIDの生成にも対応しています。

たとえば、1列目がIPアドレス、5列目が完全なURLのログなら、次のようにネットワークで絞り込み、IPアドレスとURLのパスを出力できます。

```console
$ cat access.log
192.168.1.10 2026-08-02T09:00:00Z GET 200 https://example.com/api/users
8.8.8.8 2026-08-02T09:01:00Z GET 200 https://example.com/health
192.168.1.20 2026-08-02T09:02:00Z GET 404 https://example.com/missing

$ cat access.log | cho '(f (cidr/contains? "192.168.0.0/16" $1)) (p $1 (url/path $5))'
192.168.1.10 /api/users
192.168.1.20 /missing
```

この例では、`cidr/contains?` に渡す `$1` はIPアドレスとして、`url/path` に渡す `$5` はURLとして解釈されます。

型変換に失敗した場合はその時点でエラー終了します。不正な値を黙って読み飛ばし、ログの形式が崩れていることを見落とすのを防ぐためです。

たとえば、URLとして解釈できない文字列を `url/path` に渡すと、どのレコードのどの引数で失敗したかが表示されます。

```console
$ echo 'not-a-url' | cho '(p (url/path $1))'
cho: record 1: url/path: argument 1 expects Url (absolute URL), but "not-a-url" is not a valid absolute URL
```

この場合、終了コードは `1` になります。エラー時に代わりの値を使って処理を続けたいなら、`default` で明示できます。

```console
$ echo 'not-a-url' | cho '(p (default (url/path $1) "INVALID_URL"))'
INVALID_URL
```

### 5. helpとaproposで、調べながら式を組み立てられる

ワンライナーを書いている途中で、関数名や引数の順番を調べるためにブラウザを開くのは面倒です。

choには、ターミナルで調べながら式を組み立てられるように、関数を検索する `--apropos` と、詳しい使い方を表示する `--help` があります。

関数名には、文字列を扱う `s/`（String）、日付を扱う `d/`（Date）、日時を扱う `dt/`（DateTime）などの接頭辞を付けています。`s/upper` は文字列の大文字化、`dt/>=` は日時の比較、といった具合です。IPアドレスには `ip/`、CIDRには `cidr/` が付くので、関数を探すときの手がかりにもなります。

「CIDRに関する関数は何があったか」「文字列のトリムはどうやるか」といった場合、`-k`（または `--apropos`）で関連する関数の一覧と概要を検索できます。

以下は出力の抜粋です。

```console
$ cho -k cidr
...
cidr             create a CIDR normalized to its network address
cidr/from-mask   create a CIDR from an IP address and netmask
cidr/netmask     return the network mask
...
cidr/contains?   test CIDR membership
cidr/network     return the network address
cidr/prefix      return the prefix length
...
```

使いたい関数が見つかったら、`cho --help <関数名>` でその関数の型シグネチャと実行可能なサンプルコードを確認できます。

```console
$ cho --help cidr/contains?
cidr/contains? — test CIDR membership

Signatures:
  (cidr/contains? CIDR IPADDR) -> BOOLEAN

Examples:
  echo 10.20.30.40 | cho '(cidr/contains? "10.0.0.0/8" $1)'  # => true
```


このサンプルのように、値を返す式を1つだけ書いた場合は、`p` を付けなくても結果が自動的に出力されます。

## インストール

インストールにはRustの開発環境（Cargo）が必要です。
choはまだ試行錯誤中なので、構文や動作が変わる可能性があります。

```console
$ cargo install --git https://github.com/k0f1sh/cho.git
```

- [k0f1sh/cho (GitHub)](https://github.com/k0f1sh/cho)

## まとめ

choは、awkのように列を取り出し、S式で条件や加工を書くテキスト処理ツールです。日時・IPアドレス・URL・バイトサイズなども扱えます。集計やソートは既存コマンドに任せ、ワンライナーで手軽に書ける範囲に機能を絞っています。

最初はawkっぽい自作Lispからスタートしましたが、awkとはまた違った便利なツールになったのではないかと思っています。

ちなみに、choという名前は「兆」から来ています。(awkが「億」っぽいので)
