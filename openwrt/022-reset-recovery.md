<!-- mirror-source: articles/022-reset-recovery.md -->

# リセットボタンを押す前に｜LN6001-JPでLuCI・SSHへ戻る復旧手順【OpenWrt集中連載022】

リセットボタンって、すごく強いです。

強いからこそ、最初に押すものではありません。

LAN IPを変えたらLuCIへ入れない。  
VLANを触ったら管理画面に戻れない。  
ゲストWi-Fi端末がIPを取れない。  
firewallを変えたらインターネットへ出られない。  
Wi-Fi名を変えたら、どのSSIDにつなげばいいか分からない。

こうなると、つい思います。

```txt
もう初期化かな……
```

分かります。

でも、まだ早いです。

「壊れた」のではなく、管理画面の住所が変わっただけかもしれません。  
Wi-Fiでは入れないけど、有線LANなら入れるかもしれません。  
LuCIは無理でも、SSHには入れるかもしれません。  
firewallだけ戻せば復旧するかもしれません。  
wirelessだけ戻せばSSIDが復活するかもしれません。

LN6001-JPはOpenWrtベースなので、LuCI、SSH、設定バックアップ、個別設定ファイルのロールバックを使えば、**初期化せずに直前の変更だけ戻せる** ことがあります。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で「壊れたかも」と思った時に、初期化前に見ること、SSHで戻せる可能性、LuCIバックアップからの復元、ソフトリセット、ハードリセット、初期化後の再設定順を整理します。

初期化は最後の手段です。

まず、帰り道を探します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、リセット方法を覚えることではありません。

まずは、ここまでできればOKです。

- 初期化する前に確認する順番が分かる
- 「壊れた」のではなく「アクセス先が変わっただけ」を見分けられる
- 有線LANでLuCIへ戻る方法が分かる
- SSHへ入れる場合に、現在状態を保存できる
- network、wireless、firewall、dhcpを個別に戻す考え方が分かる
- LuCIバックアップから復元できる
- `firstboot` とハードリセットの違いが分かる
- 初期化後に、どの順番で再設定すればよいか分かる
- 復旧後に、次回のためのバックアップを残せる

復旧で大事なのは、いきなり全部を戻すことではありません。

```txt
まだ入れる道があるか
どの設定を最後に触ったか
その設定だけ戻せないか
```

を順番に見ることです。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-01.png)

## 先にざっくり結論

リセット前に、まずこの順番で確認します。

```txt
有線LANでつなぐ
  ↓
PC側のIPアドレスを見る
  ↓
Default Gatewayを見る
  ↓
現在のLAN IPを探す
  ↓
LuCIへ入れるか確認する
  ↓
SSHへ入れるか確認する
  ↓
最後に変更した設定を思い出す
  ↓
network / wireless / firewall / dhcp を個別に戻せないか見る
  ↓
LuCIバックアップから復元する
  ↓
それでもだめなら初期化する
```

最初からハードリセットしないほうがよい理由は、設定を全部失うからです。

特に、次のような設定を積み上げている場合、初期化後の復旧に時間がかかります。

- IPoE / オートIPoE
- PPPoE
- ゲストWi-Fi
- VLAN
- firewall zone
- DHCP予約
- Adblock
- VPN Assistant
- Tailscale / WireGuard
- DNS / Family DNS
- 店舗や小さなオフィス向けのStaff / Guest / Device分離

初期化は最後の手段です。

まずは、

```txt
LuCIへ入れるか
SSHへ入れるか
直前に触った設定だけ戻せるか
```

を見ます。

## こういう時に向いています

この記事は、次のような時に役立ちます。

- LuCIへ入れなくなった
- Wi-FiのSSIDが見えなくなった
- ゲストWi-Fiだけ通信できない
- VLAN設定後に管理画面へ戻れない
- firewall変更後にインターネットへ出られない
- LAN IPを変更したあと、どこへアクセスすればよいか分からない
- 管理者パスワードを忘れた
- 初期化する前に、まだ残せるバックアップがないか確認したい
- リセット後に何から戻せばよいか整理したい
- 店舗や小さなオフィスで、初期化による業務停止を避けたい

