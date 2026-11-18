# 負のマージンではみ出させたボタンが隣との gap を打ち消す

## 症状

グリッド (`display: grid`) の行で、左に開閉ボタン、右にチェックボックスを並べた。ボタンのホバーの背景を文字より少し広く見せるため、左右に負の `margin-inline` を付けた。

```css
.row {
  display: grid;
  grid-template-columns: minmax(0, 1fr) auto;
  column-gap: 8px;
}

.trigger {
  padding: 0 8px;
  margin-inline: -8px; /* 左右に 8px ずつはみ出す */
}
```

すると、ホバーの背景が右のチェックボックスにぴったり接してしまった (2026年11月に確認)。

## 原因

右側の `margin-inline-end: -8px` で、ボタンの背景がグリッドのトラックの外へ 8px はみ出す。`column-gap` も 8px なので、はみ出した分で隙間がちょうど埋まり、背景が隣の要素に接する。

左側のはみ出しは、文字の位置をほかの行と揃えたまま背景だけを広げるためのもので、問題はない。

## 対処

右に別の要素が並ぶボタンでは、右へはみ出さないようにする。

```css
.trigger {
  padding: 0 8px;
  margin-inline: -8px;
}

/* 右に要素が並ぶ行では、column-gap を隙間として残す */
.rowTrigger {
  margin-inline-end: 0;
}
```

これで、背景と隣の要素の間に `column-gap` の 8px が残る。右に何も並ばない (行の幅いっぱいの) ボタンは、両側にはみ出したままでよい。

## 確かめ方

ホバーの見た目は VRT では撮られないことが多い (マウスを外して撮る) ので、Playwright で `locator.hover()` してから要素のスクリーンショットを撮り、修正前後を見比べると分かりやすい。
