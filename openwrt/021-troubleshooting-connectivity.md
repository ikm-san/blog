<!-- mirror-source: articles/021-troubleshooting-connectivity.md -->

# ネットが死んだ、の前に｜WAN・DNS・Wi-Fiを3分で切り分ける【OpenWrt集中連載021】

「Wi-Fiにはつながっているのに、Webサイトが開けない」

これ、かなり嫌な症状です。

スマホのWi-Fiマークは出ている。  
PCもSSIDにはつながっている。  
でもGoogleが開かない。  
YouTubeも重い。  
仕事のチャットもつながらない。

こうなると、つい言いたくなります。

```txt
ネットが死んだ
```

分かります。

でも、ここで全部まとめて「ネットが死んだ」と考えると、かえって遠回りになります。

実際には、止まっている場所はいろいろあります。

WANかもしれない。  
DNSかもしれない。  
DHCPかもしれない。  
Wi-Fiだけかもしれない。  
firewallかもしれない。  
IPv6だけかもしれない。  
AdblockやFamily DNSが効きすぎているだけかもしれない。

ネットワークのトラブルは、見た目はだいたい全部「つながらない」です。

でも、切り分けるとかなり落ち着いて対応できます。

LN6001-JPはOpenWrtベースなので、LuCIの管理画面とSSHの読み取りコマンドを組み合わせると、勘で再起動する前に状態を見られます。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で接続トラブルが起きた時に、最初に見る順番、3分で取る診断ログ、症状別の切り分け、設定変更後の安全な戻し方を整理します。

目標は、いきなり直すことではありません。

まず、

```txt
どこまでは生きていて
どこから止まっているのか
```

を分けます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、すべてのトラブルを一発で直すことではありません。

まずは、ここまでできればOKです。

- 「つながらない」をWAN、DNS、DHCP、Wi-Fi、firewallへ分けて見られる
- LN6001-JPで最初に確認したいLuCI画面が分かる
- SSHで読み取り中心の診断コマンドを実行できる
- IPv4 / IPv6 / DNSを分けて確認できる
- ゲストWi-Fi、VLAN、VPN、Adblock変更後の切り分けができる
- 設定変更後にどこから戻すべきか分かる
- サポートや詳しい人へ相談する時に伏せるべき情報が分かる

トラブル対応で大事なのは、いきなり直そうとしないことです。

まず見る。  
分ける。  
1か所だけ直す。  
もう一度確認する。

この順番です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-01.png)

## 先にざっくり結論

トラブル時は、最初から全部のログを読まなくて大丈夫です。

まずは次の順番で見ます。

```txt
物理・LED・ONU/HGW
  ↓
有線でLuCIへ入れるか
  ↓
WAN / WAN6
  ↓
IP直打ち疎通
  ↓
DNS名前解決
  ↓
DHCP
  ↓
Wi-Fi
  ↓
firewall / VLAN / VPN / Adblock
  ↓
直近ログ
```

最初の目標は、「直すこと」ではなく、**どこで止まっているかを分けること**です。

見るべきコマンドを最小にすると、まずはこの5つです。

```sh
ifstatus wan
ifstatus wan6
ping -c 4 8.8.8.8
nslookup www.google.co.jp
cat /tmp/dhcp.leases
```

この5つで、かなりのことが分かります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-01.png)

「全部壊れた」と考えるより、「どこまでは正常か」を見るほうが早いです。

全員ダメならWAN。  
特定SSIDだけならDHCPかfirewall。  
特定端末だけなら端末側。  
VPNだけならVPNとルート。  
ゲストWi-Fiだけならguest network。

こう分けると、かなり落ち着けます。

## こういう時に向いています

この記事は、次のような時に役立ちます。

- Wi-Fiにはつながるのにインターネットが使えない
- 有線は使えるがWi-Fiだけ不安定
- ゲストWi-Fiだけつながらない
- VLANやfirewallを触ったあと通信できなくなった
- AdblockやFamily DNSを入れたあと、一部サイトだけ壊れた
- IPoE環境でIPv4だけ、またはIPv6だけ不安定
- WireGuard / Tailscaleだけつながらない
- ファームウェア更新後に、どこから確認すればよいか分からない
- とりあえず再起動したいけれど、原因も残しておきたい

