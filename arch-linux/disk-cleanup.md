# ルートパーティションの容量を空ける

Arch Linux でルート (`/`) の使用率が高くなったときに、何が容量を使っているかを調べて片付けた手順のメモ。

## 何が容量を使っているかを調べる

`-x` を付けると別のファイルシステム (NAS や `/tmp` など) を数えずに済む。

```sh
du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -15
du -xh --max-depth=2 /var 2>/dev/null | sort -h | tail -12
du -xh --max-depth=1 ~ 2>/dev/null | sort -h | tail -15
```

sudo なしで実行すると、root しか読めないディレクトリ (例: `/var/lib/docker`) は数えられない。`df` の使用量と `du` の合計がずれている場合は、その差の多くが読めなかったディレクトリにある。Docker は `docker system df` で確認できる。

## 容量を使っていたものと対処

| 場所 | 中身 | 対処 |
| --- | --- | --- |
| `/var/cache/pacman/pkg` | pacman がダウンロードした古いパッケージ | `paccache` で整理する (後述) |
| `/var/lib/ollama` | ollama のモデル | 使わなくなったのでパッケージごと削除 |
| `~/.cache` | yarn / yay / pnpm / playwright などのキャッシュ | 中身を削除 |
| `~/.local/share/pnpm/store` | pnpm のパッケージストア | 削除 |

### ollama の削除

```sh
sudo pacman -Rns ollama ollama-cuda ollama-vulkan
sudo rm -rf /var/lib/ollama
```

パッケージを消してもモデルの置き場所 (`/var/lib/ollama`) は残ることがあるので、別に消す。

### `~/.cache` と pnpm store の削除

```sh
find ~/.cache -mindepth 1 -maxdepth 1 -exec rm -rf {} +
rm -rf ~/.local/share/pnpm/store
```

- `~/.cache` の中身は使えば再生成される。ただし Playwright のブラウザは `npx playwright install` で入れ直しが必要。
- pnpm の `node_modules` は store とハードリンクでファイルを共有している。そのため store を消しても、プロジェクトから使われているファイルの分は容量が空かない。次の `pnpm install` で再ダウンロードされる。
- yay のビルド用キャッシュ (`~/.cache/yay`) も消える。過去にビルドした PKGBUILD との差分を見られなくなる点に注意。

## pacman のキャッシュ (`/var/cache/pacman/pkg`)

`pacman -S` や `-Syu` でダウンロードしたパッケージファイル (`.pkg.tar.zst`) を置いておく場所。自動では消えないので、更新のたびに古い版がたまっていく。

使い道は次の2つ。

- 更新で不具合が出たときに、古い版へ戻す (`sudo pacman -U /var/cache/pacman/pkg/<file>`)
- 同じ版を入れ直すときに、ダウンロードを省く

### `paccache` で整理する

`paccache` は `pacman-contrib` に入っている。

```sh
sudo pacman -S pacman-contrib
sudo paccache -dk3   # 試しに実行 (dry run)。消える量だけを表示する
sudo paccache -rk3   # 各パッケージの新しい版を3つ残して削除
sudo paccache -ruk0  # アンインストール済みのパッケージのキャッシュを全部削除
```

- `-k<N>` は残す版の数。`-rk1` が一番容量を空けられるが、戻せる版がなくなる。
- `-rk3` だけではアンインストール済みのパッケージの分が残る。`-ruk0` と組み合わせると、版を残しつつ不要な分を消せる。
- 定期的に自動で整理したいときは `sudo systemctl enable --now paccache.timer` を使う (週1回 `paccache -r`)。

`paccache` がまだ入っていないときは、ファイルの日付順で残す版を決めると仮定して、消える量を見積もれる。

```sh
find /var/cache/pacman/pkg -maxdepth 1 -name '*.pkg.tar.*' ! -name '*.sig' -printf '%T@ %s %f\n' \
  | sort -rn \
  | awk '{f=$3; sub(/-[^-]+-[^-]+-[^-]+\.pkg\.tar\..*$/,"",f); n[f]++; tot+=$2;
          for(k=1;k<=3;k++) if(n[f]>k) d[k]+=$2}
         END{printf "合計 %.1fG\n",tot/2^30; for(k=1;k<=3;k++) printf "-rk%d: 削除 %.1fG\n",k,d[k]/2^30}'
```

見積もりと実際の `paccache -rk3` の削除量はほぼ一致した (約26G)。

## 使われていない依存パッケージ

```sh
pacman -Qdtq                         # 依存として入ったが、今はどこからも使われていないもの
pacman -Rs --print <pkg>             # 一緒に消えるものを、消す前に確認する
sudo pacman -Rns $(pacman -Qdtq)     # 中身を確認してから実行する
```

`pacman -Rns --print` はオプションの組み合わせでエラーになる。確認するときは `-Rs --print` を使う。
