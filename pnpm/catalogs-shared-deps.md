# モノレポで共通の依存のバージョンを pnpm の catalogs でまとめる

2026年11月に、ルートのアプリとサブパッケージ（`packages/<name>`）の両方で使っている依存のバージョンを、[pnpm の catalogs](https://pnpm.io/ja/catalogs) にまとめた。pnpm 12 で確かめた。

## なぜまとめるのか

同じパッケージを複数の `package.json` に書くと、指定がずれていく。実際に、片方が `^0.45.2`、もう片方が `^0.45.4` のようにずれていた。catalog に書けば、バージョンを決める場所が 1 つになる。

## 手順

1. 両方の `package.json` を見比べて、共通のパッケージを洗い出す（`dependencies` と `devDependencies` の両方）。
2. `pnpm-workspace.yaml` に既定の catalog を書く。指定がずれていたものは、どちらかに揃える（解決済みのバージョンを満たす方にすると lockfile が変わらない）。

   ```yaml
   packages:
     - "packages/*"

   catalog:
     "@types/node": ^26.0.0
     typescript: ^5
     vitest: ^5.0.0
     wrangler: ^4.135.0
   ```

3. 各 `package.json` では、バージョンの代わりに `"catalog:"` と書く。

   ```json
   "devDependencies": {
     "typescript": "catalog:",
     "vitest": "catalog:"
   }
   ```

4. `pnpm install` で lockfile を更新し、`pnpm install --frozen-lockfile` が通ることを確かめる。

## lockfile の変化

`pnpm-lock.yaml` には先頭に `catalogs:` の節が増え、各 importer の `specifier` が `catalog:` に変わるだけだった。解決されるバージョンは変わらなかった。差分は次のように確かめられる。

```sh
git diff pnpm-lock.yaml | grep '^[-+]' | grep -v specifier
```

`catalogs:` の節しか出てこなければ、解決済みのバージョンは変わっていない。

## 注意

- 片方だけで使うパッケージは catalog に入れなかった。全部を catalog に寄せる方針もありうる。
- 今後両方で使うパッケージを足すときは、catalog に追加することを README などに書いておく。
- Renovate は pnpm の catalog に対応していて、更新の PR は `pnpm-workspace.yaml` を書き換える形になる（Renovate のドキュメントによる。この変更のあとの更新 PR はまだ見ていない）。
- サブパッケージの変更のときだけ CI やデプロイを動かすパスの絞り込みをしている場合は、`pnpm-workspace.yaml` も対象に入れる。catalog の変更はサブパッケージの依存も変えるため。
