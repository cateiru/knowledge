# Drizzle の select に書いた相関サブクエリで、外側の列の表名が消える

Drizzle（`drizzle-orm` 0.45、SQLite）で、親の行ごとに「子の最も新しい日時」を 1 つの列として読もうとしたとき（2026年11月）に確かめたこと。

## 起きること

1 つの表だけを読む `select` では、フィールドに書いた `sql` の中の列から表名が省かれる。

```ts
db.select({
  id: parents.id,
  last: sql`(
    SELECT max(${logs.at}) FROM ${logs}
    WHERE ${logs.parentId} = ${parents.id}
  )`,
}).from(parents);
```

生成される SQL は次のようになる（`toSQL()` で確認）。

```sql
select "id", (SELECT max("created_at") FROM "logs" WHERE "parent_id" = "id") from "parents"
```

サブクエリの中の `"id"` は、内側の `logs.id` に解決される。`logs` にも `id` 列があると、エラーにならずに常に一致しない（いつも `NULL` になる）ため気づきにくい。

## 書き方

サブクエリの中は、表に別名を付けて列名を直接書く。外側の列は、列ではなく表を埋め込んでから列名を書くと、表名が残る。

```ts
last: sql<number | null>`(
  SELECT max(l.created_at) FROM ${logs} AS l
  WHERE l.parent_id = ${parents}.id
)`,
```

- 値は DB の生の値で返る。`mode: "timestamp"` の列でも `Date` にはならないので、自分で変換する。
- `(parent_id, created_at)` のインデックスがあれば、`max` はインデックスの末尾だけを読む（SQLite の min/max の最適化。`GROUP BY` で全件をまとめるより読む行が少ない、という理解で選んだ。行数は計測していない）。
- 値が `NULL` のままになる失敗は、テストで「子がある親」「子がない親」の両方を並べると見つけられる。
