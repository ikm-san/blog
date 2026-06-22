<!-- mirror-source: articles/029-store-configuration.md -->

# 店舗のWi-Fi、全部同じで大丈夫？｜Staff・Guest・Deviceを分ける小規模店舗ネットワーク例【OpenWrt集中連載029】

カフェ。  
小売店。  
美容室。  
サロン。  
小さなクリニック。  
個人事務所。

こういう小規模店舗のネットワークって、気づくと全部混ざります。

スタッフPC。  
業務タブレット。  
POS。  
レシートプリンター。  
監視カメラ。  
録画機。  
スマートロック。  
予約端末。  
ゲストWi-Fi。  
スタッフのスマートフォン。

最初は、全部同じWi-Fiでも動きます。

むしろ最初はそのほうが楽です。  
SSIDは1つ。  
パスワードも1つ。  
プリンターも見える。  
カメラも見える。  
「とりあえず全部つながる」状態にはなります。

でも、あとから少し怖くなります。

```txt
ゲスト端末から業務PCが見えてない？
監視カメラがStaff LANと同じ場所にいて大丈夫？
POSや決済端末って、このネットワークでいいの？
ゲストWi-Fiのパスワードを変えたいけど、Staffにも影響する？
トラブル時に、どこから切り分ければいいの？
```

ここでいきなり本格的なVLAN設計やゼロトラストっぽい構成を作ろうとすると、だいたい大変です。

店舗では、ネットワークが止まると普通に業務が止まります。

決済できない。  
予約端末が動かない。  
スタッフPCがクラウドへ入れない。  
カメラが見えない。  
ゲストWi-FiだけのつもりがStaff側までおかしくなる。

これはしんどいです。

なので、小規模店舗で最初にやることは、高度なVLAN設計ではありません。

まず、**Staff / Guest / Deviceを分けること**です。

Staffは業務用。  
GuestはゲストWi-Fi。  
Deviceはカメラや設備機器。  
POSや決済端末は、ベンダー要件を確認して別扱い。

このくらいからで十分です。

LN6001-JPはOpenWrtベースなので、SSID、network、DHCP、firewall zone、VPNを組み合わせて、店舗向けの構成を作れます。

ただし、自由度が高いぶん、最初は小さく作るのが大事です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）を小規模店舗で使う場合の、現実的なStaff / Guest / Device構成例をまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、店舗ネットワークを一気に完璧にすることではありません。

まずは、ここまで分かればOKです。

- 小規模店舗でStaff / Guest / Deviceをどう分けるか分かる
- ゲストWi-Fiを業務LANから分離する考え方が分かる
- POS、決済端末、監視カメラを扱う時の注意点が分かる
- DHCP予約で業務機器のIPを固定する考え方が分かる
- 店舗でVPNを使う時にWireGuard / Tailscaleをどう選ぶか分かる
- DNS広告ブロックやフィルタリングを店舗で強くしすぎない理由が分かる
- 設定前後に取るべきバックアップと確認コマンドが分かる
- 店舗運用で残しておきたいネットワーク管理メモが分かる

店舗ネットワークで大事なのは、強い制限をいきなり入れることではありません。

```txt
GuestとStaffを混ぜない
Deviceを説明できる場所に置く
POSや決済端末は雑に触らない
```

まずはこの3つです。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-01.png)

## 先にざっくり結論

小規模店舗では、まず3系統で十分です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-01.png)

最初の成功条件は、次の3つです。

```txt
GuestからStaffへ届かない
GuestからDeviceへ届かない
Staffから必要な業務機器へアクセスできる
```

POSや決済端末は、Deviceへ雑に混ぜないほうが安全です。

ベンダー指定の通信要件、サポート条件、固定IP要件、専用ルーター要件がある場合があります。

必要なら、POS専用ネットワークとして別に設計します。

店舗ネットワークで最初に守りたいのは、これです。

```txt
ゲストWi-Fiと業務端末を混ぜない
```

これだけでも、トラブル対応と日常運用はかなり楽になります。

## こういう店舗向けです

この記事は、次のような店舗に向いています。