逆に、すでに高度な監視基盤やログ収集を組んでいる人には、かなり基本寄りです。

でも、家庭や小さなオフィス、店舗では、この基本が一番効きます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-02.png)

## 最初に言葉だけそろえる

トラブル対応で出てくる言葉を、ざっくり整理しておきます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-02.png)

最初は、次だけ覚えれば大丈夫です。

```txt
WAN = 回線側
DNS = 名前解決
DHCP = IP配布
Wi-Fi = 無線接続
firewall = 通す/止める境界
```

## まずやらないほうがいいこと

トラブル時は焦ります。

でも、次をいきなりやると、原因が分かりにくくなります。

- 設定を何か所も同時に変更する
- 何度も再起動する
- バックアップなしでfirewallやnetworkを書き換える
- WAN、LAN、DNS、Wi-Fiを同時に触る
- ログを見ずにファームウェア更新する
- リセットする前に現在状態を控えない
- `uci set` コマンドを意味が分からないまま貼り付ける
- 「たぶんDNS」と決めつけてDNSだけ変える
- 「たぶんWi-Fi」と決めつけてSSIDや暗号化方式を変える

まずは読み取りです。

状態を見てから直します。

```txt
見る
  ↓
分ける
  ↓
1か所だけ直す
  ↓
確認する
```

この順番が大事です。

ここで焦ってリセットボタンを押したくなります。

分かります。

でも、まだ押さなくて大丈夫です。

## 3分で集める診断ログ

原因が分からない時は、設定を直し始める前に、まず状態を保存します。

以下は読み取り中心の診断セットです。

```sh
DIAG_DIR="/tmp/ln6001-diag-$(date +%Y%m%d-%H%M)"
mkdir -p "$DIAG_DIR"

echo "### system"
date > "$DIAG_DIR/date.txt"
ubus call system board > "$DIAG_DIR/system-board.json" 2>/dev/null || true
uptime > "$DIAG_DIR/uptime.txt"
free > "$DIAG_DIR/free.txt"
df -h > "$DIAG_DIR/df-h.txt"

echo "### interfaces"
ifstatus wan > "$DIAG_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$DIAG_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$DIAG_DIR/ip-addr.txt"
ip route show > "$DIAG_DIR/route-v4.txt"
ip -6 route show > "$DIAG_DIR/route-v6.txt"

echo "### dns-dhcp"
cat /tmp/resolv.conf.d/resolv.conf.auto > "$DIAG_DIR/resolv-auto.txt" 2>/dev/null || true
cat /etc/resolv.conf > "$DIAG_DIR/resolv-conf.txt" 2>/dev/null || true
cat /tmp/dhcp.leases > "$DIAG_DIR/dhcp-leases.txt" 2>/dev/null || true
uci show dhcp > "$DIAG_DIR/dhcp.uci.txt" 2>/dev/null || true

echo "### wifi"
uci show wireless > "$DIAG_DIR/wireless.uci.txt" 2>/dev/null || true
wifi status > "$DIAG_DIR/wifi-status.json" 2>/dev/null || true
iw dev > "$DIAG_DIR/iw-dev.txt" 2>/dev/null || true

echo "### firewall"
uci show firewall > "$DIAG_DIR/firewall.uci.txt" 2>/dev/null || true
iptables-save > "$DIAG_DIR/iptables-save.txt" 2>/dev/null || true
ip6tables-save > "$DIAG_DIR/ip6tables-save.txt" 2>/dev/null || true

echo "### logs"
logread | tail -n 300 > "$DIAG_DIR/logread-tail.txt" 2>/dev/null || true
dmesg | tail -n 120 > "$DIAG_DIR/dmesg-tail.txt" 2>/dev/null || true

echo "### done"
ls -l "$DIAG_DIR"
```

PCへコピーする場合は、PC側から次を実行します。

```sh
scp -r root@192.168.1.1:/tmp/ln6001-diag-* ./
```

LAN IPを変えている場合は、IPを置き換えます。