「もうだめ」と思った時ほど、まず有線LANです。

有線で入れるなら、まだかなり希望があります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-02.png)

## 最初に言葉だけそろえる

復旧まわりの言葉を、ざっくり整理します。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-01.png)

最初は、これだけ覚えれば大丈夫です。

```txt
LuCIへ入れるなら画面から復旧
SSHへ入れるなら設定ファイル単位で復旧
どちらも無理ならリセットを検討
```

## リセット前に考えること

リセットは便利です。

でも、便利すぎて強すぎます。

リセットすると、多くの設定が初期状態へ戻ります。

そのため、まずは次を考えます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-02.png)

「入れない」と「壊れた」は同じではありません。

設定変更で、管理画面への道が変わっただけのことがあります。

まずは道を探します。

## まず有線LANでつなぐ

Wi-Fiが消えた。  
ゲストWi-FiからLuCIへ入れない。  
SSID変更後に戻れない。  
VLANを触ったら端末が迷子になった。

こういう時は、まず有線LANです。

```txt
PC
  ↓ 有線LAN
LN6001-JPのLANポート
```

ブラウザでLuCIへアクセスします。

```txt
https://192.168.1.1
```

LAN IPを変更している場合は、変更後のIPでアクセスします。

例:

```txt
https://192.168.10.1
```

有線LANで入れるなら、ルーター本体はかなりの確率で生きています。

この場合、いきなりハードリセットする必要はありません。

Wi-Fi設定だけ。  
firewallだけ。  
networkだけ。

そういう部分復旧で済む可能性があります。

## PC側のIPアドレスを確認する

LuCIへ入れない時は、PCがどのIPアドレスを取っているか確認します。

Windows:

```powershell
ipconfig
```

macOS / Linux:

```sh
ip addr show
```

見るポイントです。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-03.png)

`169.254.x.x` の場合、ルーターからDHCPでIPを取得できていません。

この時点では、WANやWi-Fiよりも、LAN側DHCPや物理接続を見ます。

「管理画面が壊れた」のではなく、PCが住所をもらえていないだけかもしれません。

## ルーターのIPを探す

LAN IPを変えたあとに忘れた場合、PC側のDefault Gatewayを見ると見つかることがあります。

Windows:

```powershell
ipconfig
```

`Default Gateway` を確認します。

macOS / Linux:

```sh
ip route show default
```

例:

```txt
default via 192.168.10.1 dev en0
```

この場合、LuCIのURLは次です。

```txt
https://192.168.10.1
```

`192.168.1.1` に入れないだけで、実際には `192.168.10.1` へ移動していることがあります。

これは本当にありがちです。

```txt
壊れたのではなく、引っ越しただけ
```

と思って、まずDefault Gatewayを見ます。

## LEDと回線状態を見る

リセット前に、LEDと回線側も見ます。

LN6001-JP / MBE70系のLEDは、ざっくり次のように見ます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-04.png)

赤点灯だからといって、すぐ初期化ではありません。

WANケーブル。  
ONU。  
ホームゲートウェイ。  
IPoE。  
PPPoE。  
回線障害。

このあたりが原因かもしれません。

初期化しても、回線側が原因なら直りません。

赤点灯の時ほど、まずWAN側を見ます。

## 初期化前チェックリスト

リセットボタンを押す前に、次を確認します。

```txt
□ 有線LANでPCを接続した
□ PC側のIPアドレスを確認した
□ Default Gatewayを確認した
□ 192.168.1.1だけでなく、変更後のLAN IPも試した
□ LuCIへ入れるか確認した
□ SSHへ入れるか確認した
□ 最後に変更した設定を思い出した
□ LuCIバックアップがあるか確認した
□ CLIで現在状態を保存できるか確認した
□ 追加パッケージ一覧を保存できるか確認した
□ 本当に管理者パスワードを忘れているか確認した
```

このうち1つでも手がかりがあれば、初期化前に戻せる可能性があります。

リセットボタンは強いです。

だから、最後に使います。

## SSHへ入れる場合は現在状態を保存する

SSHへ入れるなら、初期化前に現在状態を保存します。