- カフェや美容室でゲストWi-Fiを出したい
- スタッフ用Wi-FiとゲストWi-Fiを分けたい
- 監視カメラや録画機を業務LANから少し分けたい
- POSやプリンターのIPアドレスを固定したい
- ゲストWi-FiからNASやプリンターへ入れないようにしたい
- 外出先から店舗の録画機やルーター状態を確認したい
- 固定IPやVPNを後から足したい
- いきなり大規模なVLAN設計までは作りたくない
- 小規模店舗でも、説明できるネットワーク構成にしたい

逆に、すでに管理スイッチ、複数AP、POS専用回線、監査要件、拠点間VPNまである環境では、この記事は入口寄りです。

その場合は、この記事の考え方をベースにしつつ、業務要件・ベンダー要件・セキュリティ要件を優先してください。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-02.png)

## 最初に言葉だけそろえる

店舗構成で出てくる言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-02.png)

最初は、これだけで大丈夫です。

```txt
Staff = 業務用
Guest = お客さん用
Device = カメラや設備用
POS = ベンダー要件を先に見る
```

ここが整理できると、設定がかなり読みやすくなります。

## 店舗ネットワークの基本方針

小規模店舗では、最初からすべての機器を厳密に分けようとすると大変です。

まずは、次の方針で十分です。

```txt
Staff:
  業務の中心。
  スタッフPC、管理端末、NAS、プリンター。

Guest:
  ゲストWi-Fi。
  インターネットだけ。
  StaffやDeviceには入れない。

Device:
  カメラ、録画機、設備機器。
  Staffから必要な機器だけ管理。
  DeviceからStaffへは原則入れない。

POS / Payment:
  ベンダー要件を確認。
  Deviceへ雑に混ぜない。
```

店舗で一番避けたいのは、ゲストWi-Fiと業務端末が同じネットワークにいる状態です。

ゲスト端末は、インターネットだけ使えれば十分です。

NASやプリンター、POS、管理PC、カメラ管理画面へ届く必要はありません。

```txt
ゲストWi-Fiは“親切なインターネット出口”
業務LANへの入口ではない
```

このくらいで考えると分かりやすいです。

## 店舗ネットワーク設計図

この記事では、次の構成を例にします。

```txt
インターネット
    │
ONU / HGW
    │
LN6001-JP
    │
    ├── Staff: 192.168.1.0/24
    │     SSID: Shop_Staff
    │     用途: スタッフPC、管理端末、NAS、プリンター
    │
    ├── Guest: 192.168.2.0/24
    │     SSID: Shop_Guest
    │     用途: ゲストWi-Fi
    │     方針: インターネットのみ
    │
    └── Device: 192.168.3.0/24
          SSID: Shop_Device
          用途: カメラ、録画機、スマートロック、設備機器
          方針: LANへは原則入れない
```

POSや決済端末は、この図のDeviceへ自動的に入れるのではなく、別枠で考えます。

ベンダーから、

```txt
このネットワーク構成にしてください
このポートを開けてください
このDNSを使ってください
専用ルーターを使ってください
固定IPにしてください
```

といった案内がある場合は、それを優先します。

店舗では「ネットワークとしてきれい」より「決済や業務が止まらない」ほうが大事です。

## まず現状を棚卸しする

設定を始める前に、店舗内の端末を棚卸しします。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-03.png)

まずは、どの端末をどのネットワークに置くかを決めます。

ここを曖昧にしたまま設定すると、あとからこうなります。

```txt
この端末、Staffにいるべき？
Deviceにいるべき？
ゲストWi-Fiへつながってるけど大丈夫？
```

店舗ネットワークでは、設定より先に棚卸しです。

## 設定前にバックアップを取る

店舗構成では、`network`、`wireless`、`firewall`、`dhcp` をまとめて変更します。

作業前に必ずバックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-store-config-20260622.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/store-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップにはSSID、Wi-Fiパスワード、MACアドレス、店舗機器名、IPアドレス、VPN情報などが含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

店舗構成では、バックアップがあるだけでかなり落ち着けます。

特に営業時間中のトラブルでは、落ち着けることが強いです。

## ステップ1: Staffネットワークを安定させる

Staffは、店舗の業務で一番大事なネットワークです。

まずは既存の `lan` をStaffとして使います。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-04.png)

Staffが不安定な状態でGuestやDeviceを作ると、切り分けが一気に難しくなります。

まずStaffが普通に使える状態を作ります。

## Staff SSIDを作る

LuCIで設定します。

