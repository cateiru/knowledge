# 全ページの title を「ページ名 | サイト名」にそろえる

App Router のアプリで、ブラウザのタブや履歴に出る title を全ページで「ページ名 | サイト名」の形にそろえたときの手順と注意点。2026年11月に Next.js 16 で確かめた。

## ルートレイアウトにテンプレートを置く

ルートレイアウトの `metadata`（または `generateMetadata` の戻り値）の `title` をオブジェクトにする。

```tsx
export const metadata: Metadata = {
  title: {
    default: "<サイト名>",
    template: "%s | <サイト名>",
  },
};
```

- 子のページは `title: "ページ名"` だけを書けばよい。`%s` に入って「ページ名 | サイト名」になる。
- title を書かないページや、`notFound()` で出る 404 は `default` の「サイト名」になる。
- ページに「ページ名 | サイト名」を直接書いたままにすると「ページ名 | サイト名 | サイト名」になる。テンプレートを入れるときは、既存の直書きから ` | サイト名` を外す。

## 名前を含む title は generateMetadata のヘルパーで作る

`/items/[id]/...` のように、見出しに対象の名前（例: 猫の名前）が入るページが多い場合、ページごとに `generateMetadata` を書くと同じコードが並ぶ。名前から title を作る関数を受け取ってヘルパーにまとめると、各ページは 1 行で済む。

```ts
export function itemPageMetadata(toTitle: (name: string) => string) {
  return async ({ params }: { params: Promise<{ id: string }> }): Promise<Metadata> => {
    const { id } = await params;
    const item = await getAccessibleItem(id);
    // 参照できなければ title を付けない（ページ本体が 404 にする）
    return item ? { title: toTitle(item.name) } : {};
  };
}
```

```ts
export const generateMetadata = itemPageMetadata((name) => `${name}の記録`);
```

- `params` に `id` 以外のキーがあるページ（`/items/[id]/records/[recordId]/edit` など）でも、`Promise<{ id: string; recordId: string }>` は `Promise<{ id: string }>` に代入できるので、そのまま使えた（`tsc` が通った）。
- 1 つのページで `metadata` と `generateMetadata` は併用できない。また `"use client"` のページには書けない。
- 取得の関数を React の `cache` で包むと、`generateMetadata` とページ本体の両方から呼んでも、1 回のリクエストの中では結果が共有される。Server Action からも呼ぶ関数は、更新のあとに古い値を返さないか考えてから包む。
- 子のレイアウトに別の `template` を置く方法は使わなかった。子の `template` は親のテンプレートを置き換えるうえ、同じセグメントの `page.tsx` には効かないため、名前を入れる用途には向かない。

## 一括置換で消しすぎないよう、テストで守る

既存の直書きから ` | サイト名"` を `sed` で一括で外したところ、同じ文字列を含むルートレイアウトの `template: "%s | <サイト名>"` まで `"%s"` に削ってしまった。型チェックも Biome も通るため、気づけたのはテンプレートの値を確かめる単体テストを足していたから。

- ルートレイアウトの `title` が期待どおりのオブジェクトであることをテストする。
- 全ページの `page.tsx` を読み、`export const metadata` か `generateMetadata` があること、` | サイト名` を直書きしていないことを確かめるテストを置くと、新しいページで title を付け忘れたときにも気づける。リダイレクトだけのページなどは除外リストに入れる。
- 最後に dev サーバーで、静的なページ・名前入りのページ・404 の `<title>` を curl で取り出して確かめた。
