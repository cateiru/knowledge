# curl で取った画像が gzip のまま保存される

2026年11月に、ストレージに置かれた VRT（reg-suit）のレポートの PNG を curl で取ったところ、画像として開けなかった。

## 症状

`curl -o <file>.png <URL>` で保存したファイルを `file` で見ると、PNG ではなく gzip と出る。

```text
<file>.png: gzip compressed data, from Unix, original size modulo 2^32 ...
```

## 原因

サーバーが `Content-Type: image/png` に加えて `Content-Encoding: gzip` を返していた。アップロード時に gzip で圧縮して置かれていたためとみられる（推測）。curl は既定では `Content-Encoding` を解かずに、受け取ったバイト列をそのまま保存する。ブラウザは自動で解くため、ブラウザでは普通に表示できる。

ヘッダーは次のように確かめられる。

```sh
curl -sI <URL> | grep -i 'content-type\|content-encoding'
```

## 対処

`--compressed` を付けると、curl が `Content-Encoding` を解いてから保存する。

```sh
curl -s --compressed -o <file>.png <URL>
file <file>.png  # PNG image data と出る
```

すでに保存してしまったファイルは、拡張子を `.gz` にしてから `gunzip` しても戻せる。

```sh
mv <file>.png <file>.png.gz
gunzip <file>.png.gz
```

## 画像以外も同じ

2026年11月に、同じレポートの `out.json` も `Content-Type: application/json` と `Content-Encoding: gzip` で返り、`--compressed` を付けずに取ると gzip のバイト列のまま表示された。レポートの中のファイルは、種類によらず `--compressed` を付けて取る。