これはめちゃくちゃ大事です。

初期化後に、

```txt
あれ、前のLAN IPなんだっけ？
どのSSIDを作ってたっけ？
Tailscale入れてたっけ？
DHCP予約どうしてたっけ？
```

となるのを防げます。

```sh
BACKUP_DIR="/root/recovery-before-reset-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

echo "### system"
ubus call system board > "$BACKUP_DIR/system-board.json" 2>/dev/null || true
uptime > "$BACKUP_DIR/uptime.txt"
df -h > "$BACKUP_DIR/df-h.txt"
free > "$BACKUP_DIR/free.txt"

echo "### network status"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/ip-route.txt"
ip -6 route show > "$BACKUP_DIR/ip6-route.txt"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true

echo "### config files"
for cfg in network wireless firewall dhcp system dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

echo "### leases and packages"
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt" 2>/dev/null || true

echo "### logs"
logread | tail -n 300 > "$BACKUP_DIR/logread-tail.txt" 2>/dev/null || true
dmesg | tail -n 120 > "$BACKUP_DIR/dmesg-tail.txt" 2>/dev/null || true

echo "### sysupgrade backup"
sysupgrade -b "$BACKUP_DIR/config-backup.tar.gz"

echo "saved: $BACKUP_DIR"
ls -l "$BACKUP_DIR"
```

このバックアップはルーター内部にあります。

本当に初期化する前に、PCへコピーします。

PC側のターミナルで実行します。

```sh
scp -r root@192.168.1.1:/root/recovery-before-reset-* ~/Downloads/
```

LAN IPを変更している場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/root/recovery-before-reset-* ~/Downloads/
```

この診断データには、SSID、Wi-Fiパスワード、MACアドレス、IPv6 prefix、VPN情報、PPPoE IDなどが含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

## 最後に触った設定だけ戻す

SSHへ入れる場合、全部初期化しなくても、直前に触った設定だけ戻せることがあります。

よくある対象は次です。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-05.png)

大事なのは、**今のファイルを保存してから戻す** ことです。

戻すつもりでさらに壊すのは、ちょっと悲しいです。

## networkだけ戻す

LAN IPやVLANを触った直後に戻す場合です。

```sh
echo "### save current network"
cp /etc/config/network /etc/config/network.before-rollback.$(date +%Y%m%d-%H%M)

echo "### show backup candidates"
ls -l /etc/config/network*

echo "### restore previous network config"
cp /etc/config/network.backup.20260620-1430 /etc/config/network

echo "### restart network"
/etc/init.d/network restart
```

`network.backup.20260620-1430` は実際のバックアップファイル名に置き換えます。

`/etc/init.d/network restart` 後は、LuCIやSSHが一度切れることがあります。

PC側のIPを取り直し、変更後のLAN IPへアクセスします。

networkは影響範囲が大きいです。

有線LANで作業している状態で実行するのがおすすめです。

## firewallだけ戻す

firewall変更後に通信できなくなった場合です。

```sh
echo "### save current firewall"
cp /etc/config/firewall /etc/config/firewall.before-rollback.$(date +%Y%m%d-%H%M)

echo "### show backup candidates"
ls -l /etc/config/firewall*

echo "### restore previous firewall config"
cp /etc/config/firewall.backup.20260620-1430 /etc/config/firewall

echo "### restart firewall"
/etc/init.d/firewall restart

echo "### verify"
uci show firewall | head -n 120
logread | grep -Ei 'firewall|reject|drop' | tail -n 80
```

firewallだけなら、networkを戻すより影響範囲が小さいことが多いです。

ゲストWi-Fi、VLAN、VPN、ポート開放を触った直後なら、まずfirewallを疑います。

## wirelessだけ戻す

SSIDや暗号化方式を変えて、Wi-Fi端末がつながらなくなった場合です。

```sh
echo "### save current wireless"
cp /etc/config/wireless /etc/config/wireless.before-rollback.$(date +%Y%m%d-%H%M)

echo "### restore previous wireless config"
cp /etc/config/wireless.backup.20260620-1430 /etc/config/wireless

echo "### reload wifi"
wifi reload

