# next dev（getPlatformProxy）では、同じ Worker の Workflow を呼べない

2026年11月に、wrangler 4.164 と @opennextjs/cloudflare 1.20 で確かめた。

## 起きること

OpenNext の Next.js アプリは、`next dev` のときに `initOpenNextCloudflareForDev()` で wrangler の `getPlatformProxy` からバインディングを受け取る。`wrangler.toml` に `script_name` のない `[[workflows]]`（同じ Worker が export する Workflow）を書いていると、`getPlatformProxy` は次の警告を出して、そのバインディングを `env` から消す。

```text
You have defined bindings to the following Workflows without a script_name:
  ...
These are not available in local development, ...
```

`getPlatformProxy` は Worker のコードを動かさないので、Worker が export する Workflow のクラスも動かせないためである（wrangler の `getMiniflareOptionsFromConfig` で、該当するバインディングを `delete` している）。そのため `next dev` では `getCloudflareContext().env.<WORKFLOW>` が `undefined` になる。

Worker Previews でも、本番の Workflow を共有しないようにバインドしていなければ、同じく `undefined` になる。

## 対応

Server Action から Workflow を起動する処理では、バインディングがないときに、同じ処理をリクエストの後に実行するようにした。

```ts
const workflow: Env["MY_WORKFLOW"] | undefined = getCloudflareContext().env.MY_WORKFLOW;
if (workflow) {
  await workflow.create({ id, params: { id } });
} else {
  // バインディングは after() の中でも使えるよう、先に取っておく
  const env = getMyEnv();
  after(async () => {
    await runJob(id, env);
  });
}
```

- 処理の本体は、`getCloudflareContext()` に頼らず D1・R2 のバインディングを引数で受け取る形にする。Workflow のステップの中では `getCloudflareContext()` が使えないため、どちらからも同じ関数を呼べる
- `after()` は Workers では `waitUntil` になるので、時間のかかる処理は途中で止まることがある。止まった処理がいつまでも「処理中」に見えないよう、一定時間（例: 1 時間）を過ぎたものは失敗として扱うようにした
- `env` の型は `Env["MY_WORKFLOW"] | undefined` と書いて、`undefined` になりうることを型で明示する（`wrangler types` の `Env` では必ずある型になっている）

## R2 のマルチパートアップロードは getPlatformProxy でも動く

`getPlatformProxy` で受け取った R2 のバインディングでも、`createMultipartUpload`・`uploadPart`・`complete` は動いた（ローカルで 5 MiB のパートと短い最後のパートを送り、`get` で読み直して大きさが合うことを確かめた）。`get` の `body` は小さな塊（数 KB）に分かれて届いた。
