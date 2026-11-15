# Next.js と @opennextjs/cloudflare は一緒に上げる

Cloudflare Workers で Next.js を動かすのに `@opennextjs/cloudflare`（OpenNext）を使っていると、Next.js だけを上げたときに本番が全ページ 500 になることがある。2026年10月に、Next.js 16.4.0 で起きた。

## 何が起きたか

Next.js を 16.3 系から 16.4.0 に上げると、本番で次のエラーが出て全ページが 500 になった。

```text
Unexpected loadManifest(/.next/server/preview-props.json)
```

いったん 16.3 系に戻し、OpenNext 1.20.12 が出てから、Next.js 16.4.2 と一緒に上げ直した。OpenNext 1.20.12 の `peerDependencies` は `next >=16.4.1` になっている。

## 注意

- Renovate は Next.js と OpenNext の更新を別々の PR にする。片方だけマージすると壊れるので、1 つの PR にまとめて上げる（Renovate の設定で同じグループにしておくとよい）。
- 上げる前に、手元で OpenNext のビルド（`opennextjs-cloudflare build` を呼ぶスクリプト）と `wrangler dev` を動かし、ページが 200 を返すことを確かめた。`next dev` だけでは Workers の上の挙動を確かめられない。
- OpenNext の `peerDependencies` に書かれた Next.js の範囲を、上げる前に確認する。

```sh
npm view @opennextjs/cloudflare@<version> peerDependencies
```
