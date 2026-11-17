# CSS だけで家系図のような図の接続線を描く

2026年11月に、ヘルプのページで「家・人・ねこ・記録」の関係を家系図のような図で表したときのメモ。画像や SVG を使わず、入れ子のリスト（`ul` と `li`）と疑似要素（`::before`・`::after`）だけで、親と子を線でつないだ。

## この方法にした理由

- 画像ではないので、ダークモード・テーマカラー・文字サイズの設定に追従する。
- 入れ子のリストなので、スクリーンリーダーでも親子の構造が伝わる。
- 線の位置は子の数や幅に合わせて自動で決まり、座標を計算しなくてよい。

最初は「枠の中に枠を入れる」入れ子の図にしたが、関係が分かりづらいと言われ、線でつなぐ家系図の形に作り直した。さらに「人とねこが同じ段に並ぶのはよくない」と言われ、最終的には家を真ん中に置き、上に人、下にねこと記録をつなぐ形にした。

## 下向きの線（1 つの親から複数の子へ）

1 つの親の下に子を並べる場合は、次のように線を引く。

- 子のリスト（`ul`）の `::before` で、親から下へ縦の線を引く。
- 子（`li`）ごとに `::before`（左半分）と `::after`（右半分）で、上端に兄弟をつなぐ横の線を引く。`::after` の左の枠線は、その子へ下りる縦の線を兼ねる。
- 両端の子は外側へ線を伸ばさない。外側の疑似要素の枠線を消し、内側の角を丸める。
- 子が 1 つだけのときは横の線を出さず、親からまっすぐ下ろす。

```css
.figure {
  --line-width: 2px;
  --line-height: 16px;
  --line-color: #999;
}

/* 子の並び。親から下へ縦の線を引く */
.children {
  position: relative;
  display: flex;
  justify-content: center;
  padding-top: var(--line-height);
  list-style: none;
}

.children::before {
  position: absolute;
  top: 0;
  left: calc(50% - var(--line-width) / 2);
  height: var(--line-height);
  border-left: var(--line-width) solid var(--line-color);
  content: "";
}

/* 子 1 つ分 */
.branch {
  position: relative;
  display: flex;
  flex: 1 1 0;
  flex-direction: column;
  align-items: center;
  min-width: 0;
  padding-top: var(--line-height);
}

.branch::before,
.branch::after {
  position: absolute;
  top: 0;
  width: 50%;
  height: var(--line-height);
  border-top: var(--line-width) solid var(--line-color);
  content: "";
}

.branch::before {
  right: calc(50% - var(--line-width) / 2);
}

.branch::after {
  left: calc(50% - var(--line-width) / 2);
  border-left: var(--line-width) solid var(--line-color);
}

/* 両端の子は外側へ線を伸ばさず、角を丸めて下ろす */
.branch:first-child::before,
.branch:last-child::after {
  border: 0;
}

.branch:first-child::after {
  border-top-left-radius: 10px;
}

.branch:last-child::before {
  border-right: var(--line-width) solid var(--line-color);
  border-top-right-radius: 10px;
}

/* 子が 1 つだけなら、親からまっすぐ下ろす */
.branch:only-child {
  padding-top: 0;
}

.branch:only-child::before,
.branch:only-child::after {
  display: none;
}
```

HTML は、親のノードのすぐ後ろに子のリストを置く。子の中にさらに `.children` を入れれば、何段でもつなげられる。

```html
<div class="node">親</div>
<ul class="children">
  <li class="branch"><div class="node">子 1</div></li>
  <li class="branch">
    <div class="node">子 2</div>
    <ul class="children">
      <li class="branch"><div class="node">孫</div></li>
    </ul>
  </li>
</ul>
```

## 上向きの線（複数の親を下の 1 つにまとめる）

親を上に並べ、下の 1 つのノードへ線を集める場合は、上下を入れ替えて同じことをする。

- `top` を `bottom` に、`border-top` を `border-bottom` に、`padding-top` を `padding-bottom` に替える。
- 角の丸めは `border-bottom-left-radius`・`border-bottom-right-radius` にする。
- リストの `::after` で、下のノードへ向けて縦の線を引く。

## 注意すること

- 線の太さの半分（`var(--line-width) / 2`）だけずらして、縦の線をノードの中心にそろえる。ずらさないと、線の太さの分だけ中心から外れる。
- 横の線は、各子の幅の中心から中心までになる。子を `flex: 1 1 0` で同じ幅にしておくと、線が左右対称になる。
- 1 段に 4 つ並べると、幅 320px の画面ではノードの文字が折り返して崩れた。このときは文字と余白を小さくして収めたが、最終的には 1 段を 2 つにする配置に変えたため、縮小は不要になった。狭い画面では、1 段に並べる数を減らすのが確実。
- ノードの文字には `white-space: nowrap` を付けると、日本語が 1 文字ずつ折り返して縦長になるのを防げる。ただし幅が足りないとはみ出すので、狭い画面での表示を確かめる。
