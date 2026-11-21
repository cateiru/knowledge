# knowledge

作業の中で得た知識のメモ。

## Arch Linux

- [ルートパーティションの容量を空ける](arch-linux/disk-cleanup.md)
- [pacman と yay でシステムを更新する](arch-linux/system-update.md)
- [yay で AUR のパッケージを入れる (Google Chrome を例に)](arch-linux/install-aur-package.md)
- [パッケージ更新時のサプライチェーンの確認](arch-linux/supply-chain.md)

## Cloudflare

- [Cloudflare Worker Previews をプレビュー専用のリソースで使う](cloudflare/worker-previews.md)
- [Workers Builds で pnpm モノレポのサブパッケージの Worker をデプロイする](cloudflare/workers-builds-monorepo.md)
- [D1 の batch でまとめて削除する処理を sql.js でテストする](cloudflare/d1-batch-delete-sqljs-test.md)
- [外部キーのない参照先が論理削除される競合を、文の中の条件で防ぐ](cloudflare/d1-polymorphic-soft-delete-race.md)
- [D1 のマイグレーションを本番に手動で適用する](cloudflare/d1-remote-migrations.md)
- [D1（SQLite）の JSON 列から、特定のキーだけを外す](cloudflare/d1-json-remove-key.md)
- [ゾーンのキャッシュ状態をコマンドラインから集計する](cloudflare/cache-status-analytics.md)
- [D1 のバインドパラメーターの上限（100 個）を超えて 500 になる](cloudflare/d1-bound-parameter-limit.md)
- [ダッシュボードでシークレットを変えると、プレビューのコードが本番に出ることがある](cloudflare/secret-edit-deploys-latest-uploaded-version.md)
- [ハッシュで保存していたメールアドレスを、平文の列へ段階的に移す](cloudflare/d1-hashed-email-to-plain-column.md)
- [Drizzle の D1 の batch で、同じ列名を JOIN した SELECT の値が入れ替わる](cloudflare/drizzle-d1-batch-duplicate-columns.md)
- [運用スクリプトから D1 を Drizzle で読み書きする（wrangler d1 execute をやめる）](cloudflare/scripts-d1-drizzle-get-platform-proxy.md)
- [Workers のカスタムドメインを wrangler の routes で付け替える](cloudflare/workers-custom-domain-switch.md)
- [Access のアプリケーションを cf コマンドで安全に書き換える](cloudflare/access-applications-cf-cli.md)

## Claude Code

- [Claude Code に作業の知識を毎回書き留めさせる](claude-code/knowledge-recording.md)
- [バックグラウンドのコマンドの「完了」の通知が、終わる前に届くことがある](claude-code/background-task-early-notification.md)

## Next.js

- [フォームを保存したあと、遷移先のページで確認のダイアログを出す](nextjs/confirm-dialog-after-submit.md)
- [form action で保存すると react-aria のチェックボックスが元に戻る](nextjs/react-aria-checkbox-form-reset.md)
- [RSC リクエストの `_rsc` 検証と CDN の Vary の関係](nextjs/rsc-cache-busting-and-cdn-vary.md)
- [Server Action のあとに遷移しても、共有のレイアウトが描き直されない](nextjs/shared-layout-stale-after-replace.md)
- [全ページの title を「ページ名 | サイト名」にそろえる](nextjs/page-title-template.md)
- [サーバーの入力エラーを表示した react-aria の入力欄で、直しても送信できない](nextjs/react-aria-server-validation-resubmit.md)
- [Next.js と @opennextjs/cloudflare は一緒に上げる](nextjs/opennext-next-version-pairing.md)
- [App Router で 404・500 のエラーページを作る](nextjs/error-pages.md)
- [メンバー限定の画像を、招待された人にだけ招待のトークンで見せる](nextjs/invite-token-scoped-media.md)

## Nivo

- [Nivo の ScatterPlot で、日ごとの列に時刻の点を並べる](nivo/scatterplot-daily-columns.md)
- [Nivo のグラフで期間を絞り込んだとき、同じデータの点・線をつなげて動かす](nivo/keyed-transitions-on-period-change.md)

## シェル

- [find の -path に角括弧を含むディレクトリ名を書くと、一致しない](shell/find-path-brackets.md)
- [pkill -f が、自分を実行しているシェルまで止めてしまう](shell/pkill-matches-own-shell.md)
- [zsh で、変数に入れたコマンドが「command not found」になる](shell/zsh-variable-command-no-word-splitting.md)

## Docker

- [sudo を使わずに、コンテナが root で作ったファイルを消す](docker/remove-root-owned-files.md)
- [node の slim イメージで wrangler dev（workerd）が起動しない](docker/node-slim-ca-certificates.md)
- [長く動かしっぱなしの next dev がメモリを食い尽くす](docker/next-dev-memory-growth.md)

## Node.js

- [node-build の定義が古いとき、Node.js を nodenv に手で入れる](node/nodenv-install-from-tarball.md)
- [Node.js の型ストリップで、拡張子なしで import し合う .ts を読む](node/register-hooks-extensionless-ts.md)

## pnpm

- [pnpm --filter で deploy スクリプトを動かすには run が必要](pnpm/filter-run-deploy.md)
- [モノレポで共通の依存のバージョンを pnpm の catalogs でまとめる](pnpm/catalogs-shared-deps.md)

## GitHub

- [PR のレビューコメントに、スレッドの返信として答える](github/reply-to-review-comments.md)
- [Codex の PR 自動レビューが終わったかを gh で判定する](github/codex-review-status.md)
- [PR の変更検知で、version だけを上げた package.json を除く](github/ignore-version-only-package-json.md)

## Web

- [フォームの多いページが、パスワードマネージャーの拡張機能で固まる](web/password-manager-form-scan.md)
- [iOS のホーム画面アプリで、ヘッダーがぼやける](web/ios-pwa-header-blur.md)
- [curl で取った画像が gzip のまま保存される](web/curl-gzip-content-encoding.md)
- [ホーム画面への追加（PWA のインストール）を勧めるバナーを出す](web/pwa-install-banner.md)

## Playwright

- [Playwright を更新したあとにヘッドレスシェルが見つからないときの対処](playwright/install-headless-shell.md)

## Vitest

- [開閉状態を localStorage に保存する部品を Vitest でテストする](vitest/localstorage-persisted-disclosure-test.md)

## VRT

- [reg-suit の差分画像で、要素の高さが変わっただけかを見分ける](vrt/reg-suit-diff-layout-shift.md)

## CSS

- [CSS だけで家系図のような図の接続線を描く](css/family-tree-connectors.md)
- [負のマージンではみ出させたボタンが隣との gap を打ち消す](css/negative-margin-hover-eats-gap.md)

## SVG

- [インライン SVG のアイコンの一部の色を、祖先の CSS 変数で差し替える](svg/recolor-icons-with-css-variables.md)
- [数色で描かれた PNG のイラストを、マスクに分けて CSS 変数の色で塗る](svg/recolor-raster-illustration-with-masks.md)

## MCP

- [MCP のツールで「だれがいつ作成・編集したか」を返す](mcp/record-change-history-in-tools.md)

## クローラー

- [クロールした作品が、配信サイト側の ID の振り直しで重複登録される](crawler/duplicate-items-after-id-reissue.md)

## RAID

- [Linux のソフトウェア RAID (md) を Mackerel で監視する](mdraid/README.md)
