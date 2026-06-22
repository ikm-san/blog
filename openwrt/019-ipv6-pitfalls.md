<!-- mirror-source: articles/019-ipv6-pitfalls.md -->

# IPv6は“ポート開放してないから安心”ではない｜IPoE時代に見落としやすいこと【OpenWrt集中連載019】

IPv6、ちゃんと動くとかなり便利です。

IPoEで速くなる。  
PPPoEの混雑を避けやすい。  
IPv4 over IPv6で普段のWebも普通に使える。  
最近の光回線では、もうIPv6は特別な設定ではなく、かなり普通に使われています。

でも、ここに少し落とし穴があります。

IPv4の感覚のまま、

```txt
ポート開放してないから大丈夫
家庭用ルーターの内側だから外から見えないはず
IPv6は勝手に動いているだけだから気にしなくていい
```

と思っていると、見落とすことがあります。

IPv4では、家庭内の端末は `192.168.x.x` のようなプライベートIPにいて、NATの内側に隠れている感覚があります。

一方でIPv6では、LAN内端末にグローバルIPv6アドレスが付くことがあります。

つまりIPv6では、

```txt
NATで隠れているから安心
```

ではなく、

```txt
firewallでちゃんと止めているか
```

を見ることが大事になります。

怖がる必要はありません。

ただ、IPv4とIPv6は別物として見たほうがいいです。

`ifstatus wan` と `ifstatus wan6` を分ける。  
`ping` と `ping6` を分ける。  
AレコードとAAAAレコードを分ける。  
IPv4のポート開放とIPv6のfirewall許可を分ける。

これだけで、IPoE環境のトラブル切り分けはかなり楽になります。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）でIPoE / IPv6を使う時に、家庭用ルーター感覚で見落としやすいポイントを整理します。

IPv6を怖がる記事ではありません。

**IPv4とIPv6を別々に見よう** という記事です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、IPv6を完璧に理解することではありません。

まずは、ここまで分かればOKです。

- IPv4とIPv6を分けて確認する理由が分かる
- IPoEとIPv4 over IPv6を混同しない
- IPv6ではNAT感覚よりfirewallが大事だと分かる
- WAN6、IPv6 default route、Prefix Delegationを確認できる
- RA / DHCPv6がLAN端末へのIPv6配布に関係すると分かる
- ゲストWi-Fi、IoT、KidsでIPv6をどう扱うか考えられる
- IPv6側で意図せず公開しないための確認ができる
- VPNやDNS広告ブロックでIPv6が抜け道になりやすい理由が分かる

IPv6は、全部を理解してから使うものではありません。

でも、

```txt
IPv4とは見方が違う
```

ここだけは最初に押さえておくと安心です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-01.png)

## 先にざっくり結論

IPv6で最初に見るべきことは、この5つです。

1. `ifstatus wan6` でWAN6がupか見る
2. `ip -6 route show` でIPv6 default routeを見る
3. Prefix Delegationが取れているか見る
4. LAN端末にIPv6が配られているか見る
5. IPv6 firewallが意図どおりになっているか見る

IPv6を怖がる必要はありません。

ただし、IPv4と同じものとして扱わないほうがいいです。

ざっくり比較するとこうです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-01.png)

IPv4では、ポートフォワードを作らない限り、LAN内端末へ外から直接入りにくい構成が多いです。

IPv6では、端末にグローバルIPv6アドレスが付くことがあります。

だからこそ、IPv6側ではfirewallがかなり大事です。

最初はこれだけ覚えてください。

```txt
IPv4はNATの感覚が強い
IPv6はfirewallの感覚が大事
```

## こういう人向けです

この記事は、次のような人向けです。

- IPoE環境でIPv6がどう動いているかよく分からない
- IPv4はつながるのに一部サイトやアプリだけ不安定
- IPv6側で端末が外から見えないか不安
- ゲストWi-FiやIoTネットワークでIPv6を配っていいか迷っている
- KidsネットワークでFamily DNSを使っているが、IPv6側の抜け道が気になる
- TailscaleやWireGuardでIPv6が絡むと混乱する
- DNS広告ブロックやDoH対策を入れたが、IPv6側も見たい
- IPv6 firewallをどこまで確認すればよいか知りたい

