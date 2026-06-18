<!-- mirror-source: articles/020-firmware-update.md -->

> Original note.com article: [ファームウェア更新運用: バックアップしてから更新する【OpenWrt集中連載020】](https://note.com/ikmsan/n/nff6b598da354)

# ファームウェア更新運用: バックアップしてから更新する【OpenWrt集中連載020】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

OpenWrtベースのLN6001-JPでは、設定自由度が高いぶん、ファームウェア更新前のバックアップがかなり重要です。

特にIPoE、VLAN、VPN、DHCP予約などを使っている場合は、「更新後に何を確認するか」を先に決めておくだけでも、トラブル時の切り分けがかなり楽になります。

ただし、最初から全部を細かく確認しなくても大丈夫です。

まずは「バックアップを取る」「有線で更新する」「WANとWi‑Fiが戻ることを確認する」だけでも、かなり安全に進められます。

この記事では、LN6001-JPのファームウェア更新前後に確認したいポイント、バックアップの考え方、追加パッケージや設定が消えた時の戻し方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-01.png)

## この記事でわかること

- LN6001-JP更新前に何をバックアップするべきか
- 更新後にどこを確認すると切り分けしやすいか
- 追加パッケージやIPoE設定が消えた時の戻し方

## こんな人に向いています

- LN6001-JPを長く使う前提で、安全に更新したい
- 更新後にWi‑FiやVPNが壊れないか不安
- バックアップからどこまで戻せるのか知っておきたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-02.png)

## まずはここまでで十分

更新前後で全部を細かく見なくても、まずは次の3つを押さえれば十分です。

最初は「安全に戻せる状態を作る」ことを優先します。

1. バックアップを PC 側へ保存する
2. 有線接続で更新する
3. 更新後は WAN、Wi-Fi、追加パッケージの3点を確認する

特に更新前バックアップがあるだけで、失敗時の戻しやすさはかなり変わります。

最初は「全部理解してから更新」より、「戻せる状態を作ってから更新」のほうが安全です。

更新前は、「今どのバージョンなのか」を残しておくことも重要です。

あとから「どの更新で問題が起きたか」を追いやすくなります。

## ファームウェアバージョンの確認

### LuCIで確認

1. **System** → **Software** を開く
2. 「Installed packages」で `kernel` パッケージのバージョンを確認
3. または **Status** → **Overview** の「Firmware Version」を確認

### CLIで確認

```sh
# ルーターの基本情報（モデルとファームウェアバージョンを含む）
ubus call system board

# 更新前の稼働状態も保存しておく
uptime
df -h
opkg list-installed | grep -E 'base-files|kernel|busybox|dnsmasq|firewall|dropbear|uhttpd'
```

`ubus call system board` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「現在状態を記録する」用途で使うくらいでも十分です。

OpenWrt系では、「更新できること」より、「戻せること」のほうがかなり重要です。

## 更新前: バックアップを取る

### LuCIでバックアップする（推奨）

1. **System** → **Backup / Flash Firmware** を開く
2. **Backup** セクション → **Generate archive** をクリック
3. `backup-OpenWrt-日付.tar.gz` をダウンロードして保存する

このファイルには、`/etc/config/` 配下の主要設定ファイルが含まれます。

最初はLuCIバックアップだけでも十分役立ちます。

### CLIでバックアップする

```sh
# 設定ファイルと状態をまとめてバックアップ
BACKUP_DIR="/root/firmware-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

sysupgrade -b "$BACKUP_DIR/config-backup.tar.gz"

for cfg in network wireless firewall dhcp system dropbear; do
  [ -f "/etc/config/$cfg" ] && cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
done

# インストール済みパッケージ一覧を保存
opkg list-installed > "$BACKUP_DIR/packages-before-update.txt"

# ネットワーク設定のスナップショットを保存
ubus call system board > "$BACKUP_DIR/system-board-before.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan-before.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6-before.json" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status-before.json" 2>/dev/null || true
logread | tail -n 200 > "$BACKUP_DIR/logread-before.txt"

echo "backup: $BACKUP_DIR"
```

最初は「network」「wireless」「firewall」「dhcp」の4つだけ保存するだけでもかなり安心感が変わります。

バックアップファイルはLN6001-JPの中だけでなく、PC側にもダウンロードして保管します（SCPまたはLuCIバックアップ機能を使用）:

```sh
# PCからSCPでバックアップファイルを取得する（PCのターミナルで実行）
scp -r root@192.168.1.1:/root/firmware-before-* ~/Desktop/
```

ルーター内部だけへ保存していると、本体故障時に取り出せなくなることがあります。

ファームウェアは、必ず機種に合ったものを使用します。

## ファームウェアの入手

Linksys公式サポートサイトからファームウェアファイルを取得します:

