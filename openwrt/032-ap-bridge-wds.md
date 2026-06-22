<!-- mirror-source: articles/032-ap-bridge-wds.md -->

# Wi-Fiだけ足したいならAP Bridge｜WDSで延長する前に管理IPをメモしよう【OpenWrt集中連載032】

すでに自宅や店舗にルーターがある。

でも、Wi-Fiだけ強くしたい。  
離れた部屋まで電波を伸ばしたい。  
ひかり電話ホームゲートウェイは残したい。  
業務用ルーターもそのまま使いたい。  
LN6001-JPをメインルーターにするほどではない。  
でも、Wi-Fi 7アクセスポイントとして使いたい。

こういう時に出てくるのが、**AP Bridge** と **WDS子機** です。

名前だけ見ると、ちょっとネットワーク玄人っぽいです。

でも、考え方はそこまで難しくありません。

```txt
有線で既存ルーターへ足す
  → AP Bridge

有線を引けない場所へ無線で伸ばす
  → WDS子機
```

このくらいで大丈夫です。

ただし、AP BridgeやWDSは、設定そのものよりも **設定後にLuCIへ戻れるか** が大事です。

ここでよく起きるのが、管理IP迷子です。

```txt
あれ、設定後の管理画面どこ？
192.168.1.1じゃないの？
親ルーターのIP？
APのIP？
子機のIP？
どれ？
```

これ、かなり起きます。

AP BridgeやWDSでは、LN6001-JPの役割が変わります。

ルーターモードの時とは違って、DHCPを親ルーターに任せたり、同じLAN内のアクセスポイントになったり、WDS子機として別の管理IPを持ったりします。

つまり、設定後の入口が変わります。

ここをメモしておかないと、

```txt
設定はたぶん成功している
でも管理画面へ戻れない
```

という、かなりモヤッとした状態になります。

LN6001-JPはOpenWrtベースなので、LuCIやCLIで細かく手動設定できます。

でも、AP BridgeやWDSは項目が多いので、最初はLinksys公式の **Setup Assistant** を使うほうが安全です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）を既存ネットワークに追加する時の考え方、AP BridgeとWDSの違い、Setup Assistantの導入、設定後に管理IPを見失わないための確認手順をまとめます。

一言でいうと、

```txt
Wi-Fiを足す前に、帰り道をメモしよう
```

という記事です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、AP Bridge / WDSを気合いで手動設定することではありません。

まずは、戻れる形で安全に追加します。

この記事で目指すのは、ここです。

- AP BridgeとWDS子機の違いが分かる
- ルーターモード、AP Bridge、WDSをどう選ぶか分かる
- Linksys Intelligent Mesh / EasyMeshとWDSの違いが分かる
- Setup Assistantを導入する流れが分かる
- AP Bridgeで指定するGateway IPとAccess Point IPの意味が分かる
- WDS子機で指定するParent SSID / Parent IP / Child IPの意味が分かる
- 設定後にLuCIへ戻るための管理IPメモを作れる
- うまくいかない時に見るCLIコマンドが分かる
- どうしても戻れない時の復旧方法が分かる

AP Bridge / WDSで一番大事なのは、設定直後です。

```txt
どのIPでLuCIへ戻るのか
```

これをメモしておく。

本当にこれだけで、かなり救われます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-01.png)

## 先にざっくり結論

既存ルーター配下へLN6001-JPを追加するなら、まず3パターンで考えます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-01.png)

判断はシンプルです。

```txt
LN6001-JPをメインルーターにしたい
  → ルーターモード

既存ルーターは残し、有線でLN6001-JPを追加したい
  → AP Bridge

有線が引けず、LN6001-JP同士を無線でつなぎたい
  → WDS子機
```

一番安定しやすいのは、有線接続のAP Bridgeです。

WDSは、有線LANを引けない場所では便利です。

ただし、無線バックホールなので、距離、壁、家具、電子レンジ、5GHz / 6GHzの届き方に影響されます。

また、ここはかなり大事です。

Velop WRT Pro 7は、従来のLinksys Intelligent MeshやEasyMeshへ参加する構成ではありません。

Velop WRT Pro 7同士を無線でつなぐ場合は、OpenWrtのWDS機能を使う構成として考えます。

```txt
Linksysアプリでメッシュ追加
  → この記事の構成ではない

Velop WRT Pro 7同士を無線接続
  → WDS

既存ルーターへ有線AP追加
  → AP Bridge
```

ここを混ぜると、一気にややこしくなります。

## こういう人向けです

この記事は、次のような人向けです。

- 既存ルーターやホームゲートウェイを残したままWi-Fiだけ強化したい
- ひかり電話HGWや業務用ルーター配下へLN6001-JPを追加したい
- 既存ネットワークのIPアドレス体系を変えたくない
- LAN側DHCPは親ルーターに任せたい
- 有線バックホールで安定したアクセスポイントを追加したい
- 有線が引けない部屋へWDS子機としてWi-Fiを伸ばしたい
- Linksysアプリのメッシュではなく、OpenWrtベースの構成として管理したい
- 設定後に管理IPを見失わないようにしたい
- AP Bridge / WDSを手動で組む前に、まず公式Setup Assistantで安全に試したい

