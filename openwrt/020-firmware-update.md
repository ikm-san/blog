<!-- mirror-source: articles/020-firmware-update.md -->

# 更新ボタンを押す前が本番｜LN6001-JPのファームウェア更新で慌てないための準備と確認【OpenWrt集中連載020】

ファームウェア更新って、ボタン自体はわりと軽いです。

ファイルを選ぶ。  
アップロードする。  
Flash imageを押す。  
再起動を待つ。

作業だけ見ると、そんなに難しくありません。

でも、OpenWrtベースのルーターでは、更新ボタンを押す前のほうが大事です。

なぜなら、LN6001-JPにはいろいろ積み上げられるからです。

Wi-Fi名。  
ゲストWi-Fi。  
IPoE。  
VLAN。  
VPN。  
Tailscale。  
Adblock。  
DHCP予約。  
firewall zone。  
追加モジュール。  
自分で入れたパッケージ。

使い込むほど、ルーターは「ただの箱」ではなく、自分のネットワーク設定が詰まった運用環境になっていきます。

なので、ファームウェア更新で本当に怖いのは、更新そのものではありません。

```txt
更新後に何が戻っていて、
何が戻っていないのか分からないこと
```

です。

Wi-Fiは見える？  
WANは取れてる？  
IPv6は生きてる？  
IPoEモジュールは残ってる？  
VPN Assistantは残ってる？  
Adblockは動いてる？  
NASのIPは変わってない？  
ゲストWi-FiからLANへ入れない状態は維持できてる？

ここが確認できれば、かなり落ち着いて対応できます。

ファームウェア更新は、怖がりすぎなくて大丈夫です。

でも、丸腰で行くものでもありません。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）でファームウェア更新をする前に、何をバックアップし、何をメモし、更新後にどこから確認するかをまとめます。

一言でいうと、

```txt
更新ボタンを押す前に、帰り道を作っておこう
```

という記事です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、ファームウェア更新を怖がることではありません。

まずは、ここまでできればOKです。

- 更新前にLuCIバックアップをPCへ保存できる
- 追加パッケージ一覧を控えられる
- 現在のWAN / WAN6 / Wi-Fi / DHCP / firewall状態を残せる
- 公式ファームウェアを使う理由が分かる
- 有線LANで更新する理由が分かる
- 更新後に何から確認すればよいか分かる
- Adblock、VPN Assistant、オートIPoEが消えた時の考え方が分かる
- 問題が出た時に、いきなりリセットせず切り分けられる

ファームウェア更新は、イベントではなく運用です。

毎回、同じ手順で準備して、同じ順番で確認する。

これだけで、かなり安心して進められます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-01.png)

## 先にざっくり結論

LN6001-JPのファームウェア更新では、まずこの5つを守ればかなり安全です。

1. **更新前にLuCIバックアップをPCへ保存する**
2. **追加パッケージ一覧を残す**
3. **有線LAN接続のPCから作業する**
4. **公式ファームウェアだけ使う**
5. **更新後はWAN / WAN6 → DNS → Wi-Fi → DHCP → 追加機能の順で確認する**

特に大事なのは、設定バックアップと追加パッケージ一覧を分けて考えることです。

LuCIのバックアップは、主に `/etc/config/` などの設定を戻すためのものです。

一方で、`opkg` で追加したパッケージや、あとから入れたモジュールは、ファームウェア更新後に入れ直しが必要になることがあります。

つまり、更新前に残すべきものは2種類あります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/table-01.png)

更新後の最初の確認は、次の順番がおすすめです。

```txt
LuCIへ入れる
  ↓
ファームウェアバージョンを見る
  ↓
WAN / WAN6を見る
  ↓
DNSを見る
  ↓
Wi-Fiを見る
  ↓
DHCP予約を見る
  ↓
ゲストWi-Fi / VLAN / firewallを見る
  ↓
Adblock / IPoE / VPNなど追加機能を見る
```

いきなり細かいパッケージの差分を見るより、まず「普通に使える状態へ戻っているか」を見ます。

## こういう人向けです

この記事は、次のような人向けです。

- LN6001-JPを長く使う前提で、安全に更新したい
- 更新後にWi-Fi、IPoE、VPN、Adblockが壊れないか不安
- 設定バックアップとパッケージ一覧の違いを知りたい
- オートIPoEやVPN Assistantを入れている
- ゲストWi-Fi、VLAN、DHCP予約、firewall zoneを設定済み
- 小さなオフィスや店舗で、営業時間中の更新事故を避けたい
- 更新後に何を確認すればよいかチェックリスト化したい
- 何か起きた時に、いきなりリセットせず戻せる状態を作りたい

逆に、まだ買ったばかりでほぼ初期設定のままなら、ここまで細かくなくても大丈夫です。

ただし、それでもLuCIバックアップだけは取っておくのがおすすめです。

バックアップがあると、人は急に落ち着けます。

OpenWrt系ルーターでは、本当にそうです。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-02.png)

## 最初に言葉だけそろえる

ファームウェア更新で出てくる言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/table-02.png)

最初は、次だけ覚えれば十分です。

```txt
LuCIバックアップ = 設定を戻すため
opkg一覧 = 追加機能を戻すため
sysupgrade = OpenWrt系の更新の仕組み
```

## 更新は「押す前」と「押した後」が大事

