<!-- mirror-source: articles/011-vlan-office-guest.md -->

# VLANは最後でいい｜小さなオフィス・店舗でStaff / Guest / Deviceをまず分ける【OpenWrt集中連載011】

小さなオフィスや店舗のネットワークって、気づくと全部混ざります。

スタッフ用PC。  
ゲストWi-Fi。  
POS。  
レシートプリンター。  
防犯カメラ。  
録画機。  
NAS。  
予約端末。  
業務タブレット。  
スマートフォン。

最初はそれでも動きます。

むしろ、全部同じWi-Fiに入れたほうが楽です。  
プリンターも見える。  
カメラも見える。  
設定も少ない。  
スタッフにも説明しやすい。

でも、あとからこう思う場面が出てきます。

「ゲストWi-Fiから社内PCが見えてない？」  
「カメラやIoT機器が業務LANと同じ場所にいて大丈夫？」  
「POSとゲスト端末が同じネットワークにいるの、ちょっと怖くない？」  
「どの端末がどこにあるのか、もう分からない」

ここで出てくるのがVLANです。

ただし、いきなり有線ポートまでVLANで分ける必要はありません。

VLANは強い道具です。  
でも最初に触ると、だいたい管理画面への帰り道を失いがちです。

なので、この記事の方針はこうです。

```txt
まずSSID / network / DHCP / firewall zoneで分ける
有線VLANは必要になってから慎重にやる
```

小さなオフィスや店舗で最初に大事なのは、VLAN IDをきれいに振ることではありません。

**GuestからStaffへ入れないこと。**  
**DeviceからStaffへ勝手に入れないこと。**  
**Staffから必要な機器だけ管理できること。**

まずはここです。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、Staff / Guest / Deviceネットワークをどう分けるかを、LuCI中心で整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、有線ポートまで完璧にVLAN化することではありません。

まずは、ここまでできればOKです。

- Staff / Guest / Deviceを分ける考え方が分かる
- VLANとゲストWi-Fi分離の違いが分かる
- Guest用network、DHCP、SSID、firewall zoneを作れる
- Device用network、DHCP、SSID、firewall zoneを作れる
- GuestからStaffへ届かないことを確認できる
- DeviceからStaffへ勝手に入れない構成を作れる
- 有線VLANを触る前に、何を確認すべきか分かる
- 設定前にバックアップを取る理由が分かる

有線VLANは後からで大丈夫です。

まずは、無線SSIDとfirewall zoneで「混ぜない」状態を作ります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-01.png)

## 先にざっくり結論

小さなオフィスや店舗では、最初はこの3分類で十分です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-01.png)

ここで大事なのは、VLAN IDそのものではありません。

先に決めるべきなのは、

```txt
誰が
どのネットワークにいて
どこへ通信してよいか
```

です。

最初の成功条件は、これです。

- GuestからStaffへ入れない
- GuestからDeviceへ入れない
- DeviceからStaffへ勝手に入れない
- Staffから必要なDeviceだけ管理できる
- POSや決済端末はベンダー要件を優先する
- 管理画面へ戻れなくなるような有線ポート変更は後回しにする

つまり、最初から全ポートを完璧にVLAN化しなくて大丈夫です。

まずは、

```txt
SSID
network
DHCP
firewall zone
```

で分ける。

有線ポートや管理スイッチまで分けたくなったら、その時にVLANへ進む。

この順番が安全です。

## こういう人向けです

この記事は、次のような人向けです。

- 小さなオフィスでStaff用とGuest用のWi-Fiを分けたい
- 店舗でStaff、Guest、Deviceを混ぜたくない
- ゲストWi-Fiだけで足りるのか、VLANが必要なのか判断したい
- 防犯カメラや録画機を社内PCと同じLANに置きたくない
- POSや決済端末の扱いを慎重に考えたい
- 有線ポートやスイッチも含めて、将来的にネットワークを整理したい
- でも最初からVLANでLuCIへ戻れなくなるのは避けたい

逆に、すでに管理スイッチや802.1Q VLAN運用に慣れている人には、前半はかなり基本寄りです。

