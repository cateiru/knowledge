# Drizzle の D1 の batch で、同じ列名を JOIN した SELECT の値が入れ替わる

2026年11月に、Drizzle ORM（0.45 系）の `drizzle-orm/d1` で確かめた。

## 何が起きるか

Drizzle の D1 ドライバーは、ふつうの SELECT では D1 の `raw()`（列の値の配列）で結果を受け取り、位置で項目に当てはめる。

一方 `db.batch([...])` では、D1 の `batch()` が返す行（列名をキーにしたオブジェクト）を `Object.values` で配列に直してから当てはめる（`drizzle-orm/d1/session.js` の `batch` と `mapResult(result, true)`）。

そのため、別々のテーブルの同じ名前の列（`users.name` と `households.name` など）を選ぶ JOIN を `batch` に入れると、オブジェクトのキーが重なって片方が消え、値が 1 つずつずれて別の項目に入る。生成される SQL は `select "households"."name", "users"."name" ...` のように別名を付けないので、D1 側でも区別できない。

## 対処

- 読み取りは `batch` に入れず、`Promise.all` で並べる。ふつうの実行は `raw()` を使うので、列名が重なってもずれない。
- どうしても `batch` で読むなら、`sql<string>\`${households.name}\`.as("household_name")` のように別名を付けて、列名が重ならないようにする（こちらは試していない）。
- 書き込みの `RETURNING` は 1 つのテーブルの列だけなので、`batch` に入れても問題にならない。

## sql.js でテストするときの注意

D1 の代わりに sql.js を D1 と同じ形に包んでテストするなら、Drizzle が呼ぶ `prepare().bind().raw()`・`all()`・`run()`・`first()` と `batch()` を用意する。`raw()` は `statement.get()`（配列）、`all()` は `statement.getAsObject()`（オブジェクト）で返す。`batch()` は D1 と同じく `BEGIN`〜`COMMIT` で包み、失敗したら `ROLLBACK` すると、本番に近い振る舞いになる。

関連: [D1 の batch でまとめて削除する処理を sql.js でテストする](d1-batch-delete-sqljs-test.md)
