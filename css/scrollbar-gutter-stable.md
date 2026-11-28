# スクロールバーの出し入れで横幅がずれるのを scrollbar-gutter で防ぐ

Windows の PC など、スクロールバーが場所を取る環境では、短いページ（スクロールバーなし）と長いページ（スクロールバーあり）を行き来したり、折りたたみを開いてページが伸びたりするたびに、横幅がスクロールバー 1 本分（Chrome では 15px）変わり、中身が左右にずれる。

`html` に `scrollbar-gutter: stable` を付けると、スクロールできないページでもスクロールバーの分の幅を空けておくため、横幅が変わらなくなる。スクロールバーが中身に重なる環境（スマートフォン・macOS の既定の設定）では何も変わらない。

```css
html {
  scrollbar-gutter: stable;
}
```

以下は 2026年11月に Chromium 159 で確かめた内容。

## `max-width: 100vw` と一緒に使わない

Next.js の雛形の globals.css には次の指定がある。

```css
html,
body {
  max-width: 100vw;
  overflow-x: hidden;
}
```

この `max-width: 100vw` を残したまま `html` に `scrollbar-gutter: stable` を付けると、スクロールできるページで横幅がさらに 1 本分狭くなった。幅 1280px の画面で測った `body` の幅は次のとおり。

| 指定 | スクロールなし | スクロールあり |
| --- | --- | --- |
| どちらもなし（変更前） | 1280px | 1265px |
| `scrollbar-gutter: stable` ＋ `max-width: 100vw` | 1265px | 1250px |
| `scrollbar-gutter: stable` だけ | 1265px | 1265px |

ルートに `scrollbar-gutter: stable` があると、Chrome では `100vw` がスクロールバーの分を除いた幅（1265px）になる。スクロールバーが出ているときは、そこからさらに 1 本分引かれた値で `max-width` が効いているように見えた（なぜ二重に引かれるのかまでは確かめていない）。

`overflow-x: hidden` があれば横のはみ出しは切り取られるので、`max-width: 100vw` は外してよい。

## 確かめるときは `body` の幅を見る

`document.documentElement.clientWidth` は、スクロールバーが出ていないときは空けた分を数えず、`scrollbar-gutter: stable` を付けても 1280px のままだった。実際に中身が使える幅は `document.body.getBoundingClientRect().width` などで測る。

## Playwright で確かめる

Playwright はヘッドレスの Chromium を `--hide-scrollbars` 付きで起動する。スクロールバーが場所を取る状態で試すには、この引数を外す。

```js
const browser = await chromium.launch({
  ignoreDefaultArgs: ["--hide-scrollbars"],
});
```

## VRT にはほぼすべての画像に差分が出る

`--hide-scrollbars` はスクロールバーを描かないだけで、`scrollbar-gutter: stable` の幅は取る。そのため、ふだんの VRT（`--hide-scrollbars` 付き）でも、すべてのページで右端に 15px の空きができ、中央寄せの中身は約 7px、右端に寄せた要素は 15px 左へずれた。全画面（fullPage）で撮る VRT は、撮るときにスクロールバーが出ないので、変更前はどのページもスクロールバーのない幅で写っており、ほぼ全件が差分になる。

ただし、`isMobile: true` でスマートフォンを模したケースだけは差分が出なかった。模擬中はスクロールバーが中身に重なる方式になり、空きができない（実機と同じ）。PC 用の Chrome の幅を狭めてスマートフォンの代わりに撮っているケースは、実機と違って右端に空きができる点に注意する。

## react-aria のダイアログとは競合しない

react-aria（3.53.1 で確認）の `usePreventScroll` は、ダイアログを開くと `html` に `scrollbar-gutter: stable; overflow: hidden;` を直接付け、スクロールを止めたときに横幅が変わらないようにしている（`scrollbar-gutter` に対応していないブラウザでは `padding-right` で補う）。もともと `html` に `scrollbar-gutter: stable` が付いていれば、ダイアログを開いても閉じても横幅は変わらない。

## `stable both-edges` にしなかった理由

`scrollbar-gutter: stable both-edges` にすると左右に同じ幅を空けるので中身が中央に保たれるが、スクロールバーのない左側にも常に 15px の空きができ、中身が 1 本分狭くなる。右だけ空ける `stable` を選んだ。

ダイアログやセレクトの中のスクロール領域には付けなかった。付けると、スクロールしない内容でも右だけ余白が広くなり、左右が非対称になるため。