```sh
scp -r root@192.168.10.1:/tmp/ln6001-diag-* ./
```

これは復元用バックアップではありません。

今この瞬間の状態を残す診断メモです。

「再起動したら直ったけど、原因が分からない」を少し減らせます。

## 共有前に伏せる情報

診断ログには、かなりいろいろな情報が入ります。

公開コメント欄、SNS、GitHub issueなどへそのまま貼らないでください。

伏せたい情報は次です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-03.png)

伏せ字例です。

```txt
SSID: Office_Staff
→ SSID: <STAFF_SSID>

MAC: aa:bb:cc:dd:ee:ff
→ MAC: aa:bb:xx:xx:xx:ff

IPv6: 2404:xxxx:xxxx:xxxx::1
→ IPv6: 2404:xxxx:xxxx:****::1
```

相談する時は、情報を消しすぎても切り分けしづらくなります。

ただ、パスワード、秘密鍵、グローバルIPv6 prefix、DDNS名、店舗名が分かる情報は慎重に扱います。

## 症状を分類する

最初に、症状を分類します。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-04.png)

最初に見るべきなのは、「全端末か、一部端末か」です。

全端末ならWAN側を疑います。

一部端末なら、その端末がいるSSID、DHCP、DNS、firewall zoneを見ます。

ここを分けるだけで、かなり時間を節約できます。

## ステップ1: 物理・LED・ONU/HGWを確認する

最初に、物理を見ます。

```txt
電源
  ↓
LED
  ↓
WANケーブル
  ↓
ONU / HGW
  ↓
上流回線
```

LN6001-JP / MBE70のLEDは、ざっくり次のように見ます。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-05.png)

確認することです。

- 電源アダプターが抜けていないか
- 本体が起動中ではないか
- WANケーブルがInternet / WANポートに入っているか
- ONU / HGW側のLANポートが有効か
- ONU / HGWのLEDが正常か
- ONU / HGWを再起動した直後ではないか
- 回線工事や障害情報がないか

意外と多いのが、設定ではなくケーブルやONU / HGW側の問題です。

特に赤点灯の場合、まずWANケーブルとONU / HGW側を見ます。

ここでWi-Fi設定を触るのはまだ早いです。

## ステップ2: 有線でLuCIへ入れるか確認する

Wi-Fiが怪しい時でも、まず有線LANでLuCIへ入れるか確認します。

PCをLN6001-JPのLANポートへ接続し、ブラウザで開きます。

```txt
https://192.168.1.1
```

LAN IPを変更している場合は、そのIPで開きます。

例:

```txt
https://192.168.10.1
```

確認します。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-06.png)

PC側でIPアドレスを確認します。

Windows:

```powershell
ipconfig
```

macOS / Linux:

```sh
ip addr show
ip route show default
```

PCが `169.254.x.x` のようなIPになっている場合、DHCPでIPを取れていません。

その場合は、DHCPやLAN側接続を見ます。

「Wi-Fiが悪い」と思っていたら、実はPCがIPを取れていないだけ、ということもあります。

## ステップ3: WAN / WAN6の接続状態を確認する

LuCIでは、**Status** → **Overview** を見ます。

確認するものです。

- IPv4 WAN Status
- IPv6 WAN Status
- WAN側IPアドレス
- WAN6側IPv6アドレス
- 接続時間
- エラー表示

SSHでは次を見ます。

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### default routes"
ip route show default
ip -6 route show default

echo "### WAN logs"
logread | grep -Ei 'wan|wan6|netifd|dhcp|dhcpv6|odhcp6c|ppp|pppoe|ipoe|map|mape|dslite|ipip' | tail -n 120
```

IPoE環境では、`wan` と `wan6` を分けて見ます。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-07.png)

HGW配下の場合、WAN側IPv4が `192.168.x.x` になることがあります。

これは必ずしも異常ではありません。

ただし、HGWとLN6001-JPのLAN IPが重複している場合は問題になります。

## ステップ4: IP直打ちとDNSを分ける

次に、IPアドレスへ直接届くか、名前解決だけが失敗しているかを分けます。

```sh
echo "### IPv4 direct"
ping -c 4 8.8.8.8

