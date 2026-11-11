# Server Action のあとに遷移しても、共有のレイアウトが描き直されない

App Router で、レイアウトがデータ（例：猫の毛色）を読んで表示を決めていると、編集ページで保存したあとに詳細ページへ移っても、レイアウトだけ古いままになることがあった。2026年11月に Next.js 16.4 の開発サーバーで確かめた。

## 起きたこと

- `/cats/[catId]/layout.tsx` が猫を読み、配下のページ全体のアイコンの色を決めていた。
- 編集ページ（`/cats/[catId]/edit`）の Server Action で保存し、クライアントで `router.replace("/cats/[catId]")` で詳細へ移った。
- 詳細ページ（page）は新しい値で描かれたが、レイアウトは古い値のままだった。編集と詳細でレイアウトを共有しているため、遷移では描き直されない。
- 同じ保存でも、`router.back()` で戻ったあとに `router.refresh()` する流れでは、レイアウトも新しくなった。refresh はルート全体を描き直すため。

## 対処

Server Action の中で、レイアウトを含めて再検証する。

```ts
"use server";

import { revalidatePath } from "next/cache";

export async function updateCatAction(id: string, formData: FormData) {
  // 保存する
  revalidatePath(`/cats/${id}`, "layout");
  return { redirectTo: `/cats/${id}` };
}
```

- 第 2 引数を `"layout"` にすると、そのパス以下のレイアウトも対象になる。
- この行を外すと古い値のまま、入れると新しい値になることを、両方の遷移で確かめた。

## 気をつけること

- ページだけを見て動作確認すると気づきにくい。レイアウトで値を使うようにしたときは、保存したあとの遷移を通して、レイアウトの表示も変わるかを確かめる。
- `router.back()`・`router.replace()` のどちらで遷移するかは、直前の履歴によって変わる作りだった。片方の流れだけで確かめると見落とす。