ファームウェア更新は、次の3段階で考えると分かりやすいです。

```txt
更新前:
  戻せる状態を作る

更新中:
  触らず待つ

更新後:
  普段使う機能が戻っているか確認する
```

失敗しやすいのは、更新中よりも前後です。

## 更新前にやりがちな失敗

- バックアップを取っていない
- 追加パッケージ一覧を残していない
- Wi-Fi経由だけで更新している
- 公式ファームウェアか確認していない
- 店舗やオフィスの営業時間中に更新している
- VPNでしか現地へ入れないのに、VPNが切れた時の手段がない
- 更新後に何を確認するか決めていない

## 更新後にやりがちな失敗

- WANが戻る前にAdblockやVPNを疑う
- IPv6は生きているのにIPv4 over IPv6側を見ていない
- 追加パッケージが消えたのに設定だけ見ている
- DNSが原因なのにWi-Fi設定を触る
- ゲストWi-Fiの分離確認を忘れる
- Tailscaleのサブネットルート承認を見ていない
- とりあえず再更新や初期化をしてしまう

更新は、ボタンを押す作業より、前後の確認が本体です。

## 更新前に決めること

ファームウェア更新前に、まず作業条件を決めます。

## 作業時間を決める

更新は、できれば次の時間に行います。

- 家族やスタッフが使っていない時間
- 店舗なら営業時間外
- 小さなオフィスなら会議や決済がない時間
- VPNで入れなくなっても現地対応できる時間
- ルーターの電源を安定して確保できる時間
- もし戻し作業が必要になっても焦らない時間

更新中は、インターネット接続やWi-Fiが一時的に止まります。

家庭なら、家族の動画視聴やオンライン会議の時間を避けます。

店舗なら、POS、予約端末、決済端末、監視カメラ、ゲストWi-Fiへの影響を考えます。

小さなオフィスなら、クラウド業務やVPN利用の少ない時間にします。

ファームウェア更新は、深夜テンションでやるより、戻せる時間にやるほうが安全です。

## 有線LANで作業する

更新作業は、有線LAN接続のPCから行うのがおすすめです。

Wi-Fiでも操作できることはあります。

でも、更新中にWi-Fiが再起動したり、SSID設定が一時的に切れたりすると、画面の状態を追いにくくなります。

理想はこの形です。

```txt
PC
  ↓ 有線LAN
LN6001-JP
  ↓ WAN
ONU / HGW / 回線機器
```

特に次のような環境では、有線LANがかなり大事です。

- Wi-Fi名を変更している
- ゲストWi-Fiを作っている
- VLANを設定している
- firewall zoneを細かく分けている
- AP BridgeやWDSを使っている
- VPNでしか外から入れない
- 店舗や小さなオフィスで運用している

Wi-Fiが切れても、有線LANでLuCIへ戻れる。

この安心感はかなり大きいです。

## 電源を安定させる

更新中に電源を切るのは避けます。

作業前に、次を確認します。

- 電源アダプターがしっかり挿さっている
- 電源タップが不安定でない
- 更新中に誰かが電源を抜かない場所に置いている
- 店舗やオフィスでは清掃や移動で触られない
- ONU / HGWも安定して電源が入っている
- 可能ならUPS配下に置いている

更新中にルーターだけでなく、ONUやHGWの電源が落ちても切り分けがややこしくなります。

ルーター、ONU、HGWはセットで安定させます。

## 現在の状態をメモする

更新前に、今の状態を軽くメモします。

完璧な台帳でなくて大丈夫です。

まずはこのくらいです。

```txt
現在のファームウェア:
  1.2.0.15

LAN IP:
  192.168.1.1

回線:
  HGW配下 / ONU直下 / PPPoE / オートIPoE

追加モジュール:
  オートIPoE
  VPN Assistant
  Adblock

主要ネットワーク:
  lan: 192.168.1.0/24
  guest: 192.168.2.0/24
  device: 192.168.3.0/24

重要機器:
  NAS: 192.168.1.10
  プリンター: 192.168.1.20
  録画機: 192.168.3.20

VPN:
  Tailscale / WireGuard / 未使用

バックアップ:
  backup-LN6001-before-firmware-update-20260621.tar.gz
```

更新後に困るのは、

```txt
前はどうなってたっけ？
```

です。

このメモがあるだけで、復旧がかなり楽になります。

## ファームウェアバージョンを確認する

更新前に、現在のファームウェアバージョンを確認します。

## LuCIで確認する

1. ブラウザでLuCIへログインする
2. **Status** → **Overview** を開く
3. Firmware Versionを確認する

## CLIで確認する

```sh
echo "### system board"
ubus call system board

echo "### uptime"
uptime

echo "### storage"
df -h

echo "### base packages"
opkg list-installed | grep -E 'base-files|kernel|busybox|dnsmasq|firewall|dropbear|uhttpd'
```

`ubus call system board` は、機種情報やファームウェア情報を見る入口として便利です。

更新前後でこの出力を保存しておくと、比較しやすくなります。

## LuCIバックアップを取る

まず、LuCIで設定バックアップを取ります。

Linksys公式手順でも、設定バックアップは **System → Backup / Flash Firmware** から行う流れです。

1. **System** → **Backup / Flash Firmware** を開く
2. **Backup** セクションの **Generate archive** をクリックする
3. `.tar.gz` ファイルをダウンロードする
4. ファイル名に日付と状態を入れて保存する

