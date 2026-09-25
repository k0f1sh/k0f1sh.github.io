+++
title = "自作のミニ言語用Emacsモードを作り、シェルから呼び出す"
description = "choをシェル上で楽に書くためにcho-mode.elを作った話"
date = 2026-09-25
+++

[cho](../cho01/) は、Lisp風の構文でawkのような行指向のテキスト処理を行うコマンドラインツールです。

例えば、名前と `role=developer` の2列を受け取り、職種を取り出して大文字にする処理は、`cho`では次のように書けます。

```console
$ echo 'alice role=developer' | cho '(p $1 (s/upper (s/after $2 "=")))'
alice DEVELOPER
```

私が普段EmacsでLispを書くときは、Paredit、rainbow-delimiters、ElDocを使っています。

シェル上で入力する場合はそれらの拡張の支援がないので、コードが少し複雑になると辛いなと思うことがあります。
そこで、シェル上でEmacsをすぐに開いて編集できる仕組みを整えることにしました。そのために、choの式を含むシェルコマンド用のEmacsメジャーモード[`cho-mode`](https://github.com/k0f1sh/cho/tree/main/editors/emacs)も作り、 `.bashrc` に設定を追加しました。

![シェル上でEmacsを起動しcho-modeで編集する様子](/cho-mode.gif)

1. Bashの入力中にショートカットを入力し、入力中のコマンドをEmacsで開く
2. `cho-mode` で編集し、`C-c C-c` で保存してEmacsを終了する
3. 保存した内容をBashに戻す
4. 内容を確認し、Enterで実行する


## 実現方法

### 入力途中の内容をEmacsで開く

`.bashrc` に `bind -x` の設定を書き、`C-x C-l` を押すと入力中のコマンドをEmacsで開くようにしています。

`bind -x` から呼ばれる関数では、`READLINE_LINE` で入力中のコマンドを取得できます。この内容を一時ファイルに書き出し、cho用の設定を読み込んだEmacsで開きます。`cho-mode` で編集して正常終了した場合だけ、一時ファイルから編集結果を読み戻します。

```bash
__cho_edit_readline_line() {
    local cho_edit_file
    cho_edit_file=$(mktemp -t cho-readline.XXXXXX) || return
    printf '%s\n' "$READLINE_LINE" >"$cho_edit_file"

    if command emacs --init-directory "$HOME/.config/cho-emacs" -nw \
        "$cho_edit_file" --eval '(cho-mode)'; then
        IFS= read -r -d '' READLINE_LINE <"$cho_edit_file" || :
        READLINE_LINE=${READLINE_LINE%$'\n'}
        READLINE_POINT=${#READLINE_LINE}
    fi

    command rm -f -- "$cho_edit_file"
}
bind -x '"\C-x\C-l":__cho_edit_readline_line'
```

`~/.config/cho-emacs/init.el` には、例えば次のように設定します。`load-path` のパスは、手元にあるchoリポジトリの場所に合わせて変更してください。

`--init-directory` で専用の設定ディレクトリを使うため、普段のEmacsの設定や追加パッケージがそのまま使えるとは限りません。Pareditとrainbow-delimitersを使う場合は、この専用設定からも読み込めるように別途導入しておきます。`cho-mode` はこれらのパッケージが読み込める場合に自動で有効にします。ElDocはEmacsに標準で含まれています。

```elisp
(add-to-list 'load-path (expand-file-name "~/src/cho/editors/emacs"))
(require 'cho-mode)

(defun cho-edit-save-and-exit ()
  (interactive)
  (save-buffer)
  (kill-emacs 0))

(defun cho-edit-cancel ()
  (interactive)
  (set-buffer-modified-p nil)
  (kill-emacs 1))

(define-key cho-mode-map (kbd "C-c C-c") #'cho-edit-save-and-exit)
(define-key cho-mode-map (kbd "C-c C-k") #'cho-edit-cancel)
```

`C-c C-c` で保存して正常終了すると、編集結果がBashの入力に戻ります。`C-c C-k` で破棄して終了した場合は元の入力を保ちます。戻った内容を確認してからEnterで実行できます。


### cho-modeの中身

`cho-mode` はEmacsの `sh-mode` を継承しています。編集するのはchoの式だけではなく、`echo` やパイプを含むシェルコマンド全体だからです。その中から `cho '(p $1)'` のように、シングルクォートで囲まれ、`(` で始まるchoの式を見つけます。

(現在は通常の一行コマンドを主な対象としており、コマンド置換やヒアドキュメントなどの複雑なシェル構文は完全には解析していません。)

choの式では、関数名、`$1` などのフィールド参照、`NR`・`NF`、真偽値、正規表現に色が付きます。

通常、`sh-mode` ではシングルクォートの中身が文字列として扱われ、その中のかっこをS式の構造として認識できません。そこで、choの式を囲むクォートについて、[Emacsが構文を認識するための属性を変更しています](https://github.com/k0f1sh/cho/blob/d165b51cfaa457e1a1d53c21cff77be1738720f1/editors/emacs/cho-mode.el#L92-L103)。コマンドに書かれたクォート自体は残したまま、中のかっこをS式として扱えるようにすることで、Pareditによる編集やrainbow-delimitersによる色分けができるようになりました。

また、式の先頭で関数名を入力してTabを押すと、補完候補が出ます。ElDocでは、編集中の関数の引数や戻り値の型を確認できます。例えば、冒頭の `s/after` を書くときも、引数の順番を確認しながら入力できます。

### 言語の定義から補完データを作る

補完やElDoc用の情報はchoのリポジトリの[`metadata.json`](https://github.com/k0f1sh/cho/blob/main/metadata.json)にあります。
このJSONには、関数名や別名、引数と戻り値の型、説明が入っています。

choでは関数を追加するときに、ASTを組み立てる処理と、説明文・使用例を、同じマクロ呼び出し（[`define_callable!`](https://github.com/k0f1sh/cho/blob/d165b51cfaa457e1a1d53c21cff77be1738720f1/src/language/mod.rs#L221)）にまとめて定義します。

この定義からパーサー用とドキュメント用の実装を生成し、ドキュメント用の情報からヘルプと `metadata.json` を作っています。
新しく関数を作るときにドキュメントが漏れないようにこのような形にしてみました。

Emacs側では [`generate-cho-mode-data.el`](https://github.com/k0f1sh/cho/blob/main/editors/emacs/generate-cho-mode-data.el) がJSONを [`cho-mode-data.el`](https://github.com/k0f1sh/cho/blob/main/editors/emacs/cho-mode-data.el) に変換します。`cho-mode` はこの生成済みのElispを読み込むので、起動するたびに `metadata.json` を解析する必要はありません。

## まとめ

`cho-mode`を作り、シェル上で専用設定のEmacsを起動することで、普段と同じ感覚で編集できるようになりました。

`cho-mode` のコードと導入方法は[choリポジトリの `editors/emacs/`](https://github.com/k0f1sh/cho/tree/main/editors/emacs) に置いています。シェルから起動する設定は私の使い方の一例です。

この記事の設定は、Emacs 31.1、Bash 5.3.15で動作を確認しました。
