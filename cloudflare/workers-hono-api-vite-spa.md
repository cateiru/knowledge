# Hono の API と Vite の React SPA を 1 つの Worker で配信する

2026年11月に、Hono の JSX でサーバーサイドレンダリングしていた管理画面を、「Hono は JSON の API だけ、画面は Vite でビルドした React の SPA」に作り替えたときのメモ。Worker は 1 つのまま、画面は Workers の静的アセット（Static Assets）として配信する。

## 構成

- `src/app.ts`：Hono。`/api/*` だけを持つ
- `src/client/`：React の SPA。`vite build` で `dist/client` に書き出す
- Worker のビルドは今までどおり wrangler（esbuild）が行い、Vite は画面だけをビルドする

`@cloudflare/vite-plugin` で Worker も Vite でビルドする方法もあるが、`wrangler preview`（Worker Previews）や既存の `wrangler.jsonc` の設定がそのまま使えるかを確かめられなかったため、Worker のビルドは wrangler に残した。

## wrangler.jsonc

```jsonc
{
  "main": "src/index.ts",
  "assets": {
    "directory": "./dist/client",
    "binding": "ASSETS",
    // 静的アセットにないパス（/users など）は index.html を返し、SPA のルーターに任せる
    "not_found_handling": "single-page-application",
    // すべてのリクエストを Worker が受ける
    "run_worker_first": true
  }
}
```

`run_worker_first` を `["/api/*"]` にすれば API だけを Worker に通せるが、その場合 HTML などの静的アセットは Worker を通らず、Hono の `secureHeaders()` のヘッダー（`X-Frame-Options` など）が付かない。管理画面のように埋め込まれたくないページでは、`true` にして Worker から静的アセットを返すほうが、ヘッダーを 1 か所で管理できる。

`wrangler.jsonc` を変えたら `wrangler types` で型を作り直す。`Env` に `ASSETS: Fetcher` が増える。

## Worker 側（Hono）

```ts
const app = new Hono<{ Bindings: Env }>();
app.use(secureHeaders());
app.route("/api", api);
// 存在しない API は index.html ではなく 404 にする
app.all("/api/*", (c) => c.json({ error: "見つかりません" }, 404));
app.all("*", async (c) => {
  const response = await c.env.ASSETS.fetch(c.req.raw);
  // 静的アセットの応答のヘッダーは変更できないので、作り直してからミドルウェアにヘッダーを付けさせる
  return new Response(response.body, response);
});
```

注意すること:

- `app.route("/api", api)` で付けた子アプリの `api.notFound()` は、親の `app.all("*")` が後ろにあると呼ばれず、`/api/missing` が index.html（200）になった。親に `app.all("/api/*", …)` を置いて 404 にする
- `ASSETS.fetch()` の `Response` はそのまま返すとヘッダーが変更できないため、`new Response(body, response)` で作り直す
- `not_found_handling: "single-page-application"` は `ASSETS.fetch()` 経由でも効き、`/users` に index.html が返る（`wrangler dev` で確かめた）

## ビルドとデプロイ

`package.json` の scripts で、wrangler の前に `vite build` を実行する。

```json
{
  "scripts": {
    "dev": "vite build && wrangler dev",
    "dev:client": "vite",
    "deploy": "vite build && wrangler deploy",
    "preview": "vite build && wrangler preview --ignore-base-config"
  }
}
```

- Workers Builds のデプロイコマンドが `pnpm run deploy`・`pnpm run preview` なら、Workers Builds 側の設定を変えずに画面もビルドされる
- `wrangler preview`（Worker Previews）でも、トップレベルの `assets` が使われ、静的アセットがアップロードされた。`previews` の設定には `assets` の項目がない（wrangler 4.164 の設定スキーマで確認）
- `dist/` は `.gitignore` に入れる
- `wrangler dev` は起動時に 1 回ビルドしたものを配るだけなので、画面を直しながら見るときは `vite` の開発サーバーを別に起動し、`server.proxy` で `/api` を `wrangler dev` に送る

```ts
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: { outDir: "dist/client", emptyOutDir: true },
  server: { proxy: { "/api": "http://localhost:8787" } },
});
```

`package.json` に `"type": "module"` がないと、Vite 8 が `vite.config.ts` を CommonJS として読もうとして警告を出す。

## 型チェックを Worker と画面で分ける