例:

```txt
backup-LN6001-before-firmware-update-20260621.tar.gz
```

このバックアップは、必ずPCへ保存します。

ルーター内部に置くだけでは足りません。

ファームウェア更新や初期化で、ルーター内部のファイルが消えることがあります。

## LuCIバックアップに過信しすぎない

LuCIバックアップはとても大事です。

ただし、万能ではありません。

主に戻しやすいのは設定ファイルです。

一方で、次のようなものは別管理が必要になることがあります。

- `opkg` で追加したパッケージ本体
- VPN Assistantなどの追加モジュール本体
- オートIPoEモジュール本体
- Adblock関連パッケージ
- banIPなど追加パッケージ
- `/root` 配下に置いた自作スクリプト
- 手動で置いた証明書や鍵
- Tailscale再認証が必要な状態
- 外部管理コンソール側の承認状態

なので、LuCIバックアップに加えて、CLIで状態スナップショットを残します。

## CLIで状態スナップショットを取る

LuCIバックアップに加えて、現在の設定と状態をテキストで残します。

SSHでLN6001-JPへ入って実行します。

```sh
BACKUP_DIR="/root/firmware-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

echo "### create sysupgrade config backup"
sysupgrade -b "$BACKUP_DIR/config-backup.tar.gz"

echo "### copy UCI config files"
for cfg in network wireless firewall dhcp system dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

echo "### package list"
opkg list-installed > "$BACKUP_DIR/packages-before-update.txt"

echo "### system and network snapshots"
ubus call system board > "$BACKUP_DIR/system-board-before.json"
uptime > "$BACKUP_DIR/uptime-before.txt"
df -h > "$BACKUP_DIR/df-h-before.txt"
free > "$BACKUP_DIR/free-before.txt"

ifstatus wan > "$BACKUP_DIR/ifstatus-wan-before.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6-before.json" 2>/dev/null || true

ip addr show > "$BACKUP_DIR/ip-addr-before.txt"
ip route show > "$BACKUP_DIR/route-v4-before.txt"
ip -6 route show > "$BACKUP_DIR/route-v6-before.txt"

wifi status > "$BACKUP_DIR/wifi-status-before.json" 2>/dev/null || true
iw dev > "$BACKUP_DIR/iw-dev-before.txt" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases-before.txt"
logread | tail -n 300 > "$BACKUP_DIR/logread-before.txt"

echo "### backup directory"
ls -l "$BACKUP_DIR"
```

これは復元用バックアップというより、更新前の状態メモです。

あとから、

```txt
更新前はWAN6どう見えてた？
ゲストWi-Fiはどのnetworkに紐づいてた？
Adblock入ってた？
Tailscaleあった？
```

を確認するために使います。

## スナップショットをPCへ保存する

ルーター内部に置くだけでは不十分です。

PCへコピーします。

PC側のターミナルから実行します。

```sh
scp -r root@192.168.1.1:/root/firmware-before-* ~/Desktop/
```

LAN IPを変えている場合は、IPを置き換えます。

```sh
scp -r root@192.168.10.1:/root/firmware-before-* ~/Desktop/
```

Windows PowerShellなら、保存先をWindowsのパスにします。

```powershell
scp -r root@192.168.1.1:/root/firmware-before-* "$env:USERPROFILE\Desktop\"
```

PC側に保存できたら、フォルダの中身を確認します。

バックアップは「取ったつもり」ではなく、「PC側にある」ところまで確認します。

## 追加パッケージ一覧を残す理由

OpenWrt系では、ファームウェア更新後に追加パッケージが残らないことがあります。

ここ、かなり大事です。

たとえば、次のようなものです。

- Adblock
- banIP
- Tailscale
- WireGuard関連
- VPN Assistant
- オートIPoE
- 追加LuCIアプリ
- 独自スクリプト
- 追加した診断ツール

設定ファイルが残っていても、パッケージ本体が消えていると機能しません。

たとえば、Adblockの設定が残っていても、Adblockパッケージがなければ動きません。

VPNの設定が残っていても、関連パッケージがなければ起動しません。

なので、更新前に一覧を保存します。

```sh
opkg list-installed > /root/packages-before-update.txt
```

更新後に比較できます。

```sh
opkg list-installed > /tmp/packages-after-update.txt
diff /root/packages-before-update.txt /tmp/packages-after-update.txt
```

ただし、更新後に古いパッケージを何でも全部入れ直すのは避けます。

ファームウェアが変わると、対応するパッケージも変わることがあります。

まず `opkg update` を行い、必要なものから順番に入れ直します。

```sh
opkg update
```

OpenWrt系では、ファームウェア更新後に雑に `opkg upgrade` をまとめて実行するより、必要なパッケージだけ追加するほうが安全です。

## 公式ファームウェアを入手する

ファームウェアは、Linksys公式サポートから取得します。

1. Linksys公式サポートのVelop WRT Pro 7 / LN6001-JPページを開く
2. ファームウェアダウンロードリンクを確認する
3. 対象モデルがLN6001-JPであることを確認する
4. ファームウェアファイルをダウンロードする
5. 可能ならリリースノートを確認する
6. ファイル名、バージョン、ダウンロード日を控える

機種違いのファームウェア、非公式サイトのファイル、出所が分からないファイルは使いません。

