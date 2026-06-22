<!-- mirror-source: articles/025-ssh-basic-commands.md -->

# SSHは怖くない｜LN6001-JPで“読むだけCLI”から始める基本コマンド集【OpenWrt集中連載025】

SSHと聞くと、ちょっと身構える人は多いと思います。

黒い画面。  
英語のログ。  
よく分からないコマンド。  
1文字間違えたら壊れそうな雰囲気。

分かります。

でも、LN6001-JPでSSHを使う時、最初から設定変更までやる必要はありません。

むしろ最初は、こう考えるのがおすすめです。

```txt
SSH = ルーターの中を“読む”ための道具
```

WANは生きているのか。  
IPv6側はどう見えているのか。  
DNSだけおかしいのか。  
ゲストWi-Fi端末はIPを取れているのか。  
Adblockが効きすぎていないか。  
Tailscaleは起動しているのか。  
firewall zoneは意図どおりなのか。

LuCIだけでは見えにくいところも、SSHで少し読むだけでかなり分かります。

逆に言うと、最初からCLIで設定を変えなくて大丈夫です。

読む。  
ログを見る。  
変更前にバックアップする。  
困った時に状態を残す。

まずはこれで十分です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）でSSHへ入り、読み取り中心で使える基本コマンド、ログ確認、UCIの見方、設定変更前のバックアップ、戻し方を整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、SSH職人になることではありません。

まずは、ここまでできればOKです。

- LN6001-JPへSSHでログインできる
- 最初に使う読み取りコマンドが分かる
- WAN、WAN6、DNS、DHCP、Wi-Fi、firewallを分けて見られる
- `logread` で直近ログを確認できる
- `uci show` で設定を読む感覚が分かる
- 設定変更前に `/etc/config/` をバックアップできる
- `uci changes` / `uci commit` の意味が分かる
- 変更後に戻すための基本パターンが分かる
- 診断ログを共有する時に、伏せるべき情報が分かる

CLIは、いきなり設定を変えるためのものではありません。

まずは、

```txt
今どうなっているかを見る
```

ところから始めます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-01.png)

## 先にざっくり結論

SSHで最初に覚えるなら、この4つだけで十分です。

```sh
ubus call system board
ifstatus wan
cat /tmp/dhcp.leases
logread | tail -n 50
```

この4つで、かなり見えます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-01.png)

IPoEやIPv6環境では、次もセットで覚えます。

```sh
ifstatus wan6
ip -6 route show
ping6 -c 4 2001:4860:4860::8888
```

おすすめの使い分けはこうです。

```txt
状態確認はSSH
設定変更はLuCI
大きな変更前だけSSHでバックアップ
```

慣れてきたら、UCIで小さく変更するくらいで十分です。

最初からCLIだけで全部やろうとしない。

これが、かなり大事です。

## こういう人向けです

この記事は、次のような人向けです。

- LuCIだけでは見えない状態を確認したい
- SSHに入れるけれど、何を打てばいいか分からない
- IPoE、IPv6、DNS、DHCPの切り分けをしたい
- ゲストWi-FiやVLAN設定後に、どこで止まっているか見たい
- AdblockやTailscale導入後のログを見たい
- サポートや詳しい人へ相談する前に、最低限の情報を集めたい
- CLIで設定変更する前に、安全な型を知りたい
- `iptables` ではなく、UCI / iptables時代の確認方法を整理したい

逆に、すでにOpenWrtのCLI運用に慣れている人には、かなり基本寄りです。

ただ、家庭や小さなオフィスでは、この基本が一番効きます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-02.png)

## 最初に言葉だけそろえる

SSHまわりで出てくる言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-02.png)

最初は、次だけで大丈夫です。

```txt
SSH = 文字で見る管理入口
UCI = 設定を見る・変える仕組み
logread = ログを見る
ifstatus = WANやinterfaceの状態を見る
```

## SSH利用の基本方針

SSHを使う時は、先にルールを決めておくと安全です。

```txt
1. 最初は読み取りだけ
2. 変更前にバックアップ
3. 変更は1つずつ
4. uci changesで差分確認
5. commitは対象configだけ
6. 反映後に状態確認
7. だめなら直前バックアップへ戻す
```

