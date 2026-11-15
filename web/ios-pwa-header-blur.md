# iOS のホーム画面アプリで、ヘッダーがぼやける

iOS 27 で、ホーム画面に追加した PWA（`display: standalone`）の上端の固定ヘッダーがぼやけて見えるという報告があった。2026年11月に調べた。

## 原因

WebKit のスクロールの端の効果（soft な scroll edge effect）が、ステータスバーの下にぼかしをかけている。

WebKit のソース（`LocalFrameView::fixedContainerEdges`、`WKWebViewIOS.mm` の `_shouldHideTopScrollPocket`）を読むと、上端の 4px の中央に、幅が 90% 以上の fixed または sticky の要素があると、その要素の色でステータスバーの下を塗り、ぼかしを出さないようになっている。

## 対処

standalone のときだけ、上端に見えない固定要素を置く。[Qiita の記事](https://qiita.com/na-trium-144/items/0add98a80ca2391e3f17)で紹介されていた方法。要点は高さ 11px と `background-clip: text` で、ほかの指定は形を示すための例。

```css
@media (display-mode: standalone) {
  .status-bar-edge {
    position: fixed;
    inset: 0 0 auto 0;
    height: 11px;
    background-color: var(--header-bg);
    background-clip: text; /* 色は WebKit に読ませるが、見た目には出さない */
    pointer-events: none;
    z-index: 100;
  }
}
```

- 高さが 10px 以下だと、WebKit は `background-color` を使わずに描画結果から色を読み取る処理に回るので、結果が不確実になる。
- この方法は実機ではまだ確かめていない（ソースを読んで判断した段階）。

## 確かめ方の注意

Chromium の DevTools Protocol では `display-mode` をエミュレートできない。デスクトップのブラウザで見た目を確かめたいときは、スクリプトで該当する `CSSMediaRule` の `media.mediaText` を `all` に書き換えた。
