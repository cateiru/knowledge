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