特に次の設定は、間違えるとLuCIやSSHへ戻れなくなることがあります。

- LAN IP
- VLAN / bridge
- firewall zone
- Wi-Fi暗号化方式
- SSH設定
- DHCP設定
- WAN / IPoE / PPPoE
- VPN関連

最初は、LuCIで設定変更し、SSHでは状態確認とバックアップに使うくらいがちょうどいいです。

SSHは、ネットワークの手術道具にもなります。

でも、最初は聴診器くらいの使い方で十分です。

## SSHをWAN側へ公開しない

SSHは、基本的にLAN側から使います。

インターネット側へSSHを直接公開するのはおすすめしません。

外出先から管理したい場合は、TailscaleやWireGuardなどのVPNを使います。

```txt
外出先PC
  ↓ VPN
LN6001-JP
  ↓ SSH / LuCI
```

SSHをWANへ直接出すより、VPN経由で管理するほうが安全に設計しやすいです。

これはかなり大事です。

LuCIもSSHも、管理用の入口です。

外から直接見える場所に置かないほうが安心です。

## SSHログイン方法

Mac / Linuxではターミナルを使います。

```sh
ssh root@192.168.1.1
```

WindowsではPowerShellまたはWindows Terminalを使います。

```powershell
ssh root@192.168.1.1
```

LAN IPを変更している場合は、そのIPへ接続します。

例:

```sh
ssh root@192.168.10.1
```

初回接続時は、fingerprintの確認が出ます。

接続先が自分のLN6001-JPであることを確認し、問題なければ `yes` と入力します。

パスワードは、LuCIの `root` パスワードと同じです。

初期状態では、本体底面ラベルのデフォルトパスワードを使う構成です。

初回設定後は、必ず管理者パスワードを変更してください。

## host key警告が出た時

初期化やファームウェア更新後に、次のような警告が出ることがあります。

```txt
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

これは、以前接続した `192.168.1.1` のSSHホスト鍵と、現在の鍵が違うという警告です。

初期化や機器交換をした直後なら、古い記録を削除して再接続します。

```sh
ssh-keygen -R 192.168.1.1
ssh root@192.168.1.1
```

LAN IPを変えている場合は、そのIPを指定します。

```sh
ssh-keygen -R 192.168.10.1
ssh root@192.168.10.1
```

ただし、身に覚えがないのにこの警告が出る場合は、接続先が本当にLN6001-JPか確認してください。

同じIPアドレスに別の機器がいる可能性もあります。

警告を反射で無視しない。

ここは、ちゃんと見ます。

## SSH接続前に確認すること

SSHへ入れない時は、まずPC側とLAN側を見ます。

```txt
PC
  ↓ 有線LANまたはメインWi-Fi
LN6001-JP
```

確認します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-03.png)

ゲストWi-FiからSSHへ入れないのは、正常な設計です。

管理作業は、メインLANまたは有線LANから行います。

「SSHできない」と思ったら、まず自分がどのネットワークにいるかを見ます。

ここ、意外とよくハマります。

## まず取っておくベースライン

SSHへ入ったら、最初に現在状態を控えます。

設定を触る前の記念写真みたいなものです。

```sh
echo "### date"
date

echo "### system board"
ubus call system board

echo "### kernel"
uname -a

echo "### uptime"
uptime

echo "### memory"
free

echo "### storage"
df -h

echo "### key packages"
opkg list-installed | grep -E 'base-files|busybox|dnsmasq|firewall|dropbear|uhttpd|opkg'
```

この出力を見るだけで、次が分かります。

- ファームウェア情報
- カーネル情報
- 起動してからの時間
- メモリ使用量
- ストレージ空き容量
- 基本パッケージの状態

特に `df -h` では、`/overlay` の空き容量を見ます。

追加パッケージを入れすぎると、ここが埋まることがあります。

`/overlay` がパンパンのルーターは、だいたい機嫌が悪くなります。

## 3分診断パック

原因が分からない時は、まず読み取り中心で状態を保存します。

設定を変える前に、今の状態を残します。

```sh
DIAG_DIR="/tmp/ln6001-ssh-diag-$(date +%Y%m%d-%H%M)"
mkdir -p "$DIAG_DIR"

