<!-- mirror-source: articles/007-guest-wifi.md -->

# ゲストWi-FiはSSIDだけでは完成しない｜LN6001-JPで家庭内LANからちゃんと分ける【OpenWrt集中連載007】

ゲストWi-Fiって、作るだけなら簡単そうに見えます。

`MyHome_Guest` みたいなSSIDを追加する。  
パスワードを設定する。  
ゲストにそれを教える。

はい、完成。

……と言いたくなるのですが、ここに少し罠があります。

OpenWrt系ルーターでは、**SSIDを分けただけでは、ネットワークまで分かれているとは限りません**。

たとえば、`MyHome_Guest` というSSIDを作っても、そのSSIDがメインの `lan` に紐づいていたら、中身は家族用Wi-Fiと同じ場所にいます。

つまり、見た目はゲストWi-Fi。  
でも、ゲスト端末からNASやプリンター、管理画面が見えてしまう可能性があります。

これはちょっと嫌ですよね。

ゲストWi-Fiで大事なのは、SSID名ではありません。

```txt
ゲスト端末を家庭内LANや業務LANから分けること
```

です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、ゲストWi-Fiを **SSID / network / DHCP / firewall zone** まで含めて作る流れをまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、ただゲスト用SSIDを追加することではありません。

ここまでできればOKです。

- ゲストWi-Fi用のSSIDを作る
- ゲスト用networkを作る
- ゲスト端末へIPアドレスを配れるようにする
- ゲスト用firewall zoneを作る
- ゲスト端末からインターネットへは出られる
- ゲスト端末からLAN、NAS、LuCI、SSHへは入れない
- 設定後に実端末で確認できる

ざっくり言うと、

```txt
ゲストにはインターネットだけ使ってもらう
家の中や店の裏側は見せない
```

という構成を作ります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-01.png)

## 先にざっくり結論

ゲストWi-Fiは、次の4点セットで作ります。

```txt
SSID
network
DHCP
firewall zone
```

SSIDだけ作っても、まだ半分です。

おすすめの基本構成はこうです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-01.png)

ここで大事なのは、ゲスト端末がIPを取るにはDHCPが必要で、ドメイン名を解決するにはDNSが必要ということです。

guest zoneのInputを `REJECT` にする場合、DHCPとDNSだけは明示的に許可します。

```txt
Allow-Guest-DHCP
Allow-Guest-DNS
```

この2つを忘れると、ゲスト端末がWi-Fiにはつながるのに、IPが取れない、名前解決できない、という状態になりがちです。

## こういう人向けです

この記事は、次のような人向けです。

- 家族以外の人にWi-Fiを使わせたい
- 店舗や小さなオフィスでゲストWi-Fiを出したい
- ゲスト端末からNASやプリンターを見せたくない
- ゲストWi-FiをSSIDだけで作ってよいのか不安
- OpenWrtのguest network / firewall zoneの考え方を知りたい
- LuCIでゲストWi-Fiを作りたい
- 設定後に何を確認すべきか知りたい

逆に、単に「家族用SSIDをもう1つ増やしたい」だけなら、ここまで分けなくてもよい場合があります。

でも、**ゲストWi-Fi** と呼ぶなら、LANから分けるところまでやっておくのがおすすめです。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-02.png)

## 最初に言葉だけそろえる

ゲストWi-Fi設定で出てくる言葉を、軽く整理しておきます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-02.png)

ここで一番大事なのは、SSIDとnetworkは別物ということです。

```txt
SSID = 見た目のWi-Fi名
network = 中身の所属先
```

`MyHome_Guest` というSSID名でも、networkが `lan` なら、ゲスト端末はメインLANにいます。

ここを間違えると、見た目だけゲストWi-Fiになります。

## ゲストWi-Fiの完成形

この記事では、次の構成を作ります。

```txt
インターネット
    │
LN6001-JP
    │
    ├── lan: 192.168.1.0/24
    │     SSID: MyHome
    │     用途: 家族PC、スマホ、NAS、プリンター
    │
    └── guest: 192.168.2.0/24
          SSID: MyHome_Guest
          用途: ゲスト端末
          方針: インターネットだけ
```

