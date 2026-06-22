<!-- mirror-source: articles/023-backup-restore.md -->

# バックアップは“壊れた時”ではなく“動いている時”に取る｜LN6001-JPの設定保存と復元【OpenWrt集中連載023】

バックアップって、ちょっと地味です。

Wi-Fi 7。  
OpenWrt。  
VLAN。  
VPN。  
Tailscale。  
Adblock。  
IPoE。  
ゲストWi-Fi。

こういう話のほうが、どう考えても派手です。

でも、OpenWrtベースのルーターを触るなら、いちばん効くのはバックアップです。

本当に地味ですが、かなり強いです。

ゲストWi-Fiを作る前。  
VLANを触る前。  
firewall zoneを変える前。  
LAN IPを変える前。  
IPoEモジュールを入れる前。  
VPN Assistantを入れる前。  
Adblockを入れる前。  
ファームウェア更新の前。

ここでバックアップを取っているかどうかで、安心感がぜんぜん違います。

設定で一番強い人は、全部のコマンドを暗記している人ではありません。

```txt
変更前に戻れる人
```

です。

LN6001-JPはOpenWrtベースなので、LuCIから設定バックアップを取れます。  
さらにSSHを使えば、`/etc/config/` の中身や `opkg` の追加パッケージ一覧も控えられます。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、設定バックアップをいつ取るべきか、LuCIでどう保存するか、SSHで何を控えるか、復元時に何へ注意するかをまとめます。

バックアップは、壊れてから取るものではありません。

**動いている時に取るもの**です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、バックアップ機能を一度押して終わりにすることではありません。

まずは、ここまでできればOKです。

- LuCIから設定バックアップを取得できる
- バックアップファイルをPC側へ保存できる
- バックアップファイルに重要情報が含まれると理解できる
- SSHで `/etc/config/` の主要設定を控えられる
- `opkg list-installed` で追加パッケージ一覧を残せる
- `sysupgrade -b` / `sysupgrade -r` の基本が分かる
- 復元後に確認すべき順番が分かる
- バックアップを丸ごと戻すべき時と、部分的に戻すべき時を分けられる

バックアップは、保険です。

でも、保険は入っているだけでは足りません。

```txt
どこに保存したか
いつの状態か
何を戻せるか
戻した後に何を確認するか
```

ここまで分かっていると、かなり安心して設定を触れます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-01.png)

## 先にざっくり結論

LN6001-JPでは、まずLuCIから設定バックアップを取ります。

```txt
System
  ↓
Backup / Flash Firmware
  ↓
Generate archive
```

この `.tar.gz` ファイルをPCへ保存します。

これが基本です。

ただし、OpenWrtベースの運用では、LuCIバックアップだけでなく、次も一緒に残すと復旧しやすくなります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-01.png)

大事なのは、バックアップを「1本だけ」ではなく、節目ごとに取ることです。

```txt
初期設定後
IPoE設定後
Wi-Fi設定後
ゲストWi-Fi設定前
ゲストWi-Fi設定後
VLAN設定前
VLAN設定後
VPN設定前
VPN設定後
ファームウェア更新前
ファームウェア更新後
```

このくらいで取っておくと、戻る場所を選べます。

バックアップは多すぎても整理が必要ですが、なさすぎるよりずっと良いです。

## こういう人向けです

この記事は、次のような人向けです。

- LN6001-JPの設定を触る前に安全策を作りたい
- ゲストWi-FiやVLANを作る前に戻せる状態を作りたい
- firewallを触るのが少し怖い
- IPoEやVPN Assistantを入れる前にバックアップしたい
- AdblockやTailscaleなど追加パッケージも控えておきたい
- ファームウェア更新前後の保存手順を固めたい
- 店舗や小さなオフィスで、設定を飛ばす事故を避けたい
- 初期化後に何から戻せばよいか整理したい

逆に、完全に初期状態のまま使っている場合は、最初はLuCIバックアップだけでも十分です。

ただし、設定を少しでも触り始めたら、バックアップ運用はかなり大事になります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-02.png)

## 最初に言葉だけそろえる

バックアップまわりの言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-02.png)

最初は、これだけ覚えれば大丈夫です。

```txt
LuCIバックアップ = 設定を戻すため
opkg一覧 = 追加機能を戻すため
状態スナップショット = 何が変わったか比べるため
```