1. https://support.linksys.com/kb/article/6274-jp/ にアクセス
2. 最新のファームウェアダウンロードリンクを確認
3. `.img` または `.bin` ファイルをダウンロード

**注意:** ファームウェアはLinksys公式サポートサイトのダウンロードリンクから取得します。

別サイトから入手したファイルや、機種違いのファームウェアは使いません。

最初はCLI更新より、LuCIから進めるほうが状態を確認しやすくなります。

## ファームウェアの更新手順（LuCI）

1. **System** → **Backup / Flash Firmware** を開く
2. **Flash image** をクリック
3. **Browse...** でダウンロードしたファームウェアファイルを選択
4. **Upload** をクリック
5. 更新が完了し、ルーターが再起動するまで電源を切らずに待つ
6. LEDが青点滅から白点灯、または更新前の状態に戻るまで待つ
7. 必要に応じてSSIDへ再接続し、**Status** ページでFirmware Versionを確認する

更新中は、電源断やブラウザ操作をできるだけ避けます。

特にWi‑Fi接続だけで更新すると、途中で切断された際に状態を追いにくくなります。

**警告:** 更新中にインターネット接続が不安定になる場合があります。

有線LAN接続で作業することを強くおすすめします。

CLI更新は、SSHやOpenWrt構成に慣れている場合向けです。

最初はLuCI更新だけでも十分です。

## ファームウェアの更新手順（CLI）

```sh
# ファームウェアファイルをルーターに転送する（PCのターミナルで実行）
scp firmware-linksys-ln6001.img root@192.168.1.1:/tmp/

# ルーターのSSHで実行
# バックアップが完了していることを確認
ls -la /root/firmware-before-*

# 可能ならファームウェアのハッシュを控える
sha256sum /tmp/firmware-linksys-ln6001.img

# ファームウェアを書き込む（設定を保持する場合）
sysupgrade -v /tmp/firmware-linksys-ln6001.img

# 設定を保持せずにクリーンインストールする場合
# sysupgrade -n /tmp/firmware-linksys-ln6001.img
```

`sysupgrade` は、OpenWrt系更新でまず覚えておきたいコマンドです。

`sysupgrade` 実行後はルーターが自動的に再起動します。

SSH接続は切断されるため、再接続できるまで数分待ちます。

更新後は、「全部を細かく確認する」より、「普段使う機能が戻っているか」を優先して確認するほうが整理しやすくなります。

## 更新後のチェックリスト

ルーターが再起動したら（約3〜5分後）、以下の順番で確認します:

### 1. 管理画面へのアクセス確認

`https://192.168.1.1` でLuCI管理画面にアクセスできるか確認します。

最初は「管理画面へ戻れるか」を確認できるだけでもかなり重要です。

### 2. WAN/WAN6の接続確認

```sh
ifstatus wan | grep -E '"up"|"ipaddr"'
ifstatus wan6 | grep -E '"up"|"ip6addr"'
ip route show
ip -6 route show
```

LuCI: **Status** → **Overview** でWAN/WAN6が接続済みか確認。

IPoE環境では、「WANは正常だがWAN6だけ落ちている」ケースもあります。

`ifstatus wan` と `ifstatus wan6` を分けて見ると切り分けしやすくなります。

### 3. Wi-Fiの確認

LuCI: **Network** → **Wireless** で設定したSSIDが存在するか確認します。

端末から接続テストも実施します。

```sh
uci show wireless | grep -E 'ssid|encryption|disabled'
wifi status
iw dev
```

### 4. 追加パッケージの確認

```sh
# 更新前後のパッケージを比較
opkg list-installed > /tmp/packages-after-update.txt
diff /root/firmware-before-YYYYMMDD-HHMM/packages-before-update.txt /tmp/packages-after-update.txt
```

最初は「Adblock」「VPN」「IPoE関連」が残っているかを見るだけでも十分です。

### 5. IPoE設定の確認

IPoE（オートIPoEモジュール）を使っている場合:

```sh
opkg list-installed | grep -i ipoe
ifstatus wan | grep '"ipaddr"'
ifstatus wan6 | grep -E '"ip6addr"|"ip6prefix"'
```

モジュールが消えた場合は再インストールが必要です（https://support.linksys.com/kb/article/6902-jp/）。

特にIPoE環境では、「IPv4 over IPv6側だけ戻っていない」ケースもあります。

### 6. VPN設定の確認

```sh
opkg list-installed | grep -E 'wireguard|tailscale'
command -v wg >/dev/null && wg show
command -v tailscale >/dev/null && tailscale status
logread | grep -Ei 'wireguard|tailscale|vpn' | tail -n 80
```

最初は「VPNへ再接続できるか」を確認できれば十分です。

### 7. DHCP予約の確認

```sh
uci show dhcp | grep '@host'
cat /tmp/dhcp.leases
```

予約が失われた場合はバックアップから復元:

