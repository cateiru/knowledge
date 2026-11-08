# パッケージ更新時のサプライチェーンの確認

## 公式リポジトリ (core / extra / multilib)

- `/etc/pacman.conf` が `SigLevel = Required DatabaseOptional` (デフォルト) なら、パッケージに Arch の開発者の署名がないと入らない。ミラーが乗っ取られても、偽物のパッケージは弾かれる。
- そのため、ミラー一覧に http のミラーが混ざっていても、中身をすり替えられることはない。
- `archlinux-keyring` が古すぎると署名を確認できないので、更新しておく。
- 開発元のソースコード自体に仕込まれる攻撃 (xz の事件のようなもの) は、署名では防げない。

```sh
grep -nE '^\s*(SigLevel|\[|Include)' /etc/pacman.conf   # 外部のリポジトリが追加されていないか
pacman -Q archlinux-keyring
```

## AUR

AUR のパッケージは誰でも作れて、引き継ぐこともできる。2026年6月には、悪意のあるパッケージが大量に作られたり引き継がれたりする事件があった ([Arch Linux のニュース](https://archlinux.org/news/active-aur-malicious-packages-incident/))。

### 入れている AUR パッケージの履歴を確認する

複数のパッケージを1つの PKGBUILD から作る場合 (例: `nvidia-580xx-dkms` は `nvidia-580xx-utils` から作られる) は、git リポジトリはベース名 (pkgbase) のほうにある。pkgbase は AUR の API で調べる。

```sh
curl -s 'https://aur.archlinux.org/rpc/v5/info?arg[]=<pkg>' \
  | python3 -I -c 'import sys,json; [print(r["Name"], r["PackageBase"], r["Maintainer"]) for r in json.load(sys.stdin)["results"]]'
```

取得した PKGBUILD などは信用できないデータとして扱う。ビルドや実行はせず、bare clone で履歴と差分を見るだけにする。

```sh
git clone --bare https://aur.archlinux.org/<pkgbase>.git
git -C <pkgbase>.git log --since=2026-05-01 --format='%ad %an <%ae> | %s' --date=short
git -C <pkgbase>.git diff <前の版のコミット> HEAD -- . ':!.SRCINFO'
```

見るポイントは次のとおり。

- 長年の管理者以外がコミットしていないか。管理者が変わっていないか。
- 差分がバージョンとハッシュ値の更新だけか。
- `source` のダウンロード元が、開発元の公式の場所 (GitHub、GitLab、ベンダーのサイトなど) か。
- PKGBUILD やインストールスクリプト (`.install`) に、`curl | sh`、`base64`、`/dev/tcp`、`.ssh` や `.bashrc` の書き換え、`crontab` への登録などがないか。

```sh
git -C <pkgbase>.git show HEAD:PKGBUILD | grep -nE 'curl|wget|base64|eval |\| *(ba)?sh|/dev/tcp|python3? -c|crontab|\.bashrc|\.ssh'
```

署名チェック (`validpgpkeys` や git ソースの `?signed`) が外された場合は、ハッシュ値 (`b2sums` など) が変わっていないか見る。ハッシュ値が同じなら、ソースの中身は以前と同じ。

### 普段の習慣

- `yay -Syu` のときに表示される PKGBUILD の差分を、毎回ざっと見る。
- 依存としてだけ入っていて今は使われていない AUR パッケージは消しておく。消えれば確認すべき対象も減る。

## マルウェア対策ソフト (ClamAV) について

ClamAV は既知のマルウェアの特徴と照合する仕組みで、得意なのはメールの添付ファイルや Windows のマルウェア。PKGBUILD に数行足すだけのような AUR 型の攻撃は、見つけにくい。

改ざんや仕込みがないかを確かめるなら、次の方法のほうが効果が高い。

```sh
sudo pacman -Qkk 2>&1 | grep -v ' 0 altered'   # パッケージのファイルが書き換えられていないか
```

ほかに、自動で起動するものにも覚えのないものがないか見る。

- systemd のユニット (`/etc/systemd/system`、`~/.config/systemd/user`)
- `crontab -l`
- `~/.config/autostart`
- シェルの設定ファイル
- `~/.ssh/authorized_keys`