ここでのミスはかなり痛いです。

「なんとなく似た名前」ではなく、対象モデルを確認します。

## ダウンロード後に控えること

```txt
ファームウェアファイル:
  firmware-linksys-ln6001-xxxx.img

入手元:
  Linksys公式サポート

ダウンロード日:
  2026-06-21

更新前バージョン:
  1.2.0.15

更新後予定バージョン:
  x.x.x.x

リリースノート:
  確認済み / 未確認

メモ:
  更新目的: セキュリティ / 不具合修正 / 機能改善 / 検証
```

PC側でハッシュを控える場合です。

```sh
sha256sum firmware-linksys-ln6001-xxxx.img
```

macOSでは次も使えます。

```sh
shasum -a 256 firmware-linksys-ln6001-xxxx.img
```

公式ページにハッシュ値がある場合は照合します。

掲載がない場合でも、自分の作業メモとして残しておくと、あとから同じファイルか確認しやすくなります。

## 更新前チェックリスト

更新前に、次を確認します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/table-03.png)

この表を埋めてから更新に進むだけで、かなり安心です。

OpenWrt系ルーターでは、勢いより準備です。

## LuCIでファームウェアを更新する

基本は、Linksys公式手順に沿ってLuCIから更新するのがおすすめです。

1. **System** → **Backup / Flash Firmware** を開く
2. **Flash image** をクリックする
3. **Browse...** でダウンロードしたファームウェアファイルを選択する
4. **Upload** をクリックする
5. 表示内容を確認する
6. 更新を開始する
7. 更新が完了し、ルーターが再起動するまで待つ
8. LEDが更新前の正常状態に戻るまで待つ
9. 必要に応じてSSIDへ再接続する
10. LuCIへ再ログインする
11. **Status** → **Overview** でFirmware Versionを確認する

更新中は、電源を切らないでください。

ブラウザを閉じても更新自体は進むことがありますが、状態を追いにくくなります。

更新中は他の作業をせず、完了を待ちます。

## Keep settingsが表示された場合

LuCIの画面やファームウェアによって、設定を保持するオプションが表示されることがあります。

通常運用では、設定保持で進めることが多いです。

ただし、次のような場合は、クリーン更新も検討します。

- 以前から設定が壊れている
- firewallやnetworkを整理し直したい
- 古い追加パッケージ設定を持ち越したくない
- 更新後に原因不明の不具合が続いている
- Linksys公式サポートからクリーン更新を案内された
- 構成を大きく作り直したい

クリーン更新をする場合は、バックアップを丸ごと戻す前に注意します。

古い設定に問題があった場合、バックアップをそのまま戻すと問題も戻ります。

必要な設定だけ戻すほうが安全な場合があります。

## CLI更新について

基本はLuCI更新をおすすめします。

CLI更新は、OpenWrtやsysupgradeに慣れている人向けです。

CLIで更新する場合は、次を守ります。

- 公式ファームウェアを使う
- 対象モデルを確認する
- 更新前バックアップをPCへ保存する
- ファームウェアを `/tmp` に置く
- SSH切断や再起動を前提にする
- 更新後にLuCIへ戻れることを確認する

PCからルーターへファームウェアを転送します。

```sh
scp firmware-linksys-ln6001-xxxx.img root@192.168.1.1:/tmp/
```

ルーター側で確認します。

```sh
ls -lh /tmp/firmware-linksys-ln6001-xxxx.img
sha256sum /tmp/firmware-linksys-ln6001-xxxx.img
```

ここから先の書き込みは、公式LuCI更新またはLinksys公式手順を優先します。

```sh
ubus call system board
ls -lh /tmp/firmware-linksys-ln6001-xxxx.img
sha256sum /tmp/firmware-linksys-ln6001-xxxx.img
```

CLIで直接書き込むコマンドは、LN6001-JP向けの公式ファームウェア形式と手順が確認できている時だけにします。

ファイル形式を間違えると危険なので、通常運用ではLuCIから更新するほうが安全です。

CLI更新は、ファイル形式や機種対応を間違えると危険です。

迷う場合はLuCI更新に戻るのがおすすめです。

## 更新中にやらないこと

更新中は、次を避けます。

- 電源を抜く
- WANケーブルやLANケーブルを抜く
- リセットボタンを押す
- 電源スイッチを切る
- ブラウザで何度も更新ボタンを押す
- 別PCから同時にLuCIへアクセスして操作する
- 不安になって短時間で初期化する
- スマートフォンのWi-Fi接続だけで状態を判断する

更新中は数分かかることがあります。

LEDやLuCIの表示が変わっても、すぐ失敗と判断せず、しばらく待ちます。

ここで焦ってリセットボタンを押したくなります。

分かります。

でも、まだ早いです。

まず待ちます。

## 更新後に最初に確認すること

再起動後は、いきなり細かい設定を触りません。

まず基本を確認します。

## 1. LuCIへ戻れるか

初期LAN IPのままなら、次で開きます。

```txt
https://192.168.1.1
```

LAN IPを変更している場合は、そのアドレスで開きます。

例:

```txt
https://192.168.10.1
```

LuCIへ入れることが、最初の重要な確認です。

## 2. ファームウェアバージョン

LuCIの **Status** → **Overview** でFirmware Versionを確認します。

CLIでは次です。

```sh
ubus call system board
```

更新後バージョンが想定どおりか確認します。

