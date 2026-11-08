# Cloudflare Worker Previews をプレビュー専用のリソースで使う

Workers Builds（Git 連携）のブランチのプレビューは、`wrangler versions upload` の方式から
[Worker Previews](https://developers.cloudflare.com/workers/previews/)（`wrangler preview`）の方式に変わった。
既存の Worker を Worker Previews に移す準備をしたときに分かったことをまとめる（2026年11月時点。
`wrangler preview` はオープンベータ）。

ドキュメントと wrangler 4.157.0 で確かめた内容と、まだ試していない内容を分けて書く。

## 前提

- wrangler 4.135.0 以上が必要。プロジェクトの依存の wrangler が使われるので、`package.json` の指定も上げておく。
- 既存の Worker は、ダッシュボードで一度切り替えるまで以前の方式のまま動く。**この切り替えは元に戻せない。**
- 新しく作った Worker は、最初から Worker Previews を使う。

## プレビューは本番の設定を引き継がない

いちばん大事な違いは、プレビューが本番の vars・バインディング・シークレットを引き継がないこと。
使うものは、wrangler の設定の `previews` ブロックに書き直す。`previews` ブロックは空でもよいが、
ないと `wrangler preview` が失敗する。

- `assets`・`compatibility_date`・`compatibility_flags` はトップレベルにだけ書く。プレビューにも同じ値が使われる。
- `observability`・`placement`・`limits` はトップレベルの値が使われる。プレビューだけ変えたいときに `previews` に書く。
- vars・D1・R2・KV などは `previews` に書く。同じバインディング名で、プレビュー用のリソースを指定する。
- Cron Triggers はプレビューでは動かない（`scheduled` が呼ばれない）。`previews` にも書かない。
- Workflows は、バインドしても既存の Workflow を共有する。プレビュー専用の Workflow は作られない。
  本番の Workflow のインスタンスを作ってしまわないよう、プレビューで要らなければバインドしない。
- Durable Objects は、プレビューごとに新しい namespace が作られる。クラスとマイグレーションはトップレベルに残す。

TOML では次のように書く。ドキュメントの例は jsonc だけだったが、TOML で書いても
`wrangler types` で警告は出なかった。

```toml
# トップレベル（本番）
preview_urls = true

[[d1_databases]]
binding = "DB"
database_name = "<本番の DB 名>"
database_id = "<本番の DB の ID>"
migrations_dir = "drizzle"

# プレビュー
[previews]

[previews.vars]
SOME_FLAG = "true"

[[previews.d1_databases]]
binding = "DB"
database_name = "<プレビュー用の DB 名>"
database_id = "<プレビュー用の DB の ID>"
migrations_dir = "drizzle"

[[previews.r2_buckets]]
binding = "MEDIA_BUCKET"
bucket_name = "<プレビュー用のバケット名>"
```

TOML では、`preview_urls` のようなトップレベルのキーを、最初のテーブル（`[...]`）より前に書く必要がある。

## プレビューだけで機能を切り替える

Next.js（OpenNext）のようにビルドしてから載せる構成では、プレビューも本番ビルドになる。
そのため、`process.env.NODE_ENV` ではプレビューと本番を区別できない。

プレビューだけで有効にしたい機能（たとえば、ログインを飛ばすデバッグ用の機能）は、
`[previews.vars]` に置いた実行時の変数で切り替える。

- フラグは `[previews.vars]` にだけ書き、トップレベルの `[vars]` やシークレットには書かない。
- 値は `=== "true"` のように厳密に比べる。
- 認証を飛ばすような機能なら、名前に `DANGEROUSLY_` を付けて、危ないことが分かるようにする。
- プレビューの URL を Cloudflare Access などで保護していることが前提になる。
  新しいプレビュー URL（`<プレビュー名>-<Worker 名>.<サブドメイン>.workers.dev`）にアカウント全体の
  Workers の Access がかかるかどうかは、ドキュメントからは読み取れなかった。最初のプレビューで、
  ログインしていないブラウザから開いて確かめる（未確認）。

## プレビュー用 D1 のマイグレーション

`wrangler d1 migrations apply` は、トップレベルの `d1_databases` しか見ない。そのため、
`previews.d1_databases` に書いた DB にはそのままでは適用できない。

ドキュメントのとおり、プレビュー用の DB をトップレベルに書いた別の設定ファイルを作り、`--config` で指定する。

```toml
# wrangler.preview-migrations.toml
name = "<Worker 名>"
compatibility_date = "<日付>"

[[d1_databases]]
binding = "DB"
database_name = "<プレビュー用の DB 名>"
database_id = "<プレビュー用の DB の ID>"
migrations_dir = "drizzle"
```

```sh
npx wrangler d1 migrations list DB --remote --config wrangler.preview-migrations.toml
npx wrangler d1 migrations apply DB --remote --config wrangler.preview-migrations.toml
```

2つのファイルの DB は必ず揃える。`migrations list` で、未適用の一覧がプレビュー用の DB のものになっていることを確かめてから適用する。

## Previews Base（ダッシュボードのひな形）に注意する

`wrangler preview` は、既定でダッシュボードの Previews Base（プレビューのひな形）の設定も合わせて使う。
ダッシュボードには本番の設定をインポートする機能がある。ここに本番の D1・R2 が入ると、
プレビューから本番のデータに触れてしまうおそれがある。

設定をすべて wrangler の設定ファイルに書いているなら、`--ignore-base-config` を付けて
ファイルの `previews` だけを使うようにするとよい（実際のビルドではまだ試していない）。

```sh
npx wrangler preview --ignore-base-config
```

シークレットは設定ファイルに書けないため、必要なら次のコマンドで設定する。Previews Base のシークレットは、
新しく作るプレビューにだけ反映される。`--ignore-base-config` を付けると、Previews Base のシークレットも使われなくなる点に注意する。

```sh
# すべての新しいプレビュー用
npx wrangler preview base-config secret put <SECRET_NAME>

# 1つのプレビュー用（--name を省略すると今のブランチ名）
npx wrangler preview secret put <SECRET_NAME> --name <プレビュー名>
```

## preview_urls はデプロイのときだけ反映される

`preview_urls = true` は `wrangler deploy`（本番のデプロイ）のときに反映され、`wrangler preview` では反映されない。
本番を一度もデプロイしていないときは、ダッシュボードの Domains → Worker URL で Preview をオンにする。

## 切り替えの順序

既存の Worker では、次の順で進める予定にした（切り替えそのものはまだ試していない）。

1. `previews` ブロックと `preview_urls = true` を含む変更を本番にデプロイする。
2. ダッシュボードで Worker Previews に切り替え、本番以外のブランチのデプロイコマンドを `npx wrangler preview` にする。
   この設定はブランチごとにある。
3. 最初のプレビューのビルドログで、バインディングがプレビュー用のリソースを指していることを確かめる。
4. ログインしていないブラウザでプレビューの URL を開き、アクセス制限がかかっていることを確かめる。