ただ、小さな店舗やSOHOでは、この基本のほうが事故を減らします。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-02.png)

## 最初に言葉だけそろえる

VLANまわりの言葉は、最初ちょっと硬いです。

ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-02.png)

最初は、VLANを「ネットワークの部屋分け」と考えると分かりやすいです。

```txt
Staffの部屋
Guestの部屋
Deviceの部屋
WANへの出口
```

どの部屋からどの部屋へ行っていいかを決めるのがfirewall zone。

有線ポートやスイッチまで部屋分けしたくなったらVLAN。

このくらいの理解でOKです。

## VLANとゲストWi-Fi分離は同じではない

まずここを分けます。

VLANとゲストWi-Fi分離は、似ていますが同じではありません。

## ゲストWi-Fi分離

ゲストWi-Fi分離は、主に無線端末を分ける考え方です。

たとえば、こんな構成です。

```txt
Office_Staff
  → lan
  → 192.168.1.0/24

Office_Guest
  → guest
  → 192.168.2.0/24
  → guestからlanへは拒否
```

この場合、SSID、network、DHCP、firewall zoneを分けます。

ゲスト端末を無線で分けたいだけなら、これで十分なことも多いです。

つまり、最初からVLANを使わなくても、

```txt
GuestからStaffへ入れない
```

状態は作れます。

## VLAN

VLANは、有線ポートやスイッチも含めてネットワークを分けるために使います。

たとえば、こんな構成です。

```txt
LANポート1 → Staff VLAN
LANポート2 → Device VLAN
管理スイッチ → Staff / Guest / Device VLANをtaggedで運ぶ
カメラ → Device VLANのaccess portへ接続
```

無線だけならSSID分離で足りることがあります。

有線カメラ、有線POS、管理スイッチ、複数アクセスポイントまで含めて分けたいなら、VLANが必要になります。

## まずはSSID + firewall zoneでいい

小さなオフィスや店舗では、最初から有線VLANへ行かなくても大丈夫です。

まずは、

```txt
Staff SSID
Guest SSID
Device SSID
```

を作り、それぞれを別networkとfirewall zoneへ紐づけます。

有線ポートVLANは、必要になってからでOKです。

## 最初から有線VLANを触らなくてよい理由

有線VLANは便利です。

でも、設定ミスの影響が大きいです。

特に怖いのは、管理端末がつながっているポートを間違って別VLANへ移してしまうことです。

そうすると、

```txt
LuCIへ入れない
SSHへ入れない
有線でも戻れない
でもWi-Fiもまだ設定途中
```

みたいな状況になります。

これはつらいです。

なので、おすすめの順番はこうです。

1. Staff用の既存LANをそのまま使う
2. Guest用SSIDとguest networkを作る
3. Device用SSIDとdevice networkを作る
4. firewall zoneで通信を分ける
5. 必要になったら、管理スイッチや有線ポートVLANを追加する

一気に完成形を作るより、段階的に進めたほうが原因を追いやすくなります。

OpenWrtでは「一気に作る」より「戻れる状態を作りながら進める」ほうが強いです。

## 小さなオフィス・店舗の設計例

この記事では、次の構成を例にします。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-03.png)

この段階では、VLAN IDは「将来、有線スイッチへ展開する時の設計メモ」として扱います。

まずはLuCIで `guest` と `device` のnetwork、SSID、firewall zoneを作ります。

## 先に通信方針を決める

SSIDやVLAN IDを作る前に、通信方針を決めます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-04.png)

一番大事なのは、GuestからStaffへ届かないことです。

次に、Device / CameraからStaffへ勝手に入れないこと。

StaffからDeviceへは、録画機、カメラ管理画面、設備機器の管理アプリなど、本当に必要な通信だけ許可します。

POSや決済端末は、このDeviceへ雑に混ぜないほうが安全です。

決済端末やPOSは、ベンダー要件、サポート条件、通信先、固定IP要件を先に確認してください。

## POSや決済端末は別扱い

ここは強めに書いておきます。

POSや決済端末は、カメラやスマートロックと同じノリでDeviceネットワークへ入れないほうが安全です。