逆に、すでにDHCPv6-PD、RA、NDP、iptables-save、iptables/ip6tables、IPv6 prefix設計まで慣れている人には基本寄りです。

ここでは、家庭・小さなオフィス・店舗で迷子にならないための入口として整理します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-02.png)

## 最初に言葉だけそろえる

IPv6まわりは用語が急に増えます。

まず、ざっくりで大丈夫です。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-02.png)

最初は、次だけで十分です。

```txt
WAN6 = IPv6側のWAN
PD = 家の中へ配るIPv6アドレス範囲
RA / DHCPv6 = LAN端末へIPv6情報を配る仕組み
UCI / iptables = firewallの実体を見る入口
```

## IPoEとIPv4 over IPv6を混同しない

日本の回線でよく混乱するのが、IPoEとIPv4 over IPv6です。

この2つ、名前が似ているうえにセットで出てくるのでややこしいです。

でも最初はこう見ればOKです。

```txt
IPoE = IPv6側の接続
IPv4 over IPv6 = IPv6網の上でIPv4通信も使う仕組み
```

代表的な方式はこうです。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-03.png)

このため、次のような状態が起きます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-04.png)

「IPv6が生きている」ことと「IPv4サイトも全部見られる」ことは別です。

ここを分けて見るだけで、IPoEまわりの切り分けはかなり楽になります。

## IPv6で一番大きい違いはNAT感覚

IPv4の家庭内ネットワークでは、よくこうなっています。

```txt
LAN端末 192.168.1.100
  ↓ NAT
ルーターのWAN IPv4
  ↓
インターネット
```

LAN端末はプライベートIPv4アドレスを持ち、外から直接見えにくい構成です。

外部からLAN内機器へ入る時は、Port Forwardを作ることが多いです。

一方でIPv6では、LAN端末にグローバルIPv6アドレスが付くことがあります。

```txt
LAN端末 240x:xxxx:.... のIPv6アドレス
  ↓ firewall
インターネット
```

ここで大事なのは、

```txt
NATで隠れているから安心
```

という感覚に頼らないことです。

IPv6では、firewallが外部からの通信を止める重要な境界になります。

OpenWrt系の標準的なfirewallでは、WAN側からのInputやForwardは拒否する方向で構成されます。

ただし、設定を変更したり、IPv6向けの許可ルールを追加した場合は、意図せず外から届く構成になることがあります。

なので、IPv6では、

```txt
ポート開放していないから大丈夫
```

だけではなく、

```txt
IPv6 firewallで何を許可しているか
```

を見ます。

## IPv4とIPv6を分けて確認する

IPv6トラブルでは、「インターネットが不安定」とまとめずに、IPv4とIPv6を分けます。

まずこのセットです。

```sh
echo "### WAN IPv4"
ifstatus wan

echo "### WAN IPv6"
ifstatus wan6

echo "### IPv4 route"
ip route show

echo "### IPv6 route"
ip -6 route show

echo "### IPv4 ping"
ping -c 4 8.8.8.8

echo "### IPv6 ping"
ping6 -c 4 2001:4860:4860::8888

echo "### DNS A and AAAA"
nslookup -type=A www.google.co.jp
nslookup -type=AAAA www.google.co.jp
```

見るポイントです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-05.png)

判断の例です。

```txt
ping は通るが ping6 が通らない
→ IPv6側を見る

ping6 は通るが IPv4サイトが開けない
→ IPv4 over IPv6側を見る

ping / ping6 は通るがWebだけ変
→ DNSやMTU、CDN、端末側を見る
```

「全部だめ」ではなく、IPv4、IPv6、DNSを分けます。

ここが一番効きます。

## WAN6を見る

IPoE環境では、WAN6の状態がかなり重要です。

```sh
echo "### wan6 status"
ifstatus wan6

echo "### IPv6 addresses"
ip -6 addr show

echo "### IPv6 routes"
ip -6 route show

echo "### wan6 logs"
logread | grep -Ei 'wan6|odhcp6c|dhcpv6|ra|router advertisement|prefix|netifd' | tail -n 120
```

見るポイントです。

- `wan6` がupか
- IPv6アドレスが付いているか
- IPv6 default routeがあるか
- `ip6prefix` やPrefix Delegationがあるか
- `odhcp6c` や `netifd` のログにエラーがないか