echo "### verify"
wifi status
logread | grep -Ei 'wireless|wifi|hostapd' | tail -n 80
```

Wi-Fi設定を戻す時は、有線LANで入っている状態で行うのがおすすめです。

Wi-Fi経由で作業していると、戻す途中で自分が切断されます。

## dhcpだけ戻す

DHCP予約、DNS、Family DNS、Adblock関連のDHCP Optionsを触ったあとに戻す場合です。

```sh
echo "### save current dhcp"
cp /etc/config/dhcp /etc/config/dhcp.before-rollback.$(date +%Y%m%d-%H%M)

echo "### restore previous dhcp config"
cp /etc/config/dhcp.backup.20260620-1430 /etc/config/dhcp

echo "### restart dnsmasq"
/etc/init.d/dnsmasq restart

echo "### verify"
uci show dhcp | head -n 160
logread | grep -i dnsmasq | tail -n 80
```

DHCP / DNSだけがおかしいなら、まず `dnsmasq` 再起動で直ることもあります。

```sh
/etc/init.d/dnsmasq restart
```

DNS広告ブロックやFamily DNSを触った直後に一部サイトが開かない場合も、dhcp / dnsmasqまわりを確認します。

## LuCIへ入れる場合のソフトリセット

LuCIへ入れる場合は、まだ状況確認やバックアップの余地があります。

ソフトリセットは、LuCIから工場出荷状態に戻す方法です。

1. LuCIへログインする
2. **System** → **Backup / Flash Firmware** を開く
3. まず **Generate archive** でバックアップをPCへ保存する
4. 必要なら追加パッケージ一覧も保存する
5. **Reset to defaults** または **Perform reset** を実行する
6. 確認ダイアログで実行する
7. ルーターの再起動を待つ
8. 工場出荷設定で再ログインする

画面名はファームウェアやLuCIのバージョンで少し変わることがあります。

「Reset to defaults」「Perform reset」など、初期化を意味する操作は、実行前にバックアップを取ってから進めます。

ここでバックアップなしに押すのは、なかなか勇敢です。

できればやめましょう。

## ソフトリセット後に戻るもの

初期化後は、基本的に次が工場出荷状態へ戻ります。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-06.png)

初期化後は、LuCIへ次で入ります。

```txt
https://192.168.1.1
```

ログイン情報は次です。

```txt
ユーザー名: root
パスワード: 本体底面ラベルの初期パスワード
```

## ハードリセットは最後の手段

LuCIへ入れない。  
SSHへも入れない。  
管理者パスワードも分からない。  
有線LANでも戻れない。

この場合は、ハードリセットを検討します。

ただし、ハードリセットを行うと設定は消えます。

実行前に、バックアップがPCにあるか、再設定に必要な情報があるかを確認します。

## ハードリセット手順

1. LN6001-JPの電源が入った状態にする
2. 本体底面のRESETボタンを確認する
3. クリップや細いピンでRESETボタンを約10秒押し続ける
4. LEDが赤から青点滅へ変わることを確認する
5. ボタンを離す
6. ルーターが再起動するまで待つ
7. インターネット接続を検知すると白点灯になる
8. 赤点灯のままならWAN配線や回線設定を確認する

ハードリセット中に電源を抜かないでください。

再起動には時間がかかることがあります。

すぐ失敗と判断せず、LEDの状態を見ながら待ちます。

赤点灯のままなら、設定ではなくWAN側が原因かもしれません。

ONU、HGW、WANケーブル、IPoE / PPPoE設定を見ます。

## ハードリセット後の初期アクセス

初期化後は、PCまたはスマートフォンをLN6001-JPへ接続します。

有線LANがあるなら、有線がおすすめです。

LuCIへアクセスします。

```txt
https://192.168.1.1
```

ログイン情報は次です。

```txt
ユーザー名: root
パスワード: 本体底面ラベルの初期パスワード
```

初回アクセス時に証明書警告が出ることがあります。

アクセス先が `https://192.168.1.1` で、LN6001-JPへ直接つないでいることを確認して進めます。

## SSHから初期化する

SSHへ入れる場合、OpenWrt系の `firstboot` で初期化できます。