理由はシンプルです。

- ベンダー指定のネットワーク要件がある
- サポート対象外構成になる場合がある
- 決済通信へ影響すると店舗運用が止まる
- 固定IP、DNS、専用ルーターなどの要件がある場合がある
- セキュリティや監査の要件がある場合がある

この記事のDeviceは、カメラ、録画機、設備機器、IoTを想定した例です。

POSや決済端末は、必要なら専用ネットワークとして別に設計します。

## 設定前にバックアップを取る

VLANやfirewallを触る前に、必ず現在の状態を保存します。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-vlan-office-guest-20260621.tar.gz
```

SSHで状態メモを残す場合は、次を実行します。

```sh
BACKUP_DIR="/root/vlan-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ip link show > "$BACKUP_DIR/ip-link.txt"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
bridge vlan show > "$BACKUP_DIR/bridge-vlan.txt" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、SSID、Wi-Fiパスワード、ネットワーク構成、端末情報が含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

設定で一番強い人は、全部暗記している人ではありません。

変更前に戻れる人です。

## まずGuestネットワークを作る

Guestネットワークは、ゲストWi-Fi用です。

ここでは `192.168.2.0/24` を使います。

## ステップ1: guest用ブリッジデバイスを作る

LuCIで作業します。

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-05.png)

5. **Save** をクリックする

Wi-FiだけのゲストWi-Fiであれば、Bridge portsは空のままで進めます。

有線ポートもGuestへ割り当てたい場合は、後半の有線VLANの考え方を確認してから進めます。

## ステップ2: guestインターフェースを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-06.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-07.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-08.png)

9. **Save** をクリックする

これで、Guest端末へ `192.168.2.100` から `192.168.2.149` あたりのIPアドレスを配る準備ができます。

## Guest用SSIDを作る

Guest用SSIDを作り、`guest` networkへ紐づけます。

1. **Network** → **Wireless** を開く
2. 5GHz radioの **Add** をクリックする
   - LN6001-JPでは、Linksys公式手順上は `wifi1` が5GHz radioです
3. **Interface Configuration** の **General Setup** で次を設定する

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-09.png)

4. **Wireless Security** タブを開く
5. 次のように設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-10.png)

6. **Save** をクリックする

Guest用SSIDのパスワードは、Staff用SSIDと同じにしません。

店舗や小さなオフィスでは、Guest用パスワードだけ定期的に変更できるようにしておくと運用しやすいです。

## guest firewall zoneを作る

GuestからStaffへ入れないようにします。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-11.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-12.png)

6. **Save** をクリックする

この設定で、Guestからインターネットへは出られます。

一方で、GuestからStaff側の `lan` へは転送しません。

ただし、Inputを `REJECT` にしたため、DHCPとDNSは明示的に許可します。

ここを忘れると、Guest端末がWi-FiにはつながるのにIPが取れない、Webが開けない、という状態になります。

## Guest用DHCPとDNSを許可する

Guest端末がIPアドレスを取得し、DNSを使えるようにします。

## DHCPを許可する

1. **Network** → **Firewall** → **Traffic Rules** を開く
2. **Add** をクリックする
3. 次のように設定する

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-13.png)

4. **Save** をクリックする

## DNSを許可する

1. **Traffic Rules** で **Add** をクリックする
2. 次のように設定する

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-14.png)

3. **Save** をクリックする

この2つは、Guestネットワークの定番ハマりポイントです。

GuestでIPが取れないならDHCP。  
GuestでIPはあるけどWebが開けないならDNS。

まずここを見ます。

## Device / Cameraネットワークを作る

次に、防犯カメラ、録画機、IoT、設備機器向けの `device` ネットワークを作ります。

ここでは `192.168.3.0/24` を使います。

## ステップ1: device用ブリッジデバイスを作る

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-15.png)

5. **Save** をクリックする

## ステップ2: deviceインターフェースを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-16.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-17.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-18.png)

9. **Save** をクリックする

Deviceネットワークでは、カメラや録画機に固定DHCP割り当てを使うことが多いです。

最初はDHCPで接続し、動作確認後に固定DHCPへ切り替えると管理しやすくなります。

