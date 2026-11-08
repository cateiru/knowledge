# pacman と yay でシステムを更新する

## 基本

```sh
sudo pacman -Syu   # 公式リポジトリのパッケージを全部更新
yay -Syu           # AUR も含めて全部更新 (sudo なしで実行する)
```

- `-S` はパッケージを入れる (sync)、`y` はパッケージ一覧を最新にする、`u` は入っているものを全部新しい版にする、という意味。
- `pacman -Sy <pkg>` のように、一覧だけ更新して一部だけ入れるのはやめる (partial upgrade)。Arch ではサポートされておらず、ライブラリの版の食い違いで壊れることがある。
- 大きな更新の前に [Arch Linux のニュース](https://archlinux.org/news/) を見ておく。手作業が必要な更新は、ここで告知される。

## 更新内容を確認するポイント

- `linux` や NVIDIA ドライバが更新されたら、再起動が必要。
- `uname -r` (動いているカーネル) と `pacman -Q linux` (入っているカーネル) が食い違っていたら、前回の更新のあとで再起動していない。古いカーネル用のモジュールは消えているので、新しいモジュールの読み込みに失敗することがある。
- perl のような言語処理系の大きな更新では、AUR などで別に入れたモジュールの作り直しが必要になることがある。`pacman -Qm` で AUR などから入れたパッケージを確認する。

## NVIDIA ドライバを DKMS で使っている場合

カーネルを更新すると、`linux-headers` の更新に合わせて DKMS がドライバを作り直す。再起動の前に、新しいカーネル用に作り直せているか確認する。

```sh
dkms status
# 例: nvidia/580.xxx, 7.x.y-arch1-1, x86_64: installed
```

ここで失敗したまま再起動すると、画面が映らないことがある。再起動するまでは、`nvidia-smi` が `Driver/library version mismatch` を表示することがあるが、正常。

## yay でよく出るもの

### クリーンビルドと差分表示の質問

`~/.cache/yay` を消したあとは、ビルド用のファイルがすべて新しくダウンロードされる。そのため、クリーンビルドの質問は `N` でも `A` でも結果は変わらない。差分表示も、前回の PKGBUILD がないので比較できない。

### 版をそろえる必要があるパッケージのインストールに失敗する

```text
:: nvidia-580xx-utils (新しい版) をインストールすると 'nvidia-580xx-utils=古い版' が破壊され nvidia-580xx-dkms の依存関係が壊れます
 -> Failed to install layer, rolling up to next layer.
```

同じ PKGBUILD から作られ、互いに版を完全に一致させる必要があるパッケージを、yay が別々に入れようとして失敗している。yay は次の回でまとめて入れ直すので、そのまま進めればたいてい成功する。

それでも失敗する場合は、ビルド済みのファイルを pacman で一度に入れる。

```sh
sudo pacman -U ~/.cache/yay/<pkgbase>/<pkg1>-*.pkg.tar.zst ~/.cache/yay/<pkgbase>/<pkg2>-*.pkg.tar.zst
```

### `exit status 8` でビルドに失敗する

ビルドに必要なパッケージがそろっていないことが多い。依存としてだけ入っていて、今はどこからも使われていないパッケージなら、直すより消したほうが早い。

```sh
pacman -Qi <pkg> | grep -E 'Required By|要求'
sudo pacman -Rns <pkg>
```

## 更新後の確認

```sh
uname -r
nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
systemctl --failed
```
