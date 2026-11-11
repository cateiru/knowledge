# Playwright を更新したあとにヘッドレスシェルが見つからないときの対処

2026年11月に確かめた内容。

## 症状

依存の更新で Playwright のバージョンが上がったあと、テストを実行すると、ブラウザーを起動するところで次のように失敗する。

```text
Error: browserType.launch: Executable doesn't exist at ~/.cache/ms-playwright/chromium_headless_shell-<revision>/chrome-headless-shell-linux64/chrome-headless-shell
```

## 原因

Playwright は、バージョンごとに決まったリビジョンのブラウザーを `~/.cache/ms-playwright/` から探す。Playwright を上げると要求するリビジョンも変わるため、前のバージョンで入れたブラウザーがあっても使われない。

ヘッドレスで実行するとき、Chromium は通常の Chromium ではなく `chromium_headless_shell-<revision>` を使う。

## 対処

プロジェクトに入っている Playwright の CLI で、ヘッドレスシェルだけを入れる。

```sh
node_modules/.bin/playwright install chromium-headless-shell
```

- グローバルの `npx playwright` ではなく、プロジェクトの `node_modules/.bin/playwright` を使う。バージョンがずれると、別のリビジョンが入ってしまう。
- ヘッドレスでしか実行しないなら、フルの Chromium (`playwright install chromium`) は入れなくてよい。ヘッドレスシェルだけなら 130MB 程度で済む。
- 古いリビジョンのディレクトリは自動では消えないので、容量が気になるときは `~/.cache/ms-playwright/` を確認する。

## Arch Linux での注意

Arch Linux では次の警告が出るが、Ubuntu 24.04 向けのフォールバックビルドが入り、そのままテストを実行できた。

```text
BEWARE: your OS is not officially supported by Playwright; downloading fallback build for ubuntu24.04-x64.
```