WAN6がupでも、LAN側へPrefix Delegationが配られていないと、端末側ではIPv6が使えないことがあります。

つまり、

```txt
ルーター自身はIPv6で外へ出られる
でもLAN端末にはIPv6が来ていない
```

という状態がありえます。

## Prefix Delegationを見る

IPv6では、プロバイダから受け取ったプレフィックスをLAN側へ配ります。

この仕組みがPrefix Delegationです。

確認します。

```sh
echo "### wan6 prefix"
ifstatus wan6 | grep -Ei 'ip6prefix|prefix|ip6addr|up'

echo "### network IPv6 settings"
uci show network | grep -Ei 'ip6assign|ip6hint|ip6class|delegate|wan6|lan'

echo "### lan IPv6 address"
ip -6 addr show br-lan 2>/dev/null || ip -6 addr show | grep -A5 -Ei 'br-lan|lan'

echo "### odhcp logs"
logread | grep -Ei 'odhcp6c|odhcpd|dhcpv6|prefix|ra' | tail -n 120
```

見るポイントです。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-06.png)

よくある状態です。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-07.png)

## RA / DHCPv6を見る

LAN端末にIPv6情報を配るには、RAやDHCPv6が関係します。

OpenWrt系では、`odhcpd` がLAN側のRA / DHCPv6に関係します。

確認します。

```sh
echo "### DHCP config IPv6 hints"
uci show dhcp | grep -Ei 'ra|dhcpv6|ndp|ra_management|interface|lan'

echo "### odhcpd status"
if [ -x /etc/init.d/odhcpd ]; then
  /etc/init.d/odhcpd status
fi

echo "### odhcpd logs"
logread | grep -Ei 'odhcpd|ra|dhcpv6|router advertisement' | tail -n 120
```

LAN側でIPv6を配るかどうかは、ネットワーク方針によります。

すべてのネットワークへIPv6を配る必要はありません。

特に、ゲストWi-Fi、Kids、IoT、Deviceなどでは、IPv6を配るならfirewall設計もセットで考えます。

ここを忘れると、

```txt
IPv4では分離できている
でもIPv6では別の道が残っている
```

という状態になりかねません。

## IPv6 firewallで最初に見ること

IPv6 firewallで最初に見るのは、この3つです。

1. WAN zoneのInputがどうなっているか
2. WAN zoneのForwardがどうなっているか
3. IPv6向けの許可ルールを追加していないか

確認します。

```sh
echo "### firewall zones"
uci show firewall | grep -E 'zone|name|network|input|output|forward|masq|mtu_fix'

echo "### firewall rules"
uci show firewall | grep -E 'rule|src|dest|proto|dest_port|target|family|ip6'

echo "### iptables rules"
iptables-save 2>/dev/null | grep -Ei 'ip6|wan|reject|drop|accept' -A3 -B3 | head -n 200 || true

echo "### ip6tables rules"
ip6tables-save 2>/dev/null | grep -Ei 'ip6|icmpv6|tcp dport|udp dport|reject|drop|accept' -A3 -B3 | head -n 200 || true
```

古いOpenWrt記事では、`ip6tables -L -n -v` を使う例もあります。

LN6001-JPでは、iptables/ip6tables系の表示を見たほうが実機に合わせやすいです。

ただし、互換表示として `ip6tables` が見える環境もあるため、確認用として次も使えます。

```sh
ip6tables -L -n -v 2>/dev/null || true
ip6tables-save 2>/dev/null | head -n 120 || true
```

最初は、`uci show firewall` と `iptables-save` を中心に見れば十分です。

## IPv6で公開されていないか確認する

IPv6側で意図せずサービスを公開していないか確認します。

まずルーターやLAN内端末がIPv6アドレスを持っているか見ます。

```sh
echo "### IPv6 addresses on router"
ip -6 addr show

echo "### neighbor table"
ip -6 neigh show
```

次にfirewall側を見ます。

```sh
echo "### firewall IPv6 rules"
uci show firewall | grep -Ei 'ip6|ipv6|family|src|dest|port|target|wan'

echo "### ip6tables IPv6 rules"
ip6tables-save 2>/dev/null | grep -Ei 'ip6|tcp dport|udp dport|accept|reject|drop' -A3 -B3 | head -n 200 || true
```

見るポイントです。