## 3. WAN / WAN6

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show
```

IPoE環境では、WANとWAN6を分けて見ます。

IPv6は戻っているけれど、IPv4 over IPv6側が戻っていないことがあります。

ここで「インターネットが全部だめ」とまとめず、

```txt
IPv4
IPv6
DNS
```

を分けて見ます。

## 4. DNS

```sh
echo "### IPv4 reachability"
ping -c 4 8.8.8.8

echo "### IPv6 reachability"
ping6 -c 4 2001:4860:4860::8888

echo "### DNS"
nslookup www.google.co.jp
```

`ping 8.8.8.8` は通るのに `nslookup` が失敗する場合はDNSを見ます。

Adblock、Family DNS、DoH対策、Tailscale DNS、WireGuard DNSを使っている場合は、更新後にDNSまわりを確認します。

## 5. Wi-Fi

LuCIの **Network** → **Wireless** を開き、SSIDが戻っているか確認します。

CLIでは次です。

```sh
uci show wireless | grep -E 'ssid|encryption|network|disabled'
wifi status
iw dev
```

スマートフォンやPCから、普段使うSSIDへ再接続します。

確認する順番は、使っているものからでOKです。

- メインSSID
- 2.4GHz IoT用SSID
- 5GHz Staff / Main
- 6GHz用SSID
- ゲストWi-Fi
- Kids
- Device
- MLO

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

## 6. DHCPと固定IP

```sh
echo "### static leases"
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='

echo "### active leases"
cat /tmp/dhcp.leases
```

NAS、プリンター、監視カメラ、録画機、ミニPCなどのIPアドレスが想定どおりか確認します。

DHCP予約が戻っていないと、NASやプリンターのIPが変わることがあります。

「NASが消えた」と見えて、実はIPが変わっただけ、ということがあります。

## 更新後チェックリスト

自分の環境で使っているものだけ確認すればOKです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/table-04.png)

全部を完璧に見る必要はありません。

ただ、普段使っている機能は必ず見ます。

## 追加パッケージを確認する

更新後、追加パッケージが残っているか確認します。

```sh
echo "### installed packages after update"
opkg list-installed > /tmp/packages-after-update.txt

echo "### compare package list"
diff /root/firmware-before-YYYYMMDD-HHMM/packages-before-update.txt /tmp/packages-after-update.txt 2>/dev/null || true
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えます。

ざっくり確認するなら次です。

```sh
opkg list-installed | grep -Ei 'adblock|banip|tailscale|wireguard|vpn|ipoe|luci-app'
```

追加パッケージが消えている場合は、更新後のファームウェアで `opkg update` を実行し、その環境に合ったパッケージを入れ直します。

```sh
opkg update
```

必要なものから順番に戻します。

一気に全部を戻さないほうが、トラブル時に切り分けやすいです。

## オートIPoEを確認する

NTTフレッツ系のONU直下構成でオートIPoEを使っている場合、更新後にモジュールが残っているか確認します。

```sh
opkg list-installed | grep -Ei 'ipoe|internet|assistant|velop' || true
ifstatus wan
ifstatus wan6
logread | grep -Ei 'ipoe|map|mape|dslite|ipip|wan|wan6' | tail -n 100
```

LuCIでは、**Services** → **Internet Assistant** が表示されるか確認します。

表示されない場合は、次を試します。

- ブラウザをリロードする
- LuCIからログアウトして再ログインする
- ルーターを再起動する
- モジュールが消えていないか `opkg list-installed` で確認する
- 必要ならLinksys公式サポートから再導入する

HGW配下でDHCP自動接続している場合は、まずWANがDHCPでIPを取れているか確認します。

## VPN Assistantを確認する

WireGuardやTailscaleを使っている場合は、VPN Assistantも確認します。

```sh
opkg list-installed | grep -Ei 'vpn|tailscale|wireguard' || true
ls /etc/init.d | grep -Ei 'tailscale|wireguard|vpn' || true
logread | grep -Ei 'vpn|tailscale|wireguard' | tail -n 100
```

Tailscaleを使っている場合です。

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route' || true
```

WireGuardを使っている場合です。

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true
```

更新後にVPNが動かない場合、次を確認します。

- パッケージが残っているか
- サービスが起動しているか
- firewallルールが残っているか
- Tailscaleの再認証が必要か
- サブネットルートが承認済みか
- WireGuardのPeerが残っているか
- VPN Assistantの画面が残っているか

Tailscaleは、ルーター側だけでなく、Tailscale管理コンソール側も確認します。

## Adblockを確認する

Adblockを使っている場合は、更新後に状態を確認します。

```sh
/etc/init.d/adblock status 2>/dev/null || true
opkg list-installed | grep -Ei 'adblock|luci-app-adblock' || true
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100
```

LuCIでは、**Services** → **Adblock** が表示されるか確認します。

表示されない場合は、パッケージが消えている可能性があります。

必要なら再インストールします。

```sh
opkg update
opkg install adblock luci-app-adblock luci-i18n-adblock-ja
```

日本語パッケージが見つからない場合は、`adblock` と `luci-app-adblock` だけでも構いません。

更新後にDNSが不安定な場合は、Adblockを一時停止して切り分けます。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

確認後、必要なら戻します。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

## firewall / VLAN / ゲストWi-Fiを確認する