## Device用SSIDを作る

カメラや設備機器がWi-Fi接続の場合、Device用SSIDを作ります。

IoT機器や一部のカメラは2.4GHzしか対応していないことも多いので、最初は2.4GHz側に作ると扱いやすいです。

1. **Network** → **Wireless** を開く
2. 2.4GHz radioの **Add** をクリックする
   - LN6001-JPでは、Linksys公式手順上は `wifi0` が2.4GHz radioです
3. **Interface Configuration** の **General Setup** で次を設定する

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-19.png)

4. **Wireless Security** タブを開く
5. 次のように設定する

![表画像 table-20](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-20.png)

6. **Save** をクリックする

古い機器では、WPA3、日本語SSID、記号の多いSSIDで相性が出ることがあります。

最初は英数字中心のSSIDと、互換性重視の暗号化設定にしておくと切り分けしやすくなります。

Deviceは速さより、まず安定接続です。

## device firewall zoneを作る

DeviceからStaffへ勝手に入れないようにします。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-21](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-21.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-22](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-22.png)

6. **Save** をクリックする

カメラや設備機器がインターネット接続を必要としない場合、device → wan は許可しない構成もあります。

ただし、スマートカメラ、クラウド録画、スマート家電、ファームウェア更新などでWAN接続が必要な機器もあります。

Deviceネットワークは、

```txt
とりあえず全部閉じる
```

でも、

```txt
とりあえず全部出す
```

でもなく、機器の要件を見て決めます。

## Device用DHCPとDNSを許可する

Device zoneのInputを `REJECT` にする場合、DHCPとDNSを許可します。

## DHCPを許可する

![表画像 table-23](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-23.png)

## DNSを許可する

![表画像 table-24](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-24.png)

Deviceネットワークでは、最初から強いDNS広告ブロックを入れすぎないほうが安全です。

メーカーのクラウド接続やファームウェア更新が止まることがあります。

まずは通信要件を確認し、必要ならDNS広告ブロックやAllowlistで調整します。

## StaffからDeviceへ必要な通信だけ許可する

Staff側から防犯カメラや録画機の管理画面へアクセスしたい場合があります。

その場合、device → staffを許可するのではなく、StaffからDeviceへの必要な通信だけ許可します。

LuCIでは、**Network** → **Firewall** → **Traffic Rules** で追加します。

例:

![表画像 table-25](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-25.png)

最初は、Device全体へ広く許可するのではなく、録画機や管理対象のIPアドレスだけ許可するほうが安全です。

カメラや録画機によって必要なポートは異なります。

ベンダーのマニュアルやサポート情報を確認してください。

## 有線ポートVLANは後から慎重に

ここまでの設定では、主にSSID分離とfirewall zoneを扱っています。

有線ポートもVLANで分けたい場合、次のような設計が必要になります。

![表画像 table-26](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-26.png)

有線ポートVLANを触る時に一番大事なのは、管理アクセスを失わないことです。

最初は、管理端末をつないでいるLANポートをStaff側に残し、そこからLuCIへ戻れる状態を維持してください。

有線VLANの設定は、LN6001-JPの実機ポートマップ、接続しているスイッチ、ファームウェアのネットワーク構成によって変わります。

コピペで一気に変更するより、まずは現在の構成を読むところから始めます。

## 有線VLANの現在状態を確認する

SSHで次を確認します。

```sh
echo "### Linux interfaces"
ip link show

echo "### IP addresses"
ip addr show

echo "### network UCI hints"
uci show network | grep -E 'device|bridge|ports|vlan|switch|ifname|proto'

echo "### bridge VLAN table, if available"
bridge vlan show 2>/dev/null || true

echo "### swconfig, if available"
swconfig list 2>/dev/null || true
swconfig dev switch0 show 2>/dev/null || true
```

ここでは設定を変更しません。

まず、LN6001-JPがDSA系の `bridge vlan` 表示なのか、従来の `swconfig` 系なのか、あるいは別の構成なのかを確認します。

OpenWrtでは世代やターゲットによって、VLAN設定の見え方が変わります。

