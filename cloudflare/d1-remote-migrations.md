# D1 のマイグレーションを本番に手動で適用する

Workers Builds でデプロイしていても、D1 のマイグレーションは自動では適用されない。`wrangler d1 migrations apply --remote` を手で実行したときだけ適用される。2026年11月に wrangler 4.158 で試した手順をまとめる。

## 手順

まず、未適用のマイグレーションを確かめる。

```sh
wrangler d1 migrations list <binding> --remote
```

`Migrations to be applied:` に出たものが、次の `apply` で適用される。

```sh
wrangler d1 migrations apply <binding> --remote
```

対話できない環境（AI エージェントのシェルなど）では、確認のプロンプトに `Using fallback value in non-interactive context: yes` と出て、そのまま適用される。確認なしで進むので、`list` で中身を見てから実行する。

## 列が追加されたかを確かめる

適用後にもう一度 `list` を実行すると `No migrations to apply!` になる。実際に列ができたかは、`sqlite_master` と `pragma_table_info()` を結合した SELECT で確かめられた。

```sh
wrangler d1 execute <binding> --remote --json --command \
  "SELECT m.name AS tbl, p.name AS col FROM sqlite_master m, pragma_table_info(m.name) p WHERE m.name IN ('<table1>','<table2>') AND p.name='<column>'"
```

`--json` を付けると、`results` に一致した表と列が並ぶ。付けないと統計の情報ばかりが表示されて、結果が見づらい。

`pragma_table_info()` は、使い方によっては D1 に拒否されたことがある。上の形の SELECT では通ったが、ほかの書き方で通るかは確かめていない。

## 7403 のエラーは一度再試行する

`migrations list --remote` が、一度だけ次のエラーで失敗した。

```text
The given account is not valid or is not authorized to access this service [code: 7403]
```

`wrangler whoami` ではトークンに `d1 (write)` のスコープがあり、同じコマンドをそのまま再実行したら通った。一時的なエラーだったと考えられる（原因は確かめていない）。`wrangler login` でトークンを作り直す前に、まず再試行する。

## プレビュー用の D1 も忘れない

Worker Previews でプレビュー専用の D1 を使っている場合、そちらにも同じマイグレーションを適用する必要がある。本番だけ適用すると、プレビューで新しい列を読むページが失敗する。プレビュー用の設定ファイルを分けているなら、`--config` で指定して適用する。

```sh
wrangler d1 migrations apply <binding> --remote --config <preview-config>.toml
```

プレビュー専用の D1 の作り方は [Cloudflare Worker Previews をプレビュー専用のリソースで使う](worker-previews.md) を参照。
