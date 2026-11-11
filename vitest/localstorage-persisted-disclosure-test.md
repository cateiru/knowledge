# 開閉状態を localStorage に保存する部品を Vitest でテストする

2026年11月に、react-aria-components の `Disclosure` を包んだ折りたたみの部品に、開閉状態を `localStorage` に保存する機能（`storageKey`）を付けたときに確かめたこと。テストは Vitest + Testing Library + jsdom で書いている。

## 前のテストの開閉状態が次のテストに残る

jsdom の `localStorage` は、同じテストファイルの中ではテストをまたいで残る。Testing Library の `cleanup()` は DOM を片付けるだけで、`localStorage` は消さない。

そのため、次のような流れで失敗する。

1. あるテストでグラフを開くと、`localStorage` に「開いている」が保存される。
2. 次のテストでは、部品が最初から開いた状態で描かれる。
3. そのテストで「開く」つもりで見出しを押すと、逆に閉じてしまう。
4. 閉じたパネルの中の要素が見つからず、テストが失敗する。

実際に `localStorage` を消さずに動かすと、次のように失敗した。どのテストが落ちるかは、それまでのテストで何回開閉したかによって変わる。

```text
TestingLibraryElementError: Unable to find an accessible element with the role "radiogroup" and name "表示期間"
```

## 対処

テストごとに `localStorage` を消す。

```tsx
afterEach(() => {
  cleanup();
  // 開閉状態は localStorage に保存されるため、テストごとに初期状態へ戻す
  window.localStorage.clear();
});
```

同じ部品を使うコンポーネントのテストすべてに入れる必要がある。共通の setup ファイル（`vitest.setup.ts` など）に入れてもよいが、`localStorage` を前提にした別のテストに影響しないか確認してから入れる。

保存されることそのものは、一度描いて開き、`unmount()` してからもう一度描くとテストできる。

```tsx
it("開いた状態は保存され、次に表示したときも開いている", () => {
  const { unmount } = render(<Chart {...props} />);
  fireEvent.click(screen.getByRole("button", { name: "<見出し>" }));
  unmount();

  render(<Chart {...props} />);

  expect(screen.getByRole("button", { name: "<見出し>" })).toHaveAttribute(
    "aria-expanded",
    "true",
  );
});
```

## 閉じたパネルの中身は getByRole で見つからない

react-aria-components の `DisclosurePanel` は、閉じていても中身を DOM に描く。ただし、閉じている間はアクセシビリティツリーから外れるため、`getByRole` では見つからない。上の失敗も、閉じたパネルの中の `radiogroup` を探したために起きている。

もともと折りたたみのなかった部品を折りたためるようにしたときは、中の要素を `getByRole` で探すテストすべてに、先に見出しを押して開く操作を足す。

## 補足：保存した開閉状態でページを開いたときのずれ

開閉状態を `localStorage` に保存すると、サーバーでは保存した状態が分からない。そのため、`useEffect` で読み込んで切り替えると、開いたまま保存した人だけ、読み込み後に下の内容が押し下げられる。

2026年11月に取った対処は次のとおり。

1. サーバーでは、開いた状態のまとまりを `visibility: hidden` で描き、開いた高さを確保しておく。
2. `useLayoutEffect` で、描画前に保存した状態へ切り替える。
3. 見えるようにするのは `requestAnimationFrame` で次のフレームにする。開いた状態から閉じた状態へ切り替えたとき、矢印の回転のトランジションが見えないようにするため。

Playwright で `PerformanceObserver` の `layout-shift` を集めて確かめた。開いたまま保存した場合のシフトは 0 だった。一方、閉じている場合は確保した高さが縮み、下の内容が持ち上がるシフトが残る。どちらの状態をずれなしにするかは、確保する高さで選ぶことになる。

どちらでもずれをなくしたいときは、開閉状態を cookie に保存し、サーバーで読んで最初から正しい状態で描く必要がある。