- WANからLAN内端末へのIPv6 forwardを広く許可していないか
- 特定ポートをIPv6で許可していないか
- ルーター自身のLuCI / SSHをWAN側IPv6で許可していないか
- guest / iot / kidsからlanへIPv6で抜けていないか

IPv4のPort Forwardを作っていないから安心、とは限りません。

IPv6側で許可ルールを作っていないかも見ます。

## IPv4ポート開放とIPv6公開は別物

IPv4では、よくこう考えます。

```txt
WAN TCP 8443 → NAS 192.168.1.10:443
```

これはIPv4のPort Forwardです。

OpenWrtではfirewallの `redirect` として扱われます。

一方でIPv6では、LAN内端末にグローバルIPv6アドレスが付くことがあります。

この場合、考え方はPort Forwardではなく、

```txt
そのIPv6アドレスのそのポートをWAN側から許可するか
```

です。

つまり、IPv4とIPv6では外部公開の見方が違います。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-08.png)

NAS、カメラ、ルーター管理画面、RDP、SSHは、IPv4でもIPv6でも直接公開しないほうが安全です。

自分だけが使うなら、まずVPNを考えます。

## ゲストWi-FiでIPv6をどうするか

ゲストWi-Fiでは、IPv4だけ分離して満足しがちです。

でも、IPv6を配っている場合は、ゲスト端末にもIPv6アドレスが付き、IPv6側で外へ出られることがあります。

これ自体は悪いことではありません。

ただし、ゲストWi-FiをLANから分けたいなら、IPv6側でも同じ方針になっているか確認します。

確認ポイントです。

- ゲスト端末にIPv6が付いているか
- ゲスト端末からLAN側IPv6アドレスへ届かないか
- guest zoneでIPv6 forwardingが意図どおりか
- guestからルーター自身へLuCI / SSHが開いていないか
- DHCPv6 / RAをどう扱うか

確認します。

```sh
echo "### guest network IPv6 settings"
uci show network | grep -Ei 'guest|ip6|delegate|ip6assign|ip6hint'

echo "### guest DHCP/RA settings"
uci show dhcp | grep -Ei 'guest|ra|dhcpv6|ndp'

echo "### guest firewall"
uci show firewall | grep -Ei 'guest|ip6|family|forwarding|rule'
```

ゲストWi-FiでIPv6を使わせるなら、guest → wanはIPv6でも意図どおり許可し、guest → lanは拒否する設計にします。

迷う場合は、まずゲストWi-FiをIPv4中心で安定させ、IPv6は別途設計してから有効化するほうが安全です。

## IoT / DeviceでIPv6をどうするか

IoT機器やカメラにIPv6が必要かは、機器によります。

スマート家電やカメラの中には、IPv6に対応しているものもあります。

ただし、IoT / Deviceネットワークの目的が「親のPCやNASから分けること」なら、IPv6側でも同じ分離が必要です。

確認ポイントです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-09.png)

確認します。

```sh
echo "### iot/device IPv6 settings"
uci show network | grep -Ei 'iot|device|ip6|delegate|ip6assign'

echo "### iot/device DHCP/RA"
uci show dhcp | grep -Ei 'iot|device|ra|dhcpv6|dns'

echo "### iot/device firewall"
uci show firewall | grep -Ei 'iot|device|ip6|family|forwarding|rule'
```

IoT / Deviceは、最初からIPv6を広く開けすぎないほうが安全です。

必要な機器だけ、必要な通信だけにします。

## KidsネットワークとIPv6

子ども用ネットワークでFamily DNSを配っている場合、IPv6側が抜け道になることがあります。

たとえば、IPv4ではCloudflare Family DNSの `1.1.1.3` / `1.0.0.3` を配っていても、IPv6側で別DNSやDoHが使われると、期待どおりフィルタリングされない場合があります。

Cloudflare Family DNSにはIPv6アドレスもあります。

```txt
2606:4700:4700::1113
2606:4700:4700::1003
```

ただし、KidsネットワークへIPv6を配る場合は、RA、DHCPv6、firewall、DNS方針をセットで設計します。

確認します。

```sh
echo "### kids IPv6 settings"
uci show network | grep -Ei 'kids|ip6|delegate|ip6assign'

echo "### kids DHCP/RA"
uci show dhcp | grep -Ei 'kids|ra|dhcpv6|dns|dhcp_option'

echo "### kids firewall IPv6"
uci show firewall | grep -Ei 'kids|ip6|family|forwarding|rule'
```

