# Workers Builds で pnpm モノレポのサブパッケージの Worker をデプロイする

pnpm ワークスペースのモノレポで、リポジトリのルートとは別に `packages/<name>` に置いた
2つ目の Worker を、Workers Builds（Git 連携）で自動デプロイし、Worker Previews でプレビューも
作れるようにしたときに分かったことをまとめる（2026年11月時点、wrangler 4.157.0）。

ドキュメントと wrangler の実装で確かめた内容と、まだ試していない内容を分けて書く。
Worker Previews そのものの注意点は [Cloudflare Worker Previews をプレビュー専用のリソースで使う](worker-previews.md) を参照。

## 1つのリポジトリに複数の Worker をつなぐ

ドキュメントの手順は、Worker ごとに同じリポジトリをつなぎ、Root directory をそれぞれのパスにするというもの。

- Root directory：`wrangler.jsonc` がある場所で、ビルドとデプロイのコマンドもここで実行される。
  サブパッケージの Worker なら `packages/<name>` にする。
- リポジトリのルートを Root directory にすると、ルートの wrangler の設定（別の Worker の `name`）が
  読まれてしまう。

## Build watch paths

変更されたファイルで、ビルドするかどうかを Worker ごとに絞れる（Settings → Build → Build watch paths）。

- 既定は Include が `*`（すべて）、Exclude が空。
- `*` は0文字以上に一致する。ドキュメントでは、ルールの先頭か末尾に置く例（`packages/<name>/*`）しか
  説明されていない。途中に置いたときの動きは書かれていない。
- push に含まれる各パスについて、先に Exclude を評価し、残ったパスを Include と照合する。
  1つでも一致すればビルドする。
- 変更ファイルが0件、3,000件以上、またはコミットが20件以上の push は、ルールにかかわらずビルドする。
- 本番のブランチとプレビューのブランチの両方に効くかは、ドキュメントからは読み取れなかった（未確認）。

サブパッケージの Worker では、Include を次のようにした。

```text
packages/<name>/*
pnpm-lock.yaml
pnpm-workspace.yaml
```

ルートの `package.json` は、PR ごとにバージョンを上げる運用だと毎回ビルドが走るので含めなかった。
ルートの Worker のほうは、Exclude に `packages/<name>/*` を入れると、サブパッケージだけを変えた
push でビルドが走らなくなる。

## 依存のインストール

Workers Builds は依存を自動でインストールする。ドキュメントで確かめられたのは次のことだけ。

- 使うパッケージマネージャーはリポジトリの lockfile で決まる。
- pnpm のバージョンは、Root directory の `package.json` の `packageManager` を読み、Root directory が
  リポジトリのルートでなければ、ルートの `package.json` の `packageManager` も読む（両方書くなら同じ値にする）。
- Build variables に `SKIP_DEPENDENCY_INSTALL` を `1` か `true` で設定すると、自動インストールをやめられる。

Root directory に lockfile がない（ワークスペースのルートにしかない）ときに、どこの lockfile を見て
何をインストールするのかはドキュメントに書かれていなかった。そこで、自動インストールをやめて、
ビルドコマンドで明示的にインストールする設定にした（実際のビルドではまだ試していない）。

```text
Build variables:  SKIP_DEPENDENCY_INSTALL = true
Build command:    pnpm install --frozen-lockfile
Deploy command:   npx wrangler deploy
Non-production branch deploy command:  npx wrangler preview --ignore-base-config
```

pnpm はサブディレクトリで実行してもワークスペースのルートを見つけ、ワークスペース全体をインストールする。
最初のビルドのログで、pnpm でインストールされていることを確かめる。

## Worker Previews に必要な wrangler のバージョン

Worker Previews には wrangler 4.135.0 以上が必要で、ビルドではプロジェクトの依存の wrangler が使われる。
サブパッケージ側の `package.json` の指定が古いままになっていないか確かめる。

`wrangler preview` には `--dry-run` がない。設定が読めるかどうかは `wrangler deploy --dry-run` で
確かめられるが、表示されるのは本番のバインディングだけなので、プレビューのバインディングは
最初のプレビューのビルドのログで確かめる。

## 本番を workers.dev に出さずにプレビューだけ作る

カスタムドメインだけで公開していて、`workers_dev: false`・`preview_urls: false` にしている Worker でも、
`workers_dev: false`・`preview_urls: true` の組み合わせにできる。プレビューの URL は
`<プレビュー名>-<Worker 名>.<サブドメイン>.workers.dev` になる。

ただし、`preview_urls: true` にすると、バージョンごとの URL（本番と同じバインディングで動く）も作られる。
wrangler も、この組み合わせのときに「Worker を意図せず公開しないよう、Preview URLs も無効にしたほうがよい」と
警告を出す（対話的でない環境では出ない）。

アプリの中で認証しない管理画面のような Worker では、URL を有効にする前に Access で保護しておく。

- アカウント全体の Workers の Access（Protect all Workers）は、ほかの Worker の利用者も許可している
  ことがあるので、それだけに頼らない。
- Worker 単位の Access（Worker の Access タブ → Protect this Worker behind Access → All traffic）に、
  その Worker を使ってよい人だけを許可するポリシーを付ける。Worker 単位の Access は、Worker の routes・
  カスタムドメイン・workers.dev・プレビューをまとめて保護する。
- ダッシュボードで選べるポリシーは Cloudflare account と Email domain だけ。細かく決めたいときは、
  作られた Access アプリケーションを Zero Trust で開いてポリシーを付け替える。

`preview_urls` は `wrangler deploy` のときだけ反映されるので、Git 連携なら本番のブランチに
マージした時点で URL が有効になる。**Access の設定はマージより前に済ませておく。**

## Access の優先順位

より具体的な設定が先に評価される。

1. ホスト名やパスで作った Self-hosted アプリケーション
2. Worker 単位の Access
3. アカウント全体の Workers の Access

具体的な設定を消すと、その下の広い設定が見えるようになる点に注意する。
カスタムドメインに Self-hosted アプリケーションを作っていれば、本番のドメインはそちらが優先され、
workers.dev のプレビューやバージョンの URL は Worker 単位の Access で守られる。

## 確かめること

URL を有効にしたら、次を確かめる（まだ試していない）。

- ログインしていないブラウザで、本番・プレビュー・デプロイの URL を開くと Access のログイン画面が出る。
- 許可していないアカウントで認証すると拒否される。
- プレビューのビルドログで、バインディングがプレビュー用のリソースを指している。