date > "$DIAG_DIR/date.txt"
ubus call system board > "$DIAG_DIR/system-board.json" 2>/dev/null || true
uptime > "$DIAG_DIR/uptime.txt"
free > "$DIAG_DIR/free.txt"
df -h > "$DIAG_DIR/df-h.txt"

ifstatus wan > "$DIAG_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$DIAG_DIR/ifstatus-wan6.json" 2>/dev/null || true

ip addr show > "$DIAG_DIR/ip-addr.txt"
ip route show > "$DIAG_DIR/route-v4.txt"
ip -6 route show > "$DIAG_DIR/route-v6.txt"

cat /tmp/dhcp.leases > "$DIAG_DIR/dhcp-leases.txt" 2>/dev/null || true
uci show network > "$DIAG_DIR/network.uci.txt" 2>/dev/null || true
uci show wireless > "$DIAG_DIR/wireless.uci.txt" 2>/dev/null || true
uci show dhcp > "$DIAG_DIR/dhcp.uci.txt" 2>/dev/null || true
uci show firewall > "$DIAG_DIR/firewall.uci.txt" 2>/dev/null || true

wifi status > "$DIAG_DIR/wifi-status.json" 2>/dev/null || true
iw dev > "$DIAG_DIR/iw-dev.txt" 2>/dev/null || true

iptables-save > "$DIAG_DIR/iptables-save.txt" 2>/dev/null || true
ip6tables-save > "$DIAG_DIR/ip6tables-save.txt" 2>/dev/null || true

logread | tail -n 300 > "$DIAG_DIR/logread-tail.txt" 2>/dev/null || true
dmesg | tail -n 120 > "$DIAG_DIR/dmesg-tail.txt" 2>/dev/null || true

ls -l "$DIAG_DIR"
```

PCへコピーする場合は、PC側で実行します。

```sh
scp -r root@192.168.1.1:/tmp/ln6001-ssh-diag-* ./
```

LAN IPを変更している場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/tmp/ln6001-ssh-diag-* ./
```

この診断データには、SSID、MACアドレス、内部IP、IPv6 prefix、VPN情報などが含まれることがあります。

共有前に伏せ字にしてください。

## システム状態を見る

まずはルーター本体の状態です。

```sh
# モデル・ファームウェア・targetなど
ubus call system board

# 起動時間とロードアベレージ
uptime

# メモリ使用量
free

# ストレージ使用量
df -h

# プロセス一覧
ps

# CPU負荷をざっくり見る
top -bn1 | head -n 30

# カーネル情報
uname -a
uname -r
```

見るポイントです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-04.png)

再起動したつもりがないのに `uptime` が短い場合、途中で再起動している可能性があります。

AdblockやVPN、追加パッケージを入れたあとに不安定なら、メモリやストレージも見ます。

## WAN / WAN6を見る

IPoEやIPv6環境では、WANとWAN6を分けて見ます。

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### IPv4 addresses"
ip addr show

echo "### IPv4 routes"
ip route show

echo "### IPv6 addresses"
ip -6 addr show

echo "### IPv6 routes"
ip -6 route show

echo "### WAN logs"
logread | grep -Ei 'wan|wan6|netifd|dhcp|dhcpv6|odhcp6c|ppp|pppoe|ipoe|map|mape|dslite|ipip' | tail -n 120
```

見るポイントです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-05.png)

HGW配下では、WAN側IPv4がプライベートIPになることがあります。

これは必ずしも異常ではありません。

ただし、HGWとLN6001-JPのLAN IPが重複している場合は注意します。

## DNSを見る

DNS問題は、「インターネット全体が落ちた」と誤解しやすいポイントです。

IP直打ちと名前解決を分けて確認します。

```sh
echo "### IPv4 direct ping"
ping -c 4 8.8.8.8

echo "### IPv6 direct ping"
ping6 -c 4 2001:4860:4860::8888

echo "### DNS via default resolver"
nslookup www.google.co.jp

echo "### DNS via router local resolver"
nslookup www.google.co.jp 127.0.0.1

echo "### DNS via external resolver"
nslookup www.google.co.jp 8.8.8.8

echo "### resolver config"
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 100
```

判断の目安です。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-06.png)

Adblockを使っている場合は、一時停止して切り分けることもあります。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
nslookup www.google.co.jp 127.0.0.1
```

