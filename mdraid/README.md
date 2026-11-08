# Linux のソフトウェア RAID (md) を Mackerel で監視する

## 背景

`mdmonitor.service` (`mdadm --monitor --scan`) は、`/etc/mdadm.conf` に `MAILADDR` も `PROGRAM` も書かれていないと、次のメッセージを出してすぐ終了する。

```text
mdadm: No mail address or alert command - not monitoring.
```

この状態では RAID の状態を誰も見張っていない。RAID1 のディスクが1台壊れても気づかないまま動き続け、もう1台も壊れたときにデータを失う。

メールを送る仕組み (MTA) がないマシンだったので、すでに入っている Mackerel のチェック監視を使うことにした。

## チェック用のスクリプト

[`check-mdraid`](check-mdraid) を Mackerel のチェックプラグインとして使う。

- `/sys/block/md*/md/` を読んで、ある md アレイを全部確認する。`/etc/mdadm.conf` に `ARRAY` 行がないとアレイの名前 (`md127` など) が自動で決まり、変わることがあるので、特定の名前に固定しない。
- 終了コードは `0` が OK、`1` が WARNING、`2` が CRITICAL。標準出力がアラートの本文になる。

| 状態 | 判定 |
| --- | --- |
| `degraded` が 0 でない (縮退) | CRITICAL |
| メンバーに `faulty` のマークがある | CRITICAL |
| `array_state` が `clean` / `active` などでない | CRITICAL |
| アレイが1つも見つからない (起動時に組み上がらなかった) | CRITICAL |
| `sync_action` が `resync` / `recover` / `reshape` / `repair` | WARNING |
| 正常、または定期チェック (`check`) の途中 | OK |

アレイが見つからないときに OK を返すと、一番まずい状態 (起動時に組み上がらなかった) を見逃すので、CRITICAL にしている。

### 配置

mackerel-agent は root で動くので、一般ユーザーが書き換えられない場所にコピーする。`install` はコピーなので、元のファイルを消しても影響はない。

```sh
sudo install -m755 check-mdraid /usr/local/bin/check-mdraid
```

### テスト

本物のディスクを外して (`mdadm --fail` などで) 試すと、RAID を本当に壊してしまう。代わりに、環境変数で偽の sysfs を読ませて試す。

```sh
t=$(mktemp -d)
mkdir -p $t/sys/md127/md/dev-sda1 $t/sys/md127/md/dev-sdb1
printf '%s\n' clean   > $t/sys/md127/md/array_state
printf '%s\n' 1       > $t/sys/md127/md/degraded
printf '%s\n' idle    > $t/sys/md127/md/sync_action
printf '%s\n' 2       > $t/sys/md127/md/raid_disks
printf '%s\n' in_sync > $t/sys/md127/md/dev-sda1/state
printf '%s\n' faulty  > $t/sys/md127/md/dev-sdb1/state
echo "md127 : active raid1 ..." > $t/mdstat
MDRAID_SYSFS=$t/sys MDRAID_MDSTAT=$t/mdstat ./check-mdraid; echo "exit=$?"
```

zsh で `echo 0>file` と書くと、`0>` がファイルディスクリプタ0のリダイレクトとして解釈され、ファイルに `0` が書かれない。数字を書くときは `printf` を使うか、`>` の前に空白を入れる。

## mackerel-agent の設定

```toml
[plugin.checks.mdraid]
command = "/usr/local/bin/check-mdraid"
max_check_attempts = 1
notification_interval = 60
```

- `max_check_attempts = 1` は、1回の異常ですぐ通知する設定。
- `notification_interval = 60` は、直るまで60分ごとに通知し直す設定。RAID の縮退は対応が必要なので、1回で終わらせない。
- 古い mackerel-agent でも動くように、`command` は配列ではなく文字列で書く。

設定を変えたら、検証してから再起動する。

```sh
mackerel-agent configtest --conf=/path/to/mackerel-agent.conf
sudo systemctl restart <mackerel のユニット名>
```

設定ファイルには API キーが入っているので、確認するときは中身をそのまま表示しないようにする。

Mackerel の画面では、次の3つを確認する。

- ホストにチェック監視の `mdraid` が表示されていること
- ホストの状態が `working` になっていること
- チェック監視のアラートが、普段見ている通知先に届く設定になっていること

## mdmonitor を止める

監視を Mackerel に任せたので、`mdmonitor.service` は止める。このサービスは状態が `static` で、RAID を検出したときに udev から起動されるため、`disable` では止まらない。`mask` で起動そのものを禁止する。

```sh
sudo systemctl mask mdmonitor
sudo systemctl reset-failed mdmonitor
```

元に戻すときは `sudo systemctl unmask mdmonitor` を実行する。