## バックアップを取るタイミング

バックアップは、壊れた後に取るものではありません。

壊れる前に取ります。

もっと正確に言うと、

```txt
動いている状態を保存する
```

のがバックアップです。

おすすめのタイミングは次です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-03.png)

とくに強くおすすめするのは、この4つです。

```txt
ゲストWi-Fi前
VLAN前
firewall前
ファームウェア更新前
```

ここは事故った時のダメージが大きいです。

## バックアップファイル名の付け方

バックアップは、取るだけでは足りません。

あとから見て、何のバックアップか分かる名前にします。

悪い例です。

```txt
backup.tar.gz
config.tar.gz
download.tar.gz
```

半年後に見ても何も分かりません。

おすすめはこうです。

```txt
backup-LN6001-initial-20260621.tar.gz
backup-LN6001-before-guest-wifi-20260621.tar.gz
backup-LN6001-after-guest-wifi-ok-20260621.tar.gz
backup-LN6001-before-vlan-20260621.tar.gz
backup-LN6001-before-firmware-update-20260621.tar.gz
backup-LN6001-after-recovery-20260621.tar.gz
```

ファイル名に入れたい情報です。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-04.png)

バックアップは、未来の自分へのメッセージです。

未来の自分は、たぶん細かい設定を忘れています。

親切に名前を付けておきましょう。

## LuCIでバックアップを取る

まずは公式手順に沿って、LuCIからバックアップを取ります。

1. ブラウザでLuCIへログインする
2. **System** → **Backup / Flash Firmware** を開く
3. **Backup** セクションを確認する
4. **Generate archive** をクリックする
5. `.tar.gz` ファイルをPCへ保存する
6. ファイル名を分かりやすく変更する

例:

```txt
backup-LN6001-before-vlan-20260621.tar.gz
```

ここで大事なのは、**PCへ保存すること**です。

ルーター内部に置いたままだと、初期化やファームウェア更新で消える可能性があります。

```txt
バックアップはPCへ
できればクラウドや外部ストレージにも
ただし公開場所には置かない
```

このくらいの運用が安全です。

## バックアップファイルに含まれる情報

設定バックアップには、かなり重要な情報が含まれます。

たとえば、次のようなものです。

- LAN IP
- WAN設定
- Wi-Fi設定
- SSID
- Wi-Fiパスワード
- firewall設定
- DHCP予約
- DNS設定
- PPPoE情報
- VPN関連設定
- Adblock関連設定
- 端末名や内部IP
- 店舗名や拠点名が分かるホスト名

つまり、バックアップファイルは秘密情報です。

SNS、GitHub、公開Google Drive、ブログ記事の添付などへそのまま置かないでください。

設定バックアップは、ルーターの合鍵に近いです。

保管場所はちゃんと選びます。

## LuCIで復元する

保存したバックアップを戻す場合は、LuCIから復元できます。

1. LuCIへログインする
2. **System** → **Backup / Flash Firmware** を開く
3. **Restore backup** または **Upload archive** を選ぶ
4. 保存していた `.tar.gz` ファイルを選択する
5. アップロードして復元する
6. 再起動を待つ
7. LuCIへ再ログインする
8. WAN / WAN6、Wi-Fi、DHCP、firewallを確認する

復元後、LAN IPがバックアップ時点のものへ戻ることがあります。

たとえば、初期化直後に `192.168.1.1` で入れていても、復元後に `192.168.10.1` へ戻る場合があります。

復元後にLuCIへ入れない時は、PC側のDefault Gatewayを確認します。

Windows:

```powershell
ipconfig
```

macOS / Linux:

```sh
ip route show default
```

「復元に失敗した」のではなく、管理画面の住所が戻っただけかもしれません。

## 復元後に確認すること

バックアップを戻したら、必ず動作確認します。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-05.png)

復元は、戻したら終わりではありません。

```txt
戻った設定で、実際に通信できるか
```

まで確認します。

特に、ゲストWi-FiやVLANは「届くべきところ」と「届かないべきところ」の両方を確認します。

## SSHで設定バックアップを取る

LuCIバックアップに加えて、SSHで設定ファイルを控えておくと、部分復旧しやすくなります。

```sh
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
logread | tail -n 300 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップは、ルーター内部にあります。

PCへコピーします。

```sh
scp -r root@192.168.1.1:/root/config-backup-* ~/Downloads/
```

LAN IPを変えている場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/root/config-backup-* ~/Downloads/
```