ただし、実行すると設定は消えます。

先にバックアップを取ります。

```sh
echo "### create backup before firstboot"
sysupgrade -b /tmp/before-firstboot-$(date +%Y%m%d-%H%M).tar.gz
opkg list-installed > /tmp/opkg-before-firstboot-$(date +%Y%m%d-%H%M).txt

echo "### check backup files"
ls -lh /tmp/before-firstboot-*.tar.gz /tmp/opkg-before-firstboot-*.txt
```

PCへコピーします。

```sh
scp root@192.168.1.1:/tmp/before-firstboot-*.tar.gz ~/Downloads/
scp root@192.168.1.1:/tmp/opkg-before-firstboot-*.txt ~/Downloads/
```

その後、初期化します。

```sh
firstboot -y
reboot
```

`firstboot` はかなり強いコマンドです。

「設定ファイル単位で戻せる可能性」があるなら、先に部分ロールバックを検討します。

## failsafeについて

OpenWrtにはfailsafe modeという復旧手段があります。

一般的なOpenWrtでは、起動時に特定操作を行い、最小限の状態で起動して設定を直すことがあります。

ただし、LN6001-JPでfailsafeを使うかどうかは、公式手順や実機の挙動を確認してからにします。

この連載では、通常運用の復旧手順としては、次を優先します。

1. 有線LANでLuCIへ入る
2. SSHへ入る
3. 設定ファイル単位で戻す
4. LuCIバックアップから復元する
5. ソフトリセットする
6. ハードリセットする
7. それでもだめなら公式サポートへ相談する

failsafeは、OpenWrtに慣れた人向けの最後寄りの手段として考えます。

まずはLuCIとSSHで戻れる道を探しましょう。

## LuCIバックアップから復元する

バックアップファイルがある場合は、初期化後に復元できます。

## LuCIで復元する

1. LuCIへログインする
2. **System** → **Backup / Flash Firmware** を開く
3. **Restore backup** または **Upload archive** を選ぶ
4. 保存していた `.tar.gz` ファイルを選択する
5. アップロードして復元する
6. 再起動を待つ
7. LuCIへ再ログインする
8. WAN、WAN6、Wi-Fi、DHCP、firewallを確認する

復元後、LAN IPがバックアップ時点のものへ戻ることがあります。

たとえば、初期化直後は `192.168.1.1` でも、復元後に `192.168.10.1` へ戻る場合があります。

復元後にLuCIへ入れない場合は、PC側のDefault Gatewayを確認します。

また住所が変わっているだけかもしれません。

## SSHで復元する

PCからバックアップをルーターへ送ります。

```sh
scp backup-LN6001-before-reset-20260620.tar.gz root@192.168.1.1:/tmp/
```

ルーター側で復元します。

```sh
sysupgrade -r /tmp/backup-LN6001-before-reset-20260620.tar.gz
reboot
```

復元後、LuCIへ再ログインして状態を確認します。

## バックアップ復元で注意すること

バックアップ復元は便利ですが、万能ではありません。

問題の原因になっていた設定も戻ることがあります。

たとえば、firewall設定を壊した状態でバックアップを取っていた場合、そのバックアップを戻すと同じ問題が戻ります。

次のような場合は、バックアップを丸ごと戻すより、必要な設定だけ手で戻すほうがよいことがあります。

- firewallやVLAN設定が原因で入れなくなった
- 古い追加パッケージ設定が壊れている
- ファームウェア更新後に古い設定を戻して不安定になった
- 以前から原因不明の不具合があった
- どの設定が壊れているかある程度分かっている

バックアップは「復元の材料」です。

常に丸ごと戻すもの、とは考えなくて大丈夫です。

## 追加パッケージは別に戻す必要がある

初期化後やファームウェア更新後、設定は戻っても、追加パッケージ本体が戻らないことがあります。

代表例です。

- Adblock
- VPN Assistant
- Tailscale
- WireGuard関連
- オートIPoE
- banIP
- 追加LuCIアプリ
- 独自スクリプト

更新前や初期化前に保存したパッケージ一覧を見ます。

```sh
opkg list-installed > /tmp/opkg-after-recovery.txt
```

