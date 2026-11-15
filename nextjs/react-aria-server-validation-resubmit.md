# サーバーの入力エラーを表示した react-aria の入力欄で、直しても送信できない

## 症状

Server Action（`useActionState`）が返した入力エラーを、react-aria-components の `TextField` に `isInvalid`（または `errorMessage` から `isInvalid` を付ける自作の部品）で渡して表示する。エラーが出たあとに値を正しく直して送信ボタンを押しても、フォームが送信されない。Server Action も呼ばれない。

2026年11月に、`type="date"` の入力欄で確かめた。

## 原因

react-aria の `validationBehavior` は既定で `"native"` になっている。このとき `isInvalid` が `true` だと、react-aria は入力欄に `setCustomValidity` でエラーを設定する。

`isInvalid` はサーバーの結果（前回の送信の状態）から作っているため、値を直しても次の送信までは `true` のまま残る。その間は入力欄の `validity.valid` が `false` になり、ブラウザの検証が送信を止める。

Playwright で、値を直した直後の入力欄を調べると次のようになっていた。

```js
await input.evaluate((el) => [el.validity.valid, el.validationMessage, el.value]);
// [false, "Invalid value.", "2024-06-20"]
```

## 対処

サーバーで検証するフォームの入力欄では、`validationBehavior="aria"` を指定する。エラーは `aria-invalid` などで伝わるだけになり、ブラウザの検証で送信を止めなくなる。

```tsx
<FormField
  name="eventDate"
  type="date"
  isRequired
  validationBehavior="aria"
  errorMessage={state.fieldErrors?.eventDate?.[0]}
/>
```

指定したあとは、値を直した時点で `validity.valid` が `true` になり、送り直せることを確かめた。

## 注意

- `isRequired` のようなブラウザの必須チェックも、`"aria"` では送信を止めなくなる。必須かどうかはサーバー側で必ず検証する。
- 単体テストで Server Action を直接呼ぶテストでは再現しない。実際のブラウザで「エラーを出す → 直す → 送り直す」まで操作して確かめる。
