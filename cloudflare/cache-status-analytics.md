# ゾーンのキャッシュ状態をコマンドラインから集計する

Cloudflare のゾーンで、パスや応答の種類ごとにキャッシュの HIT / MISS / BYPASS の割合を集計した方法。2026年11月に、Free プランのゾーンと Cloudflare の CLI（`cf` v0.15.0）で確かめた。

## `cf analytics sql` ではキャッシュ状態が取れない

`cf analytics sql post` は `events.httpRequests` というデータセットに SQL を投げられる。しかし、このデータセットには `cacheStatus` の列が無い。

```sh
cf analytics sql introspection get --account-tag <account_id> \
  --dataset-name events.httpRequests --include-columns
```

ほかにも次の点でこの用途には向かなかった。

- `clientRequestQuery` の列は、読み取ると `not permitted to access column` の 403 になる。
- `originResponseStatus` は、オリジンが Worker のゾーンでは常に `0` だったので、オリジンに届いたかどうかの判定にも使えない。
- ClickHouse 風の関数（`multiIf`、`starts_with` など）は使えない。`CASE WHEN` と `LIKE` は使える。

`cf analytics query top-n` に HTTP リクエストのデータセット名を渡しても、500 エラーが返るだけだった。

## GraphQL の `httpRequestsAdaptiveGroups` を使う

`cf` には GraphQL のコマンドが無いので、GraphQL API（`https://api.cloudflare.com/client/v4/graphql`）を直接呼ぶ。`cf auth login` で取得した OAuth トークンは `~/.config/cloudflare/config/default.json` の `oauth_token` に入っていて、スコープに `analytics.read` が含まれていれば GraphQL も呼べた。

- トークンは1時間ほどで期限が切れる。`cf auth whoami` など何か1つ `cf` のコマンドを実行すると更新される。
- トークンを画面やログに出さないよう、スクリプトの中でファイルから読んで使う。

```python
import json, os, sys, urllib.request

tok = json.load(open(os.path.expanduser("~/.config/cloudflare/config/default.json")))["oauth_token"]
query = open(sys.argv[1]).read()
req = urllib.request.Request(
    "https://api.cloudflare.com/client/v4/graphql",
    data=json.dumps({"query": query}).encode(),
    headers={"Authorization": "Bearer " + tok, "Content-Type": "application/json"},
)
print(urllib.request.urlopen(req).read().decode())
```

応答の種類とキャッシュ状態ごとに数えるクエリの例。

```text
{ viewer { zones(filter: {zoneTag: "<zone_id>"}) {
  g: httpRequestsAdaptiveGroups(limit: 100,
    filter: {datetime_geq: "2026-11-03T00:00:00Z", datetime_lt: "2026-11-10T00:00:00Z", requestSource: "eyeball"},
    orderBy: [count_DESC]) {
    count
    dimensions { edgeResponseContentTypeName cacheStatus }
  }
} } }
```

`count` はサンプリングを補正した推定値になっている。

## 注意点

- **`clientRequestQuery` は Free プランでは使えない**: フィルターに使うと `does not have access to the field 'clientrequestquery'` というエラーになる。クエリ文字列で見分けたいリクエスト（Next.js の `_rsc` 付きリクエストなど）は、ほかの手がかりで見分ける必要がある。
- **RSC は `htc` で数えられる**: `edgeResponseContentTypeName` は拡張子に近い短い名前で返る。Next.js の RSC 応答（`text/x-component`）は `htc` になっていた。
- **行数の上限**: `limit` の上限は 10,000 行で、パスごとに分けると件数の多いゾーンでは打ち切られる。その場合、合計は下限として扱う。
- **ボット**: `requestSource: "eyeball"` で絞っても、`/admin` や `/login` などの存在しないパスへのスキャンが含まれる。パス別に見るときは除いて考える。
