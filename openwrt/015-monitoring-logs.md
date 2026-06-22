<!-- mirror-source: articles/015-monitoring-logs.md -->

# ルーターが止まる前に｜小さなオフィス・店舗で見るべきログと状態確認【OpenWrt集中連載015】

小さなオフィスや店舗では、ルーターが止まると普通に業務が止まります。

レジがつながらない。  
予約端末が動かない。  
スタッフ用PCがクラウドへ入れない。  
ゲストWi-Fiだけ不安定。  
防犯カメラは見えるのにVPNで店舗へ入れない。  
スマート決済だけ失敗する。

こういう時に一番困るのは、

```txt
何が壊れているのか分からない
```

ことです。

回線なのか。  
Wi-Fiなのか。  
DNSなのか。  
DHCPなのか。  
ゲストWi-Fiだけなのか。  
VPNだけなのか。  
それとも、昨日変えたfirewall設定なのか。

ここが分からないと、だいたい全部再起動したくなります。

分かります。

でも、いきなり全部再起動する前に、まず見たい場所があります。

```txt
WAN / WAN6
DNS
DHCP
Wi-Fi
直近ログ
```

この5つです。

最初からSNMP、Grafana、Prometheus、外部監視サービスまで作らなくて大丈夫です。

もちろん、そういう監視は便利です。  
でも、その前にまず「いつもの状態」を知るほうが大事です。

いつもWANはどう見えているのか。  
普段のWi-Fi接続台数はどれくらいか。  
NASやプリンターはどのIPを取っているのか。  
ログに普段から出ている警告なのか、今日から出たエラーなのか。

監視の第一歩は、グラフを作ることではありません。

**平常時を知ること** です。

LN6001-JPはOpenWrtベースなので、LuCIの管理画面とSSHの両方で状態確認できます。

この記事では、小さなオフィスや店舗でまず見るべき状態、平常時メモ、障害時の切り分け順、サポートへ共有する時の注意点をまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、本格監視基盤を作ることではありません。

まずは、ここまでできればOKです。

- LuCIでWAN / WAN6 / Wi-Fi / DHCPを確認できる
- SSHで `ifstatus wan`、`logread`、`cat /tmp/dhcp.leases` を使える
- 障害時に「何から見るか」の順番を決められる
- 平常時の状態スナップショットを残せる
- ゲストWi-Fi、Device、VPNなど、ネットワークごとに切り分けられる
- ログを共有する時に、伏せるべき情報が分かる
- 本格監視へ進む前に、最低限の運用メモを作れる

監視というと大げさに聞こえます。

でも最初は、

```txt
いつもの状態をメモしておく
困ったら同じ場所を見る
```

だけで十分です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-01.png)

## 先にざっくり結論

小さなオフィスや店舗では、まず次の5つを見られれば十分です。

1. **WAN / WAN6 がupか**
2. **DNS名前解決ができるか**
3. **DHCPで端末にIPアドレスを配れているか**
4. **Wi-Fi端末がSSIDへ接続できているか**
5. **直近ログにerror / fail / warnが大量に出ていないか**

障害時は、いきなりログを全部読もうとしなくて大丈夫です。

まずはこの順番で見ます。

```txt
全体障害か、1台だけか
  ↓
WAN / WAN6
  ↓
DNS
  ↓
DHCP
  ↓
Wi-Fi
  ↓
firewall / VLAN / VPN
  ↓
直近ログ
```

ログは大事です。

でも、ログだけを見ても迷子になります。

先に、

```txt
どこまで生きているか
```

を見たほうが早いです。

全員ダメならWAN。  
特定SSIDだけならDHCPかfirewall。  
特定端末だけなら端末側。  
VPNだけならVPNとルート承認。  
ゲストWi-Fiだけならguest network。

こうやって順番を決めておくと、障害時にかなり落ち着けます。

## こういう人向けです

この記事は、次のような人向けです。

- 小さなオフィスや店舗で最低限の監視を始めたい
- ルーター障害時に何から見ればよいか決めておきたい
- IPoE、ゲストWi-Fi、VLAN、VPNを入れたあと、状態確認の型を作りたい
- サポートへ相談する時に、何を伝えればよいか整理したい
- 高度な監視基盤の前に、LuCIとSSHでできる確認を覚えたい
- 平常時メモや変更履歴を残して、トラブル対応を速くしたい
- 店舗スタッフや家族に「まずここを見て」と伝えたい