逆に、次のような人には少し違うかもしれません。

- Linksysアプリで普通のメッシュノードとして追加したい
- 従来Velop / MXシリーズのIntelligent Meshへ混ぜたい
- EasyMesh互換で自動構成したい
- とにかくアプリだけでノード追加したい
- 有線も無線も全部おまかせで自動最適化してほしい

この記事では、LN6001-JPをOpenWrtベースの機器として、AP BridgeまたはWDSで使う前提で進めます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-02.png)

## 最初に言葉だけそろえる

AP Bridge / WDSまわりの言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-02.png)

最初は、これだけ覚えればOKです。

```txt
Gateway IP = 親ルーターの住所
Access Point IP = AP化したLN6001-JPの住所
Child IP Address = WDS子機LN6001-JPの住所
```

AP Bridge / WDSでは、この“住所”が超重要です。

## AP BridgeとWDSの違い

まず、AP BridgeとWDSを混同しないようにします。

## AP Bridge

AP Bridgeは、有線で既存ルーターへ接続する構成です。

```txt
インターネット
  ↓
既存ルーター / HGW
  ↓ 有線LAN
LN6001-JP
  ↓
Wi-Fi端末
```

この構成では、既存ルーター側がDHCPやルーティングを担当し、LN6001-JPはWi-Fiアクセスポイントとして動きます。

向いているケースです。

- 有線LANを引ける
- 安定性を優先したい
- 既存ルーターを残したい
- Wi-Fiだけ強化したい
- 二重ルーターを避けたい
- 家庭や店舗の既存ネットワークへ自然に追加したい
- ひかり電話HGWや業務用ルーターをそのまま使いたい

まず試すなら、WDSよりAP Bridgeがおすすめです。

有線は地味です。

でも、ネットワークでは地味な有線がだいたい強いです。

## WDS子機

WDS子機は、親機へ無線で接続する構成です。

```txt
インターネット
  ↓
親機LN6001-JP
  ↓ 無線バックホール
子機LN6001-JP
  ↓
子機側Wi-Fi / 有線LAN端末
```

向いているケースです。

- 有線LANを引けない
- 離れた部屋へWi-Fiを伸ばしたい
- Velop WRT Pro 7同士を無線でつなぎたい
- 一時的に無線中継したい
- 配線工事なしでカバー範囲を広げたい
- 子機側のLANポートも使いたい

WDSは便利ですが、親機と子機の電波状態に依存します。

壁、距離、家具、電子レンジ、近隣Wi-Fiの影響を受けます。

安定性を優先するなら、まずAP Bridgeを検討します。

```txt
引けるなら有線
引けないならWDS
```

このくらいの割り切りでOKです。

## Linksys Intelligent MeshやEasyMeshとは違う

ここは重要です。

Velop WRT Pro 7は、従来のVelop / MXシリーズのLinksys Intelligent Meshへ追加する構成ではありません。

また、EasyMesh的なメッシュへ参加する製品として考えるものでもありません。

Velop WRT Pro 7同士を無線でつなぐ場合は、OpenWrtのWDS機能を使う構成として扱います。

つまり、次のように整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-03.png)

「メッシュWi-Fi」という言葉は便利ですが、かなり雑に使われます。

この記事では、アプリのメッシュではなく、AP BridgeとWDSとして扱います。

ここを分けておくと、かなり迷いにくくなります。

## 二重ルーターは必ず悪いのか

既存ルーター配下にLN6001-JPをルーターモードのまま置くと、二重ルーター構成になります。

```txt
既存ルーター
  ↓
LN6001-JP ルーターモード
  ↓
端末
```

二重ルーターは、ポート開放、VPN着信、オンラインゲーム、外部アクセスなどで問題になることがあります。

ただし、必ず悪いわけではありません。

たとえば、プライベート用と仕事用を分けたい、既存ネットワークからあえて隔離したい、という目的では二重ルーターにすることもあります。

ただし、**Wi-Fiだけ強化したい** なら、AP Bridgeのほうが分かりやすいです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-04.png)

二重ルーターを見たら、すぐ悪者にしなくて大丈夫です。

目的次第です。

ただ、目的が「Wi-Fiを足すだけ」ならAP Bridgeがすっきりします。

## 設定前に決めること

AP BridgeやWDSを始める前に、次を決めます。

## 既存ネットワークのIPを確認する

既存ルーターのLAN IPを確認します。

よくある例です。

```txt
192.168.1.1
192.168.10.1
192.168.0.1
```

PCが既存ルーターへ接続している状態で確認します。

Windows:

```powershell
ipconfig
```

macOS / Linux:

```sh
ip route show default
```

Default Gatewayが親ルーターのIPです。

例:

```txt
default via 192.168.1.1
```

この場合、親ルーターのIPは `192.168.1.1` です。

まず親ルーターの住所を知ります。

ここが分からないと、AP BridgeもWDSも始まりません。

## LN6001-JPへ割り当てるIPを決める

AP BridgeやWDSでは、LN6001-JPにも管理用IPが必要です。

たとえば、親ルーターが `192.168.1.1` の場合です。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-05.png)

ポイントは、親ルーターと同じサブネットで、ほかの機器と重複しないIPを選ぶことです。