必要なものだけ再導入します。

```sh
opkg update
```

Adblockなら例です。

```sh
opkg install adblock luci-app-adblock luci-i18n-adblock-ja
```

オートIPoEやVPN Assistantは、LN6001-JP向けのLinksys公式サポートで案内されているモジュールを使います。

一般的なOpenWrt記事のコマンドをそのまま流用する前に、LN6001-JP向け手順を確認してください。

## 初期化後の再設定順

初期化後は、全部を一気に戻そうとしないほうが安全です。

おすすめの順番は次です。

```txt
管理パスワード
  ↓
LAN IP
  ↓
WAN / WAN6 / IPoE
  ↓
メインWi-Fi
  ↓
初期バックアップ
  ↓
DHCP予約
  ↓
ゲストWi-Fi
  ↓
VLAN / firewall zone
  ↓
Adblock / DNS
  ↓
VPN
  ↓
監視・ログ
```

最初に戻すべきなのは、基本の足場です。

インターネット。  
メインWi-Fi。  
管理画面。  
バックアップ。

ゲストWi-Fi、Adblock、VPN、VLANはそのあとで大丈夫です。

一気に全部戻すと、問題が起きた時に原因が分かりにくくなります。

## 初期化後チェックリスト

```txt
□ 本体底面ラベルのSSIDと初期パスワードを確認した
□ https://192.168.1.1 へアクセスできた
□ rootでログインできた
□ 管理者パスワードを変更した
□ タイムゾーンをAsia/Tokyoへ設定した
□ ホスト名を設定した
□ WAN / WAN6を確認した
□ 必要ならオートIPoEを再導入した
□ メインSSIDを設定した
□ Wi-Fiパスワードを設定した
□ LuCIバックアップを1本作成した
□ DHCP予約を戻した
□ ゲストWi-Fiを戻した
□ VLAN / firewall zoneを戻した
□ Adblockを再導入した
□ VPN Assistant / Tailscale / WireGuardを戻した
□ 最後にバックアップをPCへ保存した
```

最初は、インターネットとメインWi-Fiが戻ることを優先します。

細かい機能は、そのあと一つずつ戻します。

## よくある「壊れていないのに入れない」ケース

## LAN IPを変更しただけ

症状:

```txt
https://192.168.1.1 に入れない
```

原因:

```txt
LAN IPを 192.168.10.1 などへ変更している
```

確認:

```sh
ip route show default
```

またはPC側のDefault Gatewayを確認します。

対処:

```txt
https://192.168.10.1
```

のように、新しいLAN IPでアクセスします。

## ゲストWi-FiからLuCIへ入ろうとしている

ゲストWi-Fiでは、LuCIやSSHへのアクセスを拒否する設計が多いです。

症状:

```txt
Wi-FiにはつながるがLuCIへ入れない
```

原因:

```txt
ゲストWi-Fiに接続している
```

対処:

- メインSSIDへ接続する
- 有線LANで接続する
- 管理用PCからアクセスする

ゲストWi-Fiから管理画面に入れないのは、むしろ正しい設計です。

## firewallでLuCIを閉じた

症状:

```txt
インターネットは使えるがLuCIへ入れない
```

原因:

```txt
lan zoneのInputをREJECTにした
管理端末をguest/device側へ移した
Traffic Ruleで管理通信を止めた
```

対処:

SSHへ入れる場合、firewallを戻します。

```sh
cp /etc/config/firewall.before-rollback.20260620-1430 /etc/config/firewall
/etc/init.d/firewall restart
```

SSHへ入れない場合は、有線LANで管理用ネットワークへ戻れるか確認します。

## Wi-Fi暗号化を変えて古い端末がつながらない

症状:

```txt
新しいスマートフォンはつながるが、古いプリンターやIoT機器がつながらない
```

原因:

```txt
WPA3-onlyにした
2.4GHz用SSIDを消した
SSID名や記号で相性が出ている
```

対処:

- IoT用2.4GHz SSIDを戻す
- 互換性重視の暗号化に戻す
- 有線LANでLuCIへ入り、Wireless設定を戻す