この方法のメリットは、個別に戻しやすいことです。

たとえば、firewallだけ壊したなら `/etc/config/firewall` だけ戻せます。

Wi-Fiだけ壊したなら `/etc/config/wireless` だけ戻せます。

## `sysupgrade -b` でバックアップを取る

OpenWrt系では、CLIから `sysupgrade -b` でバックアップを作れます。

```sh
sysupgrade -b /tmp/backup-LN6001-$(date +%Y%m%d-%H%M).tar.gz
ls -lh /tmp/backup-LN6001-*.tar.gz
```

PCへコピーします。

```sh
scp root@192.168.1.1:/tmp/backup-LN6001-*.tar.gz ~/Downloads/
```

`/tmp` は再起動で消える一時領域です。

必ずPCへコピーします。

PCへ保存できたら、ルーター側の一時ファイルは消しても構いません。

```sh
rm /tmp/backup-LN6001-*.tar.gz
```

## `sysupgrade -r` で復元する

CLIで復元する場合は、バックアップファイルをルーターへ送ります。

```sh
scp backup-LN6001-before-vlan-20260621.tar.gz root@192.168.1.1:/tmp/
```

ルーター側で復元します。

```sh
sysupgrade -r /tmp/backup-LN6001-before-vlan-20260621.tar.gz
reboot
```

復元後、LuCIへ再ログインして状態を確認します。

注意点です。

- 復元後にLAN IPが変わる場合がある
- Wi-Fi設定が戻るため、SSIDやパスワードも戻る
- firewall設定も戻るため、アクセス範囲が変わる
- 追加パッケージ本体は戻らない場合がある
- バックアップ時点で壊れていた設定も戻る

CLI復元は強力です。

でも、戻すバックアップを間違えると、古い問題も戻ります。

ファイル名と作成日をよく見ます。

## バックアップ対象を確認する

OpenWrt系では、`sysupgrade -l` でバックアップ対象に含まれるファイルを確認できます。

```sh
sysupgrade -l
```

基本的な設定ファイルは含まれます。

ただし、自分で `/root` に置いたスクリプト、証明書、鍵、独自設定ファイルなどは、バックアップ対象に含まれないことがあります。

必要に応じて、`/etc/sysupgrade.conf` に追加します。

例:

```sh
echo "/root/scripts/my-check.sh" >> /etc/sysupgrade.conf
echo "/etc/tailscale/custom-note.txt" >> /etc/sysupgrade.conf
sysupgrade -l | grep -E 'my-check|custom-note'
```

ただし、秘密鍵や証明書をバックアップに含める場合は、保管に注意してください。

バックアップファイルの重要度がさらに上がります。

## 追加パッケージはバックアップだけでは戻らないことがある

ここ、かなり大事です。

LuCIバックアップや `sysupgrade -b` は、主に設定ファイルを戻すものです。

追加パッケージ本体まで完全に戻るとは限りません。

たとえば、次のようなものです。

- Adblock
- banIP
- Tailscale
- WireGuard関連
- VPN Assistant
- オートIPoE
- 追加LuCIアプリ
- 診断ツール
- 独自スクリプト

設定ファイルが戻っていても、パッケージ本体がなければ動きません。

そのため、必ず追加パッケージ一覧を保存します。

```sh
opkg list-installed > /root/opkg-list-installed-$(date +%Y%m%d-%H%M).txt
```

PCへコピーします。

```sh
scp root@192.168.1.1:/root/opkg-list-installed-*.txt ~/Downloads/
```

復旧後、必要なものを確認します。

```sh
opkg list-installed | grep -Ei 'adblock|banip|tailscale|wireguard|vpn|ipoe|luci-app'
```

必要なものだけ、更新後の環境に合わせて入れ直します。

```sh
opkg update
```

ただし、雑に `opkg upgrade` をまとめて実行するのは避けたほうが安全です。

ルーターはPCではありません。

必要なパッケージだけ入れます。

## 何をバックアップすべきか

最低限のバックアップ対象は次です。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-06.png)

特に重要なのはこの4つです。

```txt
network
wireless
firewall
dhcp
```

OpenWrt系の設定トラブルは、この4つのどれかにいることが多いです。