1. **Network** → **Wireless** を開く
2. 5GHz radioの **Edit** または **Add** をクリックする
   - Linksys公式手順上、`wifi1` が5GHz radioです
3. **Interface Configuration** → **General Setup** を開く
4. ESSIDを設定する

```txt
Shop_Staff
```

5. Networkを `lan` にする
6. **Wireless Security** を開く
7. 暗号化方式とKeyを設定する
8. **Save** → **Save & Apply** をクリックする

暗号化方式は、端末の対応状況に合わせます。

新しい端末だけならWPA3も検討できますが、業務端末やPOS周辺機器が混ざる場合は互換性確認が必要です。

スタッフSSIDは店内掲示しません。

スタッフ退職時や端末紛失時は、Staff SSIDのパスワード変更も検討します。

## Hidden SSIDは過信しない

Staff SSIDを非表示にしたくなるかもしれません。

ただし、SSIDを非表示にしても、それ自体が強いセキュリティ対策になるわけではありません。

むしろ、端末によっては接続が不安定になったり、管理が面倒になったりすることがあります。

店舗では、Hidden SSIDに頼るより、次を優先します。

- StaffとGuestを混ぜない
- Staff用Wi-Fiパスワードを強くする
- StaffとGuestでパスワードを分ける
- 不要端末を接続させない
- 退職時や端末紛失時にパスワードを見直す
- VPN端末を定期的に棚卸しする

隠すより、分ける。

このほうが運用として分かりやすいです。

## Staffで固定したい機器

Staffネットワークでは、IPが変わると困る機器をDHCP予約します。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-05.png)

LuCIでは、**Network** → **DHCP and DNS** → **Static Leases** で設定します。

CLIで確認するなら次です。

```sh
echo "### active leases"
cat /tmp/dhcp.leases

echo "### static leases"
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='
```

すべてのスタッフ端末を固定する必要はありません。

最初は、NAS、プリンター、録画機、管理PCだけで十分です。

## ステップ2: ゲストWi-Fiを作る

ゲストWi-Fiは、ゲスト端末向けのネットワークです。

目的は、インターネットだけ使わせることです。

```txt
Shop_Guest
  ↓
guest: 192.168.2.0/24
  ↓
guest → wanのみ許可
  ↓
guest → lan/deviceは拒否
```

ゲストWi-Fiは、SSID名を分けただけでは完成しません。

必要なのは、次のセットです。

```txt
SSID
network
DHCP
firewall zone
DHCP/DNS許可
guest → wan
guest → lan拒否
```

## Guestの基本方針

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-06.png)

## Guest用bridge deviceを作る

LuCIで作ります。

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-07.png)

5. **Save** をクリックする

Wi-FiだけのゲストWi-Fiなら、Bridge portsは空のままで進めます。

有線ポートもGuestへ割り当てたい場合は、VLAN設計が必要になります。

最初はWi-Fiだけで大丈夫です。

## Guest interfaceを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-08.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-09.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-10.png)

9. **Save** をクリックする

Guestは一時利用が多いので、Leasetimeは短めでもよいです。

ただし、頻繁に切れるような短さにする必要はありません。

## Guest SSIDを作る

1. **Network** → **Wireless** を開く
2. 5GHz radioの **Add** をクリックする
3. 次のように設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-11.png)

4. **Wireless Security** を開く
5. WPA2-PSKなど、ゲスト端末が接続しやすい方式を選ぶ
6. ゲストWi-Fi用パスワードを設定する
7. **Save** をクリックする

ゲストWi-Fiのパスワードは、Staff SSIDと同じにしません。

必要に応じて、定期的に変更できる運用にします。

ゲストWi-Fiを店内掲示する場合は、StaffやDeviceへ影響しないよう、必ずStaffとは別パスワードにします。

## Guest firewall zoneを作る

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-12.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-13.png)

6. **Save** をクリックする

Guest zoneのInputを `REJECT` にするため、DHCPとDNSを明示的に許可します。

## Guest用DHCPとDNSを許可する

### DHCPを許可する

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-14.png)

### DNSを許可する

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-15.png)

この2つがないと、ゲスト端末がIPを取れなかったり、DNS名前解決できなかったりします。

ゲストWi-Fiでよくあるハマりどころです。

```txt
SSIDにはつながる
でもIPがない
Webが開けない
```

となったら、まずDHCPとDNSを見ます。