```txt
親ルーターが 192.168.1.1
  → APは 192.168.1.2 など

親ルーターが 192.168.10.1
  → APは 192.168.10.2 など
```

同じサブネットに置く。

重複しない。

ここが基本です。

## DHCP配布範囲と重複しないようにする

親ルーターのDHCP配布範囲が次のようになっているとします。

```txt
192.168.1.100 - 192.168.1.199
```

この場合、LN6001-JPの管理IPは、配布範囲外に置くと管理しやすくなります。

おすすめ例です。

```txt
192.168.1.2
192.168.1.3
192.168.1.10
```

避けたい例です。

```txt
192.168.1.100
192.168.1.101
```

親ルーターがほかの端末へ配ってしまう可能性があるためです。

IP重複は、本当に面倒です。

動いたり動かなかったりします。

一番いやなタイプの不具合です。

最初から避けます。

## 設定前メモ

作業前に、次のようなメモを作ります。

```txt
親ルーター:
  IP: 192.168.1.1
  DHCP範囲: 192.168.1.100-199

AP Bridge:
  LN6001-JP IP: 192.168.1.2
  SSID: Home_AP

WDS:
  親機IP: 192.168.1.1
  子機IP: 192.168.1.3
  Parent SSID: Home_Backhaul
  Parent Password: 非公開

作業前バックアップ:
  backup-LN6001-before-ap-wds-20260622.tar.gz
```

このメモがないと、設定後にこうなります。

```txt
どのIPでLuCIへ入るんだっけ？
```

AP Bridge / WDSで一番多いトラブルは、設定ミスより管理IP迷子です。

IPアドレスは、設定前にメモします。

## 設定前にバックアップを取る

AP BridgeやWDSでは、`network`、`wireless`、`firewall`、`dhcp` が変わる可能性があります。

作業前にLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-ap-wds-20260622.tar.gz
```

SSHでも状態メモを残します。

```sh
BACKUP_DIR="/root/ap-wds-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json" 2>/dev/null || true
ifstatus lan > "$BACKUP_DIR/ifstatus-lan.json" 2>/dev/null || true
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
wifi status > "$BACKUP_DIR/wifi-status.json" 2>/dev/null || true
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、SSID、Wi-Fiパスワード、内部IP、MACアドレスなどが含まれることがあります。

公開しないようにしてください。

AP BridgeやWDSは、うまくいくと便利です。

でも、戻せる状態を作ってから触ると、心が平和です。

## Setup Assistantを使う理由

AP BridgeやWDSは、手動でも設定できます。

ただし、手動だと次をまとめて考える必要があります。

- LAN IP
- gateway
- DHCP無効化
- wireless mode
- SSID / password
- bridge
- firewall
- WDSリンク
- 管理IP
- 親機と子機の同一サブネット

ひとつひとつは理解できます。

でも、最初から全部を手で組むと、どこかで管理IP迷子になりがちです。

なので、最初はLinksys公式Setup Assistantを使うほうが安全です。

Setup Assistantは、AP BridgeやWDSに必要な項目をまとめて入力できる補助ツールです。

ただし、Setup Assistantを使っても、設定後の管理IPは自分で控える必要があります。

```txt
Setup Assistant = 設定補助
管理IPメモ = 自分の責任
```

ここは大事です。

## Setup Assistantを導入する

## LuCIで導入する

1. Linksys公式サポートページから `setup-assistant.ipk` をダウンロードする
2. LuCIへログインする
3. **System** → **Software** を開く
4. **Update lists** をクリックする
5. 完了後、**Dismiss** をクリックする
6. **Upload Package** をクリックする
7. ダウンロードした `setup-assistant.ipk` を **Browse** で選ぶ
8. **Upload** をクリックする
9. 確認画面で **Install** をクリックする
10. `installed in root is up to date.` と表示されたら **Dismiss** をクリックする
11. ブラウザをリロードするか、LuCIへ再ログインする
12. **Services** → **Setup Assistant** が表示されることを確認する

**Update lists** でエラーが出る場合は、先にWAN / DNS / 回線状態を確認します。

```sh
ifstatus wan
ifstatus wan6
nslookup downloads.openwrt.org
logread | grep -Ei 'opkg|wget|uclient|dnsmasq|wan' | tail -n 100
```

`opkg update` や `Update lists` が通らない場合、Setup Assistant以前に、ルーターが外へ出られていない可能性があります。

まずWANとDNSです。

## SSHで導入確認する

```sh
echo "### Setup Assistant package"
opkg list-installed | grep -Ei 'setup|assistant'

echo "### LuCI / opkg logs"
logread | grep -Ei 'setup|assistant|opkg|luci' | tail -n 100
```

メニューが出ない場合は、ブラウザ更新、LuCI再ログイン、または `uhttpd` 再起動を試します。

```sh
/etc/init.d/uhttpd restart
```

それでも出ない場合は、インストールログとパッケージ名を確認します。

## AP Bridge構成

AP Bridgeは、既存ルーター配下へLN6001-JPを有線接続し、アクセスポイントとして使う構成です。

## 配線イメージ