子ども用ネットワークでは、ルーター側のDNS制御だけで完結させようとしないほうが現実的です。

端末側のファミリー機能も併用します。

IPv6やDoHまでルーターだけで完全に制御しようとすると、運用がかなり複雑になります。

## DNSでIPv6を見落とさない

DNSには、IPv4アドレスを返すAレコードと、IPv6アドレスを返すAAAAレコードがあります。

IPv4のDNSだけ見ていると、IPv6側の問題を見落とすことがあります。

確認します。

```sh
echo "### A record"
nslookup -type=A www.google.co.jp

echo "### AAAA record"
nslookup -type=AAAA www.google.co.jp

echo "### resolver config"
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 100
```

よくある切り分けです。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/table-10.png)

AdblockやFamily DNSを使う場合も、IPv4側だけでなくIPv6側のDNS方針を確認します。

## DoHでIPv6のDNS制御が抜けることがある

DNS広告ブロックやFamily DNSを設定しても、端末やブラウザがDoHを使うと、ルーター側DNSを迂回することがあります。

これはIPv4でもIPv6でも起きます。

確認するポイントです。

- Chrome / Edge / FirefoxのセキュアDNS設定
- OS側のDNS設定
- VPNアプリ
- Tailscale / WireGuardのDNS設定
- Family DNSやAdblockの適用範囲
- banIPなどのDoH対策を入れているか

ルーター側では、次のようにDNS関連を確認します。

```sh
echo "### DHCP DNS settings"
uci show dhcp | grep -Ei 'dhcp_option|dns|server'

echo "### Adblock / banIP hints"
opkg list-installed | grep -Ei 'adblock|banip' || true
uci show adblock 2>/dev/null | head -n 80 || true
uci show banip 2>/dev/null | head -n 80 || true
```

DoH対策を強くすると、必要なサービスまで止まることがあります。

最初は端末側設定の確認から始めるのがおすすめです。

## IPv6とVPN

VPNでもIPv6を見落としやすいです。

たとえば、VPNではIPv4のLANだけ通しているつもりでも、端末のIPv6通信は外出先回線からそのまま出ていることがあります。

これをIPv6 leakと呼ぶことがあります。

## WireGuardの場合

WireGuardでは、クライアント側の `AllowedIPs` が重要です。

IPv4のLANだけ通す例です。

```txt
AllowedIPs = 192.168.1.0/24
```

IPv6も通す場合は、IPv6 prefixや `::/0` をどう扱うか考える必要があります。

ただし、最初からIPv6フルトンネルにする必要はありません。

まずはIPv4で目的のLANへ届くことを確認し、必要に応じてIPv6を設計します。

確認します。

```sh
wg show 2>/dev/null || true
uci show network | grep -Ei 'wireguard|wg|ip6|allowed'
uci show firewall | grep -Ei 'wireguard|wg|vpn|ip6|family'
```

## Tailscaleの場合

TailscaleはIPv6的なアドレス体系やMagicDNSも関係します。

Tailscale経由でLAN内IPv4へ入るだけなら、まずサブネットルートを確認します。

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true
```

Exit Nodeを使う場合は、外出先端末のIPv4 / IPv6やDNSがどう流れるかも確認します。

最初はExit Nodeなしで、目的のLAN内機器へ届くところまで確認するのがおすすめです。

## ICMPv6を雑に止めない

IPv4では、ICMPを雑に止める設定を見かけることがあります。

IPv6では、ICMPv6がより重要です。

近隣探索、Path MTU Discovery、エラー通知など、IPv6通信の基本動作に関係します。

そのため、意味を理解せずにICMPv6を広く拒否しないほうがよいです。

確認します。

```sh
echo "### ICMPv6 firewall hints"
uci show firewall | grep -Ei 'icmpv6|icmp|family|ip6'

