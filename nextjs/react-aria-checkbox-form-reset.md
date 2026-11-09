# form action で保存すると react-aria のチェックボックスが元に戻る

React 19 の `<form action={...}>` で保存したあと、react-aria の `Checkbox` が、ページを開いたときの状態に戻って見えることがある。保存自体は成功していて、再読み込みすると正しい状態が表示される。2026年11月に Next.js（App Router）と react-aria-components 1.22 で起き、ユニットテストと E2E で原因と直し方を確かめた。

## 原因

2 つの仕様が重なって起きる。

- React 19 は、`action` に関数を渡したフォームを、Action が終わったあとに自動で reset する（`form.reset()` を呼ぶ）。
- react-aria の `useToggle`（`Checkbox`・`Switch` が使う）は `useFormReset` でフォームの `reset` イベントを受け、マウントしたときの値で `setSelected` を呼ぶ。`setSelected` は `onChange` を呼ぶ。

そのため、`isSelected` と `onChange` で親の state から制御しているチェックボックスでも、reset のたびに `onChange(マウント時の値)` が呼ばれ、親の state が書き戻される。「制御しているから reset の影響は受けない」とは考えないほうがよい。

## 直し方

`action` ではなく `onSubmit` で送信を受け、既定の送信を止めてから、`startTransition` の中で Action を呼ぶ。React が自動で reset するのは `action` 経由で送ったときだけなので、reset が起きなくなる。

```tsx
"use client";

import { type FormEvent, startTransition, useActionState } from "react";

export function SettingsForm({ action }: Props) {
  const [state, formAction, isPending] = useActionState(action, {});

  const handleSubmit = (event: FormEvent<HTMLFormElement>) => {
    event.preventDefault();
    const formData = new FormData(event.currentTarget);
    startTransition(() => {
      formAction(formData);
    });
  };

  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
}
```

`startTransition` の中で呼ぶので、`isPending` は `action` で送ったときと同じように使える。

注意する点:

- `onSubmit` だけにすると、JavaScript が読み込まれる前（ハイドレーション前）に送信したときは、ふつうの GET の送信になる。チェックボックスの状態をクライアントの state で持つフォームなら、もともと JavaScript がないと動かないので問題になりにくい。
- 保存に成功したら別のページへ遷移するフォームでは、この症状は見えにくい。ただし入力エラーで同じページに戻ったときも reset は起きるので、同じようにチェックが戻る（こちらは未対応で、試していない）。

## E2E で確かめるときの注意

チェックが戻るのは、保存の完了のトーストが出た直後ではない。`revalidatePath` などによる再検証まで含めたトランジションが確定したときに reset が起きる。実際に測ったところ、トーストが出てから約 1.5 秒後にチェックが外れた。

トーストが出た直後にチェックを確かめると、まだ戻る前なので、不具合があってもテストが通ってしまう。送信中の表示が終わる（「保存する」ボタンが押せる状態に戻る）のを待ってから確かめる。

```ts
await page.getByRole("button", { name: "保存する" }).click();
await expect(page.getByText("保存しました")).toBeVisible();
// 再検証が終わり、送信中の表示が消えるまで待つ
await expect(page.getByRole("button", { name: "保存する" })).toBeEnabled();
await expect(checkbox).toBeChecked();
```

## ユニットテストで再現する

jsdom（Vitest と Testing Library）でも、`form.reset()` で `reset` イベントが発火するので再現できる。チェックを変えて送信し、送信中の表示が終わってからチェックの状態を確かめる。修正前はこのテストが失敗し、修正後は通ることを確かめた。

```tsx
const action = vi.fn(async () => ({}));
render(<SettingsForm action={action} />);
const checkbox = screen.getByRole("checkbox", { name: "<label>" });

fireEvent.click(checkbox);
await act(async () => {
  fireEvent.click(screen.getByRole("button", { name: "保存する" }));
});
await waitFor(() =>
  expect(screen.getByRole("button", { name: "保存する" })).toBeEnabled(),
);
expect(checkbox).toBeChecked();
```

## チェックボックスが多いときの副作用

修正前は、数百個のチェックボックスがあるフォームで、reset のときに全部の `onChange` が一斉に呼ばれ、「保存中...」の表示が 5 秒以上続いた。チェックボックスの数が多いフォームで、保存後に送信中の表示が長引くときも、この reset を疑うとよい。
