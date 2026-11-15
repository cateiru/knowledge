# PR のレビューコメントに、スレッドの返信として答える

レビューの指摘（Copilot などのインラインコメント）を直して push しただけでは、レビュアーからはどの指摘に対応したのかが分かりにくい。指摘ごとのスレッドに、何を直したかを返信する。2026年8月から使っている手順。

## 手順

まず、インラインコメントの ID を取る。

```sh
gh api repos/<owner>/<repo>/pulls/<pr>/comments --jq '.[] | {id, path, line, body}'
```

直して push したあと、コメントごとに返信する。

```sh
gh api repos/<owner>/<repo>/pulls/<pr>/comments/<comment_id>/replies -f body='<直した内容と commit>'
```

- `gh pr comment` は PR 全体へのコメントになり、スレッドには付かない。インラインのスレッドに付けるには、上の `replies` の API を使う。
- GitHub のドキュメントでは、返信への返信はできず、スレッドの最初のコメントの ID を指定することになっている（ほかの ID を指定したときの動きは確かめていない）。
- 直さないと判断した指摘にも、その理由を返信しておく。