echo "### IPv6 direct"
ping6 -c 4 2001:4860:4860::8888

echo "### DNS"
nslookup www.google.co.jp

echo "### hostname ping"
ping -c 4 www.google.co.jp
```

判断の目安です。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-08.png)

DNS問題は、「インターネットが全部落ちた」と誤解しやすいです。

IP直打ちで通るかどうかを先に見ると、切り分けが一気に楽になります。

## ステップ5: DNSを詳しく見る

IP直打ちでは通るのにドメイン名で失敗する場合は、DNSを見ます。

```sh
echo "### resolver files"
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf

echo "### dnsmasq status"
if [ -x /etc/init.d/dnsmasq ]; then
  /etc/init.d/dnsmasq status
fi

echo "### dns tests"
nslookup www.google.co.jp 127.0.0.1
nslookup www.google.co.jp 8.8.8.8

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 100
```

見るポイントです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-09.png)

Adblockを使っている場合は、一時停止して切り分けます。

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

DNSは、Adblock、Family DNS、DoH、VPN、Tailscale DNSなどが絡むと一気に複雑になります。

まずは「ルーター自身のDNSが動いているか」から見ます。

## ステップ6: DHCPとIPアドレスを確認する

Wi-Fiにはつながるのに通信できない時は、端末が正しいIPを取れているか見ます。

```sh
echo "### active DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP config summary"
uci show dhcp | grep -E 'interface|start|limit|leasetime|dhcp_option|dns|ignore|@host|\\.mac|\\.ip|\\.name'

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 100
```

見るポイントです。

- 対象端末が `/tmp/dhcp.leases` に出ているか
- 端末が想定したIP範囲を取っているか
- ゲストWi-Fi端末が `192.168.2.x` になっているか
- Device / IoT端末が想定ネットワークにいるか
- Static Leaseが重複していないか
- DHCP Serverが対象interfaceで有効か

端末側で `169.254.x.x` のようなIPになっている場合、DHCP取得に失敗しています。

ゲストWi-FiやVLANでInputを `REJECT` にしている場合、DHCP許可ルールがないとIPを取れません。

例:

```txt
Allow-Guest-DHCP
Allow-Device-DHCP
Allow-Kids-DHCP
```

Wi-Fiマークが出ていても、IPを取れていないなら通信はできません。

ここ、かなりよくあります。

## ステップ7: Wi-Fiを確認する

有線は正常でWi-Fiだけおかしい場合は、Wi-Fi状態を見ます。

LuCIでは、**Network** → **Wireless** と **Associated Stations** を確認します。

CLIでは次です。

```sh
echo "### wireless config"
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

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-10.png)

6GHzが見えない場合は、端末がWi-Fi 6E / 7に対応しているか確認します。

IoT機器がつながらない場合は、2.4GHz、WPA2-PSK、英数字SSIDなど、互換性重視で確認します。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

ログを見るのは状態確認のためです。

## ステップ8: firewall / VLAN / ゲストWi-Fiを確認する

ゲストWi-Fi、VLAN、IoT、VPNを触ったあとにつながらない場合は、firewallとnetwork紐づけを見ます。

```sh
echo "### network mapping"
uci show network | grep -E 'interface|device|proto|ipaddr|guest|device|iot|kids|vpn'

echo "### wireless mapping"
uci show wireless | grep -E 'ssid|network|guest|device|iot|kids'

echo "### firewall zones and rules"
uci show firewall | grep -E 'zone|forwarding|name|network|src|dest|target|input|output|forward|guest|device|iot|kids|vpn|Allow-'

echo "### iptables / ip6tables"
iptables-save 2>/dev/null | grep -Ei 'guest|device|iot|kids|vpn|drop|reject|accept' -A3 -B3 | head -n 200 || true
ip6tables-save 2>/dev/null | grep -Ei 'guest|device|iot|kids|vpn|drop|reject|accept' -A3 -B3 | head -n 200 || true
```

よくある失敗です。

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-11.png)

LN6001-JPでは、まず `uci show firewall`、必要に応じて `iptables-save`、`ip6tables-save` を見ます。