逆に、すでにZabbix、Prometheus、Grafana、外部死活監視まで組んでいる人には、かなり基本寄りです。

でも、小さな環境ではこの基本が一番効きます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-02.png)

## 最初に言葉だけそろえる

監視とログで出てくる言葉を、ざっくり整理します。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-01.png)

最初は、次だけ覚えれば大丈夫です。

```txt
logread = ログを見る
ifstatus = WANやinterfaceの状態を見る
dhcp.leases = 端末に配ったIPを見る
baseline = いつもの状態
```

## 最初に見るべき5項目

監視と言っても、最初から難しく考えなくて大丈夫です。

まずは次の5項目です。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-02.png)

この5つが見えるだけで、

```txt
回線側なのか
DNSなのか
Wi-Fiなのか
端末側なのか
firewallなのか
```

をかなり絞れます。

## 最初から本格監視を入れなくていい理由

SNMP、Prometheus、Grafana、Zabbix、外部監視サービス。

どれも便利です。

ただ、小さなオフィスや店舗では、最初からそこまで作り込む前に、まず手元で状態確認できることが大事です。

いきなり本格監視から始めると、こうなりがちです。

- 監視対象が多すぎて見なくなる
- 何が正常値か分からない
- アラートが多すぎて無視する
- 監視ツール自体の運用が負担になる
- 障害時に結局ルーターへSSHして確認する
- グラフはあるけど、何を直せばいいか分からない

監視ツールは、平常時と異常時の見方が分かってから入れると強いです。

まずはLuCIとSSHで、

```txt
いつもの状態
困った時に見る順番
変更前後の差分
```

を作ります。

そのあとで、必要になったらリモートsyslog、死活監視、グラフ化へ進むほうが無理がありません。

## LuCIでまず見る画面

## Status → Overview

まず見るのはここです。

LuCIへログインしたら、最初にOverviewを見ます。

ここはルーターの健康診断みたいな画面です。

確認ポイントは次です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-03.png)

小さなオフィスや店舗で、

```txt
全員インターネットへ出られない
```

と言われたら、まずOverviewです。

ここでWAN / WAN6が落ちているなら、Wi-Fiや端末側より先に回線を見ます。

## Network → Interfaces

WAN、WAN6、LAN、guest、device、kidsなどの状態を確認する画面です。

確認ポイントは次です。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-04.png)

ゲストWi-Fi、VLAN、VPN、IPoEを入れたあとに通信できない場合は、この画面で対象interfaceがupしているかを見ます。

## Network → Wireless → Associated Stations

Wi-Fi接続中の端末一覧を確認する場所です。

見るポイントは次です。

- 端末がSSIDへ接続できているか
- どのradioへ接続しているか
- signalが弱すぎないか
- TX/RX rateが極端に低くないか
- 端末が2.4GHz / 5GHz / 6GHzのどこにいるか
- MLO有効時に複数radioへ見えているか

「Wi-Fiがつながらない」という相談では、まずAssociated Stationsを見ます。

ここに端末が出ていれば、少なくともWi-Fi接続まではできています。

出ていない場合は、SSID、パスワード、暗号化方式、電波強度、端末側の対応帯域を見ます。

## Network → DHCP and DNS → Active DHCP Leases

現在、DHCPでIPアドレスを割り当てている端末一覧です。

見るポイントは次です。

- 端末名
- MACアドレス
- IPアドレス
- リース期限
- 想定したネットワークのIPアドレスを取っているか

たとえば、ゲストWi-Fi端末が `192.168.1.x` を取っていたら、guestではなくlanへ紐づいている可能性があります。

Deviceネットワークのカメラが `192.168.3.x` を取れているか。  
Kids端末が `192.168.4.x` を取れているか。  
NASやプリンターがDHCP予約通りのIPを取っているか。

ここはかなりよく見ます。

## System → System Log

LuCIからログを確認する画面です。

最初はログを全部理解しようとしなくて大丈夫です。

まずは、次の文字が大量に出ていないかを見ます。