Worker の型（`wrangler types` が作る `worker-configuration.d.ts`）と DOM の型は同じ名前のものが多く、1 つの tsconfig にまとめられない。Vite のテンプレートと同じく、参照だけの `tsconfig.json` から 2 つの設定を `tsc -b` で確かめる。

```jsonc
// tsconfig.json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.worker.json" },
    { "path": "./tsconfig.client.json" }
  ]
}
```

```jsonc
// tsconfig.worker.json（画面のディレクトリを除く）
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.worker.tsbuildinfo",
    "lib": ["ES2022"],
    "noEmit": true
    // ...
  },
  "include": ["src/**/*.ts", "worker-configuration.d.ts"],
  "exclude": ["src/client"]
}
```

```jsonc
// tsconfig.client.json
{
  "compilerOptions": {
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.client.tsbuildinfo",
    "lib": ["dom", "dom.iterable", "ES2022"],
    "jsx": "react-jsx",
    "noEmit": true,
    "types": ["vite/client"]
  },
  "include": ["src/client"]
}
```

- TypeScript 5.6 以降は、`noEmit` でも `tsBuildInfoFile` を置けば `tsc -b` で確かめられる
- 画面から import したファイルは画面の設定でも型チェックされる。API の応答の型など、画面と Worker で共用するファイルは、D1・R2 などの Workers の型に依存させない（依存すると、画面の設定で `D1Database` が見つからないエラーになる）
- API の JSON では `Date` が文字列になる。共用の型では `type Json<T> = T extends Date ? string : …` のような型で、サーバーの値を JSON にした型を作った

## 書き込みの CSRF 対策

Hono の `csrf()` ミドルウェアは、Content-Type がフォームのもの（`application/x-www-form-urlencoded`・`multipart/form-data`・`text/plain`）のときだけ `Origin` を確かめる（hono 4.13 の実装で確認）。JSON や本文のない `DELETE`・`POST` は確かめずに通す。

本文のない `POST` は、ほかのサイトからでも `fetch(…, { method: "POST", mode: "no-cors" })` で Cookie 付きで送れる。認証を Cookie（Cloudflare Access など）に任せている API では、GET・HEAD 以外のすべてのリクエストで `Origin` を自分で確かめるほうが安全。ブラウザは GET・HEAD 以外のリクエストには、同じオリジンでも `Origin` を付ける。

```ts
app.use("/api/*", async (c, next) => {
  const method = c.req.method;
  if (
    method !== "GET" &&
    method !== "HEAD" &&
    c.req.header("Origin") !== new URL(c.req.url).origin
  ) {
    return c.json({ error: "ほかのサイトからの送信は受け付けません" }, 403);
  }
  await next();
});
```

## テスト

- Worker のテストは `createApp().request(url, init, env)` で、`env.ASSETS` に `fetch` だけを持つ偽物を渡す
- Vitest の `test.projects` で、Worker（`environment: "node"`）と画面（`environment: "jsdom"`）を分けて実行した
- 画面のテストで現在の URL を確かめるために `useLocation()` を描く要素を置くなら、`<output>` は使わない。`<output>` は role が `status` なので、`getByRole("status")` が複数の要素に当たって失敗した

## 一覧と詳細の両方に同じ書き込みの操作を置く

2026年11月に、一覧ページにあった書き込みの操作（権限の変更・メンバーの追加と削除など）を、あとから足した詳細ページ（`/users/:id` など）にも置いた。

- フォーム・確認のダイアログ・結果の表示は、ページから切り出して一覧と詳細で共用する。移動先だけは使う側が決められるよう、操作が終わったら `onChanged(ok)` のようなコールバックを呼ぶ形にする
- 書き込みの結果は、一覧と同じクエリ（`?roleChange=updated&user=<id>` など）を詳細の URL に付けて表示する。結果の表示の部品は「クエリの ID に一致する項目を、渡された一覧から探す」形にしておくと、詳細では 1 件だけの配列を渡せば同じ部品が使える
- 同じ URL を開けば結果の表示を再現できるので、VRT（Playwright）では書き込まずに `?roleChange=updated...` を開いて撮れる
- 詳細の API が 404 を返したときは、`useApi` のエラーの状態を見て「見つからない」ページを描く。読み込み中は `aria-busy` を出しておき、テストと VRT はそれが消えるのを待つ
- 画面のテストで、パンくずのリンクとナビゲーションのタブが同じ名前（「ユーザー」など）だと `getByRole("link", { name })` が 2 つに当たる。`within(getByRole("navigation", { name: "パンくずリスト" }))` のように範囲を絞る