古い記事の `iptables -L -n -v` だけに依存しないほうが安全です。

互換表示として確認する場合は、次のように失敗しても止まらない形にします。

```sh
iptables -L -n -v 2>/dev/null | head -n 80 || true
iptables-save 2>/dev/null | head -n 120 || true
```

## ステップ9: VPNを確認する

TailscaleやWireGuardだけがつながらない場合は、VPN側を見ます。

## Tailscale

```sh
echo "### Tailscale status"
tailscale status 2>/dev/null || true

echo "### Tailscale IP"
tailscale ip 2>/dev/null || true

echo "### Tailscale prefs"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true

echo "### Tailscale logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

見るポイントです。

- LN6001-JPがtailnetに参加しているか
- Tailscale IPがあるか
- サブネットルートを広告しているか
- 管理コンソールでルート承認済みか
- Exit Nodeを意図せず使っていないか
- クライアント側でTailscaleが有効か

## WireGuard

```sh
echo "### WireGuard status"
wg show 2>/dev/null || true

echo "### WireGuard network/firewall hints"
uci show network | grep -Ei 'wireguard|wg|vpn' || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true

echo "### WireGuard logs"
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

見るポイントです。

- latest handshakeが更新されているか
- UDPポートが外から届いているか
- IPoE / IPv4 over IPv6環境で外部着信できる条件か
- AllowedIPsが正しいか
- firewallでWireGuardポートを許可しているか

VPNでは、「VPNに接続できる」と「LAN内機器へ届く」は別です。

Tailscaleならサブネットルート承認、WireGuardならAllowedIPsとfirewallを確認します。

## ステップ10: 直近ログを見る

最後にログを見ます。

```sh
echo "### recent logs"
logread | tail -n 150

echo "### errors"
logread | grep -Ei 'err|error|warn|fail|timeout|restart|disconnect' | tail -n 120
```

ログは、いきなり全部読むものではありません。

ここまでの切り分け結果と合わせて見ます。

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-12.png)

例:

```sh
logread | grep -Ei 'wan|wan6|netifd|odhcp6c|ipoe|map|dslite|ipip' | tail -n 120
logread | grep -Ei 'dnsmasq|dhcp' | tail -n 120
logread | grep -Ei 'hostapd|wireless|wlan|dfs|radar' | tail -n 120
logread | grep -Ei 'firewall|drop|reject' | tail -n 120
```

ログは、答えそのものというより、手がかりです。

「WANが怪しい」と分かってからWANログを見る。  
「DNSが怪しい」と分かってからdnsmasqログを見る。

この順番のほうが迷いにくいです。

## 症状別クイック対処表

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-13.png)

この表は、トラブル時の最初の地図です。

迷ったら、上から順に見ます。

## LuCIで見る主要画面

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-14.png)

LuCIだけでもかなり切り分けできます。

最初は、Overview、Interfaces、Wireless、DHCP Leases、System Logだけ見られれば十分です。

## 変更直後に戻すための安全パターン

設定変更後におかしくなった場合は、最後に触った範囲だけ戻すほうが安全です。

全部を一気に戻すより、原因を追いやすくなります。

## 変更前バックアップを作る

大きな変更前は、LuCIバックアップに加えて、SSHでも控えを取ります。

```sh
BACKUP_DIR="/root/config-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

opkg list-installed > "$BACKUP_DIR/packages.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

復元用の正式バックアップは、LuCIの **System → Backup / Flash Firmware → Generate archive** からPCへ保存します。

```txt
診断ログ = 状態を見るため
LuCIバックアップ = 戻すため
```

この2つは分けて考えます。

## DNS/DHCPだけ戻す・再起動する

IPアドレスは取れているが名前解決だけおかしい場合は、まずdnsmasqを再起動します。

```sh
/etc/init.d/dnsmasq restart
sleep 3
logread | grep -i dnsmasq | tail -n 80
nslookup www.google.co.jp 127.0.0.1
```

Adblockが原因かもしれない時は、一時停止して確認します。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

## Wi-Fiだけ反映し直す

有線は正常でWi-Fiだけおかしい場合は、Wi-Fiだけを反映し直します。

```sh
wifi status
wifi reload
sleep 5
wifi status
```

`wifi reload` 中は無線端末が一時的に切断されます。

有線LANでLuCIやSSHへ入れる状態で実行するほうが安全です。

## firewallだけ戻す

Guest Wi-Fi、VLAN、ポート開放、VPN設定のあとに通信できなくなった場合は、firewallだけ戻します。

```sh
cp /root/config-before-change-YYYYMMDD-HHMM/firewall /etc/config/firewall
/etc/init.d/firewall restart
uci show firewall | head -n 80
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えます。