```txt
error
fail
warn
timeout
disconnect
restart
dnsmasq
netifd
odhcp6c
hostapd
firewall
```

ログは「答え」ではなく「手がかり」です。

ログだけ見て判断するより、WAN、DNS、DHCP、Wi-Fi、firewallの状態と合わせて見ます。

## 3分で取る状態スナップショット

障害時に毎回コマンドを思い出すのは大変です。

まずは、次をひとまとまりで保存できるようにしておくと便利です。

```sh
SNAP_DIR="/tmp/ln6001-snapshot-$(date +%Y%m%d-%H%M)"
mkdir -p "$SNAP_DIR"

date > "$SNAP_DIR/date.txt"
ubus call system board > "$SNAP_DIR/system-board.json"
uptime > "$SNAP_DIR/uptime.txt"
free > "$SNAP_DIR/free.txt"
df -h > "$SNAP_DIR/df-h.txt"

ifstatus wan > "$SNAP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$SNAP_DIR/ifstatus-wan6.json" 2>/dev/null || true

ip addr show > "$SNAP_DIR/ip-addr.txt"
ip route show > "$SNAP_DIR/route-v4.txt"
ip -6 route show > "$SNAP_DIR/route-v6.txt"

cat /tmp/dhcp.leases > "$SNAP_DIR/dhcp-leases.txt"
uci show network > "$SNAP_DIR/network.uci.txt"
uci show wireless > "$SNAP_DIR/wireless.uci.txt"
uci show dhcp > "$SNAP_DIR/dhcp.uci.txt"
uci show firewall > "$SNAP_DIR/firewall.uci.txt"

wifi status > "$SNAP_DIR/wifi-status.json" 2>/dev/null || true
iw dev > "$SNAP_DIR/iw-dev.txt" 2>/dev/null || true

logread | tail -n 300 > "$SNAP_DIR/logread-tail.txt"
dmesg | tail -n 100 > "$SNAP_DIR/dmesg-tail.txt"

ls -l "$SNAP_DIR"
```

これは復元用バックアップではありません。

「今どうなっているか」を見るための状態メモです。

障害時にこのスナップショットを取って、平常時のものと見比べると差分が見えます。

## スナップショットに含まれる情報に注意する

このスナップショットには、かなりいろいろな情報が入ります。

たとえば、

- 端末名
- MACアドレス
- SSID
- Wi-Fiパスワード
- IPアドレス
- グローバルIPv6アドレス
- 回線情報
- VPN関連情報
- firewall設定

などです。

サポートや詳しい人へ共有する場合は、必要に応じて伏せ字にします。

特に、次はそのまま公開しないほうが安全です。

- SSID
- Wi-Fiパスワード
- MACアドレス
- グローバルIPv6アドレス
- DDNS名
- VPN設定
- TailscaleやWireGuard関連情報
- PPPoE ID
- 固定IP情報
- 店舗や自宅を特定できるホスト名

## SSHで確認する基本コマンド

ここからは、SSHで使う基本コマンドです。

最初は読み取りだけで十分です。

## システム全体を見る

```sh
echo "### system board"
ubus call system board

echo "### uptime"
uptime

echo "### memory"
free

echo "### storage"
df -h

echo "### processes"
top -bn1 | head -n 30
```

見るポイントです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-05.png)

再起動したつもりがないのにuptimeが短い場合、途中で再起動が起きているかもしれません。

ストレージが詰まっている場合は、パッケージ追加やログ保存で問題が出ることがあります。

## WAN / WAN6を見る

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### IPv4 routes"
ip route show

echo "### IPv6 routes"
ip -6 route show

echo "### WAN related logs"
logread | grep -Ei 'wan|wan6|netifd|dhcp|dhcpv6|odhcp6c|ppp|pppoe|ipoe|map|dslite|ipip' | tail -n 120
```

IPoE環境では、IPv6は生きているけれどIPv4 over IPv6側だけおかしい、ということがあります。

そのため、`wan` と `wan6` は分けて見ます。

ここを混ぜると、

```txt
IPv6は生きているのに、IPv4だけ死んでいる
```

状態を見落としやすくなります。

## DNSを見る

```sh
echo "### DNS test"
nslookup www.google.co.jp

echo "### IPv4 ping"
ping -c 4 8.8.8.8

