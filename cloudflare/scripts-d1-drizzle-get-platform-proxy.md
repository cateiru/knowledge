# 運用スクリプトから D1 を Drizzle で読み書きする（wrangler d1 execute をやめる）

2026年11月に、wrangler 4.163 と Drizzle ORM 0.45 系で試した。

## なぜ

`wrangler d1 execute --command "<SQL>"` はバインドパラメーターを渡せないため、値を SQL の文字列に埋め込むことになる。エスケープを間違えると SQL インジェクションになりうる。

wrangler の `getPlatformProxy` を使うと、Node.js のスクリプトから本物の D1 バインディング（`D1Database`）を受け取れる。それを `drizzle(d1, { schema })` に渡せば、アプリと同じスキーマで、値をすべてバインドパラメーターにして読み書きできる。

## 書き方

```ts
import { mkdtempSync, rmSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import path from "node:path";
import { drizzle } from "drizzle-orm/d1";
import { getPlatformProxy, unstable_readConfig } from "wrangler";

const config = unstable_readConfig({ config: "wrangler.toml" });
const binding = config.d1_databases.find((d1) => d1.binding === "DB");

// --remote ではバインディングに remote: true が必要なので、DB だけを持つ設定を一時的に書き出す
const dir = mkdtempSync(path.join(tmpdir(), "scripts-"));
const configPath = path.join(dir, "wrangler.json");
writeFileSync(configPath, JSON.stringify({
  name: "<任意の名前>",
  compatibility_date: config.compatibility_date,
  d1_databases: [{ ...binding, remote: !local }],
}));

const proxy = await getPlatformProxy<{ DB: D1Database }>({
  configPath,
  envFiles: [],
  // `wrangler d1 execute --persist-to <dir>` と同じ DB を使うなら <dir>/v3 を渡す
  persist: local ? { path: path.resolve(persistTo ?? ".wrangler/state", "v3") } : false,
  remoteBindings: !local,
});
const db = drizzle(proxy.env.DB, { schema });
try {
  // db.select() / db.update() / db.batch([...]) など
} finally {
  await proxy.dispose();
  rmSync(dir, { recursive: true, force: true });
}
```

## 注意すること

- `getPlatformProxy` の `persist.path` は、`--persist-to` で渡すディレクトリそのものではなく、その下の `v3`。既定値も `.wrangler/state/v3`。`database_id` を同じにしておけば、`wrangler d1 execute --local` と同じ SQLite ファイルを読み書きする（ローカルで確かめた）。
- `unstable_readConfig` などの型は `@cloudflare/workers-utils` から来ていて、解決できずに `any` になることがある。使う形（`d1_databases` の要素など）を自分で型注釈しておく。
- `--remote` はリモートのバインディングのプロキシ（一時的な Worker）経由でつながる。`wrangler login` が必要。今回は本番に触れるため、`--remote` の接続そのものは試していない。確認の前に件数を表示して止まるスクリプトなら、最初は確認で中止して接続だけを確かめるとよい。
- `process.exit` で途中終了すると `dispose` と一時ディレクトリの片付けが飛ぶ。エラーは例外にして、`try`/`finally` で片付けてから `process.exitCode` を設定する。
- 複数のテーブルの書き換えは `db.batch` にまとめると 1 つのトランザクションになり、途中で失敗しても一部だけ書き換わらない。