IoT機器は、最新Wi-Fi設定に全部ついてきてくれるとは限りません。

速さより、まずつながることです。

## LuCIだけ止まっている

症状:

```txt
SSHへは入れるがLuCIが開かない
```

確認:

```sh
/etc/init.d/uhttpd status
logread | grep -Ei 'uhttpd|luci|http|ssl' | tail -n 80
```

対処:

```sh
/etc/init.d/uhttpd restart
```

SSHも確認します。

```sh
/etc/init.d/dropbear status
```

LuCIパッケージやuhttpd設定を壊している場合は、追加確認が必要です。

SSHへ入れるなら、まだかなり復旧できます。

## 復旧後に必ずやること

復旧できたら、そこで終わりにしないほうがよいです。

次回のために、すぐバックアップを取ります。

ここで油断すると、また次回同じところで苦しみます。

## LuCIバックアップを取る

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` をPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-after-recovery-20260621.tar.gz
```

## 追加パッケージ一覧を保存する

```sh
opkg list-installed > /root/opkg-after-recovery-$(date +%Y%m%d-%H%M).txt
```

PCへコピーします。

```sh
scp root@192.168.1.1:/root/opkg-after-recovery-*.txt ~/Downloads/
```

## 復旧メモを残す

```txt
発生日:
  2026-06-21

症状:
  VLAN変更後にLuCIへ入れなくなった

原因:
  管理PCがdevice側へ移動していた

復旧:
  有線LANで接続
  /etc/config/networkを変更前バックアップへ戻した
  network restart後、192.168.1.1へ復帰

バックアップ:
  backup-LN6001-after-recovery-20260621.tar.gz

再発防止:
  VLAN変更前に有線管理ポートを残す
  network変更前にconfig backupを取る
```

こうしたメモがあると、次に同じ問題が起きた時にかなり早く戻せます。

未来の自分は、だいたい細かい設定を忘れています。

## 初期化しても解決しないケース

初期化は、設定の問題には効きます。

でも、次のような場合は初期化しても解決しないことがあります。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-07.png)

赤点灯のまま、WANが取れない、ONU / HGW側に問題がある場合は、初期化より先に回線側を見ます。

初期化は万能薬ではありません。

設定ミスには効きますが、回線障害や物理トラブルには効きません。

## リセットを避けるための普段の運用

復旧を楽にするには、普段から次をやっておくのが一番です。

1. 大きな変更前にLuCIバックアップを取る
2. SSHで `/etc/config/` を控える
3. 追加パッケージ一覧を保存する
4. DHCP予約表を作る
5. VLAN / firewall zoneの設計メモを残す
6. IPoE / PPPoE / VPN設定をメモする
7. 設定変更は1つずつ行う
8. 変更後すぐ動作確認する
9. 動いた状態でバックアップを保存する
10. バックアップをルーター内部だけでなくPCにも保存する

リセットボタンを押す機会を減らすには、変更前バックアップが一番効きます。

地味ですが、これが最強です。

## 復旧メモのテンプレート

```txt
復旧メモ:

製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

発生日時:
  2026-06-21 14:30

症状:
  LuCIへ入れない
  Wi-Fiは見える / 見えない
  WANは生きている / 不明

直前の変更:
  LAN IP変更
  firewall変更
  VLAN変更
  ゲストWi-Fi追加
  Adblock導入
  VPN設定変更
  ファームウェア更新

確認:
  有線LAN: OK / NG
  SSH: OK / NG
  Default Gateway:
  LuCI URL:
  ifstatus wan:
  ifstatus wan6:

復旧方法:
  networkを戻した
  firewallを戻した
  LuCIバックアップから復元
  ハードリセット
  手動再設定

再発防止:
  次回から変更前バックアップを取る
  有線管理ポートを残す
  firewall zone設計メモを残す

バックアップ:
  backup-LN6001-after-recovery-YYYYMMDD.tar.gz
```

## まとめ

リセットと復旧で大事なのは、すぐ初期化しないことです。

まずは次の順番で見ます。