## バックアップ前の状態メモ

バックアップファイルだけでは、「何の状態だったか」が分かりにくいことがあります。

一緒にメモを残します。

```txt
バックアップ名:
  backup-LN6001-after-guest-wifi-ok-20260621.tar.gz

作成日:
  2026-06-21

状態:
  Main Wi-Fi OK
  Guest Wi-Fi OK
  GuestからLANへアクセス不可確認済み
  WAN / WAN6 OK
  Adblock未導入
  VPN未導入

LAN IP:
  192.168.1.1

ネットワーク:
  lan: 192.168.1.0/24
  guest: 192.168.2.0/24

注意:
  次はVLAN設定前に使う
```

このメモがあると、復元する時に迷いにくくなります。

バックアップファイルだけが並んでいると、どれが安全地点なのか分からなくなります。

## バックアップの保管場所

おすすめは、最低2か所です。

```txt
PCのローカルフォルダ
+
外部ストレージまたは非公開クラウド
```

ただし、公開場所には置かないでください。

良い例:

```txt
~/Documents/router-backups/LN6001/
暗号化された外部SSD
非公開クラウドストレージ
パスワード管理された社内保管場所
```

悪い例:

```txt
公開GitHubリポジトリ
公開Google Drive
SNSのDMに貼る
記事の添付ファイルとして公開
誰でも見られる共有フォルダ
```

バックアップファイルは、設定情報の塊です。

特に、Wi-Fiパスワード、PPPoE情報、VPN情報、内部IP、端末名が含まれる可能性があります。

## 変更前バックアップの作り方

大きな変更前は、次のセットを取ると安心です。

```sh
CHANGE="before-guest-wifi"
DATE="$(date +%Y%m%d-%H%M)"
BACKUP_DIR="/root/${CHANGE}-${DATE}"
mkdir -p "$BACKUP_DIR"

sysupgrade -b "$BACKUP_DIR/config-backup-${CHANGE}-${DATE}.tar.gz"

for cfg in network wireless firewall dhcp system dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
logread | tail -n 300 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

PCへコピーします。

```sh
scp -r root@192.168.1.1:/root/before-guest-wifi-* ~/Downloads/
```

変更内容に応じて `CHANGE` を変えます。

例:

```sh
CHANGE="before-vlan"
CHANGE="before-firewall"
CHANGE="before-vpn"
CHANGE="before-adblock"
CHANGE="before-firmware-update"
```

## 変更後バックアップも取る

変更前バックアップだけでなく、成功後のバックアップも大事です。

たとえば、ゲストWi-Fiを作って、動作確認もOKだったら、すぐ取ります。

```txt
backup-LN6001-after-guest-wifi-ok-20260621.tar.gz
```

この「動作確認済み」のバックアップが一番使いやすいです。

```txt
変更前バックアップ = 戻るため
変更後バックアップ = 成功状態を保存するため
```

両方あると安心です。

## 部分復元という考え方

バックアップを丸ごと戻すだけが復元ではありません。

SSHへ入れるなら、特定の設定ファイルだけ戻すこともできます。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-07.png)

全部戻すと、他の正常な変更まで巻き戻すことがあります。

直前に触った設定が分かっているなら、部分復元のほうがよい場合があります。

## firewallだけ戻す例

```sh
echo "### save current firewall"
cp /etc/config/firewall /etc/config/firewall.before-restore.$(date +%Y%m%d-%H%M)

echo "### restore firewall from backup"
cp /root/before-firewall-20260621-1430/firewall /etc/config/firewall

echo "### restart firewall"
/etc/init.d/firewall restart

echo "### verify"
uci show firewall | head -n 120
logread | grep -Ei 'firewall|drop|reject' | tail -n 80
```

## wirelessだけ戻す例

```sh
echo "### save current wireless"
cp /etc/config/wireless /etc/config/wireless.before-restore.$(date +%Y%m%d-%H%M)

echo "### restore wireless from backup"
cp /root/before-wifi-20260621-1430/wireless /etc/config/wireless

echo "### reload wifi"
wifi reload

echo "### verify"
wifi status
logread | grep -Ei 'wireless|wifi|hostapd' | tail -n 80
```

## networkだけ戻す例

networkは影響が大きいです。

有線LANで作業している時に行うのがおすすめです。

```sh
echo "### save current network"
cp /etc/config/network /etc/config/network.before-restore.$(date +%Y%m%d-%H%M)