確認後、必要なら戻します。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

DNSは、本当に“見た目はネット全滅”に見えることがあります。

IP直打ちは通るのに名前だけ引けないなら、Wi-FiではなくDNSです。

## DHCPと端末を見る

端末がIPアドレスを取れているか確認します。

```sh
echo "### active DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP config summary"
uci show dhcp | grep -E 'interface|start|limit|leasetime|dhcp_option|dns|ignore|@host|\.mac|\.ip|\.name'

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 100
```

見るポイントです。

- 対象端末がリース一覧にいるか
- 端末が想定したIP範囲を取っているか
- ゲストWi-Fi端末が `192.168.2.x` などになっているか
- Device / IoT端末が想定ネットワークにいるか
- DHCP予約が重複していないか
- DHCP Serverが対象interfaceで有効か

端末側で `169.254.x.x` のようなIPになっている場合、DHCPでIPを取れていません。

ゲストWi-FiやDevice zoneでInputを `REJECT` にしている場合、DHCP許可ルールが必要です。

例:

```txt
Allow-Guest-DHCP
Allow-Device-DHCP
Allow-Kids-DHCP
```

Wi-Fiマークが出ていても、IPを取れていないなら通信はできません。

ここ、かなりよくあります。

## Wi-Fiを見る

Wi-Fiは、SSID、radio、暗号化、network紐づけ、接続端末を分けて見ます。

```sh
echo "### wireless config summary"
uci show wireless | grep -E 'wifi-device|wifi-iface|ssid|encryption|network|disabled'

echo "### wifi status"
wifi status

echo "### wireless interfaces"
iw dev

echo "### associated stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done

echo "### wireless logs"
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

見るポイントです。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/table-07.png)

6GHzが見えない場合は、端末がWi-Fi 6E / 7に対応しているか確認します。

IoT機器がつながらない場合は、2.4GHz、WPA2-PSK、英数字SSIDなど、互換性重視で確認します。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

## firewallを見る

firewallは、まずUCI設定を見ます。

次に、必要ならUCI / iptablesの展開結果を見ます。

```sh
echo "### firewall UCI config"
uci show firewall

echo "### zones and forwarding"
uci show firewall | grep -E 'zone|forwarding|name|network|src|dest|target|input|output|forward'

echo "### iptables rules"
iptables-save 2>/dev/null | head -n 200 || true

echo "### ip6tables rules"
ip6tables-save 2>/dev/null | head -n 200 || true

echo "### firewall-related logs"
logread | grep -Ei 'firewall|drop|reject' | tail -n 100
```

LN6001-JPでは、UCI設定とiptables/ip6tables系の表示を中心に見る方針が現実的です。

古い記事では `iptables -L -n -v` を使う例がありますが、それだけに依存しないほうが安全です。

互換表示として確認したい場合は、失敗しても止まらない形で使います。

```sh
iptables -L -n -v 2>/dev/null | head -n 80 || true
iptables-save 2>/dev/null | head -n 120 || true
```

見るポイントです。

- `lan → wan` forwardingがあるか
- `guest → wan` forwardingがあるか
- `guest → lan` を作っていないか
- guest / device / kidsのInputを `REJECT` にしている場合、DHCP / DNS許可があるか
- VPNからLANへ広く許可しすぎていないか
- Port Forwardが不要に残っていないか

firewallは、ログだけ見ても分かりにくいです。

まずzoneとforwardingを見ます。

## UCIで設定を読む

CLI設定変更の前に、まず設定を読みます。

```sh
uci show network
uci show wireless
uci show firewall
uci show dhcp
uci show system
```

よく見る設定だけ抜き出す例です。

```sh
# LAN IP
uci get network.lan.ipaddr

# SSIDとnetwork紐づけ
uci show wireless | grep -E 'ssid|network|encryption|disabled'

# firewall zoneとforwarding
uci show firewall | grep -E 'zone|forwarding|name|src|dest|input|output|forward|target'

