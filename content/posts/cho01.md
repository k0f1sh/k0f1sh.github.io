
+++
title = "S式で書く行指向DSL『cho』"
date = 2026-09-20
+++


[`cho`](https://github.com/k0f1sh/cho) というコマンドラインツールを作っています。
choはawkを参考にした小さなテキスト処理ツールです。
標準入力を1行ずつ読み込み、行の絞り込みや列の取り出し、加工をS式で記述できます。

たとえば、日時とIPアドレスが並ぶログから「2026年8月1日以降、かつIPアドレスが `10.0.0.0/8` に含まれる行」を取り出すなら、こう書けます。

```console
$ cat events.log
2026-08-02T09:00:00Z 10.1.2.3 deploy
2026-07-31T23:00:00Z 10.2.3.4 old
2026-08-03T12:00:00Z 8.8.8.8 external

$ cat events.log | cho '(f (dt/>= $1 "2026-08-01T00:00:00Z")) (f (cidr/contains? "10.0.0.0/8" $2)) (p $1 $2 $3)'
2026-08-02T09:00:00Z 10.1.2.3 deploy
```

入力はデフォルトで空白区切りの列として扱われ、各列は `$1`、`$2` のように参照します。`f` は `filter` の短縮形で、条件を満たしたレコードだけを通します。`p` は `print` の短縮形で、指定した列を出力します。

## cho の特徴

### 1. awkのように手軽に列を取り出せる

「Lispでテキスト処理をする」だけであれば、SchemeやBabashkaを使うほうがよいと思います。
しかし、1行ずつ読み込んで、列に分割して値を取り出すところまで書くとちょっと面倒です。
choではawkのように入力を分割し、`$1`、`$2` で参照できるようにするところまでをやってくれます。
（`NF`、`NR`、`$0` なども使えます）

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

awkの列を手軽に扱えるところが好きですが、いつも書き方を忘れてしまいます。
私はEmacsが好きなので、awkがLispで書けたらなぁと思っていて、それがchoを作り始めたきっかけです。


choでは、関数の呼び出しを `(関数名 引数...)` の形で書きます。

```lisp
(filter (> $2 20))
(print $1 (s/upper $3))
```

awkでいう `awk '$2 > 20 {print $1, toupper($3)}'` に相当します。

choはスクリプトを書くほどではない処理を、ワンライナーで簡単に書けるようにするために作っています。
そのため、配列、ループ、ユーザー定義関数、代入、`BEGIN` / `END` ブロックは用意していません。こうした機能が必要になる処理には、awkやスクリプト言語を使うほうがよいと思っています。

出現頻度のカウントやソートなどは、素直に `sort` や `uniq` などの既存コマンドにお任せする方針にしています。

先ほどのログから、応答時間が100ミリ秒を超えたリクエストをサービスごとに数えるなら、こう書きます。

```console
$ cat timings.log | cho '(f (> $2 100)) (p $1)' | sort | uniq -c
      2 api
      1 worker
```


### 3. 日時やIPアドレスなどを、型に応じて比較・加工できる

ワンライナーでコマンドを組み立てようとするとき、文字列や数値だけでなく、日時やIPアドレス、URL、単位付きのファイルサイズなどを扱いたい場面がよく出てきます。

日時の比較ではタイムゾーンを考慮する必要がありますし、IPアドレスがCIDRで指定したネットワークに含まれるかは、文字列の比較だけでは判定できません。ライブラリを使えば処理できますが、ワンライナーで手軽に書きたいところです。

choには、こうした値を扱う型と組み込み関数があります。

- `Date` で、`2026-08-01` のような日付を比較したり、日数を加算したりする（`d/>=`、`d/add`）
- `DateTime` / `Duration` で、RFC 3339形式の日時を比較したり（`dt/>=`、`dt/<`）、差分を計算したりする（`dt/diff`）
- `ByteSize` で、`MB` や `MiB` などの単位が付いたサイズを比較したり、バイト数に変換したりする（`bs/>`、`bs/to-b`）
- `IpAddr` / `Cidr` で、IPv4・IPv6アドレスの正規化や、ネットワークに含まれるかの判定（`cidr/contains?`）をする
- `Url` で、ホスト名やパスを取り出す（`url/host`、`url/path`）
- `SemVer` で、セマンティックバージョニングに従ってバージョンを比較する（`semver/>`）
- `UUID` / `ULID` で、IDを生成したり（`uuid/v4`、`uuid/v7`、`ulid/new`）、UUID v1・v6・v7やULIDから日時を取り出したりする（`uuid/time`、`ulid/time`）

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

各フィールドは **文字列** として読み込まれ、関数が求める型に応じて自動的に変換されます。たとえば `cidr/contains?` に渡す `$1` はIPアドレスとして、`bs/>` に渡す `$2` はバイトサイズとして解釈されます。

型変換に失敗した場合はその時点でエラー終了します。不正な値を黙って読み飛ばし、ログの形式が崩れていることを見落とすのを防ぐためです。

たとえば、URLとして解釈できない文字列を `url/path` に渡すと、どのレコードのどの引数で失敗したかが表示されます。

```console
$ echo 'not-a-url' | cho '(url/path $1)'
cho: record 1: url/path: argument 1 expects Url (absolute URL), but "not-a-url" is not a valid absolute URL
```

この場合、終了コードは `1` になります。エラー時に代わりの値を使って処理を続けたいなら、`default` で明示できます。

```console
$ echo 'not-a-url' | cho '(default (url/path $1) "INVALID_URL")'
INVALID_URL
```


### 4. helpとaproposで、調べながら式を組み立てられる

ワンライナーを書いている途中で、関数名や引数の順番を調べるためにブラウザを開くのは面倒です。

choには、ターミナルで調べながら式を組み立てられるように、関数を検索する `--apropos` と、詳しい使い方を表示する `--help` があります。

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


## まとめ

- awkのような行指向（CSV/TSVにも対応）
- S式で書ける
- 日時・IP/CIDR・URL・バイトサイズなどをそのまま比較・加工できる豊富な関数
- ターミナルから離れずに使い方を調べられる `--help` / `--apropos`
- 複雑な機能は持たせず、UNIXパイプラインと組み合わせて使う設計

最初はawkっぽい自作Lispからスタートしましたが、awkとはまた違った便利なツールになったのではないかと思っています。

## インストール

```console
$ cargo install --git https://github.com/k0f1sh/cho.git
```

- [k0f1sh/cho (GitHub)](https://github.com/k0f1sh/cho)