通信方針はこうです。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-03.png)

ゲスト端末はインターネットには出られる。

でも、家庭内LANや業務LANには入れない。

これがゲストWi-Fiの基本です。

## 設定前にバックアップを取る

ゲストWi-Fiでは、`network`、`wireless`、`dhcp`、`firewall` を触ります。

つまり、Wi-Fiとネットワークとfirewallを同時に触ります。

ここでバックアップなしは、ちょっと怖いです。

まずLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-guest-wifi-20260621.tar.gz
```

SSHでも状態メモを残すなら、次を実行します。

```sh
BACKUP_DIR="/root/guest-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status.json" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップにはSSID、Wi-Fiパスワード、内部IP、端末名などが含まれることがあります。

そのままSNSや公開リポジトリへ出さないようにしてください。

## 作業は有線LANからがおすすめ

ゲストWi-Fiの設定中は、Wi-Fiが一時的に切れることがあります。

できれば、有線LANで接続したPCから作業してください。

```txt
PC
  ↓ 有線LAN
LN6001-JP LANポート
```

スマートフォンだけで作業していると、SSIDを変更した瞬間に接続が切れて、

```txt
今、何が反映されたの？
どのSSIDにつなぎ直せばいいの？
```

となりがちです。

有線LANでLuCIへ入れる状態を残しておくと安心です。

## ステップ1: ゲスト用bridge deviceを作る

まず、ゲスト用の器を作ります。

LuCIで作業します。

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-04.png)

5. **Save** をクリックする

Wi-FiだけのゲストWi-Fiなら、Bridge portsは空のままで進めます。

有線ポートもゲスト用に分けたい場合は、VLAN設計が必要になります。

ここでは、まずWi-Fiだけのゲストネットワークを作ります。

## ステップ2: guest interfaceを作る

次に、ゲスト用networkを作ります。

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-05.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-06.png)

6. **Save** をクリックする

これで、ゲストネットワークの入口ができます。

メインLANが `192.168.1.0/24` の場合、ゲストは `192.168.2.0/24` に分けると分かりやすいです。

## ステップ3: guest DHCPを有効にする

ゲスト端末にIPアドレスを配るため、DHCPを有効にします。

1. **Network** → **Interfaces** を開く
2. `guest` interfaceの **Edit** をクリックする
3. **DHCP Server** タブを開く
4. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
5. 次のように設定する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-07.png)

6. **Save** をクリックする

この設定だと、ゲスト端末には次のようなIPが配られます。

```txt
192.168.2.100 - 192.168.2.199
```

ゲストWi-Fiでは、リース時間を短めにしておくと扱いやすいです。

家庭なら `12h` でも構いません。

店舗や一時利用のゲストが多い環境では `2h` くらいでもよいと思います。

## ステップ4: ゲストSSIDを作る

次に、実際に端末から見えるゲストWi-Fi名を作ります。

1. **Network** → **Wireless** を開く
2. ゲストWi-Fiを出したいradioの **Add** をクリックする
   - 5GHzに出すなら `wifi1`
   - 2.4GHzに出すなら `wifi0`
3. 次のように設定する

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-08.png)

4. **Wireless Security** を開く
5. 暗号化方式とKeyを設定する
6. **Save** をクリックする

ここで大事なのは、Networkを必ず `guest` にすることです。

`lan` にすると、見た目はゲストWi-Fiでも、中身はMain LANになります。

ゲストWi-Fiを作る時にいちばん多い失敗がこれです。

```txt
SSIDはGuest
でもNetworkはlan
```

これは、名前だけゲストWi-Fiです。

## ゲストWi-Fiを2.4GHzに出すか、5GHzに出すか

家庭や小さなオフィスでは、まず5GHzで出すのが扱いやすいです。

スマートフォンやPCのゲスト利用が中心なら、5GHzで十分です。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-09.png)

店舗で広くゲストWi-Fiを出したい場合、2.4GHzも検討できます。

ただし、2.4GHzは混雑しやすいので、速度重視のゲスト利用には向きません。

