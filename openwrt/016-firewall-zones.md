<!-- mirror-source: articles/016-firewall-zones.md -->

# firewall zoneは“部屋分け”で考える｜Guest・IoT・VPNを安全に通す基本【OpenWrt集中連載016】

OpenWrtでゲストWi-Fi、IoT用SSID、VLAN、VPNを作り始めると、最後に必ず出てくるのが **firewall zone** です。

名前がちょっと怖いですよね。

firewall。  
zone。  
Input。  
Output。  
Forward。  
Traffic Rules。

急にネットワーク機器っぽさが強くなります。

でも、最初の理解はもっとラフで大丈夫です。

firewall zoneは、ざっくり言うと **部屋分け** です。

```txt
lan    = 家族や社内端末の部屋
guest  = ゲストWi-Fiの部屋
iot    = スマート家電の部屋
device = カメラや設備機器の部屋
vpn    = 外から入ってくる専用通路
wan    = インターネットへの出口
```

そしてfirewallは、

```txt
どの部屋から、どの部屋へ行っていいか
```

を決める仕組みです。

ゲストWi-Fiはインターネットへ出ていい。  
でもNASやプリンターの部屋には入れない。

IoT機器はクラウドへ出ていい。  
でも親のPCや社内PCへ勝手に入らない。

VPNは便利だけど、LAN全体へ広く入れるのではなく、NASだけ、LuCIだけ、管理PCだけに絞る。

こう考えると、firewall zoneはそこまで怖くありません。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、LAN、Guest、IoT / Device、VPNをどう分けるか、Input / Output / Forwardをどう読めばよいか、そして壊しにくく設定する順番をまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、firewallを完全に理解することではありません。

まずは、ここまで分かればOKです。

- firewall zoneを「部屋分け」として考えられる
- Input / Output / Forwardの違いがざっくり分かる
- Guest、IoT / Device、VPNの基本方針を決められる
- GuestやIoTでDHCP / DNS許可が必要な理由が分かる
- LuCIでzoneとforwardingを確認できる
- UCIでfirewall設定を読む・控える・小さく変更できる
- 設定後に「届くべきところ」と「届かないべきところ」を確認できる
- firewallで迷子になった時の戻し方が分かる

firewallで大事なのは、いきなり細かいルールを大量に作ることではありません。

まず、

```txt
GuestからLANへ入れない
IoTからLANへ勝手に入れない
VPNから入れる範囲を広げすぎない
```

ここを作るだけでかなり実用的です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-01.png)

## 先にざっくり結論

firewall zoneは、ネットワークごとに通信ルールをまとめる場所です。

最初は、この方針で十分です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-01.png)

特に大事なのは、GuestやIoTの **Input** を `REJECT` にした時です。

Inputを `REJECT` にすると、そのzoneからルーター自身への通信も拒否されます。

つまり、そのままだと次も止まります。

```txt
Guest端末 → ルーターのDHCP
Guest端末 → ルーターのDNS
```

これを忘れると、

```txt
ゲストWi-FiにはつながるのにIPが取れない
IPはあるのにWebサイトが開けない
```

という状態になります。

なので、GuestやIoTでは、DHCPとDNSだけ明示的に許可します。

```txt
Allow-Guest-DHCP
Allow-Guest-DNS
Allow-Device-DHCP
Allow-Device-DNS
```

ここが分かると、ゲストWi-Fi、VLAN、VPNの記事がかなり読みやすくなります。

## こういう人向けです

この記事は、次のような人向けです。

- ゲストWi-Fiを家庭内LANや業務LANから分けたい
- IoT機器やカメラを親のPCやNASから分けたい
- VPNで入った端末のアクセス範囲を絞りたい
- Input / Output / Forwardが何を意味するのか分かりにくい
- Traffic Rulesを増やす前に、通信方針を整理したい
- VLANやゲストWi-Fiを作ったあと、本当に分離できているか確認したい
- firewall設定でLuCIに戻れなくなるのが怖い

逆に、すでにOpenWrtのUCI / iptables / zone設計に慣れている人には基本寄りです。

ただ、家庭や小さなオフィス、店舗では、この基本が一番効きます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-02.png)

## 最初に言葉だけそろえる

firewallまわりの言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-02.png)

最初は、これだけ覚えれば大丈夫です。

```txt
zone = 部屋
forwarding = 部屋から部屋への通行許可
Traffic Rules = 例外ルール
```

