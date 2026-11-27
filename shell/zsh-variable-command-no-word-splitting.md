# zsh で、変数に入れたコマンドが「command not found」になる

2026年11月に確かめた。

## 何が起きるか

bash のつもりで次のように書くと、zsh では失敗する。

```sh
N="node --some-flag"
$N script.mts
# zsh: command not found: node --some-flag
```

zsh は既定で、変数を展開した結果を空白で単語に分割しない（`SH_WORD_SPLIT` が無効）。そのため、`node --some-flag` という名前のコマンドを 1 つ探してしまう。

## 対処

関数か配列にする。

```sh
N() { node --some-flag "$@"; }
N script.mts

args=(-F "name=a b" -F "x=1")
curl "${args[@]}" <url>
```

## ファイルの一覧を変数に入れたときも同じ

コマンドの出力を変数に入れ、そのまま別のコマンドに渡しても、zsh では分割されない。

```sh
files=$(git grep -l 'pattern' -- '*.css')
sed -i 's/a/b/' $files
# sed: <改行でつないだ全体> を読み込めません: そのようなファイルやディレクトリはありません
```

改行を含む 1 つの名前として渡されるため、ファイルが見つからない（2026年11月に確かめた）。一覧はパイプで `xargs` に渡すと、zsh か bash かを気にせずに済む。ファイル名に空白があってもよいよう、NUL 区切りにする。

```sh
git grep -l -z 'pattern' -- '*.css' | xargs -0 sed -i 's/a/b/'
```