## GuestのCLI例

UCIで作る場合の例です。

最初はLuCIで作るほうが安全です。

```sh
BACKUP_DIR="/root/store-guest-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network dhcp firewall wireless; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

# guest bridge
uci -q delete network.guest_dev
uci set network.guest_dev="device"
uci set network.guest_dev.type="bridge"
uci set network.guest_dev.name="br-guest"

# guest interface
uci -q delete network.guest
uci set network.guest="interface"
uci set network.guest.proto="static"
uci set network.guest.device="br-guest"
uci set network.guest.ifname="br-guest"
uci set network.guest.ipaddr="192.168.2.1"
uci set network.guest.netmask="255.255.255.0"

# guest DHCP
uci -q delete dhcp.guest
uci set dhcp.guest="dhcp"
uci set dhcp.guest.interface="guest"
uci set dhcp.guest.start="100"
uci set dhcp.guest.limit="100"
uci set dhcp.guest.leasetime="2h"

# guest firewall zone
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

# DHCP allow
uci -q delete firewall.guest_dhcp
uci set firewall.guest_dhcp="rule"
uci set firewall.guest_dhcp.name="Allow-Guest-DHCP"
uci set firewall.guest_dhcp.src="guest"
uci set firewall.guest_dhcp.proto="udp"
uci set firewall.guest_dhcp.src_port="68"
uci set firewall.guest_dhcp.dest_port="67"
uci set firewall.guest_dhcp.family="ipv4"
uci set firewall.guest_dhcp.target="ACCEPT"

# DNS allow
uci -q delete firewall.guest_dns
uci set firewall.guest_dns="rule"
uci set firewall.guest_dns.name="Allow-Guest-DNS"
uci set firewall.guest_dns.src="guest"
uci set firewall.guest_dns.proto="tcp udp"
uci set firewall.guest_dns.dest_port="53"
uci set firewall.guest_dns.target="ACCEPT"

uci changes network
uci changes dhcp
uci changes firewall

uci commit network
uci commit dhcp
uci commit firewall

/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
```

SSIDの作成は、LuCIの **Network → Wireless** から行うほうが確認しやすいです。

CLIでSSIDも作る場合は、radio名を必ず確認してください。

```sh
uci show wireless | grep '=wifi-device'
uci show wireless | grep -E 'device|ssid|network'
```

## ステップ3: Deviceネットワークを作る

Deviceは、監視カメラ、録画機、スマートロック、設備機器などを置くネットワークです。

POSや決済端末は、Deviceへ入れる前にベンダー要件を確認します。

```txt
Shop_Device
  ↓
device: 192.168.3.0/24
  ↓
device → wanは機器要件に応じて許可
  ↓
device → lanは原則拒否
```

## Deviceの基本方針

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-16.png)

Deviceは、StaffとGuestの中間ではありません。

業務用の中心でもなく、ゲスト用でもありません。

```txt
設備機器をまとめて置く場所
```

として考えると分かりやすいです。

## Device用bridgeとinterfaceを作る

LuCIで作ります。

1. **Network** → **Interfaces** → **Devices** を開く
2. **Add device configuration** をクリックする
3. `br-device` をBridge deviceとして作る
4. **Interfaces** タブで **Add new interface** をクリックする
5. 次のように設定する

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-17.png)

6. DHCP Serverを有効にする

例:

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-18.png)

## Device SSIDを作る

Device用SSIDは、2.4GHz側に作ると接続しやすいことが多いです。

カメラや設備機器は、2.4GHzしか対応していない場合があります。

1. **Network** → **Wireless** を開く
2. 2.4GHz radioの **Add** をクリックする
   - Linksys公式手順上、`wifi0` が2.4GHz radioです
3. 次のように設定する

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-19.png)

古い機器では、WPA3、日本語SSID、記号の多いSSIDで相性が出ることがあります。

最初は英数字中心のSSIDと、互換性重視の暗号化方式にしておくと切り分けしやすくなります。

Deviceは速度よりも安定接続が大事です。

## Device firewall zoneを作る

1. **Network** → **Firewall** → **Zones** を開く
2. **Add** をクリックする
3. 次のように設定する

![表画像 table-20](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-20.png)

4. 必要なら `device → wan` のforwardingを許可する