LN6001-JPの実機でどう見えているかを確認してから進めるのが安全です。

## 有線VLANでやってはいけないこと

最初の段階では、次の操作は避けます。

- 管理端末がつながっているポートをいきなり別VLANへ移す
- LANブリッジから全ポートを外す
- VLAN IDだけ作ってfirewall zoneを作らない
- tagged / untaggedを理解せずに設定する
- 外部スイッチ側のVLAN設定と合わせずにtrunk化する
- バックアップなしで `/etc/config/network` を大きく変更する

有線VLANは、成功するとかなり便利です。

でも、失敗するとLuCIへ戻れなくなるタイプの設定です。

小さなオフィスや店舗では、営業時間外に作業し、有線で戻れる管理端末を確保してから進めるのがおすすめです。

## UCIでGuest / Deviceを作る例

ここからは上級者向けです。

最初はLuCIで作るほうが安全です。

以下は、無線SSID分離ベースで `guest` と `device` を作る例です。

有線ポートVLANは含めません。

実行前に、必ず現在のradio名と既存設定を確認してください。

LN6001-JPでは、Linksys公式手順上、`wifi0` が2.4GHz、`wifi1` が5GHz、`wifi2` が6GHzです。

ただし、実機の状態は必ず確認します。

```sh
uci show wireless | grep '=wifi-device'
uci show wireless | grep -E 'device|ssid|network'
```

以下は、5GHzにGuest、2.4GHzにDeviceを作る例です。

SSIDとパスワードは必ず変更してください。

```sh
BACKUP_DIR="/root/vlan-logical-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless dhcp firewall; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

GUEST_RADIO="wifi1"
DEVICE_RADIO="wifi0"

GUEST_SSID="Office_Guest"
GUEST_KEY="CHANGE_ME_GUEST_PASSWORD"

DEVICE_SSID="Office_Device"
DEVICE_KEY="CHANGE_ME_DEVICE_PASSWORD"

# guest bridge/interface
uci -q delete network.guest_dev
uci set network.guest_dev="device"
uci set network.guest_dev.type="bridge"
uci set network.guest_dev.name="br-guest"

uci -q delete network.guest
uci set network.guest="interface"
uci set network.guest.proto="static"
uci set network.guest.device="br-guest"
uci set network.guest.ifname="br-guest"
uci set network.guest.ipaddr="192.168.2.1"
uci set network.guest.netmask="255.255.255.0"

# device bridge/interface
uci -q delete network.device_dev
uci set network.device_dev="device"
uci set network.device_dev.type="bridge"
uci set network.device_dev.name="br-device"

uci -q delete network.device
uci set network.device="interface"
uci set network.device.proto="static"
uci set network.device.device="br-device"
uci set network.device.ifname="br-device"
uci set network.device.ipaddr="192.168.3.1"
uci set network.device.netmask="255.255.255.0"

# Guest SSID
uci -q delete wireless.guest
uci set wireless.guest="wifi-iface"
uci set wireless.guest.device="$GUEST_RADIO"
uci set wireless.guest.mode="ap"
uci set wireless.guest.network="guest"
uci set wireless.guest.ssid="$GUEST_SSID"
uci set wireless.guest.encryption="psk2+ccmp"
uci set wireless.guest.key="$GUEST_KEY"

# Device SSID
uci -q delete wireless.device
uci set wireless.device="wifi-iface"
uci set wireless.device.device="$DEVICE_RADIO"
uci set wireless.device.mode="ap"
uci set wireless.device.network="device"
uci set wireless.device.ssid="$DEVICE_SSID"
uci set wireless.device.encryption="psk2+ccmp"
uci set wireless.device.key="$DEVICE_KEY"

# DHCP
uci -q delete dhcp.guest
uci set dhcp.guest="dhcp"
uci set dhcp.guest.interface="guest"
uci set dhcp.guest.start="100"
uci set dhcp.guest.limit="50"
uci set dhcp.guest.leasetime="1h"

uci -q delete dhcp.device
uci set dhcp.device="dhcp"
uci set dhcp.device.interface="device"
uci set dhcp.device.start="100"
uci set dhcp.device.limit="50"
uci set dhcp.device.leasetime="12h"

# firewall zones
uci -q delete firewall.guest
uci set firewall.guest="zone"
uci set firewall.guest.name="guest"
uci set firewall.guest.network="guest"
uci set firewall.guest.input="REJECT"
uci set firewall.guest.output="ACCEPT"
uci set firewall.guest.forward="REJECT"

uci -q delete firewall.device
uci set firewall.device="zone"
uci set firewall.device.name="device"
uci set firewall.device.network="device"
uci set firewall.device.input="REJECT"
uci set firewall.device.output="ACCEPT"
uci set firewall.device.forward="REJECT"

# forward to wan
uci -q delete firewall.guest_wan
uci set firewall.guest_wan="forwarding"
uci set firewall.guest_wan.src="guest"
uci set firewall.guest_wan.dest="wan"

# device -> wan は必要なら有効化
uci -q delete firewall.device_wan
uci set firewall.device_wan="forwarding"
uci set firewall.device_wan.src="device"
uci set firewall.device_wan.dest="wan"

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

uci -q delete firewall.device_dhcp
uci set firewall.device_dhcp="rule"
uci set firewall.device_dhcp.name="Allow-Device-DHCP"
uci set firewall.device_dhcp.src="device"
uci set firewall.device_dhcp.proto="udp"
uci set firewall.device_dhcp.src_port="68"
uci set firewall.device_dhcp.dest_port="67"
uci set firewall.device_dhcp.family="ipv4"
uci set firewall.device_dhcp.target="ACCEPT"

# DNS allow
uci -q delete firewall.guest_dns
uci set firewall.guest_dns="rule"
uci set firewall.guest_dns.name="Allow-Guest-DNS"
uci set firewall.guest_dns.src="guest"
uci set firewall.guest_dns.proto="tcp udp"
uci set firewall.guest_dns.dest_port="53"
uci set firewall.guest_dns.target="ACCEPT"

uci -q delete firewall.device_dns
uci set firewall.device_dns="rule"
uci set firewall.device_dns.name="Allow-Device-DNS"
uci set firewall.device_dns.src="device"
uci set firewall.device_dns.proto="tcp udp"
uci set firewall.device_dns.dest_port="53"
uci set firewall.device_dns.target="ACCEPT"

uci changes network
uci changes wireless
uci changes dhcp
uci changes firewall

uci commit network
uci commit wireless
uci commit dhcp
uci commit firewall

/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
wifi reload
```

