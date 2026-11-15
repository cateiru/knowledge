# node-build の定義が古いとき、Node.js を nodenv に手で入れる

プロジェクトの `.node-version` が新しいパッチ版に上がったのに、パッケージマネージャーで入れた `node-build` の定義が追いついておらず、`nodenv install <version>` で入れられなかった。2026年10月に、nodejs.org の配布物を手で展開して入れた。`sudo` は要らない。

## 手順

配布物とチェックサムを取ってきて、検証する。

```sh
v=<version>   # 例: 24.22.1
cd "$(mktemp -d)"
curl -fsSLO "https://nodejs.org/dist/v$v/node-v$v-linux-x64.tar.xz"
curl -fsSLO "https://nodejs.org/dist/v$v/SHASUMS256.txt"
grep " node-v$v-linux-x64.tar.xz\$" SHASUMS256.txt | sha256sum -c -
```

`OK` と出たら、nodenv のバージョンのディレクトリに展開する。

```sh
mkdir -p "$(nodenv root)/versions/$v"
tar -xJf "node-v$v-linux-x64.tar.xz" -C "$(nodenv root)/versions/$v" --strip-components=1
nodenv rehash
nodenv versions
```

- `--strip-components=1` で、配布物の一番上の `node-v<version>-linux-x64/` を外す。`versions/<version>/bin/node` になれば nodenv から使える。
- corepack で入れた pnpm も、このまま動いた。

## 一時的にしのぐ

入れる前に手元のほかのバージョンで動かしたいときは、環境変数で一時的に切り替える。

```sh
NODENV_VERSION=<installed-version> pnpm test
```

メジャーバージョンが同じなら、たいていはこれで動いた。ただし、そのバージョンに pnpm の shim がない（corepack を有効にしていない）と `pnpm` が見つからない。
