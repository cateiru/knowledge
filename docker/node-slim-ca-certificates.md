# node の slim イメージで wrangler dev（workerd）が起動しない

`node:<version>-slim` のイメージで Next.js（OpenNext）の開発サーバーを動かしていたところ、wrangler を上げたら全ページが 500 になった。2026年11月、wrangler 4.151 から 4.156 に上げたときに起きた。

## 症状

workerd が次のメッセージを出して起動に失敗し、Cloudflare のコンテキストを取る処理（`getCloudflareContext`）がエラーになる。

```text
No CA certificates were loaded from the system
```

## 原因

slim イメージには `ca-certificates` が入っていない。新しい workerd はシステムの CA 証明書を読みにいき、見つからないと起動しない。

## 対処

動いているコンテナに入れて再起動すれば直る。

```sh
docker exec <container> sh -c 'apt-get update -qq && apt-get install -y -qq ca-certificates'
docker restart <container>
```

ただし、これは動いているコンテナに入れただけなので、イメージを作り直すと再発する。恒久的には Dockerfile に入れておく。

```dockerfile
RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates \
  && rm -rf /var/lib/apt/lists/*
```
