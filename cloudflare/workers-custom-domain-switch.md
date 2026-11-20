# Workers のカスタムドメインを wrangler の routes で付け替える

Cloudflare Workers の本番のドメインを別のドメインに替えたときのメモ（2026年11月）。
アプリはメインの Worker（Next.js を OpenNext で動かしたもの）と、管理画面の Worker の 2 つ。
ログインは Cloudflare Access の SaaS アプリ（OIDC）で、管理画面は Access の Self-hosted アプリで守っている。

## wrangler の routes の custom_domain の挙動

`wrangler.toml`（または `wrangler.jsonc`）に次のように書くと、`wrangler deploy` のときにカスタムドメインが作られる。

```toml
routes = [{ pattern = "<新しいホスト名>", custom_domain = true }]
workers_dev = false
```

wrangler のソース（`publishCustomDomains`）を読んで確かめたこと:

- カスタムドメインは `PUT /workers/scripts/<名前>/domains/records` に `override_scope: true` を付けて送る。
  そのため、設定に書いていないカスタムドメインは、ダッシュボードで付けたものも含めて Worker から外れる。
  実際に、マージ後のデプロイで新しいドメインが付き、古いドメインはつながらなくなった。
- `routes` が空のときの `workers_dev` の既定値は `true` になる（`getSubdomainValues` の
  `defaultWorkersDev = routes.length === 0`）。ダッシュボードで workers.dev を無効にしていても、
  `routes` を書いていない Worker はデプロイのたびに workers.dev が有効に戻りうる。`workers_dev = false` を明示しておく。
- CI のような TTY のない環境では、ほかの Worker に付いているドメインや、ぶつかる DNS レコードを上書きする設定
  （`override_existing_origin`・`override_existing_dns_record`）が自動で `true` になる。

設定ファイルが正しく読めるかは、デプロイしなくても wrangler の API で確かめられる。

```sh
node -e '
const w = require("./node_modules/wrangler");
const c = w.unstable_readConfig({ config: "wrangler.toml" });
console.log(JSON.stringify(c.routes), c.workers_dev, c.preview_urls);'
```

`wrangler deploy --dry-run` は、OpenNext のビルド成果物（`assets.directory`）がないと失敗するので、この確認には使いにくい。

## 切り替えの順序

マージすると自動でデプロイされ、古いドメインはすぐ外れる構成だったので、次の順で進めた。

1. マージ前: ログイン用の SaaS アプリ（OIDC）の Redirect URL に、新しいホスト名の
   `https://<新しいホスト名>/login/callback` を足す。古い Redirect URL はまだ消さない。
2. マージ前: 管理画面の新しいホスト名の Self-hosted アプリを作り、管理者だけを許可するポリシーを付ける。
   デプロイしてから作ると、それまでの間に管理画面が誰にでも見えてしまうおそれがある。
3. マージして、両方の Worker のビルドが成功したことを確かめる。
4. 動作確認をする（下記）。
5. 古い Redirect URL と、古いホスト名の Self-hosted アプリを消す。SaaS アプリの `logo_url`・
   `app_launcher_url` も新しいホスト名に変える（早く変えすぎると、マージまで画像やリンクが切れる）。

Access の操作は Cloudflare CLI でもできる（[Access のアプリケーションを cf コマンドで安全に書き換える](access-applications-cf-cli.md)）。

## 動作確認

- 新しいホスト名の `/login` が、Access の画面を挟まずにアプリのログイン画面になること。
- 管理画面のホスト名が、Access のログイン画面へリダイレクトされること。リダイレクト先の URL に含まれる
  `kid` が、新しく作った Self-hosted アプリの `aud` と一致することも確かめた。
- 古いホスト名がつながらないこと（curl の結果が `000`）。
- Redirect URL が登録されているかは、Access の OIDC の authorization エンドポイントに curl で
  `redirect_uri` を渡すと分かる。登録済みなら `302` でログイン画面へ、未登録なら `400` が返った。

```sh
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://<チーム名>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<client_id>/authorization?client_id=<client_id>&redirect_uri=https%3A%2F%2F<新しいホスト名>%2Flogin%2Fcallback&response_type=code&scope=openid+email&code_challenge_method=S256&code_challenge=abc&state=x"
```

## ホスト名を変えると利用者がやり直すこと

ホスト名ごとに分かれるものは、すべてやり直しになる。

- ログイン（Cookie）
- PWA（ホーム画面への追加）
- Web Push の購読（Service Worker はオリジンごと）
- MCP の OAuth のトークン（resource が `https://<ホスト名>/mcp` に結び付いている）

## 新しいゾーンで確かめること

以前のゾーンでは、Bot Fight Mode が ChatGPT から MCP への接続（Dynamic Client Registration）を 403 で止めていた。
新しいゾーンでも状態を確かめる。今回のゾーンでは、もともとオフになっていた。

```sh
cf bot-management get --zone <ゾーン名>
```

返ってくる JSON の `fight_mode` が `false` ならオフ。