## network設定を戻す

LAN IP、WAN、VLAN、bridgeを触った場合は、network設定を戻します。

これはLuCI / SSHが切れる可能性があるため、有線LAN接続で実行します。

```sh
cp /root/config-before-change-YYYYMMDD-HHMM/network /etc/config/network
/etc/init.d/network restart
```

`/etc/init.d/network restart` 後は、PC側のIP再取得が必要になることがあります。

PCの有線LANを抜き差しするか、DHCPを更新してください。

## wireless設定を戻す

Wi-Fi設定変更後にSSIDが見えない、暗号化方式が合わない、端末がつながらない場合です。

```sh
cp /root/config-before-change-YYYYMMDD-HHMM/wireless /etc/config/wireless
wifi reload
```

Wi-Fi設定を戻す時も、有線LANで入れる状態にしておくと安心です。

## LuCIバックアップから戻す

広範囲に壊れている場合は、LuCIバックアップから戻します。

1. **System** → **Backup / Flash Firmware** を開く
2. **Restore backup** または **Upload archive** を選ぶ
3. 保存していた `.tar.gz` をアップロードする
4. 復元後、再起動を待つ
5. LuCIへ再ログインする
6. WAN/WAN6、Wi-Fi、DHCP、firewallを確認する

バックアップ復元は強力ですが、問題の原因になっていた設定も戻ることがあります。

原因不明の不具合が続く場合は、必要な設定だけ手で戻すことも検討します。

## よくあるケース別の見方

## Wi-Fiにはつながるがインターネットへ出られない

見る順番です。

```sh
cat /tmp/dhcp.leases
ifstatus wan
ifstatus wan6
ip route show default
ping -c 4 8.8.8.8
nslookup www.google.co.jp
```

判断です。

- DHCPリースにいない → IPを取れていない
- WAN down → 回線側
- pingは通るがnslookup失敗 → DNS
- pingもnslookupも失敗 → WAN / route / firewall

まず、端末がIPを取れているか。

次に、IP直打ちで外へ出られるか。

最後にDNSです。

## ゲストWi-Fiだけつながらない

```sh
ifstatus guest
uci show wireless | grep -E 'guest|ssid|network'
uci show dhcp.guest
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'
logread | grep -i dnsmasq | tail -n 80
```

確認することです。

- Guest用SSIDが `guest` networkに紐づいているか
- guest interfaceがupか
- DHCP Serverが有効か
- `Allow-Guest-DHCP` があるか
- `Allow-Guest-DNS` があるか
- guest → wan forwardingがあるか
- guest → lanを許可していないか

ゲストWi-Fiは、SSIDだけでは完成しません。

SSID、network、DHCP、firewall zoneをセットで見ます。

## IoTやカメラだけ不安定

```sh
uci show wireless | grep -E 'iot|device|ssid|network|encryption'
cat /tmp/dhcp.leases
ifstatus device 2>/dev/null || true
logread | grep -Ei 'dnsmasq|hostapd|wireless|device|iot' | tail -n 120
```

確認することです。

- 2.4GHzへ接続しているか
- WPA3ではなく互換性のある暗号化か
- Device / IoTネットワークでIPを取れているか
- DNS広告ブロックが強すぎないか
- device → wanが必要な機器なのに閉じていないか
- 初期設定時だけ同じLANが必要な機器ではないか

IoTは、つながっているように見えてアプリから見えないことがあります。

クラウド接続、mDNS、初期設定時の同一LAN要件も疑います。