```sh
cp /etc/config/dhcp.backup.YYYYMMDD /etc/config/dhcp
/etc/init.d/dnsmasq restart
```

DHCP予約が消えると、NASやプリンターIPが変わってしまうことがあります。

更新後は、「普段使うものから順番に戻っているか」を確認するほうが整理しやすくなります。

## 更新後チェックリスト（まとめ）

| 確認項目 | 方法 | 状態 |
|---|---|---|
| 管理画面アクセス | https://192.168.1.1 | □ |
| WAN接続 | `ifstatus wan` | □ |
| WAN6接続 | `ifstatus wan6` | □ |
| Wi-Fi SSID | Network > Wireless | □ |
| IPoEモジュール | `opkg list-installed \| grep ipoe` | □ |
| VPNパッケージ | `opkg list-installed \| grep wireguard` | □ |
| DHCP予約 | `uci show dhcp \| grep host` | □ |
| firewall設定 | `uci show firewall` | □ |

更新失敗時も、バックアップが残っていればかなり復旧しやすくなります。

## 更新に失敗した場合

設定を保持しない更新（`-n`オプション）や工場出荷状態リセット後は、設定を全て再入力する必要があります。バックアップを使って復元します:

```sh
# バックアップファイルをルーターに転送
scp backup-OpenWrt-20260507.tar.gz root@192.168.1.1:/tmp/

# 設定を復元
sysupgrade -r /tmp/backup-OpenWrt-20260507.tar.gz
reboot
```

または LuCI: **System** → **Backup / Flash Firmware** → **Restore backup** で tar.gz ファイルをアップロードして復元します。

最初は「LuCIへ戻れるか」を優先確認すると切り分けしやすくなります。

## まとめ

更新後は、次の3つが確認できれば最初の安全確認として十分です。最初はここまで確認できれば大きく外していません。

- WAN が上がり、通常のインターネット接続が使える
- Wi-Fi で普段使う端末が再接続できる
- 追加パッケージや DHCP 予約、VPN が想定どおり残っている

更新前バックアップが最大の安全策です。

作業は有線LAN接続状態で、できれば業務外時間帯に実施します。

最初から全部を細かく確認するより、「WAN」「Wi‑Fi」「普段使う機能」が戻っていることを優先確認するほうが整理しやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-03.png)

## よくある質問

### LN6001-JPのファームウェア更新前に必ずやることは？

設定バックアップの取得です。

更新前後の確認項目もメモしておくと戻しやすくなります。

### LN6001-JPのアップデート後に追加パッケージは残る？

残らないことがあります。

`opkg list-installed` を別に保存しておくと再インストールがしやすくなります。

### LN6001-JPの更新作業はWi-Fi接続でもできる？

できなくはありませんが、有線LAN接続のほうが安全です。

更新中に無線が切れると作業しにくくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT firmware update: https://support.linksys.com/kb/article/227-en/?section_id=175

## この連載で使っているOpenWrtルーター

この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使っています。

https://amzn.to/4tVu4yG

LN6001-JPは、国内向けに展開されているOpenWrtベースのWi‑Fi 7ルーターです。一般的な「自分でOpenWrtを書き込むタイプ」と違い、OpenWrtベースで最適化されたファームウェアを搭載した状態で出荷されるため、かなり導入しやすいのが特徴です。

初期セットアップ済みで、基本的には電源を入れてすぐ利用可能。LuCIブラウザ管理画面、SSH、opkg、VLAN、OpenVPN、WireGuard、Tailscaleなど、OpenWrtらしい拡張性も備えています。

特に日本の回線環境向けとして、Linksys公式の「オートIPoE」モジュールが用意されているのが大きなポイントです。OCNバーチャルコネクト、transixなどのIPv4 over IPv6環境でも、LuCIから比較的シンプルに設定できます。

もちろんCLIで細かく調整していく楽しさもありますが、MAP-EやIPIPは設定項目がかなり多いため、「まず普通に安定して使いたい」という場合はオートIPoEを使った方がかなり楽です。

この連載では、実機開発にも関わった立場から、家庭・小規模オフィス・店舗などで“実際どう使うと快適なのか” を中心に、現実的なOpenWrt運用を紹介していきます。

### 使用機材

- 製品名: Linksys Velop WRT Pro 7
- 型番: LN6001-JP
- 用途: 家庭〜小規模オフィス・店舗向け OpenWrt ベースルーター
- 製品情報: https://support.linksys.com/kb/article/6274-jp/
- IPoE設定方法: https://support.linksys.com/kb/article/6902-jp/
- WireGuard / Tailscale設定方法: https://support.linksys.com/kb/article/8723-jp/

※設定変更前はバックアップをおすすめします。ファームウェア更新によって画面や項目名が変わる場合があるため、必要に応じてLinksys公式サポートもあわせて確認してください。