最初は5GHzで作り、必要に応じて2.4GHz側にも追加するくらいで大丈夫です。

## ステップ5: guest firewall zoneを作る

ここからが本番です。

ゲストWi-Fiを本体LANから分けるには、firewall zoneを作ります。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-10.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-11.png)

6. **Save** をクリックする

ここでやっていることは、ざっくり言うとこうです。

```txt
guestからインターネットへは出ていい
guestからLANへは行かせない
guestからルーター自身へも基本は入れない
```

ゲストWi-Fiは、これでかなり“ゲストらしく”なります。

## ステップ6: DHCPとDNSだけ許可する

guest zoneのInputを `REJECT` にすると、ゲスト端末からルーター自身への通信は基本的に拒否されます。

これは安全側の設定です。

ただし、このままだと困ることがあります。

ゲスト端末は、ルーターに対してDHCPとDNSを使う必要があります。

つまり、これだけは許可します。

```txt
DHCP = IPアドレスをもらうため
DNS = google.com などの名前を解決するため
```

この2つを許可しないと、

```txt
Wi-FiにはつながるけどIPが取れない
IPはあるけどWebサイト名が引けない
```

という状態になります。

## DHCPを許可する

**Network** → **Firewall** → **Traffic Rules** でルールを追加します。

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-12.png)

DHCPは、ゲスト端末がIPアドレスをもらうために必要です。

## DNSを許可する

同じくTraffic Rulesで追加します。

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-13.png)

DNSは、ゲスト端末がドメイン名を解決するために必要です。

この2つは、ゲストWi-Fiの定番ハマりポイントです。

ゲストWi-FiでIPが取れない、名前解決できない時は、まずこの2つを見ます。

## ステップ7: Save & Applyする

すべて設定したら、LuCI上部の保留中の変更を確認し、**Save & Apply** をクリックします。

反映中にWi-Fiが一時的に切れることがあります。

有線LANで作業していれば、そのままLuCIに戻りやすいです。

反映後、ゲスト端末を `MyHome_Guest` へ接続します。

ここからが確認です。

## 設定後に必ず確認すること

ゲストWi-Fiは、作っただけでは終わりではありません。

必ず実端末で確認します。

確認するのは、次の2つです。

```txt
つながるべきところにつながる
つながってはいけないところにつながらない
```

## ゲスト端末で確認する

スマートフォンやPCをゲストWi-Fiへ接続します。

確認項目は次です。

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-14.png)

インターネットへ出られるだけでは不十分です。

ゲスト端末からLANへ届かないことまで確認します。

## CLIで確認する

SSHでLN6001-JPへ入り、状態を確認します。

```sh
echo "### guest interface"
ifstatus guest 2>/dev/null || true

echo "### guest DHCP"
uci show dhcp.guest 2>/dev/null || true

echo "### guest firewall"
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'

echo "### wireless guest mapping"
uci show wireless | grep -E 'guest|MyHome_Guest|ssid|network'

echo "### active DHCP leases"
cat /tmp/dhcp.leases

echo "### recent logs"
logread | grep -Ei 'guest|dnsmasq|firewall|reject|drop' | tail -n 100
```

見るポイントです。

- `guest` interfaceがあるか
- `guest` DHCPが有効か
- `Allow-Guest-DHCP` があるか
- `Allow-Guest-DNS` があるか
- `guest → wan` forwardingがあるか
- `guest → lan` forwardingを作っていないか
- ゲスト端末が `192.168.2.x` を取っているか

## UCIで作る場合の例

最初はLuCIで作るほうが安全です。

ただ、設定内容をテキストで理解したい人向けに、UCI例も載せておきます。

SSIDの作成はradio名を確認してから行う必要があるため、ここでは主にnetwork、DHCP、firewallの例です。