ゲストWi-Fi、VLAN、Deviceネットワーク、Kidsネットワークを使っている場合は、更新後に対象SSIDとfirewall zoneを確認します。

```sh
echo "### network"
uci show network | grep -E 'interface|device|ipaddr|proto|guest|device|kids|iot'

echo "### wireless mapping"
uci show wireless | grep -E 'ssid|network|guest|device|kids|iot'

echo "### firewall"
uci show firewall | grep -E 'zone|forwarding|guest|device|kids|iot|Allow-'

echo "### dhcp"
uci show dhcp | grep -E 'guest|device|kids|iot|start|limit|dhcp_option'
```

実端末でも確認します。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/table-05.png)

更新後は、「届くべきところに届く」だけでなく、「届かないべきところに届かない」ことも確認します。

特にゲストWi-Fiは、SSIDが見えるだけでは不十分です。

ゲスト端末からLANへ届かないことまで確認します。

## 更新後の状態スナップショット

更新後にも、状態を保存しておくと後から比較できます。

```sh
AFTER_DIR="/root/firmware-after-$(date +%Y%m%d-%H%M)"
mkdir -p "$AFTER_DIR"

ubus call system board > "$AFTER_DIR/system-board-after.json"
uptime > "$AFTER_DIR/uptime-after.txt"
df -h > "$AFTER_DIR/df-h-after.txt"
free > "$AFTER_DIR/free-after.txt"

ifstatus wan > "$AFTER_DIR/ifstatus-wan-after.json" 2>/dev/null || true
ifstatus wan6 > "$AFTER_DIR/ifstatus-wan6-after.json" 2>/dev/null || true

ip addr show > "$AFTER_DIR/ip-addr-after.txt"
ip route show > "$AFTER_DIR/route-v4-after.txt"
ip -6 route show > "$AFTER_DIR/route-v6-after.txt"

wifi status > "$AFTER_DIR/wifi-status-after.json" 2>/dev/null || true
iw dev > "$AFTER_DIR/iw-dev-after.txt" 2>/dev/null || true
cat /tmp/dhcp.leases > "$AFTER_DIR/dhcp-leases-after.txt"
opkg list-installed > "$AFTER_DIR/packages-after-update.txt"
logread | tail -n 300 > "$AFTER_DIR/logread-after.txt"

ls -l "$AFTER_DIR"
```

PCへコピーします。

```sh
scp -r root@192.168.1.1:/root/firmware-after-* ~/Desktop/
```

更新前後のスナップショットを残しておくと、「更新で何が変わったか」を後から確認しやすくなります。

## うまくいかない時の切り分け

## LuCIへ入れない

まず、これを確認します。

- PCがLN6001-JPのLAN側へつながっているか
- LAN IPを変更していた場合、新しいIPでアクセスしているか
- Wi-Fiではなく有線LANで試したか
- ブラウザの証明書警告で止まっていないか
- PCが正しいIPアドレスを取得しているか
- ゲストWi-Fi側から管理画面へ入ろうとしていないか

Windowsなら:

```powershell
ipconfig
```

macOS / Linuxなら:

```sh
ip addr show
ip route show default
```

LN6001-JPのLAN IPを `192.168.10.1` に変えていた場合は、次で開きます。

```txt
https://192.168.10.1
```

「LuCIへ入れない」のではなく、「管理画面の住所を忘れているだけ」のこともあります。

## インターネットへ出られない

まずWAN/WAN6を見ます。

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
logread | grep -Ei 'wan|wan6|dhcp|ppp|pppoe|ipoe|map|dslite|ipip|netifd|odhcp6c' | tail -n 120
```

確認ポイントです。

- WANがupか
- WAN6がupか
- default routeがあるか
- IPoEモジュールが残っているか
- PPPoEのID / パスワードが残っているか
- HGW配下でLAN IP重複が起きていないか
- DNSだけおかしいのか、WAN自体が落ちているのか

IPoE環境では、IPv6は生きているがIPv4 over IPv6側だけ戻っていないことがあります。

ここでWi-Fi設定を触らないでください。

まずWAN/WAN6です。

## Wi-Fiが見えない・つながらない

```sh
uci show wireless | grep -E 'ssid|encryption|network|disabled'
wifi status
iw dev
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

確認ポイントです。

- SSIDが有効か
- 暗号化方式が戻っているか
- 2.4GHz / 5GHz / 6GHzが有効か
- ゲストWi-FiやIoT用SSIDが戻っているか
- MLO設定を使っている場合、条件が戻っているか
- Wi-Fiパスワードが変わっていないか
- 端末側が古い接続情報を持っていないか

端末側でWi-Fi設定を削除して、再接続すると直ることもあります。

## 追加パッケージが消えた

更新後に、Adblock、VPN Assistant、オートIPoEなどが消えている場合があります。

まず一覧を比較します。

```sh
opkg list-installed > /tmp/packages-after-update.txt
diff /root/firmware-before-YYYYMMDD-HHMM/packages-before-update.txt /tmp/packages-after-update.txt 2>/dev/null || true
```

必要なものだけ再導入します。

```sh
opkg update
```

その後、必要なパッケージや公式モジュールを入れます。

オートIPoEやVPN Assistantは、Linksys公式サポートで案内されているモジュールを使います。

一般的なOpenWrt記事のコマンドをそのまま流用する前に、LN6001-JP向け手順を確認してください。