## Input / Output / Forwardはこう見る

firewall zoneで混乱しやすいのが、Input、Output、Forwardです。

ポイントは、**ルーター自身から見た向き**で考えることです。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-03.png)

たとえば、guest zoneをこう設定したとします。

```txt
Input: REJECT
Output: ACCEPT
Forward: REJECT
Forwarding: guest → wan
```

この意味はこうです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-04.png)

ここ、かなり大事です。

Guest zoneのInputを `REJECT` にすると、管理画面やSSHを閉じられます。

でも、DHCPやDNSまで一緒に止まります。

だからDHCPとDNSだけ、Traffic Rulesで開けます。

## デフォルトのlan / wanを理解する

OpenWrt系では、基本的に `lan` と `wan` のzoneから始まります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-05.png)

さらに、`lan → wan` のforwardingがあることで、LAN端末はインターネットへ出られます。

ざっくり言うと、初期状態はこうです。

```txt
LAN端末 → ルーター自身: 許可
LAN端末 → インターネット: 許可
インターネット → ルーター自身: 原則拒否
インターネット → LAN端末: 原則拒否
```

家庭や小さなオフィスでは、この `lan` に親のPC、スマートフォン、NAS、プリンター、管理端末などが入ります。

そこへあとから、Guest、IoT、Device、VPNを追加していくイメージです。

## zone設計の基本

新しいzoneを作る前に、まず通信方針を決めます。

firewall設定で一番大事なのは、コマンドを覚えることではありません。

```txt
何を通して
何を通さないか
```

を先に決めることです。

## 家庭向けの例

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-06.png)

家庭では、最初から全部を作らなくて大丈夫です。

まずGuest。  
必要になったらIoT。  
子ども用端末を分けたくなったらKids。  
外出先から入りたくなったらVPN。

この順番で十分です。

## 小さなオフィス・店舗向けの例

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-07.png)

店舗では、GuestからStaffへ届かないことがかなり大事です。

DeviceからStaffへ勝手に入らないことも大事です。

POSや決済端末は、カメラやスマート家電と同じノリでDeviceへ混ぜないほうが安全です。

ベンダー要件を先に確認してください。

## 通信方針を表にする

設定前に、こういう表を作っておくと迷いにくくなります。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-08.png)

この表があると、あとから

```txt
このルール、何のために作ったんだっけ？
```

となりにくいです。

firewall設定は、半年後の自分がかなり忘れます。

メモは本当に大事です。

## 設定前にバックアップを取る

firewall設定は、間違えるとインターネットへ出られなくなったり、LuCIへ戻れなくなったりします。

なので、触る前に必ずバックアップを取ります。

LuCIでは次です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-firewall-zones-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/firewall-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
iptables-save > "$BACKUP_DIR/iptables-save.txt" 2>/dev/null || true
ip6tables-save > "$BACKUP_DIR/ip6tables-save.txt" 2>/dev/null || true

logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、SSID、Wi-Fiパスワード、IPアドレス、firewall設定、端末情報が含まれることがあります。

そのままSNS、公開リポジトリ、記事スクリーンショットへ出さないでください。

OpenWrtで一番強い人は、firewallを全部暗記している人ではありません。

戻せる人です。

## LuCIでguest zoneを作る

ここでは、すでに `guest` networkとゲストSSIDを作っている前提で、guest zoneを作ります。

ゲストWi-Fiの作成手順は007の記事で扱っています。

## ステップ1: guest zoneを追加する

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-09.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-10.png)

6. **Save** をクリックする

この設定で、guestからwanへは出られます。

一方で、guestからlanへは転送しません。

つまり、

```txt
ゲストWi-Fi → インターネット: OK
ゲストWi-Fi → LAN内NAS: NG
ゲストWi-Fi → 管理画面: NG
```

を目指す構成です。

ただし、このままだとDHCPとDNSも使えない可能性があります。

なので次に、DHCPとDNSだけ許可します。

## ステップ2: DHCPとDNSだけ許可する

guest zoneのInputを `REJECT` にした場合、ゲスト端末からルーター自身への通信は原則拒否されます。

ただし、ゲスト端末には最低限この2つが必要です。

```txt
DHCP = IPアドレスをもらう
DNS  = example.com のような名前を解決する
```

この2つだけ開けます。

## DHCPを許可する

1. **Network** → **Firewall** → **Traffic Rules** を開く
2. **Add** をクリックする
3. 次のように設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-11.png)