反映後、必ず実端末で確認します。

```sh
cat /tmp/dhcp.leases
ifstatus guest
ifstatus device
uci show firewall | grep -E 'guest|device|Allow-Guest|Allow-Device'
```

## 設定後に確認すること

設定後は、LuCIの画面だけで安心しないでください。

最後は必ず実端末で確認します。

## Guest端末

Guest用SSIDへ接続します。

期待する状態です。

```txt
IP address: 192.168.2.x
Gateway: 192.168.2.1
Internet: OK
Staff LAN: NG
LuCI: NG
```

Guest端末から次へアクセスできないことを確認します。

```txt
https://192.168.1.1
http://192.168.1.x
ssh root@192.168.1.1
```

ゲスト端末からLuCIやNASが見えたら、まだ分離できていません。

## Device端末

Device用SSIDへ接続します。

期待する状態です。

```txt
IP address: 192.168.3.x
Gateway: 192.168.3.1
Internet: 要件次第
Staff LAN: NG
```

カメラや録画機は、アプリや管理画面から見えるかも確認します。

Staff側から管理したい場合は、必要な通信だけ許可します。

## Staff端末

Staff側から、必要なDevice機器へアクセスできるか確認します。

ただし、Guestへアクセスする必要は基本的にありません。

Guest側は、外部利用端末向けにインターネットだけ使えれば十分です。

## CLIで状態を確認する

メインLAN側からSSHでLN6001-JPへ入り、状態を確認します。