echo "### restore network from backup"
cp /root/before-network-20260621-1430/network /etc/config/network

echo "### restart network"
/etc/init.d/network restart
```

network restart後、LuCIやSSHが切れることがあります。

PC側のIPを取り直し、Default Gatewayを確認します。

## dhcpだけ戻す例

```sh
echo "### save current dhcp"
cp /etc/config/dhcp /etc/config/dhcp.before-restore.$(date +%Y%m%d-%H%M)

echo "### restore dhcp from backup"
cp /root/before-dhcp-20260621-1430/dhcp /etc/config/dhcp

echo "### restart dnsmasq"
/etc/init.d/dnsmasq restart

echo "### verify"
uci show dhcp | head -n 160
logread | grep -i dnsmasq | tail -n 80
```

DHCP / DNS系のトラブルは、`dnsmasq` 再起動で直ることもあります。

```sh
/etc/init.d/dnsmasq restart
```

## 復元前に必ず現在状態を保存する

復元前にも、現在状態を残しておきます。

```sh
ROLLBACK_DIR="/root/before-restore-$(date +%Y%m%d-%H%M)"
mkdir -p "$ROLLBACK_DIR"

for cfg in network wireless firewall dhcp system; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$ROLLBACK_DIR/$cfg"
    uci show "$cfg" > "$ROLLBACK_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

logread | tail -n 200 > "$ROLLBACK_DIR/logread-tail.txt"
ls -l "$ROLLBACK_DIR"
```

復元しようとして、さらに別の問題を作ることもあります。

今の状態を保存してから戻せば、二重に迷子になりにくくなります。

## ファームウェア更新前のバックアップ

ファームウェア更新前は、必ずバックアップを取ります。

LuCIバックアップに加えて、CLIで追加パッケージ一覧も保存します。

```sh
FW_DIR="/root/before-firmware-update-$(date +%Y%m%d-%H%M)"
mkdir -p "$FW_DIR"

sysupgrade -b "$FW_DIR/config-backup.tar.gz"
opkg list-installed > "$FW_DIR/opkg-list-installed.txt"

for cfg in network wireless firewall dhcp system dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$FW_DIR/$cfg"
    uci show "$cfg" > "$FW_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

ubus call system board > "$FW_DIR/system-board.json"
ifstatus wan > "$FW_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$FW_DIR/ifstatus-wan6.json" 2>/dev/null || true
logread | tail -n 300 > "$FW_DIR/logread-tail.txt"

ls -l "$FW_DIR"
```

PCへコピーします。

```sh
scp -r root@192.168.1.1:/root/before-firmware-update-* ~/Downloads/
```

ファームウェア更新後は、追加パッケージが残っているか確認します。

```sh
opkg list-installed | grep -Ei 'adblock|banip|tailscale|wireguard|vpn|ipoe|luci-app'
```

設定バックアップだけでは、追加パッケージ本体が戻らないことがあります。

この点はかなり大事です。

## バックアップを戻した後の確認順

復元後は、次の順番で確認します。

```txt
LuCIへ入れる
  ↓
LAN IPを確認
  ↓
WAN / WAN6を確認
  ↓
DNSを確認
  ↓
Wi-Fiを確認
  ↓
DHCP予約を確認
  ↓
firewall zoneを確認
  ↓
ゲストWi-Fiを確認
  ↓
VLAN / Device / Kidsを確認
  ↓
Adblockを確認
  ↓
VPNを確認
```

コマンドで見るなら、まずこれです。

```sh
ubus call system board
ifstatus wan
ifstatus wan6
ping -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
nslookup www.google.co.jp
uci show wireless | grep -E 'ssid|network|disabled'
cat /tmp/dhcp.leases
uci show firewall | grep -E 'zone|forwarding|guest|device|kids|vpn|Allow-'
```

復元後に一番やりがちなのが、

```txt
戻したから大丈夫
```

と思うことです。

戻したあとに、実端末で確認します。

特にゲストWi-Fiは、インターネットへ出られるだけでなく、LANへ入れないことも確認します。

## バックアップを丸ごと戻さないほうがよいケース

バックアップは便利ですが、いつでも丸ごと戻せばよいわけではありません。

次のような場合は注意します。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/table-08.png)

この場合は、必要な設定だけ手で戻すほうが安全です。

たとえば、

```txt
DHCP予約だけ戻す
SSIDだけ参考にする
firewallは新しく作り直す
VPNは再認証する
```

という進め方です。

バックアップは材料です。

必ず丸ごと戻すものではありません。

## バックアップ保管ルールの例

家庭なら、これくらいで十分です。

```txt
保管場所:
  PCのDocuments/router-backups/LN6001/
  非公開クラウドに暗号化して保存