## DNSだけおかしい

```sh
ping -c 4 8.8.8.8
ping -c 4 google.co.jp
nslookup www.google.co.jp
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf
logread | grep -i dnsmasq | tail -n 100
```

`8.8.8.8` へは届くが `google.co.jp` が引けない場合はDNSを見ます。

Adblock、Family DNS、DoH対策、Tailscale DNS、WireGuard DNSなどが絡んでいる場合は、更新後に設定が変わっていないか確認します。

DNSが怪しい時は、Adblockを一時停止して切り分けることもあります。

## VPNだけ動かない

Tailscaleの場合です。

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

WireGuardの場合です。

```sh
wg show 2>/dev/null || true
uci show network | grep -Ei 'wireguard|wg|vpn'
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn'
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

VPN Assistantが消えている場合は、公式手順に沿って再導入します。

Tailscaleは再認証が必要になる場合があります。

サブネットルートやExit Nodeを使っていた場合は、Tailscale管理コンソール側の承認状態も確認します。

## バックアップから復元する

設定が大きく崩れた場合は、バックアップから復元します。

## LuCIで復元する

1. **System** → **Backup / Flash Firmware** を開く
2. **Restore backup** または **Upload archive** を選ぶ
3. 更新前に保存した `.tar.gz` を選択する
4. アップロードして復元する
5. 再起動を待つ
6. LuCIへ再ログインする
7. WAN/WAN6、Wi-Fi、DHCP、firewallを確認する

## CLIで復元する

PCからバックアップを転送します。

```sh
scp backup-LN6001-before-firmware-update-20260621.tar.gz root@192.168.1.1:/tmp/
```

ルーター側で復元します。

```sh
sysupgrade -r /tmp/backup-LN6001-before-firmware-update-20260621.tar.gz
reboot
```

復元後、再起動を待ちます。

ただし、バックアップ復元は万能ではありません。

ファームウェアのバージョン差やパッケージ差によって、古い設定を戻すことで問題が戻る場合もあります。

原因不明の不具合が続く場合は、クリーン状態から必要な設定だけ戻すほうがよいこともあります。

## クリーン更新を検討する場合

通常は設定保持で更新します。

ただし、次のような場合はクリーン更新も選択肢です。

- 以前から設定が複雑になりすぎている
- firewallやnetwork設定を整理し直したい
- 追加パッケージの設定が壊れている
- 更新後にバックアップ復元しても不具合が続く
- サポートから初期化を案内された
- 新しい設計に作り直したい

クリーン更新を行う場合は、事前に次を用意します。

- LuCIバックアップ
- 主要設定のテキスト控え
- SSID / Wi-Fiパスワード
- 管理者パスワード
- LAN IP
- IPoE / PPPoE設定
- DHCP予約表
- firewall zone設計
- VPN設定メモ
- Adblock / Family DNS設定メモ
- 追加パッケージ一覧

クリーン更新後は、いきなり全部を戻すのではなく、次の順番がおすすめです。

```txt
LAN IPと管理者パスワード
  ↓
WAN / WAN6 / IPoE
  ↓
メインWi-Fi
  ↓
DHCP予約
  ↓
ゲストWi-Fi / VLAN / firewall
  ↓
Adblock
  ↓
VPN
  ↓
その他の追加機能
```

一気に戻すと、何が原因で壊れたのか分かりにくくなります。

小さく戻して確認します。

## 更新運用メモのテンプレート

更新作業の記録を残しておくと、あとから助かります。

```txt
更新日:
  2026-06-21

製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

更新前バージョン:
  1.2.0.15

更新後バージョン:
  x.x.x.x

ファームウェアファイル:
  firmware-linksys-ln6001-xxxx.img

入手元:
  Linksys公式サポート

作業方法:
  LuCI / CLI

作業端末:
  有線LAN接続PC

バックアップ:
  backup-LN6001-before-firmware-update-20260621.tar.gz
  firmware-before-20260621-1430/

追加モジュール:
  オートIPoE: あり / なし
  VPN Assistant: あり / なし
  Adblock: あり / なし

更新後確認:
  LuCI: OK / NG
  WAN: OK / NG
  WAN6: OK / NG
  DNS: OK / NG
  Wi-Fi: OK / NG
  DHCP予約: OK / NG
  VPN: OK / NG
  Adblock: OK / NG

問題:
  なし / あり

対応:
  例: VPN Assistantを再導入
