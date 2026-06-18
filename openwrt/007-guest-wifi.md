<!-- mirror-source: articles/007-guest-wifi.md -->

> Original note.com article: [ゲストWi‑Fiの作り方: 家庭内LANと来客端末を分ける【OpenWrt集中連載007】](https://note.com/ikmsan/n/nacbd5d573d67)

# ゲストWi‑Fiの作り方: 家庭内LANと来客端末を分ける【OpenWrt集中連載007】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

ゲストWi‑Fiは、来客にインターネット接続を提供しつつ、家庭内LANや業務端末には触れさせないための基本機能です。

ただ、SSIDを分けるだけでは十分ではありません。同じLANネットワークにつながったままだと、来客端末からNAS、プリンター、ルーター管理画面などが見えてしまうことがあります。

LN6001-JPはOpenWrtベースなので、SSIDだけでなく、ネットワーク（IPアドレス範囲）や firewall zone まで含めて整理できます。

最初は難しく見えるかもしれませんが、やること自体はシンプルです。

- Guest用SSIDを作る
- Guest専用ネットワークを作る
- GuestからLANへ入れないようにする

この記事では、この3つをLuCIで順番に設定していきます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-01.png)

## この記事でわかること

- ゲストWi‑FiでSSIDを分けるだけでは足りない理由
- Guest用ネットワークと firewall zone の作り方
- 来客端末をLANへ入れないための確認ポイント
- 家庭・店舗でGuest Wi‑Fiをどう分けると運用しやすいか

## こんな人に向いています

- 家庭で来客用Wi-Fiを安全に分けたい
- 店舗や事務所で、お客様用Wi-Fiと業務用ネットワークを分けたい
- SSIDを分けただけで十分か不安がある

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-02.png)

## まずはここまでで十分

最初から細かなルールを増やしすぎなくても大丈夫です。むしろ、最初はシンプルな分離だけのほうがトラブルも少なくなります。

まずは次の3つができれば、ゲストWi‑Fiとしては十分に役立ちます。

1. Guest用の独立したネットワークを作る
2. Guest用SSIDをそのネットワークへ紐付ける
3. GuestからLANへ入れないよう firewall zone を分ける

まずは「インターネットには出られるが、自宅や業務の機器は見えない」状態を作ることが優先です。

Guest Wi‑Fiは、“便利な来客用回線” というより、“家庭や業務ネットワークを混ぜないための入口” と考えると整理しやすくなります。

## 設定前に現在の状態を控える

Guest Wi-Fi設定では、`network`、`wireless`、`firewall`、`dhcp` をまとめて触ります。変更前に現在の状態を保存しておくと、設定を戻したい時にかなり安心です。

```sh
BACKUP_DIR="/root/guest-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus lan > "$BACKUP_DIR/ifstatus-lan.json" 2>/dev/null || true
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status.json"
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"

ls -l "$BACKUP_DIR"
```

バックアップにはSSIDやWi-Fi関連の情報が含まれるため、そのまま公開場所へ貼り付けないようにします。

## なぜSSIDを分けるだけでは足りないか

SSIDが違っていても、同じLANネットワークにつながっていれば、来客端末から家庭内のNAS、プリンター、管理画面が見えてしまうことがあります。

「Wi‑Fi名が違う = 完全に分離されている」ではないところが、最初に少し分かりにくいポイントです。

| 要素 | 役割 |
|---|---|
| SSID | 来客が接続するWi-Fi名 |
| 独立したネットワーク | Guest用のIPアドレス範囲（例: 192.168.2.0/24） |
| firewall zone | GuestからLANへの通信を遮断する |

つまり、Guest Wi‑Fiは「SSIDだけ」ではなく、「IPアドレス範囲」と「通信ルール」まで分けて初めて意味が出てきます。

OpenWrt系では、Guest Wi‑Fiを作る時に「SSID」「ネットワーク」「firewall zone」を別々に設定します。

最初は少し複雑に見えますが、役割を分けているぶん、あとから整理しやすいのが特徴です。

### ステップ1: Guestネットワーク（インターフェース）を作る

1. **Network** → **Interfaces** を開く
2. **Add new interface** をクリック
3. 以下を設定:

| フィールド | 設定値 |
|---|---|
| Name | `guest` |
| Protocol | `Static address` |
| Device | なし（後でSSIDと紐付け） |

4. **Create interface** をクリック
5. **General Settings** タブ:
   - IPv4 address: `192.168.2.1`
   - IPv4 netmask: `255.255.255.0`
6. **DHCP Server** タブ → **Setup DHCP Server** をクリック:
   - Start: `100`
   - Limit: `50`
   - Leasetime: `1h`
7. **Save** をクリック

ここで作っているのは、Guest端末専用のIPアドレス空間です。

家庭LANとは別の `192.168.2.x` を使うことで、あとから firewall で分離しやすくなります。

### ステップ2: Guest用SSIDを作る

1. **Network** → **Wireless** を開く
2. 任意の帯域（例: 5GHz `wifi1`）の **Add** をクリック
3. **Interface Configuration** タブ:

| フィールド | 設定値 |
|---|---|
| ESSID | `Guest` または `MyHome_Guest` |
| Mode | `Access Point` |
| Network | `guest`（ステップ1で作成したネットワーク） |

4. **Wireless Security** タブ:

| フィールド | 設定値 |
|---|---|
| Encryption | `WPA2-PSK` |
| Key | 来客用パスワード（家族用SSIDとは別にする） |

5. **Save** をクリック

Guest用SSIDは、家族用SSIDと完全に同じにしないほうが分かりやすいです。

「Guest」「MyHome_Guest」のように、来客側でも見分けやすい名前にしておくと運用しやすくなります。

### ステップ3: firewall zoneを設定する

ここがGuest Wi‑Fiの“分離”を実際に行う部分です。

SSIDだけ分けても、firewall zone を分けなければLANへ到達できてしまうことがあります。

1. **Network** → **Firewall** を開く
2. **Zones** タブで **Add** をクリック
3. **General Settings** タブ:

| フィールド | 設定値 |
|---|---|
| Name | `guest` |
| Input | `REJECT` |
| Output | `ACCEPT` |
| Forward | `REJECT` |
| Covered networks | `guest` |

4. **Inter-Zone Forwarding** セクション:
   - **Allow forward to destination zones**: `wan` にチェック
   - **Allow forward from source zones**: （空白のまま）

5. **Save & Apply** をクリック

この設定では、「Guest → インターネット」は許可し、「Guest → LAN」は許可しない構成になります。

### ステップ4: GuestからLANへのアクセスを確実に遮断する

デフォルトのfirewall設定では通常GuestゾーンからLANへの転送は拒否されますが、明示的なルールを追加しておくと、あとから設定を見返した時にも分かりやすくなります。

1. **Network** → **Firewall** → **Traffic Rules** タブを開く
2. **Add** をクリック
3. 以下を設定:

| フィールド | 設定値 |
|---|---|
| Name | `Block Guest to LAN` |
| Protocol | `any` |
| Source zone | `guest` |
| Destination zone | `lan` |
| Action | `REJECT` |

4. **Save & Apply**

### ステップ5: GuestからルーターのLuCIへのアクセスを遮断する

Guest端末がルーターの管理画面（192.168.1.1）にアクセスできないようにします。

家庭用途でも、ここを分けておくと「来客に管理画面を見せない」状態を作りやすくなります。

1. **Network** → **Firewall** → **Traffic Rules** タブを開く
2. **Add** をクリック
3. 以下を設定:

| フィールド | 設定値 |
|---|---|
| Name | `Block Guest to Router Admin` |
| Protocol | `TCP` |
| Source zone | `guest` |
| Destination address | `192.168.1.1` |
| Destination port | `443, 80` |
| Action | `REJECT` |

4. **Save & Apply**

## 設定後の確認方法

設定が終わったら、必ず実際の端末で確認します。

「設定を入れた」だけで終わらず、「本当に分離できているか」を確認するところまでがGuest Wi‑Fi設定です。

以下を必ず確認します:

**1. Guest端末からインターネットにつながるか**
- Guest SSIDに接続して、ブラウザで任意のWebサイトを開く

**2. Guest端末からLAN内の機器が見えないか**
- Guest SSIDに接続した状態で、`192.168.1.1`（ルーター）にアクセスしてみる → 接続できないことを確認
- NASやプリンターのIPアドレスへアクセスしてみる → 接続できないことを確認

**3. Guest端末がDHCPでIPアドレスを取得できているか**
- Guest端末のWi-Fi設定で、IPアドレスが `192.168.2.x` になっていることを確認

## CLIで状態を確認する

```sh
echo "### Guest DHCP leases"
cat /tmp/dhcp.leases

echo "### Guest network config"
uci show network.guest

echo "### Guest DHCP config"
uci show dhcp.guest

echo "### Guest firewall hints"
uci show firewall | grep -E "guest|Allow-Guest|Block Guest"

echo "### Guest interface addresses"
ip addr show | grep -A4 -E 'br-guest|guest'

echo "### dnsmasq logs"
logread | grep -i dnsmasq | grep guest | tail -n 30
```

`uci show network.guest` と `uci show firewall` は、Guestネットワークがどこへ紐付いているかを確認する時に便利です。

最初はCLIで変更するより、「今どう設定されているか」を見る用途で使うくらいで十分です。

## 設定変更前のバックアップ

```sh
BACKUP_DIR="/root/guest-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ls -l "$BACKUP_DIR"
```

Guest Wi‑Fi設定は、network / wireless / firewall の3つへ同時に変更が入ります。変更前バックアップを残しておくと、設定を戻したい時にかなり安心です。

## よくある失敗と対処

その前に、次の3つが確認できれば大枠では成功しています。最初はここまで確認できれば十分です。

- Guest 端末が `192.168.2.x` のIPアドレスを取得している
- Guest 端末からインターネットへ出られる
- Guest 端末から NAS、プリンター、管理画面へ直接届かない

### Guest端末がIPアドレスを取得できない

**原因:** DHCPサーバーがGuestインターフェースで動いていない
**対処:** `Network > Interfaces > guest > Edit > DHCP Server` タブで「Setup DHCP Server」をクリックし、DHCPを有効化する

