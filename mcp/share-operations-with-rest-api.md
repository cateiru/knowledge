# MCP のツールの実装を、REST API と共用する

MCP サーバーで公開している読み書きのツールと同じことを、トークンで認証する REST API からもできるようにしたとき（2026年11月）に決めたこと。

## 操作を通信の方式から切り離す

ツールの実装は、はじめ `McpServer.registerTool()` を直接呼び、MCP の結果の形（`content`・`isError`）を返していた。これを、登録先の小さなインターフェースに対して登録する「操作」に変えた。

```ts
interface OperationRegistry {
  register<TShape extends ZodRawShape>(
    name: string,
    config: { title: string; description: string; inputSchema?: TShape },
    handler: (args: z.output<z.ZodObject<TShape>>) => Promise<OperationResult>,
  ): void;
}

type OperationResult =
  | { data: unknown }
  | { error: { code: "not_found" | "forbidden" | /* ... */ string; message: string } };
```

- MCP は、操作をツールとして登録し、`data` を JSON の文字列に、`error` を `isError` の結果にする。
- API は、操作を名前で引ける Map に集め、HTTP のメソッドとパスの表から呼ぶ。`error.code` を HTTP のステータスに対応させる。
- MCP ではエラーの種類は要らなかったが、HTTP では 400・403・404・409 を分けたい。文字列だけを返していたエラーに `code` を足し、文字列だけなら `invalid_request` として扱うようにすると、変更する箇所が少なく済む。
- 呼んだ人（ユーザー・許可した範囲・履歴に残す人）は、操作に渡すコンテキストの `resolveAccess()` で求める。MCP は OAuth の接続から、API はトークンから求める。

## 引数の渡し方

- パスの `{catId}` のような部分は、同じ名前の操作の引数にする。残りの引数を `z.strictObject` で検証すると、知らない項目や、パスと食い違う本文の `catId` を 400 にできる。
- `GET`・`DELETE` はクエリで受け取る。クエリは文字列なので、その引数のスキーマが文字列を受け付けず、数字の形をしているときだけ数値に直す。

## 説明とエラーの文言

ツールの説明やエラーには「`list_xxx` で取得できる」のように、ほかのツールの名前が出てくる。API では、操作の名前とエンドポイントの対応表を使って `GET /xxx` に置き換える。名前は `snake_case` なので、単語境界の正規表現（`/\b[a-z]+(?:_[a-z]+)+\b/g`）で拾い、表にあるものだけを置き換える。

OpenAPI の仕様も、操作の zod スキーマを `z.toJSONSchema(..., { io: "input" })` で変換して作れる。

## ずれを防ぐテスト

- すべての操作に、ちょうど 1 つのエンドポイントがある。
- メソッドとパスが重ならない。
- パスの引数が、操作の引数にある。
- 仕様の文章に、ツールの名前が残っていない。

## トレードオフ

ツールの引数の名前を変えると、公開している API も変わる。MCP のクライアントは毎回ツールの定義を読むので引数の変更に強いが、REST API の利用者はそうではない。公開した API を互換性のない形で変えないという決まりを、操作を変えるときのルールとして書いておく必要がある。

## アクセス履歴とトークンの削除の競合

アクセスの履歴は応答の後に書き込むので、その間にトークンが削除されることがある。`INSERT ... SELECT ... WHERE EXISTS (トークン)` にすると、削除した後に履歴が残らない。トークンの削除は、履歴の削除と 1 つの batch にする。