```

ファームウェア更新は、毎回同じ手順でできるようにしておくと安全です。

## よくある失敗

### バックアップをルーター内部にしか置いていない

更新やリセットで消える可能性があります。

必ずPC側へ保存します。

```sh
scp -r root@192.168.1.1:/root/firmware-before-* ~/Desktop/
```

### 追加パッケージ一覧を残していない

設定バックアップだけでは、追加パッケージを復元しきれない場合があります。

更新前に必ず保存します。

```sh
opkg list-installed > /root/packages-before-update.txt
```

### Wi-Fi経由で更新して途中で不安になる

Wi-Fiが一時的に切れると状態を追いにくくなります。

更新作業は有線LANをおすすめします。

### 更新直後にすぐリセットしてしまう

更新後は再起動やLED復帰に時間がかかることがあります。

すぐ失敗と判断せず、数分待ちます。

長時間戻らない場合は、現在のLED状態、PCのIPアドレス、LuCIアクセス可否、有線接続可否を確認します。

### 古い設定を丸ごと戻して不具合も戻る

バックアップ復元は便利ですが、問題の原因になっていた設定も戻ります。

更新後の不具合が設定由来に見える場合は、必要な設定だけ手で戻すことも検討します。

### 機種違いのファームウェアを使う

危険です。

LN6001-JP向けの公式ファームウェアを使います。

ファイル名、サポートページ、対象モデルを確認してから更新します。

### opkgで全部アップグレードしてしまう

ファームウェア更新後に、なんとなく `opkg upgrade` をまとめて実行するのは避けたほうが安全です。

必要なパッケージを確認し、必要なものだけ入れます。

ルーターはPCではありません。

ストレージもメモリも限られています。

## まとめ

LN6001-JPのファームウェア更新では、更新そのものより、更新前後の運用が大事です。

ポイントは次の通りです。

1. 公式ファームウェアを使う
2. 有線LANで作業する
3. LuCIバックアップをPCへ保存する
4. CLIで状態スナップショットも残す
5. 追加パッケージ一覧を保存する
6. 更新後はLuCI、WAN/WAN6、Wi-Fi、DNS、DHCPから確認する
7. オートIPoE、VPN Assistant、Adblockなど追加機能を確認する
8. 問題が出たらログを見て、必要ならバックアップから戻す
9. パッケージは更新後の環境に合わせて入れ直す
10. 更新履歴をメモしておく

ファームウェア更新は、怖がりすぎる必要はありません。

でも、何も控えずに進めるものでもありません。

```txt
戻せる状態を作ってから更新する
```

これだけで、OpenWrtベースルーターの運用はかなり安全になります。

更新ボタンを押す前が本番です。

そこまで準備できていれば、更新後に何か起きても落ち着いて見られます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/020/diagram-03.png)

## 次に読むなら

ファームウェア更新運用を固めたら、次の記事も合わせて読むと安心です。

- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)

バックアップの戻し方を詳しく見たい人は、設定バックアップと復元へ。

更新後に通信が戻らない時の切り分けを整理したい人は、つながらない時の切り分けへ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

Linksys公式ファームウェア、LuCIの画面名、sysupgradeの挙動、追加パッケージ、オートIPoE、VPN Assistant、Adblockなどの扱いは、ファームウェア更新やモジュール更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートとOpenWrtの最新ドキュメントを確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPのファームウェア更新前に必ずやることは？

LuCIバックアップをPCへ保存することです。

あわせて、`opkg list-installed` で追加パッケージ一覧を保存し、WAN/WAN6、Wi-Fi、DHCP、firewallの状態スナップショットを残しておくと復旧しやすくなります。

### Wi-Fi接続で更新してもいい？

可能な場合もありますが、有線LAN接続のほうがおすすめです。

更新中にWi-Fiが切れると、状態を追いにくくなります。

### 更新後に追加パッケージは残る？

残らない場合があります。

Adblock、VPN Assistant、オートIPoE、Tailscale、WireGuard関連などは、更新後に確認し、必要なら再導入します。

### 設定バックアップだけあれば十分？

設定バックアップは重要ですが、それだけでは不十分な場合があります。

追加パッケージ本体や外部モジュールは別途再導入が必要になることがあります。

設定バックアップとパッケージ一覧の両方を残してください。

### sysupgrade -n は使っていい？

`sysupgrade -n` は設定を保持しない更新です。

初期化に近い状態になるため、通常運用では安易に使わないほうがよいです。

構成を作り直したい場合や、サポートから案内された場合などに検討します。

LN6001-JPでは、まずLuCIの公式更新フローとバックアップ復元を優先します。

### 更新後にIPoEが戻らない時は？

`ifstatus wan`、`ifstatus wan6`、オートIPoEモジュールの有無、Internet Assistantの表示、IPoE関連ログを確認します。

必要に応じて、Linksys公式サポートからオートIPoEモジュールを再導入します。

### 更新後にVPNが戻らない時は？

VPN Assistant、Tailscale、WireGuard関連パッケージが残っているか確認します。

Tailscaleの場合は再認証やサブネットルート承認が必要になることがあります。

WireGuardの場合はPeer、firewall rule、ポート開放、AllowedIPsを確認します。

### 更新に失敗したらどうすればいい？

まず電源をすぐ抜かず、LED状態とLuCIアクセス可否を確認します。

有線LANで接続し、管理画面に入れるか確認します。

入れる場合はバックアップ復元や再設定を行います。

入れない場合は、リセットと復旧の記事を参照し、必要に応じてLinksysサポートへ相談します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ルーターファームウェアを更新する方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7033/
- Linksys OpenWRT firmware update: https://support.linksys.com/kb/article/227-en/?section_id=175
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- OpenWrt Wiki - Backup and restore: https://openwrt.org/docs/guide-user/troubleshooting/backup_restore
- OpenWrt Wiki - Sysupgrade: https://openwrt.org/docs/guide-user/installation/generic.sysupgrade
- OpenWrt Wiki - Upgrading OpenWrt firmware using CLI: https://openwrt.org/docs/guide-user/installation/sysupgrade.cli
- OpenWrt Wiki - Preserving settings during sysupgrade: https://openwrt.org/docs/guide-quick-start/admingui_sysupgrade_keepsettings

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