### Guestからインターネットへ出られない

**原因:** firewall zoneの設定でWANへの転送が許可されていない
**対処:** `Network > Firewall > Zones` でguestゾーンの「Allow forward to destination zones」に `wan` が含まれているか確認する

### Guestから管理画面にアクセスできてしまう

**原因:** firewall ruleの設定が不足している
**対処:** 上記ステップ5の「Block Guest to Router Admin」ルールを追加する

### 変更後にLANからLuCIにアクセスできなくなった

firewall設定を触ると、意図せず自分自身の管理アクセスまで止めてしまうことがあります。

最初は有線LANを1本つないだ状態で作業すると、復旧しやすくなります。

1. 有線でLANポートに接続する
2. バックアップから設定を復元する: `cp /etc/config/firewall.backup.YYYYMMDD /etc/config/firewall && /etc/init.d/firewall restart`

## UCIで設定する方法（上級者向け）

Guest Wi‑FiはCLIでも設定できますが、最初はLuCIから作るほうが安全です。

UCIで設定する場合も、まず `uci show network` や `uci show firewall` で現在設定を確認してから変更することをおすすめします。

```sh
BACKUP_DIR="/root/guest-uci-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
for cfg in network dhcp firewall wireless; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

# Guestネットワーク追加
uci set network.guest=interface
uci set network.guest.proto='static'
uci set network.guest.ipaddr='192.168.2.1'
uci set network.guest.netmask='255.255.255.0'
uci changes network
uci commit network

# DHCPサーバー追加
uci set dhcp.guest=dhcp
uci set dhcp.guest.interface='guest'
uci set dhcp.guest.start='100'
uci set dhcp.guest.limit='50'
uci set dhcp.guest.leasetime='1h'
uci changes dhcp
uci commit dhcp

# firewall zone追加
uci add firewall zone
uci set firewall.@zone[-1].name='guest'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci add_list firewall.@zone[-1].network='guest'

# GuestからWANへの転送を許可
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='guest'
uci set firewall.@forwarding[-1].dest='wan'

# Guest端末がIPアドレスを取得できるようDHCPを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Guest-DHCP'
uci set firewall.@rule[-1].src='guest'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='67-68'
uci set firewall.@rule[-1].target='ACCEPT'

# Guest端末がルーターのDNSを使えるようDNSを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Guest-DNS'
uci set firewall.@rule[-1].src='guest'
uci set firewall.@rule[-1].proto='tcpudp'
uci set firewall.@rule[-1].dest_port='53'
uci set firewall.@rule[-1].target='ACCEPT'

uci changes firewall
uci commit firewall

# サービスを再起動
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
```

反映後はGuest端末を接続し、`cat /tmp/dhcp.leases`、`uci show firewall | grep guest`、Guest端末からのWeb閲覧、Guest端末から `https://192.168.1.1` へ届かないことを確認します。

## 家庭と店舗での使い分け

Guest Wi‑Fiは、家庭でも店舗でも「端末を混ぜない」ためにかなり役立ちます。

特に最近は、スマートフォン、IoT、ゲーム機、防犯カメラなど、接続機器が増えているため、最初から分けておくほうが管理しやすくなります。

**家庭:**
- 友人・親戚の来客用
- 子どもの友人のゲーム機・スマートフォン
- 一時的な作業業者へのWi-Fi提供
- パスワードを定期的に変えて渡す運用が向く

**店舗:**
- 来客へのフリーWi-Fi提供
- スタッフ用SSIDとは必ず分ける
- POSや決済端末は別ネットワークに置く
- 業務PCや防犯カメラへのアクセスを遮断する

## まとめ

ゲストWi‑Fiの作成は、次の3つをセットで考えると整理しやすいです:

1. **独立したネットワーク作成**: Guest用のIPアドレス範囲（192.168.2.0/24）
2. **Guest用SSID作成**: Guestネットワークへ紐付け
3. **firewall zone設定**: Guest→LAN通信を遮断し、Guest→WAN通信のみ許可

設定後は必ずGuest端末から実際に確認します。インターネットへ出られること、LAN内の機器へアクセスできないことの両方を確認して初めて、分離が機能したと言えます。

最初から細かい制御を増やすより、「GuestをLANへ入れない」という基本分離を確実に作るほうが重要です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/007/diagram-03.png)

## よくある質問

### ゲストWi-FiはSSIDを分けるだけで十分？

十分ではありません。Guest用のネットワークと firewall zone も分けて、LANへの通信を止める必要があります。

### ゲストWi-Fiからルーター管理画面は見えないようにできる？

できます。Traffic Ruleで Guest から `192.168.1.1` へのアクセスを拒否する設定を入れると分かりやすいです。

### 家庭でもゲストWi-Fiは作ったほうがいい？

来客があるなら作っておく価値があります。

家庭内機器と来客端末を分けるだけでも、かなり整理しやすくなります。IoT機器が増えている家庭ほど、Guest分離のメリットが出やすいです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT WiFi settings: https://support.linksys.com/kb/article/221-en/?section_id=175

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