echo "### hostname ping"
ping -c 4 google.co.jp

echo "### resolver files"
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 80
```

切り分けの見方です。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-06.png)

Adblock、Family DNS、Tailscale、WireGuard、DoH対策を入れている場合は、DNS経路が複雑になりやすいです。

まずは端末がどのDNSを使っているかを確認します。

## DHCPを見る

```sh
echo "### active leases"
cat /tmp/dhcp.leases

echo "### DHCP config summary"
uci show dhcp | grep -E 'interface|start|limit|leasetime|dhcp_option|dns|@host|\\.mac|\\.ip|\\.name'

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 80
```

見るポイントです。

- 端末がIPアドレスを取得しているか
- 想定したネットワークのIP範囲にいるか
- Static Leaseが重複していないか
- DHCP範囲と予約IP範囲が重なっていないか
- dnsmasqが再起動ループしていないか

Wi-Fiにはつながっているのに通信できない時は、DHCPを必ず見ます。

端末が `169.254.x.x` のような自己割り当てIPになっている場合は、DHCPからIPを取れていません。

## Wi-Fiを見る

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

- SSIDが有効か
- SSIDが正しいnetworkへ紐づいているか
- 端末がradioへ接続しているか
- signalが極端に弱くないか
- 2.4GHz / 5GHz / 6GHzのどれへつながっているか
- DFSやradio再起動が起きていないか

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

ログを見るのは状態確認のためです。

## firewallを見る

```sh
echo "### firewall config"
uci show firewall

echo "### firewall zones and forwarding"
uci show firewall | grep -E 'zone|forwarding|name|network|src|dest|target|input|output|forward'

echo "### iptables rules"
iptables-save 2>/dev/null | head -n 160 || true

echo "### ip6tables rules"
ip6tables-save 2>/dev/null | head -n 160 || true

echo "### firewall logs"
logread | grep -Ei 'firewall|DROP|REJECT' | tail -n 100
```

firewallは、ゲストWi-Fi、VLAN、VPNを触ったあとに重要になります。

よくある確認ポイントは次です。

- guest → wan が許可されているか
- guest → lan を許可していないか
- guest / device / kids のInputを `REJECT` にした場合、DHCP/DNSを許可しているか
- VPNからlanへ広く許可しすぎていないか
- deviceからlanへ勝手に入れる構成になっていないか

## VPNを見る

WireGuardやTailscaleを使っている場合は、VPNも確認します。

### WireGuard

```sh
echo "### WireGuard"
wg show 2>/dev/null || echo "wg command is not available"

echo "### WG interfaces"
ip addr show | grep -A5 -Ei 'wg|wireguard' || true

echo "### WG firewall hints"
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true

echo "### WG logs"
logread | grep -Ei 'wireguard|wg|51820' | tail -n 100
```

見るポイントです。

- latest handshakeが更新されているか
- peerが見えているか
- firewallでUDPポートが許可されているか
- AllowedIPsの範囲が広すぎないか

### Tailscale

```sh
echo "### Tailscale status"
tailscale status 2>/dev/null || echo "tailscale command is not available"

echo "### Tailscale IP"
tailscale ip 2>/dev/null || true

echo "### Tailscale prefs"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route' || true

echo "### Tailscale logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

見るポイントです。

- LN6001-JPがTailnetに参加しているか
- サブネットルートを広告しているか
- Tailscale管理コンソールでルート承認済みか
- Exit Nodeを意図せず有効にしていないか

## 障害時の切り分け手順

## ステップ1: 全体障害か、1台だけかを分ける

最初に、影響範囲を見ます。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-07.png)

ここを最初に分けるだけで、かなり楽になります。

全体障害なのに端末設定を見ても遠回りです。  
1台だけの問題なのにWANを疑っても遠回りです。

まず影響範囲です。

## ステップ2: WAN / WAN6を確認する

```sh
ifstatus wan | grep -E '"up"|"ipaddr"|"error"|"pending"'
ifstatus wan6 | grep -E '"up"|"ip6addr"|"error"|"pending"'
ip route show default
ip -6 route show default
```

WAN / WAN6が落ちている場合は、LANやWi-Fiより先に回線側を見ます。

