# D1 の batch でまとめて削除する処理を sql.js でテストする

Cloudflare D1 と Drizzle ORM を使うアプリで、親の行（例: グループ）と、それを参照する子の行を 1 回の `db.batch` でまとめて削除する処理を作り、Vitest と sql.js でテストした。そのときに確かめたことを書く（2026年11月）。

## batch のすべての文に同じ条件を付ける

D1 では、複数の文を原子的に実行するには `db.batch` を使うしかない。「操作した人がオーナーであること」「子の行がまだ残っていないこと」のように、確認してから削除するまでの間に変わりうる条件があるときは、読んでから書くのではなく、batch の **すべての** 文の WHERE に条件を含める。条件を満たさなければ、どの文も行を消さない。

batch の途中で条件が変わらないように、文の順番に気をつける。

- 条件の判定に使う行（例: オーナーであることを表すメンバーの行）は、最後から 2 番目の文で消す。
- 最後の文（親の行の削除）では、オーナーの行がもう消えているため、オーナーであることを確かめられない。代わりに「メンバーの行が 1 件も残っていないこと」を条件にする。前の文が条件を満たさずにメンバーを消さなかった場合は、メンバーが残っているので、親の行も消えない。
- 結果は最後の文に `.returning()` を付けて受け取り、1 件以上返ってきたかで、削除できたかを判断する。

```ts
const results = await db.batch([
  db.delete(children).where(and(eq(children.parentId, id), canRemove)),
  // 判定に使うメンバーの行は親の直前に消す
  db.delete(members).where(and(eq(members.parentId, id), canRemove)),
  db
    .delete(parents)
    .where(
      and(
        eq(parents.id, id),
        sql`NOT EXISTS (SELECT 1 FROM ${members} WHERE ${members.parentId} = ${id})`,
      ),
    )
    .returning({ id: parents.id }),
]);
const deleted = results.at(-1);
return Array.isArray(deleted) && deleted.length > 0;
```

`canRemove` には「オーナーのメンバーの行がある」「子の猫などがいない」といった `EXISTS` / `NOT EXISTS` の条件をまとめておく。

## 同じテーブルを読む WHERE で DELETE しても大丈夫

メンバーの行を消す文の WHERE に、同じメンバーのテーブルを読むサブクエリ（オーナーの行があるか）が含まれる。SQLite は、WHERE が削除対象と同じテーブルを読む場合、消す行を先にすべて決めてから消す。そのため、オーナーの行を途中で消しても、ほかのメンバーの行が消し残されることはない。オーナーの行も含めて 1 つの文で消せることを、テストで確かめた。

## sql.js は外部キー制約が既定で無効

D1 は外部キー制約を強制するが、sql.js は既定で無効になっている。そのままテストすると、子より先に親を消すような順番の間違いがあってもテストが通ってしまい、本番の D1 で初めて失敗する。テストの準備で有効にしておく。

```ts
const SQL = await initSqlJs();
const sqlite = new SQL.Database();
const db = drizzle(sqlite);
await migrate(db, { migrationsFolder: "./drizzle" });
// D1 と同じく外部キー制約を有効にする
sqlite.run("PRAGMA foreign_keys = ON");
```

sql.js には `db.batch` がないため、テストでは文を順番に実行するだけの簡易実装で補う。この場合、本物の batch と違ってロールバックはされない。そこで、条件を満たさない場合（オーナーでない、子が残っている）のテストでは、削除対象の **すべての** テーブルに行を入れておき、どのテーブルの行も 1 件も消えていないことを確かめる。こうすると、条件を付け忘れた文が 1 つでもあればテストが失敗する。実際に 1 つの文から条件を外して、テストが失敗することを確認した。

## Vitest は型チェックをしない

Drizzle で存在しない列（例: `items.parentRecordId` のように、名前を間違えた列）を `inArray` に渡しても、Vitest は型エラーを出さずにそのまま実行する。生成される SQL では列名が空になり、`( in (select ...))` のような文になって、次のエラーで失敗する。

```text
Error: near "in": syntax error
```

エラーメッセージから原因が分かりにくい。テストより先に `tsc --noEmit` を実行すると、列名の間違いをすぐに見つけられる。生成された SQL を見たいときは、文の `.toSQL().sql` を出力する。

## テストで NOT NULL 制約に引っかかる

テストで行を入れるとき、Drizzle の `insert().values()` は型の上では必須の列を要求するが、Vitest は型チェックをしないため、必須の列が抜けていても実行時の `NOT NULL constraint failed` で初めて気づく。これも `tsc` を先に流せば防げる。