echo "### ip6tables ICMPv6 hints"
ip6tables-save 2>/dev/null | grep -Ei 'icmpv6|packet too big|neighbor|router' -A3 -B3 | head -n 160 || true
```

一部サイトだけ開けない。  
VPNだけ不安定。  
画像や動画が途中で止まる。

こういう時に、MTUやICMPv6が関係することがあります。

最初からICMPv6を疑う必要はありませんが、

```txt
IPv6ではICMPv6を雑に止めない
```

これは覚えておくと役立ちます。

## MTU / PMTUDの落とし穴

IPv4 over IPv6、VPN、PPPoE、IPoE、DS-Lite、MAP-Eなどが絡むと、MTUが問題になることがあります。

症状の例です。

- 一部サイトだけ開けない
- 小さい通信は通るが、大きい通信が不安定
- 動画や画像だけ読み込みに失敗する
- VPN経由だけ不安定
- ゲームやWeb会議だけ切れる

確認の入口です。

```sh
echo "### IPv4 MTU test"
ping -M do -s 1472 -c 4 8.8.8.8

echo "### IPv6 reachability"
ping6 -c 4 2001:4860:4860::8888

echo "### interface MTU"
ip link show | grep -E 'mtu|wan|br-lan|tailscale|wg' -A1 -B1
```

MTU変更は影響が大きいので、いきなり変更しないほうがよいです。

まず、WAN / WAN6、DNS、firewall、VPN設定が正しいかを確認し、それでも症状が残る場合にMTUやPMTUDを疑います。

## IPv6を無効化すればよいのか

トラブルが出ると、

```txt
IPv6を切ればよいのでは？
```

と思いたくなります。

短期的な切り分けとして、IPv6を一時的に止めることはあります。

ただし、日本のIPoE / IPv4 over IPv6環境では、IPv6を雑に止めるとIPv4 over IPv6側も影響を受ける場合があります。

OCNバーチャルコネクト、transix、v6プラス、クロスパスなどでは、IPv6が土台になっています。

そのため、恒久的にIPv6を無効化するより、まず次を確認します。

1. WAN6はupか
2. Prefix Delegationは取れているか
3. LAN側RA / DHCPv6は意図どおりか
4. firewallはIPv6側も意図どおりか
5. DNSやDoHで抜け道になっていないか
6. ゲストWi-FiやIoTにIPv6を配る必要があるか

「IPv6を全部切る」より、

```txt
どのネットワークへIPv6を配るか
```

を決めるほうが現実的です。

## IPv6設定変更前にバックアップを取る

IPv6 / IPoEまわりを触る前に、バックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-ipv6-change-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/ipv6-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip -6 addr show > "$BACKUP_DIR/ip-addr-v6.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
iptables-save > "$BACKUP_DIR/iptables-save.txt" 2>/dev/null || true
ip6tables-save > "$BACKUP_DIR/ip6tables-save.txt" 2>/dev/null || true

logread | grep -Ei 'ipv6|ip6|wan6|odhcp6c|odhcpd|dhcpv6|ra|prefix|netifd' | tail -n 200 > "$BACKUP_DIR/ipv6-log.txt"

ls -l "$BACKUP_DIR"
```

IPv6アドレスやprefixには、回線や拠点を推測できる情報が含まれることがあります。

共有する場合は、必要に応じて伏せ字にしてください。

## よくある症状と切り分け

## IPv4は通るがIPv6が通らない

確認します。

```sh
ifstatus wan6
ip -6 route show
ping6 -c 4 2001:4860:4860::8888
logread | grep -Ei 'wan6|odhcp6c|dhcpv6|prefix|netifd' | tail -n 120
```

見るポイントです。

- WAN6がupか
- IPv6 default routeがあるか
- Prefix Delegationが取れているか
- 上流HGWやONU構成が正しいか
- IPoE契約が有効か

## IPv6は通るがIPv4サイトが開けない

