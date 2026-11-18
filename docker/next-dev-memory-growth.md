# 長く動かしっぱなしの next dev がメモリを食い尽くす

Docker コンテナで `next dev` を何日も動かしっぱなしにしていたら、ホストのメモリとスワップがほとんどなくなっていた。そのときの調べ方と対処をまとめる。

## 起きたこと

2026年11月に起きた。

- Next.js 16 の `next dev`（Cloudflare の workerd を子プロセスとして使う構成）を、Docker Compose のコンテナで7日間動かしっぱなしにしていた。
- `next-server` プロセスの RSS が約 8.2GiB、スワップが約 1.7GiB になっていた。過去の最大値（`VmHWM`）は約 9.4GiB だった。
- ホストは物理メモリ 15GiB のうち 11GiB を使い、スワップは 4GiB のうち 3.9GiB を使っていた。
- コンテナにはメモリの上限を付けていなかった（`docker inspect` で `Memory` が `0`）。そのため、ホストのメモリをいくらでも使えた。

dev サーバーは、コンパイル結果や HMR のためのデータをメモリにためていく。長く動かすほどメモリが増えるのはそのためだと考えている（原因までは突き止めていない）。

## 調べ方

まずホスト全体の使用量を見る。

```sh
free -h
```

`used` と `Swap` の使用量が多く、`buff/cache` が少なければ、キャッシュではなくプロセスがメモリを使っている。

次に、RSS（実際に使っている物理メモリ）の多い順にプロセスを並べる。

```sh
ps -eo pid,rss,comm --sort=-rss | head -20
```

目立つプロセスがあれば、スワップに出ている量や過去の最大値も見る。

```sh
grep -E 'VmRSS|VmSwap|VmHWM' /proc/<pid>/status
ps -o pid,ppid,etime,lstart,args -p <pid>
```

`etime` を見ると、どのくらいの間動いているかがわかる。

そのプロセスがどのコンテナのものかは、cgroup かプロセスの親子関係から調べる。

```sh
cat /proc/<pid>/cgroup
pstree -ps <pid>
```

cgroup に `docker-<コンテナ ID>.scope` が出ていれば、そのコンテナのプロセス。親をたどって `containerd-shim` があるかでも判断できる。

最後に、コンテナ単位の使用量と設定を確かめる。

```sh
docker stats --no-stream --format '{{.Name}} {{.MemUsage}}'
docker inspect <container> --format '{{.Config.Cmd}} {{.HostConfig.Memory}}'
```

`HostConfig.Memory` が `0` なら、メモリの上限がない。

## 対処

コンテナを再起動すると、メモリはすぐに戻る。

```sh
docker restart <container>
```

実際にこれで、コンテナのメモリは 8.25GiB から 764MiB に下がった。ホストで使えるメモリも 4.4GiB から 12GiB に増えた。スワップに出ていた分はすぐには戻らないが、必要になったときに少しずつ戻るので放っておいてよい。

再起動すると dev サーバーが止まるので、作業中の画面は読み込み直しになる。データは消えない。

## 再発を防ぐ

ここからは試していない案。

- Compose のサービスにメモリの上限を付ける。上限を超えるとコンテナは OOM で止まるが、ホスト全体が重くなることはなくなる。

  ```yaml
  services:
    app:
      mem_limit: 4g
  ```

- 使わないときは `docker compose stop` で止めておく。
- 何日も動かし続ける場合は、ときどき `docker stats` で使用量を確かめる。