1. 有線LANで接続する
2. PC側IPとDefault Gatewayを確認する
3. 変更後のLAN IPでLuCIへアクセスする
4. LuCIへ入れるならバックアップを取る
5. SSHへ入れるなら現在状態を保存する
6. 最後に触った設定だけ戻す
7. LuCIバックアップから復元する
8. それでもだめならソフトリセットまたはハードリセットする
9. 初期化後はWAN、Wi-Fi、管理パスワードから順番に戻す
10. 復旧後すぐに新しいバックアップを取る

「壊れた」と思っても、実際にはLAN IPが変わっているだけ、ゲストWi-FiからLuCIへ入ろうとしているだけ、firewallだけ戻せばよいだけ、というケースは多いです。

初期化は最後の手段です。

戻せる余地を確認してから進めるだけで、OpenWrtベースルーターの運用はかなり安心になります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-03.png)

## 次に読むなら

復旧手順を固めたい人は、次の記事も合わせて読むと整理しやすいです。

- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [ファームウェア更新運用](https://note.com/ikmsan/n/nff6b598da354)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)

初期化前に設定を残す手順を固めたい人は、設定バックアップと復元へ。

なぜつながらないのかを切り分けたい人は、つながらない時の切り分けへ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、リセット表記、バックアップ/復元手順、`firstboot`、`sysupgrade`、`uhttpd`、`dropbear` などの挙動は、ファームウェアや追加モジュールで変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずLinksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPがおかしい時、すぐリセットしていい？

まず有線LAN、LuCI、SSHで戻せないか確認するのがおすすめです。

設定ファイル単位で戻せる場合があります。

特にLAN IP、Wi-Fi、firewall、DHCPの変更直後なら、直前の設定だけ戻せる可能性があります。

### ソフトリセットとハードリセットは何が違う？

ソフトリセットはLuCIやコマンドから初期化する方法です。

ハードリセットは本体のRESETボタンを約10秒押して初期化する方法です。

どちらも設定は消えますが、LuCIやSSHへ入れるなら、初期化前にバックアップや状態保存ができます。

### 管理者パスワードを忘れたらどうする？

LuCIにもSSHにも入れない場合、ハードリセットが必要になる可能性が高いです。

初期化後は、本体底面ラベルの初期パスワードでログインし、管理者パスワードを再設定します。

### `192.168.1.1` に入れない時は？

LAN IPを変更している可能性があります。

PC側のDefault Gatewayを確認してください。

`192.168.10.1` などへ変更している場合は、そのIPでLuCIへアクセスします。

### `firstboot` はいつ使う？

SSHへ入れる状態で、OpenWrt系の設定を初期化したい時に使います。

ただし、設定は消えるため、先に `sysupgrade -b` でバックアップを取り、PCへコピーしてから実行してください。

### LuCIバックアップを戻せば全部戻る？

主に設定ファイルは戻せます。

ただし、Adblock、VPN Assistant、オートIPoEなど、あとから入れたパッケージ本体は再導入が必要になる場合があります。

`opkg list-installed` の控えも保存しておくと復旧しやすくなります。

### ハードリセットしても赤点灯のままです

赤点灯は、モデム側からインターネット接続がない状態を示す場合があります。

初期化だけでなく、WANケーブル、ONU / HGW、回線設定、IPoE / PPPoE設定を確認してください。

### failsafeは使ったほうがいい？

OpenWrtにはfailsafe modeがありますが、LN6001-JPでの通常復旧では、まずLuCI、SSH、設定ファイル復元、LuCIバックアップ復元、ソフトリセット、ハードリセットの順で考えるのがおすすめです。

failsafeはOpenWrtに慣れた人向けの手段として扱います。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ルーターを初期化する方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7031/
- Linksys MBE70 reset: https://support.linksys.com/kb/article/220-en/?section_id=175
- Linksys MBE70 WRT Pro 7 FAQs: https://support.linksys.com/kb/article/217-en/?section_id=175
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へログインする方法: https://support.linksys.com/kb/article/7038/
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041/
- OpenWrt Wiki - Failsafe mode, factory reset, and recovery mode: https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset
- OpenWrt Wiki - Backup and restore: https://openwrt.org/docs/guide-user/troubleshooting/backup_restore

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
