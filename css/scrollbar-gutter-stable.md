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

`scrollbar-gutter: stable both-edges` にすると左右に同じ幅を空けるので中身が中央に保たれるが、スクロールバーのない左側にも常に 15px の空きができ、中身が 1 本分狭くなる。ページの左右の余白が 16px の狭い画面では、余白が左右とも 31px になる。右だけ空ける `stable` を選び、中身が左に寄る点は次の節の方法で直した。

ダイアログやセレクトの中のスクロール領域には付けなかった。付けると、スクロールしない内容でも右だけ余白が広くなり、左右が非対称になるため。

## 空けた分で中身が左に寄るのを、スクロールバーを右の余白に収めて防ぐ

右だけ空ける `stable` では、中身はスクロールバーを除いた幅の中で中央に置かれるため、画面全体が 7.5px 左に寄って見える。ページにはもともと左右の余白（ここでは 16px）があるので、スクロールバーをその右の余白の中に収め、中身は画面の中央に置くようにした。スクロールバーが中身に重なる環境（スマートフォン・macOS の既定）と同じ見た目にするということ。

CSS だけではスクロールバーの幅を取り出せない（ルートに `scrollbar-gutter: stable` があると `100vw` もスクロールバーを除いた幅になる）ので、JS で測って CSS 変数に入れる。

```ts
export function watchScrollbarWidth(): void {
  const root = document.documentElement;
  const measure = () => {
    const probe = document.createElement("div");
    probe.style.cssText =
      "position:absolute;top:0;left:0;width:100px;height:100px;overflow:scroll;visibility:hidden;pointer-events:none";
    root.appendChild(probe);
    const width = `${probe.offsetWidth - probe.clientWidth}px`;
    probe.remove();
    if (root.style.getPropertyValue("--scrollbar-width") !== width) {
      root.style.setProperty("--scrollbar-width", width);
    }
  };
  measure();
  // ズームを変えるとスクロールバーの幅（CSS の px）が変わるため、測り直す
  window.addEventListener("resize", measure);
}
```

`overflow: scroll` の要素の `offsetWidth - clientWidth` がスクロールバーの幅になる。スクロールバーが重なる環境では 0px になる（Playwright の `isMobile: true` で 0px を確認）。

ページの中身を包む要素をその幅だけ右へ広げ、広げた分を `body` で切り取る。

```css
:root {
  /* 余白より太いときは、中身がスクロールバーの下に入らないよう余白の分で止める */
  --scrollbar-overlap: min(var(--scrollbar-width, 0px), var(--space-4));
}

html {
  scrollbar-gutter: stable;
  overflow-x: hidden;
}

body {
  overflow-x: clip;
}

/* ヘッダー・本文・フッターを包む要素 */
.page {
  margin-right: calc(-1 * var(--scrollbar-overlap));
}
```

`--space-4` はページの左右の余白（16px）。幅 400px・1280px の画面で、ページの中身の左右の余白を測った結果は次のとおり（Chromium 159、スクロールバーは 15px）。

| 画面の幅 | 変更前（左・右） | 変更後（左・右） |
| --- | --- | --- |
| 400px | 16px・31px | 16px・16px |
| 1280px | 249px・264px | 256px・256px |

### `body` に負の margin を付けると、横にスクロールできてしまう

最初は `body` に `margin-right` を負にして広げた。見た目は中央に揃ったが、`body` が viewport からはみ出し、`document.documentElement.scrollWidth` が `clientWidth` より 15px 大きくなった。`html` の `overflow-x: hidden` でスクロールバーは出ないものの、`window.scrollTo(100, 0)` を呼ぶと `scrollX` が 15 になり、画面全体が横にずれた。

広げるのは `body` の内側の要素にし、`body` の `overflow-x: clip` で切り取る。`clip` で切り取ったはみ出しは、外側のスクロールできる幅に数えられないため、`scrollWidth` と `clientWidth` が一致し、`scrollTo(100, 0)` でも動かなかった。

`body` を `overflow-x: hidden` にしない理由は、`hidden` だと `overflow-y` が `auto` として扱われ、`body` 自身がスクロールできる箱になり、広げた分だけ横にずらせてしまうため（`clip` ならスクロールできる箱にならない）。また、仕様では `html` の `overflow` が `visible` のとき `body` の `overflow` が viewport に移るので、`html` には `overflow-x: hidden` を付けておく。

`overflow-x: clip` に対応していない古いブラウザ向けに `overflow-x: hidden;` を前に重ねて書くと、Biome の `lint/suspicious/noDuplicateProperties` で止められた。古いブラウザでも `html` の `hidden` で横のスクロールバーは出ないので、`clip` だけにした。

### 画面に固定した要素は、スクロールバーを除いた幅が基準になる

`position: fixed` の要素は、スクロールバーを除いた範囲を基準に置かれる。`inset: 0` のダイアログの背景は 0〜1265px になり、その中で中央に置いたダイアログの中心は 632.5px で、ページの中身（640px）と 7.5px ずれた。

固定した要素も同じ幅だけ右へ広げると揃う。

```css
/* ダイアログの背景 */
.overlay {
  position: fixed;
  inset: 0;
  right: calc(-1 * var(--scrollbar-overlap));
}

/* 画面の下の中央に固定するバナー */
.banner {
  position: fixed;
  left: calc((100% + var(--scrollbar-overlap)) / 2);
  transform: translateX(-50%);
  width: min(560px, calc(100% + var(--scrollbar-overlap) - 32px));
}
```

ダイアログの中心は 640px になり、バナーも左右の余白が揃った（幅 400px で 16px・16px）。右下に固定するトーストのように、画面の端を基準に置くものはそのままにした。

react-aria のダイアログを開くと `html` の `style` に `scrollbar-gutter: stable; overflow: hidden;` が足されるが、JS で入れた `--scrollbar-width` は残ったまま（`--scrollbar-width: 15px; scrollbar-gutter: stable; overflow: hidden;`）で、競合しなかった。

### Next.js では最初の描画の前に `<head>` で測る

描画のあとに測ると、最初の描画では中身が左に寄り、測ったあとに右へ動いてしまう。App Router のルートレイアウトの `<head>` に、測る関数をそのまま `<script>` で書いた。

```tsx
const watchScrollbarWidthScript = `(${watchScrollbarWidth.toString()})();`;

// ルートレイアウト
<html lang="ja" suppressHydrationWarning>
  <head>
    <script
      // biome-ignore lint/security/noDangerouslySetInnerHtml: 固定の文字列で、利用者の入力を含まない
      dangerouslySetInnerHTML={{ __html: watchScrollbarWidthScript }}
    />
  </head>
  <body>{/* ... */}</body>
</html>
```

- `Function.prototype.toString()` で関数のソースを文字列にするので、関数の中では外側の変数や import を使わない。`next build` の圧縮後のソースも、変数名が `a`・`b` に変わるだけで関数の中だけで完結していた。
- スクリプトが `<html>` の `style` 属性を書き換えるので、ハイドレーションの不一致の警告を避けるため `<html>` に `suppressHydrationWarning` を付けた（付けない場合に警告が出るかは確かめていない）。
- Vite の SPA のように、画面を JS で描く場合は、描画の前（`createRoot(...).render()` の前）に関数を呼べば間に合う。

狭い画面でズームを小さくすると、スクロールバーの幅（CSS の px）が余白の 16px を超えうる。そのときに中身がスクロールバーの下に入らないよう `min()` で余白の分に止めたが、ズームを変えたときの動きは実際には試していない。