```txt
インターネット
  ↓
既存ルーター / HGW
  ↓ 有線LAN
LN6001-JP
  ↓ Wi-Fi
端末
```

この構成では、親ルーターがDHCPを担当します。

LN6001-JPは、親ルーターと同じサブネット内の管理IPを持つAPとして動きます。

## AP Bridgeに向くケース

- 既存ルーターを残したい
- 既存ルーターがIPoEやひかり電話を担当している
- Wi-Fiだけ強化したい
- 有線LANを引ける
- 安定性を優先したい
- ゲストWi-FiやVLANは親ルーター側で管理している
- 店舗や小さなオフィスの既存ネットワークへAP追加したい
- 二重ルーターを避けたい

AP Bridgeは、WDSより安定しやすいです。

有線を引けるなら、まずAP Bridgeを検討します。

## AP Bridgeの設定項目

Setup AssistantのAP Bridgeでは、主に次を入力します。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-06.png)

Gateway IPとAccess Point IPは、同じサブネットにします。

例:

```txt
Gateway IP:      192.168.1.1
Access Point IP: 192.168.1.2
```

Access Point IPは、親ルーターやほかの端末と重複しないIPにします。

ここを間違えると、設定後にLuCIへ戻るのが面倒になります。

## AP Bridgeの設定手順

1. **Services** → **Setup Assistant** を開く
2. **AP Bridge** タブを選ぶ
   - 画面上は `AP Brige` のように表示される場合があります
3. SSIDを入力する
4. SSID Passwordを入力する
5. Gateway IPを入力する
6. Access Point IPを入力する
7. **Save & Apply** をクリックする
8. 念のためLN6001-JPを再起動する
9. Access Point IPでLuCIへアクセスする

例:

```txt
https://192.168.1.2
```

設定後は、元の `https://192.168.1.1` ではなく、Access Point IPで管理画面へ入ります。

ここを忘れると、「LuCIへ入れなくなった」と勘違いしやすいです。

違います。

住所が変わっただけです。

## AP Bridge後の確認

PCを親ルーター側またはLN6001-JPのAP側へ接続し、確認します。

```sh
AP_IP="192.168.1.2"

ping -c 4 "$AP_IP"
ssh root@"$AP_IP"
```

SSHへ入れたら、状態を確認します。

```sh
echo "### system"
ubus call system board

echo "### lan"
uci show network.lan
ifstatus lan

echo "### routes"
ip route show
ip -6 route show

echo "### DHCP leases"
cat /tmp/dhcp.leases 2>/dev/null || true

echo "### wireless"
uci show wireless | grep -E 'ssid|network|encryption|disabled'
wifi status

echo "### logs"
logread | tail -n 100
```

AP Bridgeでは、端末のDHCPは親ルーターから配られる想定です。

LN6001-JP側でDHCPが二重に動いていないかも確認します。

```sh
uci show dhcp | grep -E 'lan|ignore|interface|start|limit'
```

二重DHCPは、かなりやっかいです。

端末によってIPの出どころが変わり、トラブルが見えにくくなります。

AP Bridgeでは、基本的に親ルーターにDHCPを任せます。

## AP Bridgeでよくある失敗

## Access Point IPを忘れた

一番多いパターンです。

対処:

- 作業メモを確認する
- 親ルーターのDHCPリース一覧を見る
- PC側Default Gatewayと同一サブネットをスキャンする
- 有線LANで接続して確認する

親ルーターが `192.168.1.1` なら、AP IPとして `192.168.1.2` や `192.168.1.10` を指定していることが多いです。

## Gateway IPとAccess Point IPが同一サブネットではない

悪い例です。

```txt
Gateway IP:      192.168.1.1
Access Point IP: 192.168.10.2
```

この場合、親ルーター配下の端末からAPへ戻れないことがあります。

基本は同一サブネットにします。

よい例です。

```txt
Gateway IP:      192.168.1.1
Access Point IP: 192.168.1.2
```

## 親ルーターのDHCP範囲と重複している

親ルーターが `192.168.1.100-199` を配るなら、Access Point IPはそこを避けます。

おすすめです。

```txt
192.168.1.2
192.168.1.3
192.168.1.10
```

避けたい例です。

```txt
192.168.1.100
192.168.1.101
```

IP重複は、管理画面へ入れない原因になります。

動いたり動かなかったりするので、かなり面倒です。

## AP Bridge後にインターネットへ出られない

確認します。

```sh
uci show network.lan
ip route show
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
logread | grep -Ei 'netifd|dhcp|dnsmasq|gateway|lan' | tail -n 100
```

見るポイントです。

- Gateway IPが親ルーターのIPか
- Access Point IPが同一サブネットか
- IP重複していないか
- 親ルーター側のLANポートへ接続しているか
- 親ルーター側でDHCPが動いているか
- LN6001-JP側でDHCPが二重に動いていないか

AP Bridgeでは、LN6001-JPだけ見ても原因が分からないことがあります。

親ルーター側も見ます。

## WDS子機構成

WDS子機は、親機へ無線接続し、離れた場所へネットワークを延長する構成です。

## 配線イメージ

```txt
インターネット
  ↓
親機 Velop WRT Pro 7
  ↓ 無線バックホール
子機 Velop WRT Pro 7
  ↓
子機側Wi-Fi / 有線LAN端末
```