- ONU / HGWの電源
- WANケーブル
- IPoE / PPPoE設定
- オートIPoEモジュール
- 上流ルーター
- プロバイダ障害

IPoE環境では、`wan6` はupでも、IPv4 over IPv6側だけおかしいことがあります。

IPv4とIPv6は分けて見ます。

## ステップ3: DNSを確認する

```sh
ping -c 4 8.8.8.8
ping -c 4 google.co.jp
nslookup www.google.co.jp
logread | grep -i dnsmasq | tail -n 80
```

判断の目安です。

```txt
8.8.8.8へのping OK
google.co.jpへのping NG
→ DNS問題の可能性

8.8.8.8へのping NG
google.co.jpへのping NG
→ WAN、経路、firewall、回線側の可能性
```

AdblockやFamily DNSを入れている場合は、一時停止して切り分けることもあります。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

確認後、必要なら戻します。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

## ステップ4: DHCPを確認する

```sh
cat /tmp/dhcp.leases
logread | grep -i dnsmasq | tail -n 80
uci show dhcp | grep -E 'interface|start|limit|@host|dhcp_option'
```

確認することです。

- 対象端末がリース一覧にいるか
- 想定したIP範囲を取っているか
- `169.254.x.x` のような自己割り当てになっていないか
- DHCP予約が重複していないか
- DHCPサーバーが対象interfaceで有効か

Wi-Fi接続はできているのにIPアドレスが取れない場合は、DHCPやfirewallのInput設定を見ます。

## ステップ5: Wi-Fiを確認する

```sh
wifi status
iw dev
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

確認することです。

- 対象SSIDが有効か
- 対象端末がAssociated Stationsにいるか
- 2.4GHz / 5GHz / 6GHzのどれが不安定か
- 暗号化方式を変えた直後ではないか
- MLO設定を有効化してから不安定になっていないか

特定端末だけつながらない場合は、端末側のWi-Fi設定削除・再接続も試します。

## ステップ6: firewall / VLAN / VPNを確認する

設定変更直後に通信できなくなった場合は、firewallやVLANが原因のことがあります。

```sh
uci show firewall | grep -E 'zone|forwarding|guest|device|kids|vpn|wireguard|tailscale'
uci show network | grep -E 'interface|device|bridge|vlan|ports'
uci show wireless | grep -E 'ssid|network'
```

よくある失敗です。

- ゲストWi-FiのSSIDが `guest` ではなく `lan` に紐づいている
- guest zoneのInputを `REJECT` にしたが、DHCP/DNSを許可していない
- DeviceからWANへ出る必要があるのにforwardingがない
- VLAN変更で管理端末が別ネットワークへ飛ばされた
- VPNからLANへ届くルールがない
- VPNからLANへ広く入りすぎている

設定変更直後のトラブルは、直前に触った場所を疑います。

これはかなり効きます。

## ステップ7: 直近ログを見る

```sh
logread | tail -n 150
logread | grep -Ei 'err|warn|fail|timeout|restart|disconnect' | tail -n 100
```

ログは最後にまとめて見るより、ここまでの切り分け結果と合わせて見ます。

DNSが怪しいならdnsmasqログ。  
WANが怪しいならnetifdやodhcp6c。  
Wi-Fiが怪しいならhostapdやwireless関連。

ログを全部読むのではなく、疑っている場所に関係するログを見ると楽です。

## 平常時スナップショットを残す

障害時に比較できるよう、平常時の状態を残しておきます。

小さなオフィスや店舗では、この平常時メモがかなり効きます。

## 記録しておく項目

最低限、次を残します。

```txt
製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

LAN IP:
  192.168.1.1

回線:
  IPoE / PPPoE / HGW配下 / ONU直下

IPv4 over IPv6:
  OCNバーチャルコネクト / transix / v6プラス / クロスパス / 不明

主要ネットワーク:
  lan: 192.168.1.0/24
  guest: 192.168.2.0/24
  device: 192.168.3.0/24

主要機器:
  NAS: 192.168.1.10
  プリンター: 192.168.1.20
  録画機: 192.168.3.20

通常のWi-Fi接続台数:
  平日昼: 約15台
  営業終了後: 約5台

VPN:
  Tailscale / WireGuard / 未使用

最終バックアップ:
  backup-LN6001-initial-20260621.tar.gz
