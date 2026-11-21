# Drizzle で同じテーブルを 2 回 JOIN する

2026年11月に、Drizzle ORM（0.45 系）の `drizzle-orm/d1` で確かめた（テストは sql.js を D1 の形に包んだもので実行）。

## やりたかったこと

招待のテーブルに「招待した人」（`created_by_user_id`）と「参加した人」（`accepted_by_user_id`、NULL あり）の 2 つの
ユーザーへの参照があり、両方の名前を 1 回の SELECT で取りたかった。同じ `users` を 2 回 JOIN すると、
どちらの `users.name` か区別できない。

## 対処

`drizzle-orm/sqlite-core` の `alias` で、同じテーブルに別名を付けて JOIN する。

```ts
import { alias } from "drizzle-orm/sqlite-core";

const creators = alias(users, "creators");
const acceptors = alias(users, "acceptors");

const rows = await db
  .select({
    id: invitations.id,
    createdByUserName: creators.name,
    // 参加していなければ leftJoin なので null になる
    acceptedByUserName: acceptors.name,
  })
  .from(invitations)
  .innerJoin(creators, eq(creators.id, invitations.createdByUserId))
  .leftJoin(acceptors, eq(acceptors.id, invitations.acceptedByUserId));
```

- `alias` は関数の中で作ってよい（クエリごとに作っても問題なかった）
- `select({ ... })` で項目名を明示すると、結果のキーは項目名になり、`creators.name` と `acceptors.name` は混ざらない
- `leftJoin` した別名の列は、型も `string | null` になる

## 注意

D1 の `db.batch([...])` に入れると、同じ列名（どちらも `name`）が重なって値がずれる。読み取りは `batch` に入れず、
`Promise.all` で並べる。詳しくは [Drizzle の D1 の batch で、同じ列名を JOIN した SELECT の値が入れ替わる](drizzle-d1-batch-duplicate-columns.md)。
