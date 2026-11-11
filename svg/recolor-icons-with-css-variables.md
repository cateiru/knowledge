# インライン SVG のアイコンの一部の色を、祖先の CSS 変数で差し替える

インライン SVG で描いたイラストアイコン（24px グリッド）のうち、猫の毛の塗りだけを、ページごとに別の色や SVG パターンへ差し替えたときのメモ。2026年11月に Chromium（Playwright のヘッドレスシェル）で確かめた。

## 塗りは class の CSS で `var()` にする

差し替えたい形に class を付け、CSS で `fill` を CSS 変数にする。既定の色は `var()` のフォールバックに書く。

```css
.coat {
  fill: var(--icon-coat, #f5dfb7);
}
```

```tsx
<path className={styles.coat} fill="#f5dfb7" d="..." />
```

- CSS の宣言は、SVG の presentation 属性（`fill="..."`）より優先される。属性には同じ既定の色を残しておくと、CSS が読み込まれない場面でも同じ色になる。
- 色を変えたい範囲の祖先の要素に `style={{ "--icon-coat": "#9a7650" }}` を付ければ、その中のアイコンだけが変わる。アイコンのコンポーネントに props を足さなくて済む。
- presentation 属性の値に直接 `var()` を書く方法は試していない。確実に効く CSS の宣言にした。
- 単色表示用の `.monochrome .fills > * { fill: currentColor; }` のように、詳細度が高い指定が別にあれば、そちらがそのまま優先される。

## フォールバックなしの `var()` で「未設定なら親から継承」にする

目・口の線の色も同じように差し替えた。

```css
.feature {
  stroke: var(--icon-feature);
}
```

`--icon-feature` が未設定だと、この宣言は「計算値の時点で無効（invalid at computed-value time）」になり、`unset` と同じ扱いになる。`stroke` は継承するプロパティなので、親の `<svg>` の `stroke` をそのまま継承する。既定の色をここに書き写さずに済む。

## CSS 変数から別の SVG の `<pattern>` を参照する

1 色ではなく斑（まだら）の模様で塗りたいときは、変数の値を `url(#id)` にする。

```tsx
style={{ "--icon-coat": `url("#coat-<id>") #4a4442` }}
```

- `url()` のあとの色は、パターンが見つからないときのフォールバック。
- 参照先の `<pattern>` は、同じ文書の中なら別のインライン SVG に置いてよい。
- パターンを置く SVG を `display: none` にすると、参照できないことがある。大きさ 0 の絶対配置にして隠す。

```css
.defs {
  position: absolute;
  width: 0;
  height: 0;
  overflow: hidden;
}
```

- id は、CSS の `url()` にそのまま書ける文字列にする。React の `useId()` の値は記号を含むので避け、データの ID から組み立てた。

## 塗る形ごとに、正方形の絵を外接矩形いっぱいに敷く

顔・胴・前足のように大きさや縦横比が違う形を、同じ絵（64×64 で描いた見本）で塗りたかった。次の指定にすると、形ごとの外接矩形に絵が 1 枚だけ、縦横の比を保ったまま覆うように敷かれる。

```tsx
<pattern
  id="coat-<id>"
  patternUnits="objectBoundingBox"
  width="1"
  height="1"
  viewBox="0 0 64 64"
  preserveAspectRatio="xMidYMid slice"
>
  {/* 64×64 の座標で描いた絵 */}
</pattern>
```

- `viewBox` があると `patternContentUnits` は無視され、中身は `viewBox` の座標で描ける。
- `slice` なので、横長の形では絵の上下が少し切れる。

## 見た目の判断

- 24px 程度のアイコンに縞・斑点などの柄まで入れると、線が込み入って見づらくなった。ベース色だけで塗るほうが良いと判断した。
- 輪郭や目・口を濃い色の線で描いているアイコンでは、黒・こげ茶などの濃い塗りで目・口が沈む。塗りを少し明るくし、目・口を明るい色で描いた。
- ただし白い模様の上に目・口が重なると、明るい色では逆に消えた。白い模様を塗りに入れる場合は、白の割合が多いときに目・口を明るくしない条件が必要になる。
