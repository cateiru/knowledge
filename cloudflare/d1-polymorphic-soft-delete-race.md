# 外部キーのない参照先が論理削除される競合を、文の中の条件で防ぐ

Cloudflare D1 と Drizzle ORM を使うアプリで、添付ファイルの行を記録に紐付ける処理に、記録が論理削除されていないことの確認を文の中に入れた。そのときに確かめたことを書く（2026年11月）。

## 何が問題だったか

添付のテーブルは「種別 + ID」の組（polymorphic な参照）で、いろいろな種類の記録を指している。この形では参照先のテーブルが 1 つに決まらないため、外部キー制約を張れない。

書き込みの前に「記録が実在し、`deleted_at IS NULL` であること」を SELECT で確かめていたが、確認と書き込みは別の文になっている。その間に別の端末から記録が論理削除されると、削除済みの記録に新しく添付できてしまう。物理削除であれば外部キーが防いでくれるが、論理削除では行が残っているため、外部キーがあっても防げない。

## 文の WHERE に EXISTS を含める

書き込む文そのものに「参照先が実在し、削除されていない」という条件を入れる。確認の SELECT はそのまま残してよい。文の中の条件は、競合したときの最後の砦になる。

種別ごとにテーブルが違うので、種別から `EXISTS` の条件を作る関数を用意する。

```ts
function liveRecordCondition(recordType: string, recordId: string): SQL | undefined {
  switch (recordType) {
    case "record_a":
      return sql`EXISTS (SELECT 1 FROM ${recordsA} WHERE ${recordsA.id} = ${recordId} AND ${recordsA.deletedAt} IS NULL)`;
    // 論理削除しないテーブルは、実在だけを確かめる
    case "product":
      return sql`EXISTS (SELECT 1 FROM ${products} WHERE ${products.id} = ${recordId})`;
    default:
      // 記録の種別でないもの（下書きなど）には条件を付けない
      return undefined;
  }
}
```

`deleted_at` 列があるかどうかは種別ごとに違うので、推測せずにスキーマで確かめる。Drizzle の `and()` は `undefined` を無視するため、条件を付けない種別では `undefined` を返せばよい。

### UPDATE（紐付けの付け替え）

```ts
const updated = await db
  .update(attachments)
  .set({ recordType, recordId })
  .where(and(eq(attachments.id, id), liveRecordCondition(recordType, recordId)))
  .returning({ id: attachments.id });
```

`.returning()` で返った行を数えれば、条件で断られたかが分かる。`meta.changes` は D1 とテスト用の sql.js で返り方が違うため、`returning` の件数で判定するほうが扱いやすい。

### INSERT（記録へ直接追加する）

INSERT には WHERE がないので、`INSERT ... SELECT ... WHERE <条件>` の形にする。Drizzle では `db.insert(table).select(sql\`SELECT ... WHERE ...\`)` で書ける。

注意点として、`INSERT ... SELECT` では列の既定値（Drizzle の `$defaultFn` や SQL の `DEFAULT`）が使われない。テーブル定義の列の並びどおりに、渡した値か列の既定値を SELECT に並べるヘルパーを作って使った。挿入されたかどうかは、ここでも `.returning()` の件数で判定する。0 件なら「添付先が見つからない」エラーにする。

## DB の外（オブジェクトストレージ）の削除は文に含められない

フォームの保存では「新しい添付を紐付ける」と「外された添付を削除する（R2 のオブジェクトも消す）」を続けて行う。R2 の削除は D1 の文やトランザクションに含められないため、完全には原子的にできない。

次のように順番を工夫して、被害を減らした。

1. 先に紐付けの UPDATE を実行する。
2. 紐付けられなかった行があれば、記録がまだあるかをもう一度確かめる。記録が消えていたら、外された添付は削除せずにエラーを返す。
3. 記録があれば、外された添付を削除する。

「添付をすべて外して保存した」場合は紐付ける行がないので、手順 2 で記録の削除に気付けない。この競合は残るが、論理削除した記録を元に戻す機能がないアプリだったため、実害はないと判断した。

## テスト

Vitest と sql.js で、実際のマイグレーションを流した DB に対してテストした（sql.js の準備は [D1 の batch でまとめて削除する処理を sql.js でテストする](d1-batch-delete-sqljs-test.md) を参照）。

- 削除済みの記録には紐付けず、下書きのまま残ること
- 論理削除しない種別でも、実在しない ID には紐付けないこと
- 確認の後に削除された状況を作るため、確認の関数だけをモックで「削除されていない」と返させ、DB の行は削除済みにしておく。このとき、エラーを返し、外された添付の削除が呼ばれないこと

`vi.mock` で一部の関数だけ差し替えるには、`importOriginal` で本物を読み込み、差し替える関数を `vi.fn(actual.fn)` で包む。テストの中では `mockResolvedValueOnce` で 1 回目の呼び出しの結果だけを変えられる。

```ts
vi.mock("./recordOwner", async (importOriginal) => {
  const actual = await importOriginal<typeof import("./recordOwner")>();
  return { ...actual, resolveOwner: vi.fn(actual.resolveOwner) };
});
```

最後に、文の中の条件を一時的に外してテストを流し、追加したテストが失敗することを確かめた。条件がなくても通るテストでは、競合を防げているかを確かめたことにならない。
