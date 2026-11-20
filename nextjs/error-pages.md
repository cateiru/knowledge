# App Router で 404・500 のエラーページを作る

2026年11月に、Next.js 16.4（App Router）のアプリに 404 と、それ以外のエラーのページを作ったときのメモ。

## 置くファイル

- `app/not-found.tsx`: 存在しない URL と、`notFound()` を呼んだページで表示する。ルートレイアウトの中に表示され、ヘッダー・フッターは残る。
- `app/error.tsx`: ページの表示中の予期しないエラーで表示する。ルートレイアウトの中身だけを差し替える。
- `app/global-error.tsx`: ルートレイアウト自体のエラーで、ルートレイアウトの代わりに表示する。

`error.tsx` は同じ階層の `layout.tsx` を包まないので、ルートレイアウトで起きたエラーは `global-error.tsx` でしか受けられない。

## error.tsx の props は `retry`

Next.js 16.4 の `error.tsx`・`global-error.tsx` は、`reset` ではなく `retry` を使う。`retry` はエラーになった部分を取得し直して描き直す。`reset` も残っているが、取得し直さずに描き直すだけなので、ふつうは `retry` を使う。

`error` は `unknown` 型になっている。クライアントで `throw` された値は `Error` とは限らないためで、`digest`（サーバーのログと照合するための ID）は `instanceof Error` で絞ってから読む。

```tsx
"use client";

export default function ErrorPage({
  error,
  retry,
}: {
  error: unknown;
  retry: () => void;
}) {
  const digest =
    error instanceof Error && "digest" in error &&
    typeof error.digest === "string"
      ? error.digest
      : undefined;

  return (
    <main>
      <title>エラーが発生しました | サイト名</title>
      <h1>エラーが発生しました</h1>
      {digest ? <p>エラー ID: {digest}</p> : null}
      <button type="button" onClick={retry}>
        もう一度読み込む
      </button>
    </main>
  );
}
```

- エラーのページはクライアントコンポーネントなので `metadata` を export できない。タイトルは React の `<title>` を書く。
- ステータスコードは受け取れないため、404 以外は「500」としてまとめて表示した。

正確な仕様は、インストール済みの `node_modules/next/dist/docs/01-app/03-api-reference/03-file-conventions/error.md` で確かめた（Next.js は同梱のドキュメントがバージョンに合っている）。

## global-error.tsx は自前で html・body・スタイルを持つ

`global-error.tsx` はルートレイアウトの代わりに表示されるため、次のものを自分で用意する。

- `<html>` と `<body>`
- グローバルの CSS（`import "@/styles/globals.css"` など）
- テーマ。ルートレイアウトで cookie から `data-theme` を付けている場合、それは届かない。端末の設定に合わせる値（例: `data-theme="system"`）を付けておく

トップへのリンクは `next/link` ではなく通常の `<a href="/">` にして、ページ全体を読み込み直すようにした（壊れたルートレイアウトのままクライアント遷移させないため。これは判断であり、`next/link` で問題が起きることを確かめたわけではない）。

## next dev では error.tsx を VRT で撮りにくい

`next dev` で本当にエラーを起こすと、`error.tsx` の上に開発用のエラー表示（dev overlay）が重なる。VRT を `next dev` 相手に撮っている場合は、エラーのページの中身を部品に切り出し、その部品を固定の値（`digest` も固定）で描く開発用ページを作って撮ると安定する。

404 は `notFound()` で dev overlay が出ないので、存在しない URL を開いてそのまま撮れる。Playwright では、ステータスも確かめておくとよい。

```ts
const response = await page.goto("/no-such-page");
expect(response?.status()).toBe(404);
```

## 数字のイラストは文字ではなく図形で描く

「404」の大きな数字を SVG の `<text>` で描くと、端末のフォントによって形が変わり、VRT の結果も環境に左右される。数字を `<path>`・`<ellipse>` で描いた。

- 「0」は外側と内側の楕円を重ねて描くと、内側の楕円で `clipPath` を作れる。穴の中に置いた絵を穴の形で切り抜き、そのあと縁の線をもう一度上から描くと、穴の奥にあるように見える。
- 「4」の三角の穴は、外形と穴を 1 つの `path` に書いて `fill-rule="evenodd"` でくり抜く。

## Biome の useIterableCallbackReturn

数字ごとに `switch` で描き分ける処理を `map` のコールバックに書くと、すべての場合を網羅していても、Biome の `lint/suspicious/useIterableCallbackReturn` が「値を返さない経路がある」と指摘した。`switch` を部品（コンポーネント）に切り出して、`map` からはその部品を返すようにすると通った。
