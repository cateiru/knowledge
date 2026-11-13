# yay で AUR のパッケージを入れる (Google Chrome を例に)

Google Chrome は公式リポジトリにないので、AUR の `google-chrome` を yay でビルドして入れる。2026年11月に試した。

## 手順

```sh
yay -S google-chrome
```

- yay は sudo なしで実行する。ビルドは一般ユーザーで行い、最後のインストールのときだけ yay が sudo のパスワードを聞く。
- 依存として、公式リポジトリの `ttf-liberation` (フォント) も一緒に入る。
- 中身は Google が配っている .deb を展開して pacman のパッケージにしたもの。ダウンロードに少し時間がかかる。
- 入ったあとは `google-chrome-stable` コマンドかアプリランチャーから起動する。以後は `yay -Syu` で一緒に更新される。

入ったかどうかは次のように確かめる。

```sh
pacman -Q google-chrome
google-chrome-stable --version
```

## プロンプトの答え方

ビルドの前に yay が2つ質問する。

- 「パッケージをクリーンビルドしますか？」: 前回のビルドで残ったファイルを消してからビルドするかどうか。初めて入れるときは何を選んでも同じなので `N` でよい。
- 「差異を表示しますか？」: 前回から PKGBUILD がどう変わったかを見るかどうか。初めてなら比べるものがないので `N` でよい。

質問を省略したいときは、答えを先に渡しておく。このオプションは `yay --help` に載っていることを確かめたが、実際にこれを付けて入れたわけではない。

```sh
yay -S --answerclean None --answerdiff None <pkg>
```

## ハマりどころ: プロンプトで Ctrl+C を押すと失敗する

クリーンビルドの質問で Ctrl+C を押してから `N` を入力すると、そのまま先へ進んだように見えるが、最後に次のように失敗する。

```text
 -> ソースのダウンロード時にエラー: ~/.cache/yay/google-chrome
     context: context canceled
 -> Failed to install layer, rolling up to next layer.error:error installing repo packages
```

Ctrl+C でソースのダウンロードが取り消されている (`context canceled`) のが原因。`~/.cache/yay/<pkg>/` には PKGBUILD などのビルド用のファイルだけが残り、途中まで落とした壊れたファイルはなかった。そのため、もう一度 `yay -S <pkg>` を実行して、Ctrl+C を押さずに答えればよい。

## Claude Code から入れる場合

最後に sudo のパスワードが必要なので、パスワードなしの sudo を設定していなければ Claude Code の Bash ツールからは入れられない。`sudo -n true` が失敗するならパスワードが必要なので、自分のターミナルで実行する。