確認します。

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
ping -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
logread | grep -Ei 'ipoe|map|mape|dslite|ipip|wan|wan6' | tail -n 120
```

IPv6 / IPoEは生きているが、IPv4 over IPv6側が未設定または不調の可能性があります。

オートIPoEモジュール、契約中のVNE方式、HGW / ONU構成を確認します。

## LAN端末にIPv6が付かない

確認します。

```sh
ifstatus wan6 | grep -Ei 'ip6prefix|prefix|up'
uci show network | grep -Ei 'lan|ip6assign|ip6hint|delegate'
uci show dhcp | grep -Ei 'lan|ra|dhcpv6|ndp'
logread | grep -Ei 'odhcpd|odhcp6c|dhcpv6|prefix|ra' | tail -n 120
```

WAN6がupでも、LANへprefixが配られていなければ端末側にIPv6が付きません。

LAN側のRA / DHCPv6設定も見ます。

## IPv6でだけ一部サイトが遅い

確認します。

```sh
nslookup -type=A target.example
nslookup -type=AAAA target.example
ping -c 4 target.example
ping6 -c 4 target.example
traceroute6 target.example 2>/dev/null || true
logread | grep -Ei 'dnsmasq|wan6|odhcp6c|mtu|packet' | tail -n 120
```

可能性です。

- IPv6経路の問題
- DNSでAAAAを引いた後の経路問題
- MTU / PMTUD
- CDN側の経路
- 端末側のHappy Eyeballsの挙動
- firewallでICMPv6を止めすぎている

最初から設定変更せず、まずIPv4とIPv6で差が出るかを見ます。

## ゲストWi-FiでIPv6だけLANへ届く気がする

確認します。

```sh
uci show wireless | grep -E 'guest|ssid|network'
uci show network | grep -Ei 'guest|ip6|delegate|ip6assign'
uci show dhcp | grep -Ei 'guest|ra|dhcpv6'
uci show firewall | grep -Ei 'guest|lan|ip6|family|forwarding'
```

見るポイントです。

- Guest SSIDがguest networkに紐づいているか
- guestからlanへのIPv6 forwardingを作っていないか
- guest zoneのForwardが `REJECT` か
- IPv4では分離できているがIPv6だけ抜けていないか

実端末で、IPv4とIPv6の両方からLAN内機器へ届かないことを確認します。

## VPNでIPv6が漏れる気がする

WireGuardの場合:

```sh
wg show 2>/dev/null || true
uci show network | grep -Ei 'wireguard|wg|ip6|allowed'
ip route show
ip -6 route show
```

Tailscaleの場合:

```sh
tailscale status 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'DNS|ExitNode|Route|AdvertiseRoutes' || true
ip route show
ip -6 route show
```

見るポイントです。

- VPNでIPv6も通す設計か
- IPv6は通常回線へ出す設計か
- Exit Nodeを使っているか
- DNSだけVPN経由で、通信は通常経路になっていないか

VPN記事と合わせて、目的に合う設計にします。

## 運用メモのテンプレート

IPv6 / IPoEまわりは、後から見返すと忘れやすいです。

設定したら、次のようなメモを残しておきます。

```txt
回線:
  NTTフレッツ / その他

接続:
  ONU直下 / HGW配下

IPv4:
  DHCP / PPPoE / IPv4 over IPv6

IPv6:
  IPoE
  wan6: up
  prefix delegation: あり / なし

IPv4 over IPv6:
  OCNバーチャルコネクト / transix / v6プラス / クロスパス / 不明
  方式: MAP-E / DS-Lite / IPIP / 不明

LAN IPv6:
  lan: 配布する / しない
  guest: 配布する / しない
  kids: 配布する / しない
  iot: 配布する / しない

firewall:
  wan input: REJECT
  wan forward: REJECT
  guest -> lan: reject
  iot -> lan: reject

DNS:
  Adblock: あり / なし
  Family DNS: あり / なし
  DoH対策: あり / なし

VPN:
  Tailscale / WireGuard / なし
  IPv6 routing: 使う / 使わない

確認:
  ping IPv4 OK
  ping6 IPv6 OK
  A/AAAA DNS OK

バックアップ:
  backup-LN6001-before-ipv6-change-20260621.tar.gz