4. **Save** をクリックする

## DNSを許可する

1. **Traffic Rules** で **Add** をクリックする
2. 次のように設定する

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-12.png)

3. **Save** をクリックする

これで、ゲスト端末はIPアドレスを取得し、DNSを使えます。

一方で、LuCIやSSHなどの管理系通信は許可していないため、ゲスト端末から管理画面へ入りにくい構成になります。

## ステップ3: Save & Applyする

設定が終わったら、LuCI上部の保留中の変更を確認し、**Save & Apply** をクリックします。

firewall反映後、Wi-Fiや通信が一時的に不安定に見えることがあります。

すぐにリセットボタンを押さなくて大丈夫です。

まず、有線LANの管理端末からLuCIへ入れるか確認します。

## GuestからLANへ入れないことを確認する

firewallは、設定画面だけ見ても安心できません。

必ず実端末で確認します。

ゲストWi-Fiに接続したスマートフォンやPCで、次を確認します。

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-13.png)

PCから確認する例です。

```sh
ping -c 4 8.8.8.8
ping -c 4 example.com
ping -c 4 192.168.1.1
ssh root@192.168.1.1
```

期待する状態はこうです。

```txt
8.8.8.8 / example.com → 通る
192.168.1.1 / NAS / SSH → 通らない
```

「インターネットへ出られる」だけでは不十分です。

「LANへ届かない」ことまで確認します。

## IoT / Device zoneを作る考え方

IoT機器やカメラも、guestと似た考え方で分けられます。

ただし、GuestとIoT / Deviceでは通信要件が違います。

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-14.png)

IoT機器は、メーカーのクラウドへ出る必要があるものもあります。

一方で、LAN内の録画機だけに接続できればよいカメラもあります。

なので、Device zoneでは次の方針が分かりやすいです。

- IoT / DeviceからLANへは原則拒否
- IoT / DeviceからWANへは機器要件に応じて許可
- LAN / StaffからIoT / Deviceへは、必要な機器だけ許可
- カメラや録画機はDHCP予約でIPを固定する
- POSや決済端末はDeviceへ雑に混ぜない

Device zoneもInputを `REJECT` にする場合、DHCPとDNSの許可が必要です。

```txt
Allow-Device-DHCP
Allow-Device-DNS
```

この2つを忘れると、IoT機器がIPを取れなかったり、クラウドへ出られなかったりします。

## VPN zoneを作る考え方

VPNは、外出先から内側へ入る入口です。

便利ですが、アクセス範囲を広げすぎないことが大事です。

ありがちな失敗はこれです。

```txt
VPN → LAN全体を許可
VPN → Device全体も許可
VPN → 管理画面も全部許可
```

自分専用ならまだしも、家族やスタッフの端末もVPNへ入る場合は、少し広すぎることがあります。

最初は、目的ごとに小さく許可します。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-15.png)

VPN zoneを作る場合は、`vpn → lan` を広く許可する前に、どの宛先が本当に必要かを決めます。

WireGuardやTailscaleの記事と合わせて確認してください。

## Traffic Rulesは“例外”に使う

Traffic Rulesは、zone間の大きな方針を作ったあと、必要な通信だけを例外として許可するために使います。

たとえば、こんな用途です。

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-16.png)

最初からTraffic Rulesを大量に作ると、あとで自分が混乱します。

まずはzone間の大きな方針。

そのあと、必要な通信だけTraffic Rulesで追加。

この順番が安全です。

## UCIでguest zoneを作る例

ここからはCLIで作りたい人向けです。

最初はLuCIで設定するほうが安全です。

UCIで作る場合も、まず現在の状態を確認します。

```sh
echo "### current firewall"
uci show firewall

echo "### current networks"
uci show network | grep -E 'interface|device|proto|ipaddr'

echo "### current wireless mapping"
uci show wireless | grep -E 'ssid|network'
```

以下は、`guest` networkがすでに存在する前提で、guest zoneとDHCP / DNS許可を作る例です。

