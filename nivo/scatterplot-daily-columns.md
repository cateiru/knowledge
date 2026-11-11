# Nivo の ScatterPlot で、日ごとの列に時刻の点を並べる

「横軸に日付（1 日 1 列）、縦軸に時刻」の散布図を `@nivo/scatterplot`（0.99）で描いたときのメモ。2026年11月に試した。

## x を「日の番号」の linear スケールにする

期間の日を古い順に `0..N-1` と番号を振り、点の x を「その日の番号 + 0.5」にする。こうすると点が列の中央に並び、列の境界は整数になる。

```tsx
<ResponsiveScatterPlot<Datum>
  data={[{ id: "records", data: points.map((p) => ({ x: p.dayIndex + 0.5, y: p.hour })) }]}
  xScale={{ type: "linear", min: 0, max: dayCount, nice: false }}
  yScale={{ type: "linear", min: 0, max: 24, reverse: true, nice: false }}
  enableGridX={true}
  gridXValues={gridIndexes} // 列の境界（整数）
  axisBottom={{
    tickValues: tickIndexes, // 列の中央（i + 0.5）や、列の左端（i）
    format: (value) => formatDate(dayStarts[Math.floor(Number(value))]),
  }}
/>
```

- `min`・`max` を明示し、`nice: false` にする。linear スケールは `nice` が既定で効き、範囲が切りのよい値まで広がることがある。範囲を決めておくと、記録のない端の日も列として描かれる。
- 目盛りの文字は `axisBottom.format` で番号から日付に変換する。番号を `Math.floor` すると、列の中央（`i + 0.5`）でも左端（`i`）でも同じ日を引ける。
- 日の区切りや「何日目か」の計算は、描画とは別の純粋関数にしておくと単体テストが書きやすい。

time スケールを使わなかったのは、点を列の中央に置くために日時を加工する必要があり、`precision` による丸めや `useUTC` の扱いも考えることになるため。time スケールで実際に試して比べたわけではない。

## 既定のツールチップは番号を表示する

x を番号にすると、既定のツールチップには `x: 3.5, y: 8.2` のような値が出る。`tooltip` に自前のコンポーネントを渡し、datum に持たせた日時などを表示する。

```tsx
function Tooltip({ node }: ScatterPlotTooltipProps<Datum>) {
  return <div>{node.data.label}</div>;
}
```

## 期間が長いときは区切り線を間引く

半年（180 列）をスマートフォンの幅（グラフの幅が 300px 前後）で描くと、1 列が 2px 前後になる。毎日の線を引くと塗りつぶしと見分けがつかないため、1 週間の表示だけ毎日の線を引き、3 か月・半年は週の区切り（月曜）だけに線を引いた。点の大きさも、期間が長いほど小さくした。

## Playwright でツールチップを確認するときは、グラフの中の点を選ぶ

`page.locator("svg circle").first()` は、グラフより前にあるアイコンの `circle` に一致し、ツールチップが出なかった。グラフを包む要素の中に絞って点を選び、その中心へマウスを動かすと表示された。

```ts
const box = await page.locator("<グラフを包む要素> svg circle").nth(3).boundingBox();
await page.mouse.move(box.x - 20, box.y - 20);
await page.mouse.move(box.x + box.width / 2, box.y + box.height / 2, { steps: 10 });
```

## 日付に依存するグラフの VRT

「今日」から期間を切るグラフは、実行日によって軸と点が変わる。固定の「今日」と固定の記録を渡してグラフだけを描く開発用ページ（本番では `notFound()` にする）を作り、VRT ではそのページを撮った。