```

このメモがあるだけで、IPoE、VPN、DNS、firewallの切り分けがかなり楽になります。

## まとめ

IPv6 / IPoE環境では、IPv4のNAT前提とは見え方が変わります。

大事なのは、次のポイントです。

1. IPv4とIPv6を分けて確認する
2. IPoEとIPv4 over IPv6を混同しない
3. IPv6ではNAT感覚ではなくfirewallを重視する
4. WAN6、default route、Prefix Delegationを見る
5. LAN側RA / DHCPv6で端末へIPv6が配られているか見る
6. ゲストWi-Fi、IoT、KidsでもIPv6側の分離を考える
7. DNS広告ブロックやFamily DNSではIPv6側DNSやDoHも確認する
8. VPNではIPv6 leakやExit Nodeの挙動を確認する
9. ICMPv6を雑に止めない
10. IPv6を全部切るより、どこへ配るかを設計する

最初からIPv6を完璧に理解する必要はありません。

まずは、

```txt
ifstatus wan と ifstatus wan6 を分ける
ping と ping6 を分ける
AレコードとAAAAレコードを分ける
IPv6 firewallも見る
```

この4つだけでも、IPoE環境のトラブル切り分けはかなり楽になります。

IPv6は怖いものではありません。

でも、IPv4と同じ顔をした別の道です。

だから、別々に見てあげる。

それくらいの距離感がちょうどいいと思います。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-03.png)

## 次に読むなら

IPv6の落とし穴を押さえたら、次は目的に合わせて進みます。

- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [ポート開放の注意点](https://note.com/ikmsan/n/n765ffea6ea82)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

IPoE方式そのものを整理したい人は014の記事へ。

IPv6側で意図せず公開していないか不安な人は、firewall zonesとポート開放の記事へ。

VPNでIPv6が絡む人は、WireGuard / Tailscaleの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

OpenWrt系のfirewall表示は、バージョンやターゲットによって変わります。LN6001-JPでは `iptables-save`、`ip6tables-save`、iptables / ip6tables互換表示を確認します。

この記事では、まず `uci show firewall`、`iptables-save`、`ip6tables-save` を中心に確認し、必要に応じて `ip6tables` も確認用として使う方針にしています。

IPoE / IPv4 over IPv6、PPPoE、固定IP、IPIP、Prefix Delegation、RA / DHCPv6の挙動は、回線契約、プロバイダ、HGW / ONU構成、ファームウェア更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポート、OpenWrtの最新ドキュメント、契約中のプロバイダ情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### IPv6だけつながらない時は何を見ればいい？

まず `ifstatus wan6`、`ip -6 route show`、`ping6 -c 4 2001:4860:4860::8888` を見ます。

WAN6がupか、IPv6 default routeがあるか、IPv6疎通があるかを分けて確認します。

### IPv4がつながっていればIPv6は気にしなくていい？

IPoE環境では、IPv4とIPv6が別の見え方をするため、分けて確認したほうがよいです。

IPv4だけ正常、IPv6だけ正常、DNSだけ不調、という状態がありえます。

### IPv6ではポート開放は不要？

IPv4のようなDNAT型のポートフォワードとは違います。

IPv6では、LAN内端末にグローバルIPv6アドレスが付くことがあるため、外部からの通信はIPv6 firewallで許可するかどうかを見ます。

### IPv6はNATがないから危険？

NATがないこと自体が危険というより、firewall設計を見落とすことが危険です。

WAN側からのInputやForwardが意図どおり拒否されているかを確認します。

### Prefix Delegationは何のために見る？

プロバイダからLAN内へ配るためのIPv6アドレス範囲を受け取れているかを見るためです。

WAN6がupでも、Prefix Delegationが取れていないとLAN端末へIPv6が配れない場合があります。

### ゲストWi-FiにもIPv6を配っていい？

配ることはできます。

ただし、guest → lanがIPv6側でも拒否されているか、guest zoneのfirewallが意図どおりかを確認してください。

迷う場合は、まずIPv4でゲストWi-Fiを安定させ、IPv6は別途設計してから有効化するほうが安全です。

### IPv6を無効化すればトラブルは減る？

短期的な切り分けとして一時的に止めることはあります。

ただし、日本のIPoE / IPv4 over IPv6環境ではIPv6が土台になっていることがあります。

恒久的に無効化するより、WAN6、Prefix Delegation、firewall、DNSを分けて確認するほうがおすすめです。

### DNS広告ブロックやFamily DNSはIPv6でも効く？

構成によります。

IPv4側だけDNSを配っていると、IPv6側DNSやDoHで抜け道になる場合があります。

IPv6側のDNS方針、DoH、VPNのDNS設定も確認してください。

## 参考リンク

- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt IPv6 configuration: https://openwrt.org/docs/guide-user/network/ipv6/configuration
- OpenWrt firewall overview: https://openwrt.org/docs/guide-user/firewall/overview
- OpenWrt firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt odhcpd documentation: https://openwrt.org/docs/techref/odhcpd

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