```sh
BACKUP_DIR="/root/firewall-guest-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

# guest zone
uci -q delete firewall.guest
uci set firewall.guest="zone"
uci set firewall.guest.name="guest"
uci set firewall.guest.network="guest"
uci set firewall.guest.input="REJECT"
uci set firewall.guest.output="ACCEPT"
uci set firewall.guest.forward="REJECT"

# guest -> wan
uci -q delete firewall.guest_wan
uci set firewall.guest_wan="forwarding"
uci set firewall.guest_wan.src="guest"
uci set firewall.guest_wan.dest="wan"

# DHCP
uci -q delete firewall.guest_dhcp
uci set firewall.guest_dhcp="rule"
uci set firewall.guest_dhcp.name="Allow-Guest-DHCP"
uci set firewall.guest_dhcp.src="guest"
uci set firewall.guest_dhcp.proto="udp"
uci set firewall.guest_dhcp.src_port="68"
uci set firewall.guest_dhcp.dest_port="67"
uci set firewall.guest_dhcp.family="ipv4"
uci set firewall.guest_dhcp.target="ACCEPT"

# DNS
uci -q delete firewall.guest_dns
uci set firewall.guest_dns="rule"
uci set firewall.guest_dns.name="Allow-Guest-DNS"
uci set firewall.guest_dns.src="guest"
uci set firewall.guest_dns.proto="tcp udp"
uci set firewall.guest_dns.dest_port="53"
uci set firewall.guest_dns.target="ACCEPT"

echo "### pending firewall changes"
uci changes firewall

uci commit firewall
/etc/init.d/firewall restart

echo "### verify"
uci show firewall | grep -E 'guest|Allow-Guest'
logread | grep -Ei 'firewall|guest|reject|drop' | tail -n 80
```

この例では、`guest → lan` のforwardingを作っていません。

そのため、guestからlanへは原則届かない構成になります。

## UCIでDevice zoneを作る例

Deviceネットワークを作っている場合も同じ考え方です。

ここでは `device` networkがすでに存在する前提です。

```sh
BACKUP_DIR="/root/firewall-device-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/firewall "$BACKUP_DIR/firewall"
uci show firewall > "$BACKUP_DIR/firewall.uci.txt"

# device zone
uci -q delete firewall.device
uci set firewall.device="zone"
uci set firewall.device.name="device"
uci set firewall.device.network="device"
uci set firewall.device.input="REJECT"
uci set firewall.device.output="ACCEPT"
uci set firewall.device.forward="REJECT"

# device -> wan
# クラウド接続が必要な機器がある場合だけ有効化する
uci -q delete firewall.device_wan
uci set firewall.device_wan="forwarding"
uci set firewall.device_wan.src="device"
uci set firewall.device_wan.dest="wan"

# DHCP
uci -q delete firewall.device_dhcp
uci set firewall.device_dhcp="rule"
uci set firewall.device_dhcp.name="Allow-Device-DHCP"
uci set firewall.device_dhcp.src="device"
uci set firewall.device_dhcp.proto="udp"
uci set firewall.device_dhcp.src_port="68"
uci set firewall.device_dhcp.dest_port="67"
uci set firewall.device_dhcp.family="ipv4"
uci set firewall.device_dhcp.target="ACCEPT"

# DNS
uci -q delete firewall.device_dns
uci set firewall.device_dns="rule"
uci set firewall.device_dns.name="Allow-Device-DNS"
uci set firewall.device_dns.src="device"
uci set firewall.device_dns.proto="tcp udp"
uci set firewall.device_dns.dest_port="53"
uci set firewall.device_dns.target="ACCEPT"

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

DeviceからWANへ出したくない場合は、`firewall.device_wan` を作らない、またはLuCIでforwardingを外します。

防犯カメラ、録画機、スマート家電は、WAN接続が必要かどうかを機器ごとに確認してください。

## StaffからDeviceへ必要な通信だけ許可する

カメラや録画機をDeviceネットワークへ置いた場合、Staff側から管理画面へ入りたいことがあります。

この時、device → lanを許可する必要はありません。

Staffから特定のDevice機器へだけ許可します。

例: Staff LANから録画機 `192.168.3.20` のWeb管理画面だけ許可する。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-staff-device.$(date +%Y%m%d-%H%M)

uci -q delete firewall.staff_to_nvr
uci set firewall.staff_to_nvr="rule"
uci set firewall.staff_to_nvr.name="Allow-Staff-to-NVR-Web"
uci set firewall.staff_to_nvr.src="lan"
uci set firewall.staff_to_nvr.dest="device"
uci set firewall.staff_to_nvr.dest_ip="192.168.3.20"
uci set firewall.staff_to_nvr.proto="tcp"
uci set firewall.staff_to_nvr.dest_port="80 443"
uci set firewall.staff_to_nvr.target="ACCEPT"

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

カメラや録画機によって必要なポートは違います。

ベンダーのマニュアルや実機の通信要件を確認してください。

## VPNからNASだけ許可する例

VPNからLAN全体ではなく、NASだけ使わせたい場合の考え方です。

例: VPN zoneからNAS `192.168.1.10` だけ許可する。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-vpn-nas.$(date +%Y%m%d-%H%M)

uci -q delete firewall.vpn_to_nas
uci set firewall.vpn_to_nas="rule"
uci set firewall.vpn_to_nas.name="Allow-VPN-to-NAS"
uci set firewall.vpn_to_nas.src="vpn"
uci set firewall.vpn_to_nas.dest="lan"
uci set firewall.vpn_to_nas.dest_ip="192.168.1.10"
uci set firewall.vpn_to_nas.proto="tcp"
uci set firewall.vpn_to_nas.dest_port="22 80 443 445"
uci set firewall.vpn_to_nas.target="ACCEPT"

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

この例では、NASのSSH、Web管理画面、SMBを想定しています。

実際に必要なポートだけ残してください。

VPNからLAN全体へforwardingを作る場合は便利ですが、アクセス範囲は広くなります。

最初はNASだけ、LuCIだけ、管理PCだけのように狭く始めるのがおすすめです。

## 設定後に確認するコマンド

firewall設定後は、LuCIだけでなくCLIでも状態を読みます。

```sh
echo "### UCI firewall"
uci show firewall