## IPoE環境でIPv6は生きているがIPv4だけだめ

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
ping -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
logread | grep -Ei 'ipoe|map|mape|dslite|ipip|wan|wan6' | tail -n 120
```

可能性です。

- IPv4 over IPv6側が未設定
- オートIPoEモジュールが動いていない
- プロバイダ側の方式と設定が合っていない
- HGW配下 / ONU直下の構成を取り違えている
- PPPoEや固定IP設定と衝突している

ここでWi-Fiを疑わないことが大事です。

IPv6が生きていて、IPv4だけだめなら、IPoE / IPv4 over IPv6側を見ます。

## Adblock導入後に一部サイトだけ壊れた

```sh
/etc/init.d/adblock status 2>/dev/null || true
logread | grep -Ei 'adblock|dnsmasq' | tail -n 120
nslookup target.example 127.0.0.1
```

一時停止して確認します。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

一時停止で直るなら、Adblockやブロックリストが原因の可能性があります。

Allowlist追加、ブロックリスト見直し、DNS Report確認へ進みます。

## VPNだけつながらない

Tailscaleなら次です。

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

WireGuardなら次です。

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn'
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

確認することです。

- Tailscaleはtailnetに参加しているか
- サブネットルートは承認済みか
- WireGuardはlatest handshakeが更新されるか
- IPoE環境でWireGuardポートが外から届くか
- VPNからLANへのfirewallルールがあるか
- NASや管理PCのIPが変わっていないか

VPNは、「VPN接続そのもの」と「LAN内機器へ届くか」を分けて見ます。

## ファームウェア更新後につながらない

まず基本から確認します。

```sh
ubus call system board
ifstatus wan
ifstatus wan6
ping -c 4 8.8.8.8
nslookup www.google.co.jp
opkg list-installed | grep -Ei 'ipoe|vpn|tailscale|wireguard|adblock'
logread | tail -n 150
```

更新後は、追加パッケージやモジュールが消えている場合があります。

特に確認したいものです。

- オートIPoE
- VPN Assistant
- Tailscale / WireGuard
- Adblock
- banIP
- 追加LuCIアプリ

設定バックアップだけでは、追加パッケージ本体が戻らないことがあります。

020のファームウェア更新運用の記事と合わせて確認してください。

## トラブル時メモのテンプレート

サポートや詳しい人へ相談する時は、次のように整理すると話が早くなります。

```txt
製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

回線:
  NTT / auひかり / NURO / J:COM / その他

構成:
  ONU -> LN6001-JP
  ONU -> HGW -> LN6001-JP
  既存ルーター配下

接続方式:
  IPoE / PPPoE / HGW配下DHCP / 不明

問題:
  全端末でインターネット不可
  Wi-Fiだけ不可
  ゲストWi-Fiだけ不可
  VPNだけ不可
  一部サイトだけ不可

発生時刻:
  2026-06-21 14:30頃

直前の変更:
  なし
  Adblock導入
  ゲストWi-Fi追加
  VLAN変更
  VPN Assistant導入
  ファームウェア更新

確認済み:
  有線LuCI: OK / NG
  ifstatus wan: up / down
  ifstatus wan6: up / down
  ping 8.8.8.8: OK / NG
  nslookup: OK / NG
  DHCP leases: 対象端末あり / なし

バックアップ:
  あり / なし
