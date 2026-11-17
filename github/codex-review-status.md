# Codex の PR 自動レビューが終わったかを gh で判定する

Codex の GitHub 連携で PR を自動レビューさせているとき、レビューが終わって問題がなかったのか、指摘があるのか、止まっているのかを `gh` で判定する方法。2026年11月に、実際の PR のコメントとリアクションを見て確かめた。

## Codex が PR に残すもの

Codex のアカウントは `chatgpt-codex-connector[bot]`（GraphQL では `chatgpt-codex-connector`）。PR には次のものが付く。

- PR 本文へのリアクション
  - 👀 (`eyes`): レビュー中。
  - 👍 (`+1`): すべてのレビューが指摘なしで終わった。
- インラインのレビューコメント: 指摘。本文の先頭に `P1`・`P2` などの優先度のバッジがある。
- PR のコメント（issue comment）
  - `<!-- codex-pull-request-review-summary -->` で始まる要約のコメント。レビューのたびに同じコメントが編集され、`updated_at` が変わる。
  - `<!-- codex-review-receipt:v1 -->` を含み `Didn't find any major issues` と書かれたコメント。`Reviewed commit:` に、レビューしたコミットの SHA（先頭 10 文字）が書かれる。
  - `<!-- codex-code-review-finding:v1 -->` を含むコメント。インラインではなく、PR のコメントとして指摘が来ることがある。

要約のコメントには、次のような機械向けの状態が HTML コメントで埋め込まれている。

```text
<!-- codex-review-state:v1 {"code_review":{"commit_sha":"<sha>","has_any_findings":true,"has_unresolved_findings":false,"status":"completed"},"security_review":{...}} -->
```

## 判定のしかた

- 完了: 最新のコミットより後に付いた 👍 があるか、`Didn't find any major issues` のコメントの `Reviewed commit:` が最新のコミットと一致する。
  - 👍 は古いコミットのときのものが残るので、`created_at` を最新のコミットの日時と比べる。
  - 👍 の代わりに `Didn't find any major issues` のコメントだけが付くことがあった。どちらも完了として扱う。
- 指摘あり: 未解決のレビュースレッドがあるか、最新のコミットより後に `codex-code-review-finding` のコメントがある。
- 止まっている: 要約の `commit_sha` が最新のコミットで `status` が `completed` なのに、👍 もコメントも付かない。実際に、指摘を直して push したあとのレビューが完了表示のまま 👍 が付かない PR があった。
  - このときは PR に `@codex review` とコメントすると、もう一度レビューされる（要約のコメントに書かれている起動方法）。日本語で「@codex レビューして」と書いてもレビューされた。

```sh
# 👍 と 👀
gh api 'repos/<owner>/<repo>/issues/<pr>/reactions?per_page=100' \
  --jq '.[] | select(.user.login | test("chatgpt-codex-connector")) | {content, created_at}'

# 要約の状態
gh api 'repos/<owner>/<repo>/issues/<pr>/comments' \
  --jq '[.[] | select(.body | test("codex-review-state:v1"))] | last | .body' \
  | grep -o 'codex-review-state:v1 {.*}'
```

未解決のスレッドは REST では分からないので、GraphQL の `reviewThreads` の `isResolved` を見る。

```sh
gh api graphql -f query='
  query($owner: String!, $name: String!, $pr: Int!) {
    repository(owner: $owner, name: $name) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes { id isResolved comments(first: 20) { nodes { databaseId author { login } body } } }
        }
      }
    }
  }' -F owner=<owner> -F name=<repo> -F pr=<pr>
```

## 指摘に対応したあと

返信のしかたは [PR のレビューコメントに、スレッドの返信として答える](reply-to-review-comments.md) を参照。返信したら、スレッドを解決済みにする。`id` には上の GraphQL で取ったスレッドの `id`（`databaseId` ではない）を渡す。

```sh
gh api graphql -f query='mutation($id: ID!) { resolveReviewThread(input: { threadId: $id }) { thread { isResolved } } }' -F id=<thread_id>
```

新しいコミットを push すると、Codex は自動でもう一度レビューする（要約のコメントの Review trigger が `New commits` になる）。

## Claude Code に待たせる

Claude Code では、5 分ごとに上の判定をして、次に動くべき状態になったら結果を 1 行出して終了するスクリプトを書き、Bash の `run_in_background` で起動した。終了すると Claude Code に通知が来るので、`sleep` で待たせたり出力をのぞかせたりする必要がない。
