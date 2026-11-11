# RSC リクエストの `_rsc` 検証と CDN の Vary の関係

Next.js（App Router）のページを CDN でキャッシュするとき、RSC リクエスト（クライアント遷移やプリフェッチで送られるリクエスト）と通常の HTML リクエストのキャッシュが混ざらないようにする方法を調べた。2026年11月に Next.js 16.4.5 と Cloudflare で確かめた内容。

## 結論

- RSC 系のリクエストに `public` な `Cache-Control` を付けないようにする（`headers()` の `missing` 条件など）。この対策は CDN の Vary 設定では置き換えられない。
- 理由は、Next.js が `_rsc` クエリの検証に失敗したときに返す 307 に `Vary` ヘッダーが付かないため。
- 正規の RSC リクエストは `_rsc` クエリで URL が変わるので、Vary 設定がなくてもキャッシュキーは HTML と分かれる。

## `_rsc` クエリの仕組み

Next.js 16 では `experimental.validateRSCRequestHeaders` が既定で有効になっている。クライアントは次の4つのリクエストヘッダーの値からハッシュを計算し、`_rsc` クエリとして送る（`next/dist/shared/lib/router/utils/cache-busting-search-param.js`）。

- `next-router-prefetch`（無ければ `0`）
- `next-router-segment-prefetch`
- `next-router-state-tree`
- `next-url`

値を `,` でつないだ文字列の SHA-256 の先頭12バイトを base64url（末尾の `=` なし）にしたものが `_rsc` の値になる。4つとも無ければ（プリフェッチの値が `0` でもよい）値は空文字列になる。

確かめるときは、次のように期待値を計算して送ると `200 text/x-component` が返る。

```sh
H=$(printf "1,0,0,0" | openssl dgst -sha256 -binary | head -c 12 | base64 | tr "+/" "-_" | tr -d "=")
curl -sS -o /dev/null -D - -H "rsc: 1" -H "next-router-prefetch: 1" "https://<host>/<path>?_rsc=$H"
```

サーバー（`next/dist/server/base-server.js`）はこのハッシュを検証し、一致しなければ `307 Location: <path>?_rsc=<期待値>` を返す。この処理は `setVaryHeader()` より前にあるので、307 のレスポンスには `Vary` が付かない。

```sh
curl -sS -o /dev/null -D - -H "rsc: 1" "https://<host>/<path>"
```

実際に、200 の HTML には `Vary: rsc, next-router-state-tree, next-router-prefetch, next-router-segment-prefetch` が付く一方で、上のコマンドに対する 307 には `vary` の行がなかった。

## なぜ Vary 設定では防げないか

`headers()` で `/` などのパスに `public, s-maxage=...` を付けると、パスだけで判定されるので、`_rsc` 無しで `RSC: 1` を送ったリクエストへの 307 にも付いてしまう。CDN のキャッシュキーが URL だけなら、この 307 が `/` のキャッシュとして固定され、全員がリダイレクトされる。

Cloudflare の Cache Rules には、オリジン応答の `Vary` に列挙されたヘッダーをキャッシュキーに含める設定がある。しかし、これは応答に `Vary` があるときにしか働かない。307 には `Vary` が無いので、Vary 設定を入れても 307 は HTML と同じキーでキャッシュされる。したがって、RSC 系リクエストへの `public` な `Cache-Control` を止める対策が必要になる。

## Cloudflare Cache Rules の Vary 設定（参考）

2026年11月時点の公式ドキュメント（[Vary](https://developers.cloudflare.com/cache/concepts/vary/)、[Cache Rules settings](https://developers.cloudflare.com/cache/how-to/cache-rules/settings/)）で確かめた仕様。今回は採用しなかったので、実際の動作は試していない。

- 全プランで使える。
- ヘッダーごとに `normalize`（同じ意味の値をまとめる）、`passthrough`（値をそのまま使う）、`bypass`（そのヘッダーが `Vary` にあればキャッシュしない）を選ぶ。
- API では `action_parameters.vary` に書く。`default`（`headers` に無いヘッダーの扱い）は必須で、`headers` は小文字のヘッダー名を最大50件まで書ける。
- `Vary: *` の応答は設定にかかわらずキャッシュされない。
- 設定を変えてもキャッシュはパージされない。古いエントリが期限切れになるまでミスが増える。

Next.js は通常の応答に4つの RSC 系ヘッダーを常に `Vary` に入れるので、そのどれかを `bypass` にしたり、`headers` に書かずに `default` を `bypass` にしたりすると、すべての応答がキャッシュされなくなる。

## 注意: zsh で条件付きの curl オプションを組み立てない

調べている途中で、zsh で `curl ${h:+-H "$h"} ...` のようにヘッダーを条件付きで渡したところ、ヘッダーが意図どおりに送られていないような結果になった（`RSC: 1` を送ったはずのリクエストが HTML のキャッシュに HIT した）。bash で明示的に `-H` を書いて送り直すと 307 が返った。CDN の挙動を調べるときは、送ったヘッダーを `-v` で確認するか、明示的にオプションを書く。