# DHCP範囲と予約
uci show dhcp | grep -E 'interface|start|limit|@host|\.name|\.mac|\.ip'
```

最初は `uci show` と `uci get` だけで十分です。

`uci set` は設定変更です。

慣れるまでは、読み取り中心で使います。

```txt
uci show = 見る
uci get = 1つの値を見る
uci set = 変える
uci commit = 保存する
```

この違いだけでも、かなり安全になります。

## UCIで設定変更する時の安全な型

CLIで設定変更する場合は、次の型を守ります。

```sh
# 1. まず対象設定を読む
uci show network

# 2. 対象ファイルをバックアップ
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)

# 3. 変更を1つだけ入れる
uci set network.lan.ipaddr='192.168.10.1'

# 4. 未保存の変更を確認
uci changes

# 5. 問題なければ対象configだけ保存
uci commit network

# 6. 反映
/etc/init.d/network restart
```

上のLAN IP変更は例です。

実際に実行すると、管理画面のURLが変わります。

```txt
変更前: https://192.168.1.1
変更後: https://192.168.10.1
```

LAN IP、VLAN、firewall、Wi-Fi設定は、LuCIで変更するほうが安全な場合が多いです。

CLIでは、まず読み取りとバックアップを中心に使います。

## 未保存の変更を見る・取り消す

UCIでは、`uci set` しただけではまだ保存されていません。

未保存の変更を見るには次です。

```sh
uci changes
```

保存するには、対象configを指定します。

```sh
uci commit network
```

未保存の変更を取り消すには、対象configを指定してrevertします。

```sh
uci revert network
```

例です。

```sh
uci set network.lan.ipaddr='192.168.10.1'
uci changes
uci revert network
uci changes
```

CLIで作業する時は、`uci changes` を必ず挟むのがおすすめです。

ここを飛ばすと、自分が何を変えたのか分からなくなります。

## rollbackの基本

設定変更後におかしくなった場合は、バックアップしたファイルを戻して、対象サービスを再起動します。

## networkを戻す

LAN IP、WAN、VLAN、bridgeを戻す場合です。

```sh
cp /etc/config/network.backup.YYYYMMDD-HHMM /etc/config/network
/etc/init.d/network restart
```

`network restart` 後は、SSHやLuCIが切れる可能性があります。

有線LANで作業し、PC側のIPを取り直します。

## firewallを戻す

firewall zone、Traffic Rules、Port Forwardを戻す場合です。

```sh
cp /etc/config/firewall.backup.YYYYMMDD-HHMM /etc/config/firewall
/etc/init.d/firewall restart
```

firewallだけなら、networkより影響範囲が小さいことが多いです。

## Wi-Fiを戻す

SSID、暗号化、radio設定を戻す場合です。

```sh
cp /etc/config/wireless.backup.YYYYMMDD-HHMM /etc/config/wireless
wifi reload
```

`wifi reload` 中はWi-Fiが切れます。

有線LANで入っている状態がおすすめです。

## DHCP / DNSを戻す

DHCP予約、DNS、Family DNS、Adblock連携などを戻す場合です。

```sh
cp /etc/config/dhcp.backup.YYYYMMDD-HHMM /etc/config/dhcp
/etc/init.d/dnsmasq restart
```

`YYYYMMDD-HHMM` は実際のバックアップファイル名に置き換えます。

DNSだけ怪しい場合は、まず `dnsmasq` 再起動だけで直ることもあります。

```sh
/etc/init.d/dnsmasq restart
```

## サービス管理コマンド

設定変更後に、対象サービスを再起動することがあります。

```sh
# ネットワーク全体。通信断に注意
/etc/init.d/network restart

# firewallだけ
/etc/init.d/firewall restart

# DNS / DHCP
/etc/init.d/dnsmasq restart

# LuCIのWebサーバー
/etc/init.d/uhttpd restart

# Wi-Fiだけ
wifi reload
```

追加パッケージがある場合は、存在確認してから実行します。

```sh
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock status
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock restart

[ -x /etc/init.d/tailscale ] && /etc/init.d/tailscale status
[ -x /etc/init.d/tailscale ] && /etc/init.d/tailscale restart
```

`/etc/init.d/network restart` は、SSHが切れる可能性があります。

まず、できるだけ対象サービスだけ再起動します。

## パッケージ確認コマンド

パッケージ管理は、opkgを使います。

```sh
# パッケージ一覧を更新
opkg update