```

この程度でも、障害時の比較にはかなり使えます。

## 平常時コマンド

平常時に次の出力を保存しておくと便利です。

```sh
BASE_DIR="/root/baseline-$(date +%Y%m%d-%H%M)"
mkdir -p "$BASE_DIR"

date > "$BASE_DIR/date.txt"
ubus call system board > "$BASE_DIR/system-board.json"
uptime > "$BASE_DIR/uptime.txt"
ifstatus wan > "$BASE_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BASE_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BASE_DIR/ip-addr.txt"
ip route show > "$BASE_DIR/route-v4.txt"
ip -6 route show > "$BASE_DIR/route-v6.txt"
cat /tmp/dhcp.leases > "$BASE_DIR/dhcp-leases.txt"
uci show network > "$BASE_DIR/network.uci.txt"
uci show wireless > "$BASE_DIR/wireless.uci.txt"
uci show dhcp > "$BASE_DIR/dhcp.uci.txt"
uci show firewall > "$BASE_DIR/firewall.uci.txt"
logread | tail -n 300 > "$BASE_DIR/logread-tail.txt"

ls -l "$BASE_DIR"
```

これは復元用バックアップではなく、比較用の状態メモです。

復元用バックアップは、LuCIの **System** → **Backup / Flash Firmware** → **Generate archive** からPCへ保存します。

## 変更履歴を残す

障害対応でかなり効くのが、変更履歴です。

設定変更後に問題が起きた場合、「最後に何を変えたか」が分かれば原因を絞りやすくなります。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-08.png)

設定変更前に、軽い控えを残します。

```sh
CHANGE_DIR="/root/change-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$CHANGE_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$CHANGE_DIR/$cfg"
  uci show "$cfg" > "$CHANGE_DIR/$cfg.uci.txt"
done

opkg list-installed > "$CHANGE_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$CHANGE_DIR/logread-tail.txt"

ls -l "$CHANGE_DIR"
```

最初は、Markdownに1行だけでも十分です。

```txt
2026-06-21 14:30 Guest Wi-Fi追加。guest=192.168.2.0/24。変更前バックアップあり。
```

障害時に「昨日何か変えたっけ？」となるのは、かなりよくあります。

変更履歴は、未来の自分へのメモです。

## ログ共有時に伏せる情報

トラブル時に、詳しい人やサポートへログを送ることがあります。

ただし、ログや設定には個人情報やネットワーク情報が含まれます。

共有前に伏せたいものです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-09.png)

伏せ字にする例です。

```txt
SSID: Office_Staff
→ SSID: <STAFF_SSID>

MAC: aa:bb:cc:dd:ee:ff
→ MAC: aa:bb:xx:xx:xx:ff

IPv6: 2404:xxxx:xxxx:xxxx::1
→ IPv6: 2404:xxxx:xxxx:****::1

PPPoE ID: user@example.ne.jp
→ PPPoE ID: <PPPOE_ID>
```

ログを共有する前に、いったんテキストエディタで検索して伏せるのがおすすめです。

## 日常点検の頻度

小さなオフィスや店舗では、毎日ログを全部読む必要はありません。

次くらいの運用で十分です。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-10.png)

ポイントは、「異常が出た時だけ見る」のではなく、「正常時を少し知っておく」ことです。

いつもの接続台数。  
いつものWAN状態。  
いつものログの雰囲気。  
いつものIPアドレス。

この“いつも”があると、異常が見えます。

## もう一歩進んだ監視

LuCIとSSHで状態確認できるようになったら、必要に応じて次へ進みます。

## リモートsyslog

OpenWrtのログは、基本的にはルーター内で確認します。

障害発生後に再起動してしまうと、過去ログが十分に残っていない場合があります。

小さなオフィスや店舗でログを残したい場合は、リモートsyslogを検討します。

ただし、最初から必須ではありません。

まずはLuCIとSSHでログを見られるようにする。

その後に、

```txt
再起動前のログも残したい
```

と感じたら導入を検討します。

## 死活監視

外部サービスや別拠点から、次のような監視を行う方法もあります。

- インターネット疎通
- VPN疎通
- NASやカメラの応答
- DNS名前解決
- 店舗の回線停止検知

ただし、監視対象を増やしすぎると運用が大変です。

まずは、業務影響の大きいものから始めます。

例:

```txt
店舗:
  - インターネット疎通
  - POS / 予約端末の疎通
  - VPN疎通
  - 録画機の疎通