世代:
  initial
  before-major-change
  after-working-change
  before-firmware-update
  after-recovery

削除:
  古すぎるものは月1回整理
```

小さなオフィスや店舗なら、もう少し丁寧にします。

```txt
保管場所:
  管理者PC
  社内共有の非公開ストレージ
  外部SSD

ルール:
  変更前と変更後の2本を残す
  ファイル名に日付と変更内容を入れる
  バックアップ取得者をメモする
  復元テスト履歴を残す
  退職者がアクセスできない場所に置く
```

バックアップファイルの保管場所も、ネットワーク運用の一部です。

## 復元テストは必要？

理想を言えば、復元テストはしたほうがよいです。

ただし、家庭や小さなオフィスで本番ルーターを気軽に復元テストするのは少し怖いです。

現実的には、次くらいで十分です。

- バックアップファイルがPCに保存できているか確認する
- ファイルサイズが0ではないか確認する
- ファイル名とメモが合っているか確認する
- 重要設定が `uci show` の控えに残っているか確認する
- 復元手順を記事やメモで確認しておく

小さなオフィスや店舗で予備機があるなら、予備機で復元テストをしてもよいです。

ただし、本番機で気軽に復元テストする時は、作業時間と戻し方を必ず確保します。

## バックアップ運用メモのテンプレート

```txt
バックアップ運用メモ:

製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

LAN IP:
  192.168.1.1

主要ネットワーク:
  lan: 192.168.1.0/24
  guest: 192.168.2.0/24
  device: 192.168.3.0/24

追加機能:
  オートIPoE: あり / なし
  Adblock: あり / なし
  VPN Assistant: あり / なし
  Tailscale: あり / なし
  WireGuard: あり / なし

最新バックアップ:
  backup-LN6001-after-guest-wifi-ok-20260621.tar.gz

保存場所:
  管理者PC
  非公開クラウド
  外部SSD

更新履歴:
  2026-06-21 初期設定後バックアップ
  2026-06-21 ゲストWi-Fi作成前バックアップ
  2026-06-21 ゲストWi-Fi動作確認後バックアップ

注意:
  バックアップファイルは公開しない
  復元後はLAN IPとWi-Fi名が戻る可能性あり