# インストール済み一覧
opkg list-installed

# 特定パッケージ確認
opkg list-installed | grep -Ei 'adblock|tailscale|wireguard|vpn|ipoe|luci-app'

# パッケージ情報
opkg info tcpdump

# パッケージが置いたファイル
opkg files tcpdump

# ストレージ空き容量
df -h
```

ファームウェア更新前には、追加パッケージ一覧を保存します。

```sh
opkg list-installed > /root/opkg-list-installed-$(date +%Y%m%d-%H%M).txt
```

OpenWrt系では、`opkg upgrade` で一括更新する運用は基本的におすすめしません。

使うのは主にこれです。

```sh
opkg update
opkg install <package>
opkg remove <package>
opkg list-installed
```

`update` と `upgrade` は違います。

名前が似ているので、ここは本当に注意です。

## ログを見る

SSHの強みはログを直接見られることです。

```sh
# 全ログ
logread

# 末尾50行
logread | tail -n 50

# リアルタイム表示
logread -f

# error / warn / fail だけざっくり見る
logread | grep -Ei 'err|error|warn|fail|timeout|restart|disconnect' | tail -n 100
```

用途別に見るなら次です。

```sh
# WAN / IPoE / PPPoE
logread | grep -Ei 'wan|wan6|netifd|odhcp6c|ppp|pppoe|ipoe|map|dslite|ipip' | tail -n 120

# DNS / DHCP
logread | grep -Ei 'dnsmasq|dhcp' | tail -n 120

# Wi-Fi
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120

# firewall
logread | grep -Ei 'firewall|drop|reject' | tail -n 120

# Adblock
logread | grep -Ei 'adblock|dnsmasq' | tail -n 120

# Tailscale
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120

# WireGuard
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

ログは「答え」ではなく「手がかり」です。

WAN、DNS、DHCP、Wi-Fiの状態と合わせて見ます。

ログだけをじっと見つめても、だいたいログに見つめ返されます。

## ファイルを読む・編集する

設定ファイルを直接読むと、LuCIで何が保存されているか理解しやすくなります。

```sh
cat /etc/config/network
cat /etc/config/wireless
cat /etc/config/firewall
cat /etc/config/dhcp
```

長いファイルを見る時は、まず `sed -n` で先頭だけ見ると安全です。

```sh
sed -n "1,160p" /etc/config/network
```

編集は `vi` を使えます。

```sh
vi /etc/config/network
```

viの最低限の操作です。

```txt
i      挿入モード
ESC    コマンドモードへ戻る
:wq    保存して終了
:q!    保存せず終了
```

最初は `cat` や `uci show` で読むだけでも十分です。

`vi` で直接編集するのは、慣れてからにします。

設定ファイルを直接編集する時は、必ず先にコピーを取ります。

```sh
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)
```

## 設定変更前のまとめバックアップ

CLIで設定を触る前に、最低限この5つを保存しておくと復旧しやすくなります。

```sh
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップはルーター内に保存されます。

大きな変更前は、LuCIの **System → Backup / Flash Firmware → Generate archive** からPCにもバックアップを保存します。

PCへコピーするなら、PC側で次を実行します。

```sh
scp -r root@192.168.1.1:/root/config-backup-YYYYMMDD-HHMM ~/Downloads/
```

LAN IPを変更している場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/root/config-backup-YYYYMMDD-HHMM ~/Downloads/
```

## サポート相談前に集める情報

サポートや詳しい人へ相談する前に、次の読み取りコマンドを取っておくと説明しやすくなります。

```sh
echo "### system"
date
ubus call system board
uptime

echo "### interfaces"
ifstatus wan
ifstatus wan6
ip addr show
ip route show
ip -6 route show

echo "### dns-dhcp"
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf
cat /tmp/dhcp.leases
uci show dhcp | grep -E 'dns|server|dhcp_option|ignore' || true

echo "### wifi"
wifi status
iw dev

echo "### firewall"
uci show firewall
iptables-save 2>/dev/null | head -n 160 || true
ip6tables-save 2>/dev/null | head -n 160 || true

echo "### recent logs"
logread | tail -n 120
```

ただし、ログや設定には次が含まれることがあります。