```sh
echo "### interfaces"
ifstatus lan
ifstatus guest 2>/dev/null || true
ifstatus device 2>/dev/null || true

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### wireless mapping"
uci show wireless | grep -E 'ssid|network|guest|device'

echo "### firewall zones and rules"
uci show firewall | grep -E 'zone|forwarding|guest|device|Allow-Guest|Allow-Device|Staff'

echo "### addresses"
ip addr show

echo "### routes"
ip route show

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 80
```

firewallの実際の展開を見たい場合は、次も使えます。

```sh
iptables-save 2>/dev/null | grep -Ei 'guest|device' -A5 -B5
ip6tables-save 2>/dev/null | grep -Ei 'guest|device' -A5 -B5
```

最初は `uci show firewall` だけでも十分です。

## よくある失敗と対処

## GuestからStaffへ届いてしまう

これは分離としては失敗です。

次を確認します。

- Guest用SSIDのNetworkが `guest` になっているか
- guest zoneのForwardが `REJECT` になっているか
- guest → lan のforwardingを作っていないか
- Traffic RulesでGuestからLANを許可していないか

CLIでは次を確認します。

```sh
uci show wireless | grep -E 'guest|ssid|network'
uci show firewall | grep -E 'guest|lan|forwarding'
```

## Guest端末がIPアドレスを取れない

期待するIPアドレスは `192.168.2.x` です。

取得できない場合は、次を確認します。

- `guest` interfaceでDHCP Serverが有効か
- `Allow-Guest-DHCP` があるか
- Guest用SSIDのNetworkが `guest` になっているか
- 端末をWi-Fiから切断して再接続したか

```sh
uci show dhcp.guest
logread | grep -i dnsmasq | tail -n 50
cat /tmp/dhcp.leases
```

## GuestはIPを取れるがWebサイトが開けない

次を確認します。

- guest → wan forwardingがあるか
- `Allow-Guest-DNS` があるか
- WAN側インターネット接続が正常か
- 端末が別DNSやDoHを使っていないか

```sh
ifstatus wan
ifstatus guest
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'
logread | grep -Ei 'dnsmasq|firewall' | tail -n 80
```

## Device機器がアプリから見えない

Deviceネットワークへ分けると、スマート家電やカメラがアプリから見えなくなることがあります。

よくある原因です。

- 初期設定時だけ同じLANにいる必要がある
- mDNSやローカル探索が必要
- device → wanが必要なのに許可していない
- DNS広告ブロックで必要なドメインが止まっている
- StaffからDeviceへの管理通信が許可されていない

まずはDevice機器がIPアドレスを取得しているか、WANへ出る必要がある機器かを確認します。

## LuCIへ入れなくなった

有線VLANやfirewall設定を触った時に、管理アクセスを失うことがあります。

まず、有線LANでStaff側のLANポートへ直接つなぎ直します。

それでも入れない場合は、バックアップから戻します。

SSHで入れる場合は、保存していたバックアップから戻します。

```sh
cp /root/vlan-before-YYYYMMDD-HHMM/network /etc/config/network
cp /root/vlan-before-YYYYMMDD-HHMM/wireless /etc/config/wireless
cp /root/vlan-before-YYYYMMDD-HHMM/dhcp /etc/config/dhcp
cp /root/vlan-before-YYYYMMDD-HHMM/firewall /etc/config/firewall

/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
wifi reload
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えてください。

LuCIのバックアップファイルを使う場合は、**System** → **Backup / Flash Firmware** から復元します。

ここで焦ってリセットボタンを押す前に、まず有線LANと管理IPを確認します。

## 小さなオフィスでの運用メモ

設定したら、次のようなメモを残しておくと便利です。

```txt
Staff:
  network: lan
  IP: 192.168.1.0/24
  SSID: Office_Staff
  用途: 業務PC、管理端末、NAS、プリンター

Guest:
  network: guest
  IP: 192.168.2.0/24
  SSID: Office_Guest
  方針:
    guest -> wan: allow
    guest -> lan: reject

Device:
  network: device
  IP: 192.168.3.0/24
  SSID: Office_Device
  方針:
    device -> lan: reject
    device -> wan: 要件次第

POS:
  ベンダー要件確認中
  deviceへ雑に混ぜない

