# Node.js の型ストリップで、拡張子なしで import し合う .ts を読む

2026年11月に、Node.js 26 で確かめた。

## 何が困るか

Node.js は `.ts` を型だけ取り除いてそのまま実行できる（type stripping）。ただし import の解決は ESM のままなので、`export * from "./users"` のように拡張子を付けずに書いたファイル（バンドラー向けの書き方）は `ERR_MODULE_NOT_FOUND` になる。

## 対処

`node:module` の `registerHooks` で、見つからなかった相対パスの import だけ `.ts` を付けて解決し直す。

```ts
import { registerHooks } from "node:module";
import path from "node:path";

registerHooks({
  resolve(specifier, context, nextResolve) {
    try {
      return nextResolve(specifier, context);
    } catch (error) {
      if ((specifier.startsWith("./") || specifier.startsWith("../")) && path.extname(specifier) === "") {
        return nextResolve(`${specifier}.ts`, context);
      }
      throw error;
    }
  },
});

// フックを登録してから読み込むため、動的 import にする
const schema: typeof import("./schema/index.ts") = await import("./schema/index.ts");
```

## 注意すること

- 対象のファイルを静的な `import` で読むと、フックを登録する前にモジュールのグラフ全体が解決されるため、同じファイルの先頭でフックを登録しても効かない。動的 `import()` にするか、`node --import <フックを登録するファイル>` で先に読み込む。
- 型だけを使う import は `import type` で書かれている必要がある（型ストリップでは消えない名前付き import が、実行時に存在しない export を探して失敗する）。
- `tsx` などのローダーを入れれば同じことができるが、依存を増やしたくないときはこの方法で足りる。