```sh
BACKUP_DIR="/root/guest-before-uci-$(date +%Y%m%d-%H%M)"
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

SSIDの追加は、LuCIの **Network → Wireless** から行うほうが確認しやすいです。

CLIでSSIDも作る場合は、必ず先にradio名を確認します。

```sh
uci show wireless | grep '=wifi-device'
uci show wireless | grep -E 'device|ssid|network'
```

確認せずに無線設定をコピペ実行しないでください。

## IPv6の扱い

この記事では、まずIPv4のゲストWi-Fiを作ります。

IPv6をゲストWi-Fiへ配るかどうかは、別途考えます。

IPoE環境では、IPv6が絡むと少し見え方が変わります。

ゲストWi-FiでもIPv6を使わせたい場合は、次の点を確認します。

- guestにIPv6 prefixを配るか
- guest → lanがIPv6でも拒否されているか
- DNSがIPv4 / IPv6の両方で意図通りか
- Family DNSやAdblockの抜け道になっていないか

最初は、IPv4のゲストWi-Fi分離を安定させてから、IPv6を扱うのがおすすめです。

IPv6まわりは、IPv6の落とし穴の記事で詳しく扱います。

## ゲストWi-FiでAdblockやFamily DNSを使う場合

ゲストWi-FiにもAdblockやFamily DNSを適用したくなるかもしれません。

これはできます。

ただし、最初から強くかけすぎると、ゲスト端末の一部サービスが動かないことがあります。

家庭なら、ゲストWi-Fiは通常DNSでも十分な場合があります。

店舗なら、ゲストWi-Fiへ軽めのDNS制御を入れるのも選択肢です。

ただし、ゲストWi-Fiで大事なのはまず分離です。

```txt
GuestからLANへ入れない
```

これが最優先です。

DNS制御はその次で大丈夫です。

## ゲストWi-Fiのパスワード運用

ゲストWi-Fiのパスワードは、メインSSIDと同じにしないほうがよいです。

家庭なら、家族用とは別のパスワードにします。

店舗なら、ゲスト用パスワードを掲示する場合もあります。

その場合、Staff用SSIDとは必ず別にします。

おすすめの考え方です。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/table-15.png)

ゲストWi-Fiのパスワードは、必要に応じて変更できるようにしておくと安心です。

## よくある失敗

## SSIDを作っただけで安心する

一番多い失敗です。

`MyHome_Guest` というSSIDを作っても、それが `lan` に紐づいていたら、ゲスト端末はMain LANにいます。

確認します。

```sh
uci show wireless | grep -E 'MyHome_Guest|ssid|network'
```

`network='guest'` になっているか見ます。

## ゲスト端末がIPを取れない

DHCPが動いていない、またはDHCP通信がfirewallで止まっている可能性があります。

確認します。

```sh
uci show dhcp.guest
uci show firewall | grep -E 'Allow-Guest-DHCP|guest'
logread | grep -i dnsmasq | tail -n 80
```

`Allow-Guest-DHCP` を確認します。

## Webサイトが開けない

DNSが通っていない可能性があります。

確認します。

```sh
uci show firewall | grep -E 'Allow-Guest-DNS|guest'
logread | grep -i dnsmasq | tail -n 80
```

`Allow-Guest-DNS` を確認します。

ゲスト端末側でIPアドレス直打ちは通るのに、ドメイン名が開けない場合はDNSを疑います。

## ゲスト端末からLuCIが開けてしまう

guest zoneのInputやfirewallルールを確認します。

```sh
uci show firewall | grep -E 'guest|input|Allow-|22|80|443'
```

ゲスト端末から `https://192.168.1.1` や `ssh root@192.168.1.1` へ入れないことを確認します。

管理画面やSSHは、ゲストWi-Fiから見せないほうが安全です。

## ゲスト端末からNASやプリンターが見える

guest → lan forwardingを作っていないか確認します。

```sh
uci show firewall | grep -E 'forwarding|guest|lan'
```

`guest` から `lan` へのforwardingがある場合は、意図したものか確認します。

通常のゲストWi-Fiでは、guest → lanは作りません。

## SSIDを増やしすぎた

OpenWrtではSSIDを増やせます。

でも、SSIDを増やしすぎると管理が大変になります。

家庭なら、最初はこれくらいで十分です。

```txt
MyHome
MyHome_IoT
MyHome_Guest
```

店舗なら、これくらいです。

