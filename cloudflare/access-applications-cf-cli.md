# Access のアプリケーションを cf コマンドで安全に書き換える

Cloudflare CLI（`cf`、2026年11月時点の v0.15.0）の `cf zero-trust access applications` で、
Access の SaaS アプリ（OIDC）の Redirect URL を足し、Self-hosted アプリを作り、古いものを消したときのメモ。

## 使えるサブコマンド

```sh
cf zero-trust access applications list
cf zero-trust access applications get <app-id>
cf zero-trust access applications create --body '<JSON>'
cf zero-trust access applications update <app-id> --body '<JSON>'
cf zero-trust access applications delete <app-id> --force
```

どれも `--dry-run` を付けると、送る HTTP メソッド・URL・本文を表示するだけで実行しない。

## 出力に個人情報が含まれる

`get`・`list` の結果には、アプリに付いたポリシーの `include` が含まれる。メールアドレスで許可していると、
許可した人のメールアドレスがそのまま出力される。ログや会話に残さないよう、jq で必要な項目だけを取り出す。

```sh
cf zero-trust access applications list \
  | jq -c '.[] | {id, name, type, domain, redirect_uris: .saas_app.redirect_uris}'
```

`get` の結果を丸ごと保存するときは、権限を絞ったファイルに書く。

```sh
cf zero-trust access applications get <app-id> > before.json
chmod 600 before.json
```

## update は PUT なので、get の結果から本文を作る

`update` は `PUT /accounts/<account>/access/apps/<app-id>` になる。送らなかった項目は消えるので、
`get` の結果から送る項目を選び、変えたい項目だけを書き換えた本文を作る。

SaaS アプリの Redirect URL を 1 つ足したときの例:

```sh
jq '(.result // .) | {
  name, type, logo_url, app_launcher_visible, allowed_idps,
  auto_redirect_to_identity, session_duration, tags,
  policies: [.policies[] | {id, precedence}],
  saas_app: (.saas_app | {
    auth_type, client_id,
    redirect_uris: (.redirect_uris + ["https://<新しいホスト名>/login/callback"]),
    app_launcher_url, grant_types, scopes, access_token_lifetime
  })
}' before.json > update.json
```

注意すること:

- `policies` は `get` ではポリシーの中身ごと返ってくるが、送るときは `{id, precedence}` の形にする。
  再利用できるポリシー（reusable）を参照する形になる。
- SaaS アプリの Client secret は `get` の結果に含まれない。`saas_app` に含めずに送っても、
  Client ID と secret は変わらなかった（更新後も同じ client_id のままで、ログインも通った）。
- `public_key` などの読み取り専用の項目は送らない。

## dry-run、更新、get で比べる

```sh
cf zero-trust access applications update <app-id> --body "$(cat update.json)" --dry-run
cf zero-trust access applications update <app-id> --body "$(cat update.json)"
cf zero-trust access applications get <app-id> > after.json
```

変えた項目と更新日時を除いて比べ、それ以外が変わっていないことを確かめる。

```sh
F='(.result // .) | del(.updated_at, .saas_app.updated_at, .saas_app.redirect_uris)'
jq -S "$F" before.json > a.json
jq -S "$F" after.json > b.json
diff a.json b.json && echo "変えた項目以外の差分なし"
```

zsh では `diff` が関数として定義されていて「function definition file not found」で失敗することがあった。
そのときは `command diff` を使う。

## Self-hosted アプリを、既存のものと同じ設定で作る

既存のアプリの `get` の結果から、ドメインとポリシーの参照だけを差し替えて `create` に渡す。

```sh
jq '(.result // .) | {
  name: "<新しい名前>", type, domain: "<新しいホスト名>",
  destinations: [{type: "public", uri: "<新しいホスト名>"}],
  app_launcher_visible, allowed_idps, tags, auto_redirect_to_identity, session_duration,
  enable_binding_cookie, http_only_cookie_attribute, options_preflight_bypass,
  policies: [.policies[] | {id, precedence}]
}' old-app.json > create.json
cf zero-trust access applications create --body "$(cat create.json)"
```

切り替えのあいだは古いアプリと並ぶので、名前を変えておくと取り違えにくい。

## delete には --force が要る

TTY のない環境で `delete` を実行すると、確認を求めて「Aborted.」で止まる。

```text
? This permanently deletes the resource. Continue?
  (non-interactive; pass --force to confirm)
Aborted.
```

`--force` を付けると消える。取り消せないので、直前に `get` で ID・名前・ドメインが合っているかを確かめる。

## Worker 単位の Access の対象を確かめる

Workers & Pages で Worker ごとに付けた Access は、Zero Trust のアプリとしても見える。`destinations` を見ると、
何を守っているかが分かる。

```sh
cf zero-trust access applications get <app-id> | jq -c '(.result // .) | {name, destinations}'
```

`destinations` が `[{"type":"preview_worker", ...}]` だけなら、守っているのはプレビューの URL だけで、
本番のカスタムドメインは対象外。2026年11月には、このアプリの名前が「<Worker 名> - Cloudflare Workers」から
「<Worker 名> (preview)」に変わっていた（自分では変えていない）。