```

このくらい整理されていると、原因をかなり絞りやすくなります。

「なんかつながらない」より、ずっと相談しやすいです。

## まとめ

接続トラブルは、順番を決めて見ると落ち着いて対応できます。

おすすめの順番は次です。

1. LED、ケーブル、ONU/HGWを見る
2. 有線LANでLuCIへ入れるか確認する
3. `ifstatus wan` / `ifstatus wan6` でWAN状態を見る
4. `ping` と `ping6` でIPv4 / IPv6疎通を分ける
5. `nslookup` でDNS問題を分ける
6. `cat /tmp/dhcp.leases` でDHCP状態を見る
7. `wifi status` とAssociated StationsでWi-Fiを見る
8. `uci show firewall` でfirewall / VLAN / ゲストWi-Fiを見る
9. Tailscale / WireGuardを使っているならVPN状態を見る
10. `logread` で直近ログを見る

「つながらない」は、ひとつの症状に見えます。

でも、中身はWAN、DNS、DHCP、Wi-Fi、firewall、VPN、IPv6に分かれます。

最初から全部を直そうとせず、どこで止まっているかを分ける。

それが一番の近道です。

困った時ほど、いきなり直さない。

まず見る。

それだけで、かなり変わります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-03.png)

## 次に読むなら

接続トラブルの切り分けを深めたい人は、次の記事も合わせて読むと整理しやすいです。

- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)
- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)

ログの見方を整えたい人は監視とログへ。

設定を戻す手順を固めたい人はバックアップと復元へ。

IPoEやIPv6で詰まっている人は、IPoE記事とIPv6記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、`ifstatus`、`wifi status`、`logread`、`iptables-save`、`ip6tables-save`、Tailscale / WireGuard関連の出力は、ファームウェアや追加モジュールの更新で変わることがあります。

OpenWrt系のfirewall表示は、バージョンやターゲットによって変わります。LN6001-JPでは `iptables-save`、`ip6tables-save`、iptables互換表示を確認します。

この記事では、まず `uci show firewall`、`iptables-save`、`ip6tables-save` を中心に確認し、必要に応じて `iptables` を確認用として使う方針にしています。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPがつながらない時は何から確認すればいい？

まずLED、WANケーブル、ONU/HGWを確認します。

そのあと、有線LANでLuCIへ入れるかを見ます。

LuCIへ入れるなら、次は `ifstatus wan`、`ifstatus wan6`、`ping`、`nslookup` でWANとDNSを分けます。

### Wi-Fiにはつながるのにインターネットが使えない時は？

まず端末がIPアドレスを取れているか確認します。

LN6001-JP側では `cat /tmp/dhcp.leases`、端末側ではIPアドレス表示を見ます。

IPがあるなら、`ping 8.8.8.8` と `nslookup www.google.co.jp` でWANかDNSかを分けます。

### IPアドレスでは通るのにドメイン名で開けない時は？

DNS問題の可能性が高いです。

`nslookup www.google.co.jp 127.0.0.1` と `nslookup www.google.co.jp 8.8.8.8` を比べます。

Adblock、Family DNS、DoH、dnsmasq設定も確認してください。

### ゲストWi-Fiだけつながらない時は？

guest interface、DHCP、DNS、firewallを見ます。

特に、guest zoneのInputを `REJECT` にしている場合、`Allow-Guest-DHCP` と `Allow-Guest-DNS` が必要です。

### 設定変更後におかしくなった時はどう戻せばいい？

最後に触った設定だけ戻すのが安全です。

DNSだけならdnsmasq、Wi-Fiだけならwireless、firewall変更ならfirewall、LAN IPやVLANならnetworkを確認します。

大きく崩れている場合は、LuCIバックアップから復元します。

### 再起動すれば直る？

直る場合もあります。

ただし、原因が見えなくなることもあります。

まず診断ログを取り、WAN、DNS、DHCP、Wi-Fi、firewallのどこで止まっているか確認してから再起動するほうが安全です。

### firewall確認は何を見ればいい？

LN6001-JPでは、`iptables-save` と `ip6tables-save` を見たほうが実機に合わせやすいです。

まず `uci show firewall` で設定を確認し、必要に応じて `iptables-save` や `ip6tables-save` を使います。

古い記事の `iptables` コマンドは、互換表示として見える場合もありますが、それだけに依存しないほうが安全です。

### 診断ログはそのままSNSやGitHubへ貼っていい？

おすすめしません。

SSID、MACアドレス、IPv6 prefix、VPN情報、Wi-Fiパスワード、PPPoE ID、DDNS名などが含まれる場合があります。

共有前に必ず伏せ字にしてください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 LED状態: https://support.linksys.com/kb/article/216-en/?section_id=175
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- OpenWrt Wiki - Logging messages: https://openwrt.org/docs/guide-user/base-system/log.essentials
- OpenWrt Wiki - DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt Wiki - Firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt Wiki - Firewall overview: https://openwrt.org/docs/guide-user/firewall/overview

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