バックアップ:
  backup-LN6001-before-vlan-office-guest-20260621.tar.gz
```

ネットワーク分離は、作ったあとに説明できることが大事です。

誰がどのSSIDを使うのか。  
どの機器がどのIP範囲にいるのか。  
何を許可して、何を拒否しているのか。

これを残しておくと、後からかなり助かります。

半年後の自分は、たぶん設定内容を忘れています。

## まとめ

VLANやネットワーク分離は、小さなオフィスや店舗でとても役に立ちます。

ただし、最初から有線ポートまで全部VLAN化する必要はありません。

おすすめの順番はこうです。

1. Staff、Guest、Deviceの役割を決める
2. Guest用SSIDとguest networkを作る
3. Device用SSIDとdevice networkを作る
4. firewall zoneでGuest / DeviceからStaffへ入れないようにする
5. DHCP / DNSだけ必要な分を許可する
6. StaffからDeviceへは必要な機器だけ許可する
7. 有線VLANは実機ポートマップと管理スイッチを確認してから進める

LN6001-JPはOpenWrtベースなので、SSID、DHCP、firewall zone、VLANを組み合わせた柔軟な設計ができます。

でも、自由度が高いぶん、最初は小さく作るのが大事です。

まずはGuestからStaffへ入れない。

次にDeviceを分ける。

有線VLANはその後。

この順番なら、実用しながら少しずつ安全に整理できます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-03.png)

## 次に読むなら

VLANやネットワーク分離を進めるなら、次の記事が役に立ちます。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

Guestの基本分離をまだ作っていない人は、ゲストWi-Fiの記事へ。

zoneやforwardingの意味を整理したい人は、firewall zonesの記事へ。

カメラ、POS、プリンターなどのIP管理を整理したい人は、固定IPとDHCP予約の記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

有線VLANやbridge VLAN filteringの表示・設定方法は、ファームウェア、スイッチ構成、実機ポートマップによって変わることがあります。

記事の内容と実際の画面やコマンド出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

## よくある質問

### ゲストWi-FiだけならVLANは不要？

無線のゲスト端末を分けるだけなら、SSID、network、DHCP、firewall zoneで十分な場合があります。

有線ポートや管理スイッチを含めてStaff、Guest、Deviceを分けたい場合はVLANを検討します。

### VLANは家庭でも必要？

必須ではありません。

家庭なら、まずゲストWi-Fi、子ども用SSID、IoT用SSIDを分けるだけでも十分なことが多いです。

有線カメラ、NAS、管理スイッチなどが増えてきたらVLANを検討するとよいです。

### 店舗ではPOSもDevice VLANに入れていい？

慎重に判断してください。

POSや決済端末は、ベンダーのネットワーク要件やサポート条件がある場合があります。

この記事のDevice例にそのまま混ぜるのではなく、必要ならPOS専用ネットワークとして別に設計してください。

### 有線VLAN設定はどこから始めるのが安全？

まず現在のポート構成を確認します。

```sh
ip link show
bridge vlan show 2>/dev/null || true
uci show network | grep -E 'device|bridge|ports|vlan|switch|ifname'
```

管理端末がつながっているポートを残し、LuCIへ戻れる状態を維持してから、小さく変更します。

### guest zoneのInputをREJECTにしたら通信できなくなった

DHCPとDNSを明示的に許可してください。

Input `REJECT` では、ゲスト端末からルーター自身への通信が原則拒否されます。

そのため、`Allow-Guest-DHCP` と `Allow-Guest-DNS` のようなTraffic Ruleが必要です。

### StaffからDeviceへアクセスしたい場合は？

DeviceからStaffへ広く許可するのではなく、Staffから必要なDevice機器へ必要なポートだけ許可します。

録画機、カメラ、管理画面など、宛先IPアドレスとポートを絞るのがおすすめです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- OpenWrt DSA Mini-Tutorial: https://openwrt.org/docs/guide-user/network/dsa/dsa-mini-tutorial
- OpenWrt network configuration: https://openwrt.org/docs/guide-user/network/network_configuration
- OpenWrt firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration

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
