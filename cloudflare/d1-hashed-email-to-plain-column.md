# ハッシュで保存していたメールアドレスを、平文の列へ段階的に移す

2026年11月に、Cloudflare Workers + D1 + Drizzle ORM のアプリで、ログインに使うメールアドレスを
argon2id のハッシュ（`users.email_hash`）で保存していたのをやめ、メールアドレスそのものを
`users.email` に保存するように移した。そのときの手順と注意点のメモ。

## なぜ移すのか

ハッシュ（SALT 付き）で保存していると、元のメールアドレスが分からないため、運用者がメールアドレスを
変えることも、だれのアカウントかを確かめることもできない。メールアドレスを変えられるようにするには、
平文で持つ必要がある。

ただし、既存のユーザーのメールアドレスは DB からは復元できない。本人がもう一度ログインしたとき
（OIDC の ID トークンでメールアドレスが分かったとき）に埋めるしかない。

## 3 段階で移す

1. 新しい列 `email`（一意・NULL 可）を追加する。
2. ログインの処理を次のように変え、`email` を持たないユーザーを強制ログアウトさせる。
3. 全員の `email` がそろったら、`email_hash` の列とハッシュの処理・SALT を削除する（別のリリースで行う）。

ログインの処理（2 段階目）は次の順にする。

- 正規化（前後の空白を除いて小文字に）したメールアドレスで `email` を探す。見つかればそのユーザー。
- 見つからなければ、従来どおりハッシュにして、`email_hash` が一致し **`email` が NULL の**ユーザーを探す。
  見つかれば `email` を保存して、そのユーザーとしてログインさせる。
- どちらでも見つからなければ、新しいユーザーを `email` だけで作る（`email_hash` は保存しない）。

argon2id のハッシュは CPU を 200 ms 以上使うため、`email` で見つかったときは計算しない。移行が進むほど
ハッシュの計算は減る。

`email` を保存する UPDATE には `WHERE id = ? AND email IS NULL` を付け、同じユーザーの同時ログインで
上書きし合わないようにした。UPDATE のあとに `email` で引き直し、別のユーザーが返ってきたらエラーにする。

## 強制ログアウトはコードで行う

既存のセッションを使い続けられると、そのユーザーはログインし直さず、いつまでも `email` がそろわない。
はじめは「デプロイ後に `DELETE FROM sessions` を手で実行する」ことを考えたが、次の理由でやめた。

- マイグレーションに `DELETE FROM sessions` を入れると、デプロイ前に適用したときに旧コードで
  ログインし直され、`email` が埋まらない。
- デプロイ後に手で実行する手順は、抜けやすい。

代わりに、セッションからユーザーを引くクエリに `users.email IS NOT NULL` の条件を足した。
デプロイした瞬間に、`email` を持たないユーザーのセッションだけが無効になり、ログインし直すと
`email` が埋まって以降は普通に使える。3 段階目でこの条件は外してよい。

```ts
const [row] = await db
  .select({ user: users })
  .from(sessions)
  .innerJoin(users, eq(sessions.userId, users.id))
  .where(
    and(
      eq(sessions.tokenHash, await hashSessionToken(token)),
      gt(sessions.expiresAt, new Date()),
      isNotNull(users.email),
    ),
  )
  .limit(1);
```

ローカルの開発サーバーで、`email` を持たないユーザーの既存のセッションで開くとログイン画面に戻され、
大文字を混ぜたメールアドレスでログインし直すと、新しいユーザーは作られずに既存のユーザーに
`email` が保存されることを確かめた。

## 列の追加はデプロイより先に適用する

Drizzle の `db.select().from(users)` は `SELECT *` ではなく、スキーマに書いた列を 1 つずつ並べた SQL になる。
そのため、スキーマに `email` を足したコードを、列を追加するマイグレーションより先にデプロイすると、
`users` を読むすべての処理が「no such column」で失敗する（仕組みからの推測で、本番では試していない）。

列と一意のインデックスを追加するだけのマイグレーションなら、旧コードはそのまま動くので、
マージ・デプロイの前に本番 D1 へ適用しておく。

```sh
pnpm exec wrangler d1 migrations apply <DB のバインディング名> --remote
```

全員がそろったかは、次で確かめる（0 になれば 3 段階目に進める）。

```sh
pnpm exec wrangler d1 execute <DB のバインディング名> --remote --command "SELECT COUNT(*) FROM users WHERE email IS NULL"
```

## find-or-create を 1 文にする

新しいユーザーの作成は、同じメールアドレスで同時にログインしても 2 件できないよう、一意制約と
`ON CONFLICT DO NOTHING` で 1 つの INSERT 文にしている。SQLite では `INSERT ... SELECT` に
`ON CONFLICT` を付けるとき、構文のあいまいさを避けるため `WHERE true` が必要になる。

```sql
INSERT INTO users (id, name, role, email)
SELECT
  ?,
  CASE WHEN EXISTS (SELECT 1 FROM users) THEN 'ユーザー' ELSE '管理者' END,
  CASE WHEN EXISTS (SELECT 1 FROM users) THEN 'member' ELSE 'admin' END,
  ?
WHERE true
ON CONFLICT (email) DO NOTHING
```

挿入のあとに `email` で引き直して、挿入した行か、同時に作られた行のどちらかを返す。

## 運用者がメールアドレスを変えるとき

画面からは変えられないようにし、問い合わせを受けたら運用者がスクリプトで `email` を書き換える。
書き換える前に、別のユーザーが同じメールアドレスを使っていないかを確かめる（一意制約で UPDATE も
失敗するが、分かりやすいエラーにするため）。書き換えたあとに古いメールアドレスでログインすると、
新しいユーザーが作られる点に注意する。