echo "### zones and forwarding"
uci show firewall | grep -E 'zone|forwarding|name|network|src|dest|target|input|output|forward'

echo "### iptables rules"
iptables-save 2>/dev/null | head -n 200 || true

echo "### ip6tables rules"
ip6tables-save 2>/dev/null | head -n 200 || true

echo "### firewall logs"
logread | grep -Ei 'firewall|drop|reject|guest|device|vpn' | tail -n 120
```

LN6001-JPでは、まずUCI設定と `iptables-save`、`ip6tables-save` の表示を見ます。

古い記事では `iptables -L -n -v` を使う例もありますが、まずは `uci show firewall`、`iptables-save`、`ip6tables-save` を中心に見るほうが分かりやすいです。

互換レイヤーや環境によって `iptables` が見える場合もあります。

確認用として使う場合は、次のように失敗しても止まらない形にします。

```sh
iptables -L -n -v 2>/dev/null || true
iptables-save 2>/dev/null | head -n 120 || true
```

## 実端末で確認する

firewallは、設定画面だけ見ても安心できません。

実際の端末で確認します。

## Guest端末で確認する

Guest Wi-Fiに接続した端末から確認します。

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-17.png)

## Device端末で確認する

Deviceネットワークにつないだ機器で確認します。

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-18.png)

## VPN端末で確認する

VPN接続端末で確認します。

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-19.png)

ここで大事なのは、つながる確認だけではありません。

```txt
届くべきところに届く
届かないべきところに届かない
```

この両方を確認します。

## よくある失敗と対処

## Guest端末がIPアドレスを取れない

よくある原因は、DHCPを許可していないことです。

確認します。

```sh
uci show dhcp.guest
uci show firewall | grep -E 'guest|Allow-Guest-DHCP'
logread | grep -i dnsmasq | tail -n 80
cat /tmp/dhcp.leases
```

対処:

- guest interfaceでDHCP Serverが有効か確認する
- `Allow-Guest-DHCP` を作る
- Source zoneが `guest` になっているか確認する
- Source port `68`、Destination port `67` を確認する
- 端末のWi-Fiを切断して再接続する

## Guest端末はIPを取るがWebサイトが開けない

DNSまたはguest → wan forwardingを確認します。

```sh
ifstatus guest
ifstatus wan
uci show firewall | grep -E 'guest|Allow-Guest-DNS|forwarding'
logread | grep -Ei 'dnsmasq|firewall' | tail -n 100
```

判断の目安です。

```txt
8.8.8.8へping OK / example.com NG
→ DNS問題

