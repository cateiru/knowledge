# find の -path に角括弧を含むディレクトリ名を書くと、一致しない

## 症状

Next.js の動的ルートのような `[id]` を含むパスを、`find` の `-path` / `! -path` で指定すると、意図したとおりに一致しない。

```sh
# app/items/[id]/edit/page.tsx を除外したつもりが、除外されない
find "app/items/[id]" -path "*/edit/page.tsx" ! -path "app/items/[id]/edit/page.tsx"
```

2026年11月に、除外したつもりのファイルまで一括置換の対象になって気づいた。

## 原因

`-path` のパターンはシェルの glob と同じ規則で解釈される。`[id]` は「`i` または `d` の 1 文字」という文字クラスになり、文字どおりの `[id]` には一致しない。

`find` の起点（最初の引数）はパターンではないため、そちらは引用符で囲めば問題なく動く。そのため、起点は正しく動いて除外だけが効かず、気づきにくい。

## 対処

パターンの中の角括弧を `\` でエスケープする。

```sh
find "app/items/[id]" -path "*/edit/page.tsx" ! -path 'app/items/\[id\]/edit/page.tsx'
```

または、`find` では絞り込みすぎず、結果を `grep -v -F`（固定文字列として比べる）で除外する。

```sh
find "app/items/[id]" -path "*/edit/page.tsx" | grep -v -F "app/items/[id]/edit/page.tsx"
```

## 注意

- 一括置換の前に、対象のファイルの一覧を出して、除外したいファイルが含まれていないか目で確かめる。
- zsh では、引用符で囲まない `[...]` を含む引数自体もファイル名展開され、一致しないと `no matches found` で失敗する。引数は引用符で囲む。