カメラやスマートロックがクラウド接続を必要とする場合は、device → wanを許可します。

ローカル録画機だけで完結する機器なら、WANを閉じる設計もあります。

最初は、機器要件を確認しながら決めます。

## Device用DHCPとDNSを許可する

Device zoneのInputを `REJECT` にする場合、DHCPとDNSを明示的に許可します。

![表画像 table-21](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-21.png)

DNSを許可しないと、クラウド接続が必要な機器が動かないことがあります。

Deviceは「閉じれば安全」ではありません。

動くために必要な通信を確認して、必要な分だけ通します。

## Deviceの固定IP例

![表画像 table-22](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-22.png)

カメラや録画機は、DHCP予約で固定すると管理しやすくなります。

```txt
録画機のIPが変わった
アプリから見えない
```

みたいなトラブルを減らせます。

## StaffからDeviceへ必要な通信だけ許可する

DeviceからStaffへ広く許可する必要はありません。

Staffから録画機やカメラ管理画面へアクセスしたい場合は、必要な機器だけ許可します。

例: Staffから録画機 `192.168.3.10` のWeb管理画面だけ許可する。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-staff-device.$(date +%Y%m%d-%H%M)

uci -q delete firewall.staff_to_nvr
uci set firewall.staff_to_nvr="rule"
uci set firewall.staff_to_nvr.name="Allow-Staff-to-NVR-Web"
uci set firewall.staff_to_nvr.src="lan"
uci set firewall.staff_to_nvr.dest="device"
uci set firewall.staff_to_nvr.dest_ip="192.168.3.10"
uci set firewall.staff_to_nvr.proto="tcp"
uci set firewall.staff_to_nvr.dest_port="80 443"
uci set firewall.staff_to_nvr.target="ACCEPT"

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

実際に必要なポートは、録画機やカメラの仕様に合わせます。

必要な機器だけ、必要なポートだけ許可するのが基本です。

## POS / 決済端末は別扱いにする

ここは強めに書いておきます。

POSや決済端末は、カメラやスマートロックと同じ感覚でDeviceへ入れないほうが安全です。

理由は次の通りです。

- ベンダー指定のネットワーク要件がある
- 固定IPやDNSの指定がある場合がある
- サポート対象外の構成になる可能性がある
- 決済通信に影響すると店舗運用が止まる
- 監査やセキュリティ要件がある場合がある
- 専用ルーターや専用回線が前提のことがある

POSや決済端末を扱う時は、次を確認します。

```txt
□ ベンダー指定のネットワーク要件
□ 固定IPが必要か
□ DHCPでよいか
□ 専用ルーターが必要か
□ ゲストWi-Fiとは分離されているか
□ AdblockやFamily DNSを通してよいか
□ VPNからアクセスしてよいか
□ Port Forwardが必要と言われていないか
```

原則として、POSや決済端末には強いAdblockや実験的なDNS制御をかけないほうが安全です。

店舗で一番怖いのは、広告が少し残ることではありません。

決済が止まることです。

## ステップ4: VPNで店舗へ安全に入る

店舗ネットワークを外出先から確認したい場合、ポート開放よりVPNを優先します。

代表的な用途です。

- ルーターのLuCIを確認する
- 録画機を見る
- 管理PCへSSHする
- NASへアクセスする
- 一時的に保守作業をする

LuCIや録画機、NAS、カメラ管理画面をインターネットへ直接公開するのは避けます。

```txt
外から直接公開
  → 入口がインターネットに見える

VPN経由
  → 認証された端末だけが入る
```

この違いはかなり大きいです。

## WireGuardとTailscaleの選び方

![表画像 table-23](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-23.png)

LN6001-JPでは、Linksys公式VPN AssistantでWireGuard / Tailscaleを扱えます。

最初はVPN Assistantを使う方針がおすすめです。

## 店舗でTailscaleを使う例

```txt
管理者のノートPC
  ↓ Tailscale
LN6001-JP
  ↓
録画機 192.168.3.10
```

Tailscaleを使う場合は、LN6001-JPをサブネットルーターとして使い、必要なサブネットを広告します。

ただし、最初から店舗LAN全体を広告しなくても構いません。

録画機だけ必要なら、録画機のIPだけに絞ります。

```txt
192.168.3.10/32
```

Tailscale管理コンソールで、広告したルートを承認する必要があります。