## WDSに向くケース

- 有線LANを引けない
- ルーター同士を無線でつなぎたい
- 離れた部屋へWi-Fiを伸ばしたい
- 子機側LANポートも使いたい
- Velop WRT Pro 7同士で構成したい
- 配線工事が難しい場所で使いたい

WDSは便利ですが、無線バックホールの品質に左右されます。

最初の設定時は、親機と子機を近くに置いて接続確認します。

安定確認後に設置場所へ移動するのがおすすめです。

最初から離れた部屋で設定すると、原因が分かりにくくなります。

```txt
設定ミス？
距離？
壁？
SSID？
暗号化？
```

一気に候補が増えます。

まず近くで成功させます。

## WDSでそろえる条件

WDSでは、親機と子機の無線条件が重要です。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-07.png)

WDS接続できない時は、SSIDやパスワードだけでなく、暗号化方式と周波数帯も確認します。

特に、2.4GHz / 5GHz / 6GHzのどれを親子間で使うかは重要です。

## WDSの設定項目

Setup AssistantのWDS Configurationでは、主に次を入力します。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-08.png)

Child IP Addressは、親機と同じサブネットで、ほかの端末と重複しないIPにします。

例です。

```txt
Parent IP Address: 192.168.1.1
Child IP Address:  192.168.1.3
```

設定後、子機のLuCIは `https://192.168.1.3` で開く想定です。

これも必ずメモします。

## WDS子機の設定手順

1. 親機と子機を近い距離に置く
2. 子機LN6001-JPへ有線LANでPCを接続する
3. LuCIへログインする
4. **Services** → **Setup Assistant** を開く
5. **WDS Configuration** タブを選ぶ
6. Parent SSIDを入力する
7. Parent SSID Passwordを入力する
8. Parent IP Addressを入力する
9. Child IP Addressを入力する
10. **Save & Apply** をクリックする
11. 念のため子機を再起動する
12. Child IP AddressでLuCIへアクセスする

例:

```txt
https://192.168.1.3
```

設定後は、子機側の新しいIPアドレスで管理画面へ入ります。

元の `192.168.1.1` に入れなくても、すぐリセットしないでください。

Child IP Addressへアクセスします。

## WDS後の確認

子機へSSHできる場合は、次を確認します。

```sh
echo "### system"
ubus call system board

echo "### wireless"
wifi status
iw dev

echo "### routes"
ip route show
ip -6 route show

echo "### network"
uci show network | grep -E 'lan|wds|sta|bridge|ipaddr|gateway'

echo "### wireless config"
uci show wireless | grep -E 'ssid|mode|wds|network|encryption|disabled'

echo "### WDS / wireless logs"
logread | grep -Ei 'wds|wifi|wireless|hostapd|wpa|sta' | tail -n 150
```

無線リンクの状態を見るには、`iw dev` でInterface名を確認し、該当interfaceで確認します。

```sh
iw dev

WLAN_IF="ここにInterface名を入れる"
iw dev "$WLAN_IF" link 2>/dev/null || true
```

Interface名は環境によって変わります。

記事の例をそのまま `ath00` などで決め打ちしないほうが安全です。

まず `iw dev` で見ます。

## WDSでよくある失敗

## Parent SSIDやパスワードが違う

WDSでつながらない時の最初の確認ポイントです。

確認するものです。

- SSIDの大文字小文字
- 半角スペース
- 記号
- パスワード
- 暗号化方式
- 周波数帯

スマートフォンが親SSIDへ接続できるかも確認します。

ただし、スマホが接続できるからWDSも必ず成功する、という意味ではありません。

でも、SSIDやパスワードの切り分けには使えます。

## 親機と子機が遠すぎる

設定時から離れた場所に置くと、切り分けが難しくなります。

最初は親機の近くで設定し、接続確認後に移動します。

移動後に不安定になる場合は、距離や壁の影響が大きい可能性があります。

WDSは電波でつなぐので、物理環境がかなり効きます。

```txt
設定は合ってる
でも場所が悪い
```

これは普通にあります。

## 6GHzを過信している

6GHzは高速ですが、壁や距離の影響を受けやすいです。

WDSバックホールとして使う場合、見通しがよい近距離では有利です。

でも、部屋をまたぐと5GHzのほうが安定する場合があります。

「速い帯域ほど遠くまで届く」わけではありません。

6GHzは速い。  
でも、繊細。

このくらいの見方がちょうどいいです。

## Child IP Addressが重複している

子機側IPがほかの端末と重複すると、管理画面へ入れなくなることがあります。

親ルーターや親機のDHCP範囲を確認し、重複しないIPを指定します。

避けたい例です。

```txt
親ルーターDHCP範囲: 192.168.1.100-199
Child IP Address: 192.168.1.100
```

おすすめです。

```txt
Child IP Address: 192.168.1.3
```

## Wireless初回確認を済ませていない

Setup Assistant導入後や設定後に、Wireless画面で初回確認が必要になる場合があります。

うまくいかない場合は、次を確認します。

