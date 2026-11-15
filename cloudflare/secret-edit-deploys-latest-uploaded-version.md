# ダッシュボードでシークレットを変えると、プレビューのコードが本番に出ることがある

Workers Builds でプレビュー（main 以外のブランチ）もビルドしている Worker で、ダッシュボードからシークレットを追加・変更したところ、プレビューのコードが本番にデプロイされた。2026年10月に起きた。

## 何が起きたか

- Workers Builds の非本番ブランチのビルドは、本番と同じ Worker に新しい version を upload する（デプロイはしない）。
- ダッシュボードでシークレットを編集すると、新しい version が作られてそのままデプロイされる。このとき元になるのは「いまデプロイされている version」ではなく、「最後に upload された version」のコードだった。
- そのため、本番で問題を直してデプロイしたあとでシークレットを編集したら、依存を上げたプレビューのブランチ（Renovate の PR）のコードが本番に出て、全ページが 500 に戻った。

## 確かめ方

本番で動いている version のコードがどのビルドのものかは、スクリプトの etag で比べられる。

```sh
wrangler deployments list
wrangler versions view <version-id> --json
```

`--json` の出力の `.resources.script.etag` を、main のビルドで upload された version の etag と比べる。違っていれば、ほかのブランチのコードが出ている。

## 対策

- シークレットを変えたら、デプロイされた version の etag を必ず確かめる。
- 違うコードが出ていたら、main のコードをデプロイし直す。`wrangler secret put` でも同じことが起きるかは確かめていないので、こちらで設定した場合も etag を確かめる。
- プレビューを本番と別の Worker に分けている場合（Worker Previews でプレビュー専用の設定にしている場合など）は、この競合は起きにくいと考えられる（確かめてはいない）。[Cloudflare Worker Previews をプレビュー専用のリソースで使う](worker-previews.md) も参照。