8.8.8.8へping NG / example.com NG
→ guest → wan forwardingやWAN側を確認
```

## GuestからLANへ届いてしまう

これは分離としては失敗です。

確認します。

```sh
uci show wireless | grep -E 'guest|ssid|network'
uci show firewall | grep -E 'guest|lan|forwarding'
```

見るポイントです。

- Guest用SSIDのNetworkが `lan` になっていないか
- guest → lan forwardingを作っていないか
- Traffic Ruleでguestからlanを許可していないか
- guest zoneのForwardが `ACCEPT` になっていないか

## GuestからLuCIが開けてしまう

guest zoneのInputが広すぎる可能性があります。

確認します。

```sh
uci show firewall | grep -A8 -E "name='guest'|name=\"guest\""
uci show firewall | grep -E '80|443|22|guest'
```

対処:

- guest zoneのInputを `REJECT` にする
- DHCP / DNSだけTraffic Rulesで許可する
- 80 / 443 / 22をguestから許可するルールがないか確認する

## LAN端末がインターネットへ出られなくなった

`lan → wan` forwardingが消えていないか確認します。

```sh
uci show firewall | grep -E 'forwarding|src|dest'
ifstatus wan
ip route show default
```

`lan → wan` がない場合は、LuCIでForwardingを戻すか、CLIで追加します。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-restore-lan-wan.$(date +%Y%m%d-%H%M)

uci add firewall forwarding
uci set firewall.@forwarding[-1].src="lan"
uci set firewall.@forwarding[-1].dest="wan"

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

## LuCIへ入れなくなった

LAN側Inputを誤って `REJECT` にしたり、管理端末が別ネットワークへ移ったりした可能性があります。

まず、有線LANでLN6001-JPのLANポートへ直接つなぎます。

それでも入れない場合、SSHで入れるならバックアップから戻します。

```sh
cp /root/firewall-before-YYYYMMDD-HHMM/firewall /etc/config/firewall
/etc/init.d/firewall restart
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えます。

LuCIバックアップを使う場合は、**System** → **Backup / Flash Firmware** から復元します。

本当に戻れない場合は、リセットと復旧の記事を参照してください。

ここで焦ってリセットボタンを押したくなります。

分かります。

でも、まだ有線LAN、管理IP、バックアップを確認する余地があります。

## Device機器がクラウドへつながらない

Device zoneからWANへ出る必要がある機器なのに、device → wan forwardingがない可能性があります。

確認します。

```sh
ifstatus device
uci show firewall | grep -E 'device|forwarding|Allow-Device'
logread | grep -Ei 'dnsmasq|firewall|device' | tail -n 100
```

対処:

- device → wan forwardingを許可するか検討する
- Device用DNS許可を確認する
- AdblockやDNS制御が強すぎないか確認する
- 機器のクラウド接続要件を確認する

カメラやスマート家電は、メーカーのクラウドへ出る必要があることがあります。

「DeviceはWAN禁止」と決め打ちせず、機器要件を見て決めます。

## firewall変更の進め方

firewallは、段階的に進めるのが安全です。

おすすめの順番です。

1. 現在のlan / wan設定を確認する
2. LuCIバックアップを取る
3. Guest networkとguest SSIDを作る
4. guest zoneを作る
5. DHCP / DNSを許可する
6. guest → wanを許可する
7. guest → lanが届かないことを確認する
8. Device / IoTを追加する
9. VPNのアクセス範囲を必要最小限にする
10. 変更履歴を残す

一気に完成形を作るより、1つ作って確認するほうがトラブル時に戻りやすくなります。

OpenWrtでは、強い設定より、戻れる設定のほうが強いです。

## 設定メモのテンプレート

firewall設定を作ったら、次のようなメモを残しておくと便利です。

```txt
firewall zones:

lan:
  input: ACCEPT
  output: ACCEPT
  forward: ACCEPT
  forwarding: lan -> wan

guest:
  input: REJECT
  output: ACCEPT
  forward: REJECT
  forwarding: guest -> wan
  rules:
    Allow-Guest-DHCP
    Allow-Guest-DNS
  blocked:
    guest -> lan
    guest -> LuCI/SSH

device:
  input: REJECT
  output: ACCEPT
  forward: REJECT
  forwarding:
    device -> wan: allowed / disabled
  rules:
    Allow-Device-DHCP
    Allow-Device-DNS
    Allow-Staff-to-NVR-Web

vpn:
  purpose:
    remote admin
  allowed:
    VPN -> NAS only
    VPN -> LuCI only if needed

backup:
  backup-LN6001-before-firewall-zones-20260621.tar.gz
```

半年後に見ると、firewall設定はかなり忘れます。