家庭:
  - NAS
  - インターネット
  - VPN
```

## SNMPやGrafanaは後でいい

SNMP、Prometheus、Grafana、Zabbixなどは便利です。

ただし、LN6001-JP側へパッケージを追加して監視を作り込む場合は、ストレージ、メモリ、CPU負荷、運用負荷も考える必要があります。

最初は次で十分です。

- LuCI Overviewを見る
- `logread` を見る
- 平常時スナップショットを残す
- 設定変更履歴を残す
- 重要機器のIPアドレスをDHCP予約する

本格監視は、そのあとで必要になったら検討します。

## サポートへ伝える情報

Linksysサポートや詳しい人に相談する時は、次を整理すると話が早くなります。

```txt
製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

回線:
  NTT / auひかり / NURO / J:COM / その他

プロバイダ:
  OCN / IIJ / BIGLOBE / その他

接続方式:
  IPoE / PPPoE / HGW配下 / ONU直下 / 不明

問題:
  全端末でインターネット不可
  Wi-Fiだけ不可
  ゲストWi-Fiだけ不可
  VPNだけ不可
  特定端末だけ不可

発生時刻:
  2026-06-21 14:30頃

直前の変更:
  ゲストWi-Fi追加
  Adblock導入
  VPN Assistant導入
  ファームウェア更新
  変更なし

確認済み:
  有線でも発生 / Wi-Fiだけ
  ifstatus wan
  ifstatus wan6
  logread tail
```

ログや設定を送る場合は、前述の伏せ字を忘れないようにしてください。

## よくある問題と見る場所

## 全端末でインターネットにつながらない

まずWAN/WAN6を見ます。

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
logread | grep -Ei 'wan|wan6|netifd|ppp|ipoe|odhcp6c' | tail -n 120
```

確認することです。

- ONU / HGWが生きているか
- WANケーブルが抜けていないか
- IPoE / PPPoE設定が変わっていないか
- WAN/WAN6がupか
- default routeがあるか

全端末でダメなら、まず回線側です。

Wi-Fi設定を触る前に、WANを見ます。

## Wi-Fiだけつながらない

