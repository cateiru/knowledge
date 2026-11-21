# PR の変更検知で、version だけを上げた package.json を除く

## 状況

GitHub Actions で、PR で変わったファイルを見て一部のジョブ（管理画面の VRT など）を動かすかどうかを決めていた。
依存関係の更新でも見た目が変わりうるため、パターンに `package\.json$` を入れている。

```sh
PATTERN='^(packages/admin/|package\.json$|pnpm-lock\.yaml$)'
FILES="$(gh api --paginate "repos/${GITHUB_REPOSITORY}/pulls/${PR_NUMBER}/files" \
  --jq '.[].filename')"
if grep -qE "$PATTERN" <<<"$FILES"; then
  echo "run=true" >> "$GITHUB_OUTPUT"
fi
```

ところが、そのリポジトリでは「ユーザーに見える変更をした PR では、リリースノートに合わせて `package.json` の `version` を上げる」という決まりがあった。
そのため、対象と関係のない PR でも `package.json` に一致して、毎回ジョブが動いていた。

## 対処

PR のファイル一覧の API（`pulls/<番号>/files`）は、ファイルごとの差分を `patch` に入れて返す。
`package.json` の差分の変更行（`+`・`-` で始まる行）がすべて `"version":` の行なら、一覧から除く。

```yaml
- name: Detect changes
  id: filter
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    PR_NUMBER: ${{ github.event.pull_request.number }}
    FILES_JQ: >-
      .[] | if .filename == "package.json" and .patch != null
      and ([.patch | split("\n")[] | select(test("^[-+]"))]
      | length > 0 and all(test("^[-+]\\s*\"version\":")))
      then empty else .filename end
  run: |
    set -euo pipefail
    FILES="$(gh api --paginate "repos/${GITHUB_REPOSITORY}/pulls/${PR_NUMBER}/files" \
      --jq "${FILES_JQ}")"
    # 以降は今までどおり grep -qE "$PATTERN" で判定する
```

注意すること:

- `patch` には `---`・`+++` の見出しが含まれない（`@@` のハンクと文脈の行から始まる）ので、`^[-+]` で変更行だけを取り出せる。
- 差分が大きすぎると、API は `patch` を返さない（`null`）。そのときは中身が分からないので、除かずに一覧に残す（`.patch != null` の条件）。
- `all` は空の配列に対して `true` を返す。変更行が 0 行のとき（名前の変更だけなど）に誤って除かないよう、`length > 0` も確かめる。
- ルートの `package.json` だけを対象にしている。`packages/<pkg>/package.json` のようなサブパッケージの変更はそのまま残る。

## jq の式を env に置く理由

jq の式には `"` と `\` が多く、`run:` の中のシェルの引用符に直接書くと、エスケープが二重になって読みにくい。
`env:` に YAML のブロックスカラー（`>-`）で書くと、YAML はブロックスカラーの中のバックスラッシュを処理しないので、jq に渡したい文字列をそのまま書ける。

- `split("\n")` の `\n` は、YAML ではなく jq が改行として解釈する。
- 正規表現の `\s` は jq の文字列の中なので `\\s` と書く。
- `>-` は改行を空白に折りたたみ、末尾の改行を落とす。jq の式は複数行に分けて書いても 1 行として渡る。

YAML として読んだときの実際の文字列は、次のように確かめた。

```sh
python3 -c 'import yaml; d = yaml.safe_load(open(".github/workflows/ci.yml")); print(repr(d["jobs"]["<job>"]["steps"][0]["env"]["FILES_JQ"]))'
```

## 確かめたこと

2026年11月に、YAML から取り出した式を過去の PR のファイル一覧に当てて確かめた。

```sh
gh api --paginate "repos/<owner>/<repo>/pulls/<番号>/files" --jq "$JQ"
```

- `version` だけを上げた PR: `package.json` が一覧から除かれた。
- 依存関係を変えた PR: `package.json` が一覧に残った。
- サブパッケージの `package.json` を変えた PR: そのまま残った。

ワークフローの変更自体（`ci.yml`）もパターンに含めている場合、この変更を入れる PR ではジョブが動く。