「なぜこのルールを作ったか」を残しておくと、未来の自分が助かります。

## まとめ

firewall zoneは、OpenWrt系ルーターでネットワークを分けるための中心です。

SSIDやVLANでネットワークを分けても、zoneとforwardingを設計しないと、通信方針は完成しません。

最初に押さえるポイントは次の通りです。

1. zoneは通信ルールをまとめる部屋
2. Inputは「そのzoneからルーター自身へ」
3. Forwardは「そのzoneから別zoneへ」
4. GuestやIoTのInputを `REJECT` にしたらDHCP / DNSを許可する
5. Guest → WANは許可、Guest → LANは拒否
6. IoT / Device → LANは原則拒否
7. VPN → LANは必要な宛先だけ許可
8. Traffic Rulesは例外に使う
9. 設定前に必ずバックアップを取る
10. 設定後は「届くべきところ」と「届かないべきところ」を両方確認する

最初から細かなルールを大量に作る必要はありません。

まずはGuestからLANへ届かない。

次にIoT / Deviceを分ける。

最後にVPNのアクセス範囲を絞る。

この順番で進めると、家庭・小さなオフィス・店舗でも壊しにくく運用できます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-03.png)

## 次に読むなら

firewall zoneの考え方が見えてきたら、次は目的に合わせて進みます。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)

GuestやIoTをまだ作っていない人は、ゲストWi-FiやVLANの記事へ。

VPNからどこまで入れるかを整理したい人は、WireGuard / Tailscaleの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

OpenWrt系のfirewall表示は、バージョンやターゲットによって変わります。LN6001-JPでは `iptables-save`、`ip6tables-save`、iptables互換表示を確認します。

この記事では、まず `uci show firewall` で設定を確認し、必要に応じて `iptables-save` や `ip6tables-save` を読む方針にしています。

LuCIの画面名、firewall設定項目、コマンド出力は、ファームウェア更新や追加モジュールで変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### firewall zoneは何のために使う？

LAN、Guest、IoT、VPNなど、役割の違うネットワーク同士の通信ルールを分けるために使います。

たとえば、Guestからインターネットへは出すが、LAN内NASやプリンターへは入れない、といった制御ができます。

### Input、Output、Forwardの違いは？

ルーター自身から見た向きです。

Inputはzoneからルーター自身へ入る通信、Outputはルーター自身からzoneへ出る通信、Forwardはzoneから別zoneへ通過する通信です。

### Guest zoneのInputはREJECTでいい？

多くの場合、GuestではInputを `REJECT` にするのが分かりやすいです。

ただし、そのままだとDHCPやDNSも拒否されるため、`Allow-Guest-DHCP` と `Allow-Guest-DNS` を追加します。

### GuestからLANへ届かないようにするには？

guest → lan のforwardingを作らないこと、guest zoneのForwardを `REJECT` にすること、Guest用SSIDが `guest` networkに紐づいていることを確認します。

実端末から `192.168.1.1` やNASへ届かないことも確認してください。

### IoTとGuestは同じzoneでいい？

分けたほうが管理しやすいです。

Guestは一時利用端末向け、IoTはスマート家電やカメラ向けで、必要な通信が違います。

特に、Staffからカメラや録画機へ管理アクセスしたい場合は、Device zoneとして分けると整理しやすいです。

### VPNはlan zoneに入れていい？

自分だけが使う小さな環境では動きますが、アクセス範囲が広くなりやすいです。

VPN専用zoneを作り、NASだけ、LuCIだけ、Staff LANだけのように必要な範囲へ絞る設計も検討してください。

### firewall確認は何を見ればいい？

LN6001-JPでは、`iptables-save` と `ip6tables-save` を見るほうが実機に合わせやすいです。

まず `uci show firewall` で設定を確認し、必要に応じて `iptables-save` や `ip6tables-save` を使います。

古い記事の `iptables -L -n -v` は、互換表示として見える場合もありますが、それだけに依存しないほうが安全です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- OpenWrt Wiki - Firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt Wiki - Firewall overview: https://openwrt.org/docs/guide-user/firewall/overview
- OpenWrt Wiki - Guest Wi-Fi basics: https://openwrt.org/docs/guide-user/network/wifi/guestwifi/guest-wlan
- OpenWrt Wiki - Guest Wi-Fi using LuCI: https://openwrt.org/docs/guide-user/network/wifi/guestwifi/configuration_webinterface

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