```sh
wifi status
iw dev
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

確認することです。

- SSIDが有効か
- 端末がAssociated Stationsに出ているか
- 2.4GHz / 5GHz / 6GHzのどれが不安定か
- 暗号化方式を変えた直後ではないか
- MLO設定を変えた直後ではないか

有線はOKでWi-Fiだけダメなら、回線よりWi-Fiを見ます。

## ゲストWi-Fiだけつながらない

```sh
ifstatus guest
uci show wireless | grep -E 'guest|ssid|network'
uci show dhcp.guest
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'
logread | grep -i dnsmasq | tail -n 80
```

確認することです。

- ゲストSSIDが `guest` networkに紐づいているか
- DHCP Serverが有効か
- `Allow-Guest-DHCP` があるか
- `Allow-Guest-DNS` があるか
- guest → wan forwardingがあるか
- guest → lanを許可していないか

ゲストWi-Fiは、SSIDだけ作っても完成しません。

network、DHCP、firewall zoneまで見ます。

## カメラやIoTだけ不安定

```sh
ifstatus device 2>/dev/null || true
cat /tmp/dhcp.leases
uci show wireless | grep -E 'device|iot|ssid|network'
logread | grep -Ei 'dnsmasq|wireless|hostapd|device|iot' | tail -n 120
```

確認することです。

- Device / IoTネットワークでIPを取れているか
- 2.4GHzに接続しているか
- DNS広告ブロックが強すぎないか
- WANへ出る必要がある機器なのにdevice → wanを閉じていないか
- アプリ初期設定時だけ同じLANが必要ではないか

IoT機器は、クラウド接続やアプリの探索が絡むことがあります。

最初から強く閉じすぎると、動いているようで動いていない状態になりがちです。

## VPNだけつながらない

WireGuardなら次を見ます。

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn'
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

Tailscaleなら次を見ます。

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route' || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

確認することです。

- VPNサービスが起動しているか
- 外からポートが届いているか
- Tailscale管理コンソールでサブネットルートが承認されているか
- VPNからLANへのfirewallルールがあるか
- AllowedIPsやサブネットルートが正しいか

VPNは「接続できた」だけでは不十分です。

接続後に、どこへ入れるのかまで確認します。

## DNS広告ブロック後に一部サイトが壊れた

```sh
/etc/init.d/adblock status 2>/dev/null || true
logread | grep -Ei 'adblock|dnsmasq' | tail -n 120
```

一時的にAdblockを止めて確認します。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

問題が解消するなら、Adblockやブロックリストが原因の可能性があります。

必要なドメインをAllowlistへ追加するか、ブロックリストを減らします。

確認後、必要なら戻します。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

## まとめ

小さなオフィスや店舗では、最初から本格監視を作る必要はありません。

まずは、次の状態を見られるようにします。

- WAN / WAN6
- DNS
- DHCP
- Wi-Fi
- firewall zone
- VPN
- 直近ログ

障害時は、この順番で切り分けます。

```txt
全体障害か1台だけか
→ WAN / WAN6
→ DNS
→ DHCP
→ Wi-Fi
→ firewall / VLAN / VPN
→ ログ
```

監視の第一歩は、平常時を知ることです。

いつものWAN状態。  
いつものWi-Fi接続台数。  
主要機器のIPアドレス。  
最後に変更した設定。  
直近のバックアップ。

これがあるだけで、障害時の対応はかなり速くなります。

LN6001-JPはLuCIとSSHの両方で状態を確認できます。

最初はログを全部読む必要はありません。

見る場所と順番を決めておく。

それだけで、小さなオフィスや店舗のネットワーク運用はかなり落ち着きます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-03.png)

## 次に読むなら

監視とログの基本を押さえたら、次は目的に合わせて進みます。

- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [ファームウェア更新運用](https://note.com/ikmsan/n/nff6b598da354)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)

障害対応の流れをさらに整理したい人は、つながらない時の切り分けへ。

設定変更前後の戻し方を固めたい人は、設定バックアップと復元の記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、ログ出力、`ifstatus`、`wifi status`、`iptables-save`、`ip6tables-save`、Tailscale / WireGuard関連の出力は、ファームウェアや追加モジュールの更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### 小さなオフィスで最初に監視すべきものは？

まずはWAN/WAN6、DNS、DHCP、Wi-Fi、直近ログです。

これだけで、回線側なのか、DNSなのか、Wi-Fiなのか、端末側なのかをかなり切り分けられます。

### 毎日ログを全部見る必要はある？

ありません。

普段はLuCIのOverviewでWAN、Wi-Fi、メモリ、接続台数を見るくらいで十分です。

障害時に `logread | tail -n 100` や関連ログを見られるようにしておくことが大事です。

### 障害時は何から見ればいい？

まず全体障害か1台だけかを分けます。

その後、WAN/WAN6、DNS、DHCP、Wi-Fi、firewall、ログの順で見ると切り分けしやすくなります。

### 平常時メモには何を残せばいい？

LAN IP、回線方式、WAN/WAN6の状態、主要機器のIPアドレス、SSID、通常の接続台数、VPNの有無、最後のバックアップ名を残しておくと役立ちます。

### logreadの出力をそのまま共有していい？

そのまま共有しないほうが安全です。

SSID、MACアドレス、グローバルIPv6アドレス、DDNS名、VPN情報、PPPoE ID、店舗名や自宅名が分かるホスト名などを伏せてから共有してください。

### 本格監視はいつ入れるべき？

LuCIとSSHで平常時と障害時の確認ができるようになってからで十分です。

そのうえで、再起動前のログを残したいならリモートsyslog、複数拠点を見たいなら外部監視、グラフ化したいならSNMPやGrafanaを検討します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/
- OpenWrt Wiki - Logging messages: https://openwrt.org/docs/guide-user/base-system/log.essentials
- OpenWrt Wiki - ubus system: https://openwrt.org/docs/guide-developer/ubus/system
- OpenWrt Wiki - System configuration: https://openwrt.org/docs/guide-user/base-system/system_configuration
- OpenWrt Wiki - DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt Wiki - Firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration

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