1. **Network** → **Wireless** を開く
2. Wi-Fi関連インターフェース名の最適化画面が表示されたら **Continue** をクリックする
3. Wi-Fiサービス再起動を待つ
4. 再度Wireless画面が正しく表示されることを確認する

設定後にWireless表示がおかしい場合は、まずこの初回確認を済ませます。

焦って設定を増やす前に、LuCIのWireless表示を整えます。

## 設定後の確認チェックリスト

AP Bridge / WDS設定後は、次を確認します。

## AP Bridge

```txt
□ Access Point IPへpingできる
□ Access Point IPでLuCIへ入れる
□ 親ルーター配下の端末と同一サブネットにいる
□ 端末が親ルーターからIPを取れている
□ インターネットへ出られる
□ LN6001-JP側でDHCPが二重に動いていない
□ Wi-Fi端末がSSIDへ接続できる
□ 親ルーター側のDHCPリースに不自然な重複がない
```

## WDS

```txt
□ Child IP Addressへpingできる
□ Child IP AddressでLuCIへ入れる
□ 親SSIDへ子機が接続できている
□ 子機側Wi-Fiへ端末が接続できる
□ 子機側LANポートの端末が通信できる
□ 親機と子機のSSID / パスワード / 暗号化方式 / 周波数帯が合っている
□ 設置場所へ移動後も安定している
□ WDSが不安定なら、設置場所や5GHz/6GHzを見直した
```

## CLIでまとめて確認する

```sh
echo "### system"
ubus call system board

echo "### network"
uci show network | grep -E 'lan|wan|gateway|ipaddr|proto|bridge|wds|sta'

echo "### wireless"
uci show wireless | grep -E 'ssid|mode|wds|network|encryption|disabled'

echo "### wifi status"
wifi status

echo "### interfaces"
ip addr show
ip route show
ip -6 route show

echo "### DHCP leases"
cat /tmp/dhcp.leases 2>/dev/null || true

echo "### logs"
logread | grep -Ei 'setup|assistant|wds|wifi|wireless|hostapd|wpa|netifd|dhcp' | tail -n 150
```

ここでは、設定が存在するかだけでなく、実際にどのIPを持っているかを見ます。

AP Bridge / WDSではIPアドレスが大事です。

管理IPを見失うと、全部が急に不安になります。

## 管理IPを見失った時

AP BridgeやWDS後にLuCIへ入れない時は、まず「IPが変わっただけ」を疑います。

## 親ルーターのDHCPリースを見る

親ルーターの管理画面で、接続中端末一覧やDHCPリースを見ます。

次のような名前を探します。

```txt
LN6001
Linksys
OpenWrt
設定したホスト名
```

AP BridgeやWDSで固定IPを設定している場合、DHCPリースに出ないこともあります。

その場合は、作業メモのAccess Point IP / Child IP Addressを確認します。

## PC側から探す

親ルーターが `192.168.1.1` なら、同じサブネット内で探します。

macOS / Linux例です。

```sh
for i in $(seq 1 254); do
  ping -c 1 -W 1 192.168.1.$i >/dev/null 2>&1 && echo "192.168.1.$i is up"
done
```

Windows PowerShell例です。

```powershell
1..254 | ForEach-Object {
  $ip = "192.168.1.$_"
  if (Test-Connection -ComputerName $ip -Count 1 -Quiet) {
    Write-Output "$ip is up"
  }
}
```

ただし、pingに応答しない機器もあります。

親ルーター側のDHCPリース一覧を見るほうが確実なこともあります。

## 旧IPへ入ろうとしていないか確認する

AP Bridge後:

```txt
旧IP: 192.168.1.1
新IP: Access Point IP
```

WDS後:

```txt
旧IP: 192.168.1.1
新IP: Child IP Address
```

設定後は、新しいIPでLuCIへ入ります。

ここを忘れがちです。

本当に忘れがちです。

## 戻し方

SSHへ入れる場合は、作業前バックアップから戻せます。

```sh
BACKUP_DIR="/root/ap-wds-before-YYYYMMDD-HHMM"

cp "$BACKUP_DIR/network" /etc/config/network
cp "$BACKUP_DIR/wireless" /etc/config/wireless
cp "$BACKUP_DIR/firewall" /etc/config/firewall
cp "$BACKUP_DIR/dhcp" /etc/config/dhcp

/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
wifi reload
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えます。

network restartやwifi reloadでSSHが切れることがあります。

有線LANで作業するのがおすすめです。

## LuCIバックアップから戻す

LuCIへ入れる場合は、バックアップから復元できます。

1. **System** → **Backup / Flash Firmware** を開く
2. **Restore backup** または **Upload archive** を選ぶ
3. 作業前に保存した `.tar.gz` を選ぶ
4. 復元する
5. 再起動を待つ
6. 旧IPまたは復元後IPでLuCIへアクセスする

復元後、LAN IPがバックアップ時点へ戻ることがあります。

PC側のDefault Gatewayを確認します。

```sh
ip route show default
```

Windowsなら次です。

```powershell
ipconfig
```

復元後にLuCIへ入れない時は、まずDefault Gatewayです。

管理画面の住所が戻っただけかもしれません。

## ハードリセットする前に

どうしても戻れない時は、リセットも選択肢です。

ただし、ハードリセットの前に次を確認します。

```txt
□ 有線LANで接続した
□ Access Point IP / Child IP Addressを試した
□ 親ルーターのDHCPリースを見た
□ PC側Default Gatewayを確認した
□ SSHへ入れるか確認した
□ LuCIバックアップがあるか確認した
□ Setup Assistantの入力値を見直した
□ 親機と子機を近くに置いてWDSを再確認した
```

初期化は最後の手段です。

リセット後は、本体底面ラベルの初期SSID・初期パスワード・初期管理パスワードで再設定します。

詳しくはリセットと復旧の記事で扱っています。

## AP Bridge / WDS運用メモ

設定後は、次のようなメモを残しておくと便利です。

```txt
AP / WDS運用メモ:

