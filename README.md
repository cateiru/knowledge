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

## Claude Code

- [Claude Code に作業の知識を毎回書き留めさせる](claude-code/knowledge-recording.md)

## Next.js

- [フォームを保存したあと、遷移先のページで確認のダイアログを出す](nextjs/confirm-dialog-after-submit.md)
- [form action で保存すると react-aria のチェックボックスが元に戻る](nextjs/react-aria-checkbox-form-reset.md)
- [RSC リクエストの `_rsc` 検証と CDN の Vary の関係](nextjs/rsc-cache-busting-and-cdn-vary.md)
- [Server Action のあとに遷移しても、共有のレイアウトが描き直されない](nextjs/shared-layout-stale-after-replace.md)
- [全ページの title を「ページ名 | サイト名」にそろえる](nextjs/page-title-template.md)
- [サーバーの入力エラーを表示した react-aria の入力欄で、直しても送信できない](nextjs/react-aria-server-validation-resubmit.md)

## Nivo

- [Nivo の ScatterPlot で、日ごとの列に時刻の点を並べる](nivo/scatterplot-daily-columns.md)
- [Nivo のグラフで期間を絞り込んだとき、同じデータの点・線をつなげて動かす](nivo/keyed-transitions-on-period-change.md)

## シェル

- [find の -path に角括弧を含むディレクトリ名を書くと、一致しない](shell/find-path-brackets.md)

## Playwright

- [Playwright を更新したあとにヘッドレスシェルが見つからないときの対処](playwright/install-headless-shell.md)

## Vitest

- [開閉状態を localStorage に保存する部品を Vitest でテストする](vitest/localstorage-persisted-disclosure-test.md)

## SVG

- [インライン SVG のアイコンの一部の色を、祖先の CSS 変数で差し替える](svg/recolor-icons-with-css-variables.md)
- [数色で描かれた PNG のイラストを、マスクに分けて CSS 変数の色で塗る](svg/recolor-raster-illustration-with-masks.md)

## RAID

- [Linux のソフトウェア RAID (md) を Mackerel で監視する](mdraid/README.md)
