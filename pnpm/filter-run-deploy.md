# pnpm --filter で deploy スクリプトを動かすには run が必要

モノレポのサブパッケージに `deploy` という npm スクリプト（`wrangler deploy` など）を定義して、次のように実行したら失敗した。2026年10月、pnpm 12 で確かめた。

```sh
pnpm --filter <package> deploy
```

```text
ERR_PNPM_INVALID_DEPLOY_TARGET
```

## 原因

`pnpm deploy` は pnpm の組み込みのコマンド（パッケージを依存ごと別のディレクトリに書き出すもの）で、同じ名前のスクリプトより優先される。`pnpm <name>` でスクリプトを呼べるのは、組み込みのコマンドと名前がぶつからないときだけ。

## 対処

`run` を付けて、スクリプトだと明示する。

```sh
pnpm --filter <package> run deploy
```

ドキュメントや README に `pnpm --filter <package> deploy` と書いていたら、`run` 付きに直しておく。`install`・`publish`・`pack` なども組み込みのコマンドと名前がぶつかるので、スクリプトを呼ぶときは `run` を付ける習慣にしておくと安全。