構成:
  AP Bridge / WDS

親ルーター:
  IP: 192.168.1.1
  DHCP範囲: 192.168.1.100-199

LN6001-JP:
  管理IP: 192.168.1.2
  モード: AP Bridge
  SSID: Home_AP

WDS:
  親機IP: 192.168.1.1
  子機IP: 192.168.1.3
  Parent SSID: Home_Backhaul
  周波数帯: 5GHz / 6GHz
  暗号化方式: WPA3-SAE / WPA2-PSK

Setup Assistant:
  導入済み
  導入日: 2026-06-22

バックアップ:
  backup-LN6001-before-ap-wds-20260622.tar.gz

戻し方:
  LuCIバックアップから復元
  または /root/ap-wds-before-... のnetwork/wireless/firewall/dhcpを戻す
```

AP BridgeやWDSは、設定が動いたあとも、半年後に管理IPを忘れがちです。

管理IPだけでもメモしておくと助かります。

未来の自分、だいたい忘れています。

## よくある失敗

## AP Bridge後にインターネットへ出られない

確認します。

```sh
uci show network.lan
ip route show
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
logread | grep -Ei 'netifd|dhcp|dnsmasq|gateway|lan' | tail -n 100
```

見るポイントです。

- Gateway IPが親ルーターのIPか
- Access Point IPが同一サブネットか
- IP重複していないか
- 親ルーター側のLANポートへ接続しているか
- 親ルーター側でDHCPが動いているか
- LN6001-JP側でDHCPが二重に動いていないか

## AP Bridge後に管理画面へ入れない

ほとんどの場合、Access Point IPへアクセスしていないことが原因です。

```txt
https://<Access Point IP>
```

で開きます。

親ルーターのDHCPリース一覧も確認します。

## WDSで接続できない

確認します。

```sh
wifi status
uci show wireless | grep -E 'ssid|mode|wds|encryption|network|disabled'
logread | grep -Ei 'wds|wifi|wireless|wpa|hostapd|auth|assoc' | tail -n 150
```

見るポイントです。

- Parent SSIDが正確か
- パスワードが正確か
- 暗号化方式が一致しているか
- 周波数帯が一致しているか
- 親機と子機が近い距離にあるか
- 親機側SSIDへ通常端末が接続できるか

## WDSが不安定

よくある原因です。

- 親機と子機が遠い
- 壁や家具が多い
- 6GHzが距離や壁に弱い環境
- 電子レンジや近隣Wi-Fiの干渉
- バックホールと端末接続で同じ帯域を使っている
- 子機設置場所の電波が弱い

対処です。

- まず親機と子機を近くに置いて確認する
- 設置場所を変える
- 5GHzと6GHzを比較する
- 可能なら有線AP Bridgeへ変更する
- 子機側端末数を減らす

WDSは便利です。

でも、電波には物理があります。

根性では壁を貫通しません。

## WDSはメッシュと同じだと思っていた

WDSはメッシュとは違います。

Velop WRT Pro 7は、Linksys Intelligent MeshやEasyMeshへ参加する構成ではありません。

Velop WRT Pro 7同士を無線接続したい場合は、WDSとして考えます。

ここを混ぜると、期待する動作と実際の構成がズレます。

## Setup AssistantがServicesに出ない

確認します。

```sh
opkg list-installed | grep -Ei 'setup|assistant'
logread | grep -Ei 'setup|assistant|opkg|luci|uhttpd' | tail -n 100
```

対処です。

- ブラウザをリロードする
- LuCIへ再ログインする
- `uhttpd` を再起動する

```sh
/etc/init.d/uhttpd restart
```

- パッケージがインストール済みか確認する
- 必要ならSetup Assistantを再インストールする

## 変更する時の順番

AP Bridge / WDSは、次の順番で進めると安全です。

1. 既存ルーターのIPを確認する
2. DHCP配布範囲を確認する
3. LN6001-JPへ割り当てる管理IPを決める
4. 設定前バックアップを取る
5. Setup Assistantを導入する
6. AP BridgeまたはWDSを設定する
7. 新しい管理IPでLuCIへ入る
8. Wi-Fi端末が通信できるか確認する
9. 親ルーター側のDHCPやリースも確認する
10. 動作確認後に運用メモを残す

特に大事なのは、3と7です。

```txt
管理IPを決める
新しい管理IPで入る
```

ここを忘れないだけで、かなりトラブルを減らせます。

## まとめ

LN6001-JPは、既存ネットワークへAP BridgeやWDS子機として追加できます。

考え方はシンプルです。

1. 既存ルーターを残して有線APにするならAP Bridge
2. 有線を引けない場所へ無線で延長するならWDS
3. Linksys Intelligent Mesh / EasyMeshとは別物として考える
4. 最初は手動設定よりSetup Assistantを使う
5. Gateway IP、Access Point IP、Child IP Addressを必ず控える
6. 管理IPは親ルーターのDHCP範囲と重複しないものにする
7. WDSは親機と子機のSSID、パスワード、暗号化方式、周波数帯をそろえる
8. 設定後は新しい管理IPでLuCIへ入る
9. WDSが不安定なら設置場所や有線AP Bridgeを検討する
10. 作業前に必ずバックアップを取る

AP BridgeやWDSは、うまく使うとLN6001-JPの活用範囲を広げてくれます。

ただし、最初から手動で複雑に作らず、Setup Assistantで小さく始めるのがおすすめです。

一番大事なのは、設定後の管理IPを見失わないことです。

ここさえ押さえれば、既存ネットワークへの追加はかなり進めやすくなります。

Wi-Fiだけ足したいなら、まずAP Bridge。

有線が無理なら、WDS。

そして、どちらでも管理IPをメモ。

これでかなり平和です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-03.png)

## 次に読むなら

AP BridgeやWDSを試す前後には、次の記事も合わせて読むと整理しやすいです。

- [ハードウェア概要](https://note.com/ikmsan/n/nf5e2df270ea3)
- [Wi-Fi設定](https://note.com/ikmsan/n/n4dbf8f72744c)
- [SSH基本コマンド集](https://note.com/ikmsan/n/n99897dd3cae6)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

設定前のバックアップを固めたい人は、設定バックアップと復元へ。

設定後に管理画面へ戻れない人は、リセットと復旧、つながらない時の切り分けへ進むと戻しやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

Setup Assistant、AP Bridge、WDS Configuration、LuCIの画面名、Wi-Fi radio名、`wifi status`、`iw dev`、`logread` の出力は、ファームウェアや追加モジュール更新で変わることがあります。

AP BridgeやWDSは、既存ルーター、HGW、親機SSID、暗号化方式、周波数帯、設置環境に強く依存します。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートと実機状態を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### AP BridgeとWDSは何が違う？

AP Bridgeは有線で既存ルーター配下へ追加する構成です。

WDSは親機へ無線接続してネットワークを延長する構成です。

有線を引けるならAP Bridgeのほうが安定しやすいです。

### LN6001-JPはLinksys Intelligent Meshに参加できますか？

この記事の構成では参加しません。

Velop WRT Pro 7は、従来のVelop / MXシリーズのLinksys Intelligent MeshやEasyMesh互換として追加する構成ではなく、Velop WRT Pro 7同士を無線でつなぐ場合はWDSとして扱います。

### Setup AssistantなしでもAP BridgeやWDSを設定できますか？

できます。

LuCIやCLIで手動設定することも可能です。

ただし、最初はSetup Assistantを使ったほうが、入力項目を整理しやすく、切り分けもしやすくなります。

### AP Bridge後にLuCIへ入れなくなりました

Access Point IPでアクセスしてください。

AP Bridge後は、元の `192.168.1.1` ではなく、Setup Assistantで指定したAccess Point IPが管理画面の入口になります。

### WDS子機のLuCIへ入れません

Child IP Addressでアクセスしてください。

親機やほかの機器とIPアドレスが重複していると、管理画面へ入れないことがあります。

### WDSは6GHzで使うべきですか？

6GHzは高速ですが、距離や壁の影響を受けやすいです。

親機と子機の距離が近く、見通しがよい場合は有力ですが、安定性重視なら5GHzや有線AP Bridgeも検討してください。

### 二重ルーターは必ず避けるべきですか？

必ずではありません。

ポートフォワーディング、VPN着信、オンラインゲームなどで問題がなければ、そのまま使える場合もあります。

ただし、Wi-Fiだけ追加したいならAP Bridgeのほうが分かりやすいです。

### WDSが不安定な時はどうすればいい？

まず親機と子機を近くに置いて確認します。

その後、設置場所、周波数帯、壁や家具、電子レンジなどの干渉を見ます。

安定性を重視するなら、有線AP Bridgeを検討してください。

### AP BridgeにしたらゲストWi-Fi分離はどうなりますか？

AP Bridgeでは、基本的に親ルーター側のネットワーク設計に乗ります。

LN6001-JP側で単にSSIDを出すだけなら、親ルーター配下の同一LANに入る形になります。

ゲストWi-FiやVLANをどこで分離するかは、親ルーター側の機能や構成も含めて設計してください。

### AP Bridge / WDS後にリセットすれば戻りますか？

戻せますが、初期化で設定は消えます。

まずAccess Point IP / Child IP Address、親ルーターのDHCPリース、SSH接続、LuCIバックアップ復元を試してください。

ハードリセットは最後の手段です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt アクセスポイント・ブリッジモードおよびWDS子機としてVelop WRT Pro 7を設定する方法: https://support.linksys.com/kb/article/7195-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041-jp/

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