ここはハマりやすいです。

```txt
Tailscale上ではルーターが見える
でも録画機へ届かない
```

という時は、まずサブネットルート承認を確認します。

## VPN運用で注意すること

- 退職したスタッフの端末を削除する
- 紛失端末を無効化する
- Tailscale管理コンソールを定期的に確認する
- WireGuard Peerは端末ごとに分ける
- QRコードや秘密鍵を共有しない
- VPNからPOSや決済端末へ入る設計は慎重にする
- VPNからGuestへ入る必要は基本的にない
- 保守担当に一時アクセスを渡したら、作業後に削除する

VPNは便利ですが、入口を作ることでもあります。

「誰が、どの端末から、どこまで入れるか」をメモしておきます。

## ステップ5: DNS / Adblockを慎重に使う

店舗では、DNS広告ブロックやFamily DNSを入れたくなることがあります。

ただし、業務端末に強いブロックをかけると、思わぬトラブルが出ることがあります。

![表画像 table-24](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-24.png)

店舗では、広告ブロックの強さより、業務アプリ、決済、予約、会計、カメラ、スマートロックが安定して動くことを優先します。

Adblockを入れる場合は、まずStaffまたはGuestで軽めに試し、POSや決済端末には影響させないようにします。

```txt
広告ゼロより、業務が止まらないこと
```

店舗ではここが本当に大事です。

## 設定完了後の確認チェックリスト

設定後は、実端末で確認します。

LuCIの画面だけ見て終わりにしないほうが安全です。

## Staff

![表画像 table-25](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-25.png)

## Guest

![表画像 table-26](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-26.png)

## Device

![表画像 table-27](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-27.png)

## VPN

![表画像 table-28](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-28.png)

確認で大事なのは、つながることだけではありません。

```txt
届くべきところに届く
届かないべきところに届かない
```

この両方を確認します。

## CLIで状態を確認する

まとめて確認するなら次です。

```sh
echo "### network interfaces"
uci show network | grep -E 'lan|guest|device|proto|ipaddr'

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP static leases"
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='

echo "### firewall zones and rules"
uci show firewall | grep -E 'zone|forwarding|guest|device|lan|wan|Allow-|Staff'

echo "### Wi-Fi mapping"
uci show wireless | grep -E 'ssid|network|encryption|disabled'

echo "### Wi-Fi status"
wifi status

echo "### WAN/WAN6"
ifstatus wan
ifstatus wan6

echo "### logs"
logread | grep -Ei 'dnsmasq|firewall|guest|device|reject|drop|wireless' | tail -n 120
```

ここでは、「設定が存在するか」だけでなく、「実際の端末が想定IPを取っているか」を見ます。

たとえば、Guest端末が `192.168.1.x` を取っていたら、GuestではなくStaff側に入っている可能性があります。

SSID名だけでは分離になりません。

networkの紐づきまで見ます。

## 店舗運用メモ

設定したら、次のようなメモを残しておきます。

```txt
店舗ネットワーク:

Staff:
  SSID: Shop_Staff
  network: lan
  IP: 192.168.1.0/24
  用途: スタッフPC、管理端末、NAS、プリンター

Guest:
  SSID: Shop_Guest
  network: guest
  IP: 192.168.2.0/24
  方針:
    guest -> wan: allow
    guest -> lan: reject
    guest -> device: reject

Device:
  SSID: Shop_Device
  network: device
  IP: 192.168.3.0/24
  用途: カメラ、録画機、設備機器
  方針:
    device -> lan: reject
    device -> wan: 機器要件に応じて許可

固定IP:
  NAS: 192.168.1.20
  プリンター: 192.168.1.30
  録画機: 192.168.3.10
  カメラ1: 192.168.3.11
  カメラ2: 192.168.3.12

POS / Payment:
  ベンダー要件確認中
  GuestやDeviceへ雑に混ぜない

VPN:
  Tailscale / WireGuard / 未使用
  許可範囲:
    録画機のみ / Staff LAN / LuCIのみ

バックアップ:
  backup-LN6001-store-ok-20260622.tar.gz
```

このメモがあると、スタッフ交代、機器追加、トラブル時にかなり助かります。

半年後の自分は、たぶん細かい設定を忘れています。

## 店舗運用のルール

日常運用では、次を決めておくと安全です。