- SSID
- Wi-Fiパスワード
- MACアドレス
- 内部IP
- グローバルIPv6 prefix
- DDNS名
- VPN情報
- PPPoE ID
- 店舗名や住所が分かるホスト名

公開の場所へそのまま貼らないでください。

必要な部分だけ伏せ字にします。

```txt
SSID: Office_Staff
→ SSID: <STAFF_SSID>

MAC: aa:bb:cc:dd:ee:ff
→ MAC: aa:bb:xx:xx:xx:ff

IPv6 prefix: 2404:xxxx:xxxx:xxxx::/64
→ IPv6 prefix: <IPv6_PREFIX>
```

## 目的別コマンド早見表

## インターネットへ出られない

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
ping -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
nslookup www.google.co.jp
```

## DNSだけ怪しい

```sh
ping -c 4 8.8.8.8
nslookup www.google.co.jp
nslookup www.google.co.jp 127.0.0.1
nslookup www.google.co.jp 8.8.8.8
logread | grep -i dnsmasq | tail -n 100
```

## DHCPが怪しい

```sh
cat /tmp/dhcp.leases
uci show dhcp | grep -E 'interface|start|limit|@host|\.mac|\.ip|\.name'
logread | grep -i dnsmasq | tail -n 100
```

## Wi-Fiが怪しい

```sh
uci show wireless | grep -E 'ssid|encryption|network|disabled'
wifi status
iw dev
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

## ゲストWi-Fiが怪しい

```sh
ifstatus guest
uci show wireless | grep -E 'guest|ssid|network'
uci show dhcp.guest
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'
logread | grep -i dnsmasq | tail -n 80
```

## firewallが怪しい

```sh
uci show firewall
iptables-save 2>/dev/null | head -n 200 || true
ip6tables-save 2>/dev/null | head -n 200 || true
logread | grep -Ei 'firewall|drop|reject' | tail -n 100
```

## Tailscaleが怪しい

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

## WireGuardが怪しい

```sh
wg show 2>/dev/null || true
uci show network | grep -Ei 'wireguard|wg|vpn' || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

## 更新後に追加パッケージを確認したい

```sh
opkg list-installed | grep -Ei 'adblock|banip|wireguard|tailscale|vpn|ipoe|internet|assistant|luci-app' || true
df -h
logread | tail -n 100
```

## やってはいけないこと

SSHに慣れるまでは、次は避けたほうが安全です。

- 意味を理解せずに `uci set` を貼り付ける
- `uci commit` 前に `uci changes` を見ない
- バックアップなしで `/etc/config/network` を編集する
- firewallを複数箇所まとめて変更する
- VLANを有線管理ポートなしで変更する
- `opkg upgrade` を実行する
- 出所不明のスクリプトを `curl | sh` で実行する
- WAN側へSSHを公開する
- ログやバックアップをSNSへそのまま貼る

SSHは強力です。

だからこそ、最初は読み取りだけでも十分に価値があります。

読むだけSSH、かなり強いです。

## SSH運用メモのテンプレート

```txt
SSH運用メモ:

接続先:
  ssh root@192.168.1.1

管理ルール:
  SSHはLAN側のみ
  外出先からはTailscale / WireGuard経由

よく使うコマンド:
  ubus call system board
  ifstatus wan
  ifstatus wan6
  cat /tmp/dhcp.leases
  logread | tail -n 50

変更前バックアップ:
  /root/config-backup-YYYYMMDD-HHMM
  LuCI backup: backup-LN6001-before-change-YYYYMMDD.tar.gz

注意:
  network restartでSSHが切れる
  wifi reloadで無線が切れる
  firewall変更前にバックアップ
  opkg upgradeは使わない