```txt
Shop_Staff
Shop_Guest
Shop_Device
```

役割を説明できないSSIDは、作らないほうが管理しやすいです。

## 運用メモを残す

設定したら、簡単なメモを残しておきます。

```txt
ゲストWi-Fi設定:

SSID:
  MyHome_Guest

network:
  guest

IP:
  192.168.2.1/24

DHCP:
  192.168.2.100-199
  leasetime: 2h

firewall:
  guest input: REJECT
  guest output: ACCEPT
  guest forward: REJECT
  guest -> wan: allow
  guest -> lan: reject

Traffic Rules:
  Allow-Guest-DHCP
  Allow-Guest-DNS

確認:
  ゲスト端末でWeb OK
  192.168.1.1へアクセス不可
  NASへアクセス不可
  SSH不可

バックアップ:
  backup-LN6001-before-guest-wifi-20260621.tar.gz
```

こういうメモがあると、半年後の自分が助かります。

本当に助かります。

## まとめ

ゲストWi-Fiは、SSIDを追加するだけでは完成しません。

大事なのは、ゲスト端末を本体LANから分けることです。

そのためには、次の4点セットで考えます。

```txt
SSID
network
DHCP
firewall zone
```

基本方針はこうです。

- guest → wan は許可
- guest → lan は拒否
- guest → LuCI / SSH は拒否
- DHCPとDNSだけ許可
- 実端末でLANへ届かないことを確認する

ゲストWi-Fiは、名前を作るだけではなく、ちゃんと“ゲストの部屋”を作るイメージです。

ゲストにはインターネットだけ使ってもらう。  
家の中や店の裏側には入れない。

この考え方ができると、家庭でも店舗でもかなり安心してWi-Fiを出せるようになります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-03.png)

## 次に読むなら

ゲストWi-Fiを作ったら、次は目的に合わせて進みます。

- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

ゲストWi-Fiのfirewallをちゃんと理解したい人は、firewall zonesの記事へ。

店舗や小さなオフィスでStaff / Guest / Deviceまで分けたい人は、VLANや店舗向け構成例へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、firewall表示、Wi-Fi radio名、`uci show` の出力は、ファームウェア更新や設定状態によって変わることがあります。

無線の国設定、送信出力、DFS関連の値は変更しません。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

## よくある質問

### ゲストWi-FiはSSIDを追加するだけでいい？

いいえ。

SSIDを追加しただけでは、メインLANから分離できていない場合があります。

ゲストWi-Fiとして使うなら、guest network、DHCP、firewall zoneまで作るのがおすすめです。

### ゲストWi-Fiからインターネットだけ使わせたい時は？

guest zoneを作り、guest → wanだけ許可します。

guest → lanは許可しません。

また、guest zoneのInputを `REJECT` にする場合は、DHCPとDNSだけTraffic Rulesで許可します。

### ゲスト端末がIPアドレスを取れません

DHCP設定かfirewallのDHCP許可を確認します。

`Allow-Guest-DHCP` があるか、guest DHCP Serverが有効かを見てください。

### ゲスト端末でWebサイトが開けません

DNSが通っていない可能性があります。

`Allow-Guest-DNS` があるか、ゲスト端末が `192.168.2.x` のIPアドレスを取れているか確認してください。

### ゲストWi-FiからLuCIへ入れてしまいます

guest zoneのInputやfirewall ruleを確認してください。

通常のゲストWi-Fiでは、LuCIやSSHへ入れないようにします。

### ゲストWi-FiにAdblockを適用してもいい？

できます。

ただし、最初は分離を優先します。

AdblockやFamily DNSは、その後で必要に応じて追加するのがおすすめです。

### ゲストWi-Fiを2.4GHzにも出したほうがいい？

環境によります。

スマートフォンやPC中心なら5GHzで十分なことが多いです。

遠い部屋や古い端末向けに使わせたい場合は、2.4GHzにも出す選択肢があります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- OpenWrt Wiki - Guest Wi-Fi using LuCI: https://openwrt.org/docs/guide-user/network/wifi/guestwifi/configuration_webinterface
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