```

## よくある失敗

### バックアップをルーター内部にしか置いていない

`/root` や `/tmp` に置いただけでは不十分です。

初期化やファームウェア更新で消える可能性があります。

必ずPCへコピーします。

```sh
scp -r root@192.168.1.1:/root/config-backup-* ~/Downloads/
```

### どのバックアップが何か分からない

ファイル名が `backup.tar.gz` だけだと、あとから分かりません。

日付と状態を入れます。

```txt
backup-LN6001-before-vlan-20260621.tar.gz
backup-LN6001-after-vpn-ok-20260621.tar.gz
```

### 設定バックアップだけで追加パッケージも戻ると思っている

追加パッケージ本体は戻らない場合があります。

`opkg list-installed` を保存し、必要なものは復旧後に再導入します。

### 壊れた状態のバックアップを戻してしまう

不具合発生後に取ったバックアップを戻すと、不具合も戻ることがあります。

「正常に動いていた時のバックアップ」を残しておくことが大事です。

### バックアップを公開場所に置いてしまう

設定バックアップには、SSID、パスワード、内部IP、VPN情報などが含まれることがあります。

公開GitHubやSNSへ置かないでください。

### 復元後にアクセスできず焦る

復元後、LAN IPがバックアップ時点のものへ戻ることがあります。

`192.168.1.1` で入れない場合は、PC側のDefault Gatewayを確認します。

## まとめ

LN6001-JPでOpenWrtベースの設定を触るなら、バックアップはかなり重要です。

ポイントは次の通りです。

1. バックアップは壊れた時ではなく、動いている時に取る
2. LuCIの **Generate archive** で設定バックアップをPCへ保存する
3. バックアップファイルには秘密情報が含まれるので公開しない
4. SSHで `/etc/config/` と `uci show` の控えも残す
5. `opkg list-installed` で追加パッケージ一覧を保存する
6. `sysupgrade -b` / `sysupgrade -r` も使えるようにしておく
7. 大きな変更の前後でバックアップを取る
8. ファイル名に日付と状態を入れる
9. 復元後はWAN、Wi-Fi、DHCP、firewall、VPNを確認する
10. 丸ごと戻すべきか、部分的に戻すべきかを考える

OpenWrtベースルーターの設定は、自由度が高いです。

だからこそ、戻れる状態を作ってから触るのが一番安全です。

バックアップは地味です。

でも、VLANで迷子になった時、firewallでLuCIへ入れなくなった時、ファームウェア更新後にVPNが消えた時、めちゃくちゃ効きます。

設定を変える前にバックアップ。

動いたらまたバックアップ。

この習慣だけで、LN6001-JPはかなり安心して育てられます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-03.png)

## 次に読むなら

バックアップ運用を固めたら、次の記事も合わせて読むと安心です。

- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)
- [ファームウェア更新運用](https://note.com/ikmsan/n/nff6b598da354)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)

初期化前にどう戻すかを知りたい人は、リセットと復旧へ。

ファームウェア更新前後の確認を固めたい人は、ファームウェア更新運用へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、バックアップ/復元ボタンの表記、`sysupgrade`、`opkg`、`/etc/sysupgrade.conf`、追加モジュールの扱いは、ファームウェアやモジュール更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずLinksys公式サポートやOpenWrtの最新ドキュメントを確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPの設定バックアップはどこから取れますか？

LuCIへログインし、**System → Backup / Flash Firmware** を開き、**Generate archive** をクリックします。

`.tar.gz` ファイルがダウンロードされるので、PCへ保存してください。

### バックアップファイルには何が入りますか？

主にルーター設定が含まれます。

SSID、Wi-Fiパスワード、LAN IP、firewall、DHCP予約、DNS、VPN関連設定など、重要情報が含まれる可能性があります。

公開場所へ置かないでください。

### バックアップを取るタイミングは？

初期設定後、IPoE設定後、ゲストWi-Fi設定前後、VLANやfirewall変更前、VPN導入前、Adblock導入前、ファームウェア更新前後がおすすめです。

### LuCIバックアップだけで十分ですか？

最低限はLuCIバックアップで大丈夫です。

ただし、追加パッケージ本体は戻らない場合があります。

`opkg list-installed` も保存しておくと復旧しやすくなります。

### SSHでもバックアップできますか？

できます。

`sysupgrade -b /tmp/backup.tar.gz` で設定バックアップを作成できます。

作成後は必ずPCへコピーしてください。

### 復元後に `192.168.1.1` へ入れません

バックアップ時点のLAN IPへ戻っている可能性があります。

PC側のDefault Gatewayを確認してください。

`192.168.10.1` などへ変更していた場合、そのIPでLuCIへアクセスします。

### 追加パッケージも自動で戻りますか？

戻らない場合があります。

Adblock、VPN Assistant、Tailscale、WireGuard、オートIPoEなどは、必要に応じて再導入します。

### 壊れたあとに取ったバックアップを戻していいですか？

注意が必要です。

壊れた設定も一緒に戻る可能性があります。

正常に動いていた時のバックアップがあるなら、そちらを優先します。

原因が分かっている場合は、network、wireless、firewall、dhcpなどを部分的に戻す方法もあります。

### バックアップはどこに保存すればいいですか？

PCのローカルフォルダ、暗号化された外部ストレージ、非公開クラウドなどがおすすめです。

公開GitHub、SNS、誰でも見られる共有フォルダには置かないでください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041/
- Linksys OpenWRT router configuration backup and restore: https://support.linksys.com/kb/article/218-en/?section_id=175
- OpenWrt Wiki - Backup and restore: https://openwrt.org/docs/guide-user/troubleshooting/backup_restore
- OpenWrt Wiki - Preserving settings during firmware upgrade: https://openwrt.org/docs/guide-quick-start/admingui_sysupgrade_keepsettings
- OpenWrt Wiki - Sysupgrade: https://openwrt.org/docs/guide-user/installation/generic.sysupgrade

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