![表画像 table-29](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/table-29.png)

「何かあったら全部再起動」ではなく、どのネットワークで起きているかを見る運用にします。

```txt
Staffだけ？
Guestだけ？
Deviceだけ？
全体？
```

ここを分けるだけで、トラブル対応はかなり早くなります。

## よくある失敗

## GuestからStaffへ届いてしまう

これは店舗構成としては失敗です。

確認します。

```sh
uci show wireless | grep -E 'guest|Shop_Guest|ssid|network'
uci show firewall | grep -E 'guest|lan|forwarding'
```

見るポイントです。

- Guest SSIDのNetworkが `guest` になっているか
- guest → lan forwardingを作っていないか
- guest zoneのForwardが `REJECT` か
- Guest端末が `192.168.2.x` を取っているか

SSIDが `Shop_Guest` でも、Networkが `lan` のままだと中身はStaffです。

名前だけGuestになってしまいます。

ここは必ず確認します。

## GuestでIPが取れない

Guest zoneのInputを `REJECT` にしている場合、DHCP許可が必要です。

```sh
uci show dhcp.guest
uci show firewall | grep -E 'Allow-Guest-DHCP|guest'
logread | grep -i dnsmasq | tail -n 80
```

`Allow-Guest-DHCP` と `Allow-Guest-DNS` を確認します。

よくある症状です。

```txt
SSIDにはつながる
でもWebが開けない
IPが169.254.x.xになっている
```

この場合はDHCPを疑います。

## Device機器がアプリから見えない

Deviceを分けると、カメラやスマートロックがアプリから見えなくなることがあります。

よくある原因です。

- 初期設定時だけ同じLANにいる必要がある
- mDNSやローカル探索が必要
- device → wanを閉じている
- DNSが通っていない
- Adblockでクラウド接続先が止まっている
- 2.4GHzしか対応していない
- WPA3に対応していない

まずはDevice機器がIPを取れているか、WANへ出る必要がある機器かを確認します。

いきなりfirewallを全部開ける前に、必要な通信を見ます。

## POSが不安定になった

POSや決済端末にネットワーク変更を入れた直後なら、すぐにベンダー要件を確認します。

見るポイントです。

- POSをどのネットワークに置いたか
- IPアドレスは変わっていないか
- DNSやAdblockを通していないか
- 必要な通信先やポートがあるか
- ベンダーのサポート対象構成か
- 専用ルーターや専用回線が必要ではないか

POSや決済端末は、店舗運用への影響が大きいです。

不安があれば、GuestやDeviceとは別に、ベンダー指定に合わせた専用設計を検討します。

## VPNで店舗へ入れない

Tailscaleの場合:

```sh
tailscale status 2>/dev/null || true
tailscale ip 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

WireGuardの場合:

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn'
logread | grep -Ei 'wireguard|wg|51820' | tail -n 100
```

確認することです。

- Tailscale管理コンソールでルート承認済みか
- WireGuardのlatest handshakeが更新されているか
- IPoE環境でWireGuardポートが外から届くか
- VPNから目的の機器へfirewallで許可しているか
- 接続先機器のIPが変わっていないか

VPN接続できることと、録画機やNASへ届くことは別です。

Tailscaleならサブネットルート。  
WireGuardならAllowedIPsとfirewall。  
ここを見ます。

## SSIDを増やしすぎた

店舗では、SSIDを増やしすぎるとスタッフが混乱します。

最初は次で十分です。

```txt
Shop_Staff
Shop_Guest
Shop_Device
```

役割を説明できないSSIDは作らないほうが管理しやすいです。

SSIDを増やすほど、管理も確認も増えます。

店舗では、分かりやすさもセキュリティの一部です。

## 変更する時の順番

店舗では、設定変更を一気に行わないほうが安全です。

おすすめの順番です。

1. 現状バックアップを取る
2. Staffを安定させる
3. Guestを作る
4. GuestからStaffへ届かないことを確認する
5. バックアップを取る
6. Deviceを作る
7. Device機器が動くことを確認する
8. Staffから必要なDevice機器だけ許可する
9. DHCP予約を整理する
10. VPNを必要な範囲で追加する
11. 安定後にバックアップを取る

1つ作って、確認して、バックアップ。

店舗ではこれが一番安全です。

「一気に全部やって、最後に確認」は、原因が分からなくなりやすいです。