```

こうしたメモを残しておくと、あとから自分が助かります。

未来の自分は、だいたい細かいコマンドを忘れています。

## まとめ

LN6001-JPでSSHを使えるようになると、LuCIだけでは見えにくい状態を確認できます。

ただし、最初からCLIで全部を設定する必要はありません。

まずは、この4つからで十分です。

```sh
ubus call system board
ifstatus wan
cat /tmp/dhcp.leases
logread | tail -n 50
```

慣れてきたら、次を追加します。

```sh
ifstatus wan6
wifi status
uci show firewall
iptables-save 2>/dev/null || true
ip6tables-save 2>/dev/null || true
```

おすすめの使い分けは次です。

```txt
設定変更: LuCI中心
状態確認: SSH
大きな変更前: SSHでバックアップ
トラブル時: SSHで診断ログ
```

SSHは、設定を壊すためのものではありません。

状態を見て、落ち着いて切り分けるための道具です。

まずは読み取りから始めるだけで、OpenWrtベースルーターの運用はかなり見通しが良くなります。

黒い画面は、敵ではありません。

ただ、最初から全力で殴りにいかないだけです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-03.png)

## 次に読むなら

SSH基本コマンドを押さえたら、次の記事も合わせて読むと運用しやすくなります。

- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [パッケージ管理](https://note.com/ikmsan/n/n2bc5e8447e77)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)

トラブル時に何から見るかを整理したい人は、つながらない時の切り分けへ。

バックアップや復元の型を固めたい人は、設定バックアップと復元へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、`ifstatus`、`wifi status`、`logread`、`iptables-save`、`ip6tables-save`、Tailscale / WireGuard関連の出力は、ファームウェアや追加モジュールの更新で変わることがあります。

OpenWrt系のfirewall表示は、バージョンやターゲットによって変わります。LN6001-JPでは `iptables-save`、`ip6tables-save`、iptables互換表示を確認します。

この記事では、まず `uci show firewall`、`iptables-save`、`ip6tables-save` を中心に確認し、必要に応じて `iptables` を確認用として使う方針にしています。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPのSSHで最初に覚えるべきコマンドは？

`ubus call system board`、`ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` の4つからで十分です。

WAN状態、端末のIP取得、直近ログが見えるだけでも、かなり切り分けしやすくなります。

### SSHで設定変更までやるべき？

慣れるまでは、状態確認とバックアップ中心で十分です。

日常的な設定変更はLuCIのほうが安全に進めやすいです。

SSHでは、まず読み取り、ログ確認、バックアップから始めるのがおすすめです。

### SSHで入れない時は何を確認する？

PCがLN6001-JP配下のIPを取れているか、Default GatewayがLN6001-JPか、LuCIへ入れるか、SSHがLAN側で有効かを確認します。

ゲストWi-FiからはSSHへ入れない設計にするのが普通です。

### `uci set` を使ってもいい？

使えますが、慣れるまでは慎重に使います。

実行前に対象設定をバックアップし、`uci changes` で差分を見てから、対象configだけ `uci commit` します。

### firewall確認は `iptables` でいい？

LN6001-JPでは、`uci show firewall`、`iptables-save`、`ip6tables-save` を中心に見るほうが実機に合わせやすいです。

`iptables` は互換表示として見える場合もありますが、それだけに依存しないほうが安全です。

### `network restart` を実行してもいい？

実行できますが、SSHやLuCIが切れる可能性があります。

WAN、LAN、VLAN、bridgeを触った後に必要になることがありますが、有線LANで復旧できる状態にしてから行うのがおすすめです。

### SSHログや診断結果をSNSへ貼っていい？

そのまま貼らないでください。

SSID、MACアドレス、内部IP、IPv6 prefix、VPN情報、PPPoE ID、Wi-Fiパスワードなどが含まれることがあります。

必要な部分だけ伏せ字にして共有してください。

### SSHをWAN側に開けてもいい？

おすすめしません。

SSHはLAN側から使い、外出先から管理したい場合はTailscaleやWireGuardなどのVPN経由にします。

### LuCIとSSHはどちらを使えばいい？

最初は、設定変更はLuCI、状態確認はSSHという使い分けがおすすめです。

SSHに慣れてきたら、UCIで小さな変更や差分確認を行うとよいです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へログインする方法: https://support.linksys.com/kb/article/7038-jp/
- OpenWrt Wiki - ubus: https://openwrt.org/docs/techref/ubus
- OpenWrt Wiki - Logging messages: https://openwrt.org/docs/guide-user/base-system/log.essentials
- OpenWrt Wiki - DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt Wiki - Firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt Wiki - Firewall overview: https://openwrt.org/docs/guide-user/firewall/overview
- OpenWrt Wiki - Opkg package manager: https://openwrt.org/docs/guide-user/additional-software/opkg

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