## まとめ

小規模店舗では、まずStaff、Guest、Deviceを分けるだけでも運用はかなり安定します。

最初から高度なVLANやVPNまで全部作り込む必要はありません。

大事なのは次です。

1. Staffを業務の中心にする
2. ゲストWi-Fiはインターネットだけにする
3. GuestからStaffやDeviceへ入れない
4. Deviceはカメラや設備機器用に分ける
5. POSや決済端末はベンダー要件を優先する
6. 重要機器はDHCP予約で固定する
7. 外出先から見るならポート開放よりVPNを優先する
8. AdblockやDNS制御は業務影響を見ながら慎重に使う
9. 設定変更前後にバックアップを取る
10. 店舗向けの運用メモを残す

店舗ネットワークでまず目指すのは、完璧なゼロトラスト構成ではありません。

ゲストWi-Fiと業務端末を混ぜない。

POSやカメラを説明できる場所に置く。

必要な機器だけ固定IPにする。

このくらいでも、トラブル対応と日常運用はかなり楽になります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-03.png)

## 次に読むなら

店舗向け構成を作るなら、次の記事も合わせて読むと整理しやすいです。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

まずGuestを分けたい人はゲストWi-Fiの記事へ。

Deviceや有線ポートまで整理したい人はVLANの記事へ。

外出先から店舗へ安全に入りたい人はVPNの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、Wi-Fi radio名、`ifstatus`、`wifi status`、`uci show firewall`、VPN Assistant、Tailscale / WireGuard関連の出力は、ファームウェアや追加モジュール更新で変わることがあります。

POSや決済端末、監視カメラ、スマートロックなどの業務機器は、ベンダー指定のネットワーク要件がある場合があります。

記事の例と実際の店舗機器要件が違う場合は、ベンダー情報を優先してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### 小規模店舗ではネットワークをいくつに分ければいい？

最初はStaff、Guest、Deviceの3つで十分です。

Staffは業務用、GuestはゲストWi-Fi、Deviceはカメラや設備機器用です。

### ゲストWi-FiはSSIDを分けるだけでいい？

十分ではありません。

SSIDだけでなく、guest network、DHCP、firewall zoneを分け、guest → wanだけ許可し、guest → lanを拒否します。

### POSや決済端末はDeviceへ入れていい？

ベンダー要件を確認してから判断してください。

POSや決済端末は、カメラやスマートロックと同じ扱いにしないほうが安全です。

必要ならPOS専用ネットワークとして別に設計します。

### 監視カメラはStaff LANに置いてもいい？

動かすだけならできますが、Deviceへ分けると管理しやすくなります。

Staffから録画機やカメラへ必要な通信だけ許可する構成がおすすめです。

### ゲストWi-Fiのパスワードはどれくらいの頻度で変えるべき？

店舗の運用に合わせて決めます。

掲示している場合や不特定多数へ共有している場合は、必要に応じて変更できる運用にしておくと安心です。

Staff SSIDと同じパスワードにしないことが重要です。

### 店舗へ外出先から入るならポート開放でいい？

おすすめはVPNです。

固定IPやポート開放を管理できるならWireGuard、固定IPなしやIPoE環境ならTailscaleが始めやすいです。

LuCI、録画機、NAS、管理PCを直接インターネットへ公開しないほうが安全です。

### Adblockを店舗全体に入れていい？

慎重に進めてください。

業務アプリ、決済、予約、会計、カメラ、スマートロックが誤ブロックで動かなくなる可能性があります。

最初はStaffまたはGuestで軽めに試し、POSや決済端末には影響させない設計がおすすめです。

### 有線ポートまでVLANで分けるべき？

最初から必須ではありません。

まずはSSID、network、firewall zoneで分けるだけでも効果があります。

有線カメラ、管理スイッチ、POS専用線などが必要になったら、VLAN記事へ進むとよいです。

### StaffからDeviceへ全部アクセスできるようにしていい？

最初から広く許可しないほうが安全です。

録画機やカメラ管理画面など、必要な機器とポートだけ許可します。

### GuestにAdblockを入れるべき？

入れても構いませんが、最初は軽めで十分です。

店舗のゲストWi-Fiで一番大事なのは、広告を消すことより、StaffやDeviceへ入れないことです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- OpenWrt Wiki - Firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
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
