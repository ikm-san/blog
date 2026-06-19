<!-- mirror-source: articles/010-family-filtering.md -->

> Original note.com article: [家族向けフィルタリング: 端末別にDNSと通信ルールを分ける【OpenWrt集中連載010】](https://note.com/ikmsan/n/n284bb49cd1e3)

# 家族向けフィルタリング: 端末別にDNSと通信ルールを分ける【OpenWrt集中連載010】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

家族向けフィルタリングの第一歩は、「強く制限する」ことではなく、「端末を分類する」ことです。

家族用、子ども用、IoT用、来客用でネットワークを分けると、DNS設定や firewall rules を端末グループごとに適用しやすくなります。

LN6001-JPはOpenWrtベースなので、SSID、DHCP、DNS、firewall zone を組み合わせた設計を作れます。

ただし、最初から細かな制限ルールを大量に作る必要はありません。まずは「Kids用SSIDを分ける」「KidsだけフィルタリングDNSを使う」くらいでも、家庭内ネットワークはかなり整理しやすくなります。

この記事では、家庭向けの現実的なネットワーク分離として、Kids、IoT、Guestをどう分けると扱いやすいかを順番に見ていきます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-01.png)

## この記事でわかること

- 家族用、子ども用、IoT用でネットワークを分ける考え方
- フィルタリングDNSをどう配ると整理しやすいか
- firewall zone を使った基本的な分離方法
- ルーター側でできることと、端末側に任せるべきこと

## こんな家庭に向いています

- 子ども用タブレットやゲーム機だけ、少し強めに見守りたい
- スマート家電やカメラを、家族のPCやNASと分けておきたい
- 来客用Wi-Fiも含めて、家庭内ネットワークを整理したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-02.png)

## まずはここまでで十分

最初から全部を細かく作り込む必要はありません。むしろ、最初はシンプルな分類だけのほうが運用しやすくなります。

まずは次の3つだけできれば、家庭向けの整理としては十分に効果があります。

1. Kids用のネットワークとSSIDを分ける
2. KidsへフィルタリングDNSを配る
3. IoTは必要になった時に追加する

最初の一歩としては、「子ども用だけ分ける」で十分です。GuestやIoTはあとから足しても遅くありません。

ネットワーク分離は、“最初から完璧に制御する” より、“混ぜない” ところから始めるほうが整理しやすいです。

## 設定前に現在の状態を控える

Kids、IoT、Guestを分ける設定では、`network`、`wireless`、`firewall`、`dhcp` をまとめて触ります。まずは現在の状態を保存してから進めます。

```sh
BACKUP_DIR="/root/family-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
ip addr show > "$BACKUP_DIR/ip-addr.txt"

ls -l "$BACKUP_DIR"
```

Wi-Fiパスワードや端末情報が含まれるため、このバックアップをそのまま共有しないようにします。

## 端末グループの分類から始める

最初に考えるのは、細かな制限内容よりも「どの端末をどこへ置くか」です。

先に分類を決めておくと、あとからDNSやfirewallルールを整理しやすくなります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-01.png)

家庭では、「親用LANに全部入れる」状態から始まりがちですが、子ども用端末やIoT機器だけでも分けておくと、トラブルや管理がかなり整理しやすくなります。

## ネットワーク設計の全体像

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-02.png)

全部を最初から作る必要はありません。

まずは `LAN + Kids` の2系統だけでも十分意味があります。IoTやGuestは、必要になった時にあとから追加していくほうが無理がありません。

家庭でのざっくりした置き方の例:

- 親のスマートフォン、PC、NAS は LAN
- 子どものタブレット、ゲーム機は Kids
- テレビ、見守りカメラ、スマートスピーカーは IoT
- 友人や親族の端末は Guest

## 子ども用ネットワークの作り方

Kids用ネットワークでは、「別SSIDを作る」だけでなく、「別IPアドレス帯」と「別firewall zone」を作るところまで行うと、あとから管理しやすくなります。

### ステップ1: Kidsネットワークを作る

1. **Network** → **Interfaces** → **Add new interface**
2. Name: `kids`、Protocol: `Static address`
3. **General Settings** タブ:
   - IPv4 address: `192.168.3.1`
   - IPv4 netmask: `255.255.255.0`
4. **DHCP Server** タブ → **Setup DHCP Server**:
   - Start: `100`、Limit: `50`、Leasetime: `2h`
5. **DHCP Server** → **Advanced Settings** タブ:
   - **DHCP-Options**: `6,1.1.1.3,1.0.0.3`（Cloudflare Family DNSを配布）
6. **Save**

ここで作っているのは、Kids端末専用のIPアドレス空間です。

親用LANと別の `192.168.3.x` を使うことで、あとから通信ルールを整理しやすくなります。

### ステップ2: Kids用SSIDを作る

1. **Network** → **Wireless** → 任意帯域の **Add**
2. ESSID: `MyHome_Kids`
3. Network: `kids`（作成したネットワーク）
4. Encryption: `WPA2-PSK`、Key: 子ども用パスワード
5. **Save** → **Apply**

Kids用SSIDは、家族用SSIDと完全に同じ名前にしないほうが管理しやすいです。

「MyHome_Kids」のように、親側から見ても分かりやすい名前にしておくと整理しやすくなります。

### ステップ3: firewall zoneを設定する

ここが「KidsネットワークをLANから分ける」部分です。

SSIDだけ分けても、firewall zone を分けなければLANへそのまま届いてしまうことがあります。

1. **Network** → **Firewall** → **Zones** → **Add**
2. Name: `kids`、Input: `REJECT`、Output: `ACCEPT`、Forward: `REJECT`
3. Covered networks: `kids`
4. Allow forward to destination zones: `wan`（インターネットへの接続を許可）

この設定では、「Kids → インターネット」は許可し、「Kids → LAN」は許可しない構成になります。

5. **Save & Apply**

### UCIコマンドで設定する場合

KidsネットワークはCLIでも設定できますが、最初はLuCIから設定したほうが構成を理解しやすいです。

UCIを使う場合も、まず `uci show network` や `uci show firewall` で現在設定を確認してから変更するほうが安全です。

```sh
BACKUP_DIR="/root/kids-before-uci-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
for cfg in network dhcp firewall wireless; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

# Kidsネットワーク
uci set network.kids=interface
uci set network.kids.proto='static'
uci set network.kids.ipaddr='192.168.3.1'
uci set network.kids.netmask='255.255.255.0'
uci changes network
uci commit network

# KidsDHCP（Cloudflare Family DNSを配布）
uci set dhcp.kids=dhcp
uci set dhcp.kids.interface='kids'
uci set dhcp.kids.start='100'
uci set dhcp.kids.limit='50'
uci set dhcp.kids.leasetime='2h'
uci add_list dhcp.kids.dhcp_option='6,1.1.1.3,1.0.0.3'
uci changes dhcp
uci commit dhcp

# Kidsファイアウォールゾーン
uci add firewall zone
uci set firewall.@zone[-1].name='kids'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci add_list firewall.@zone[-1].network='kids'

uci add firewall forwarding
uci set firewall.@forwarding[-1].src='kids'
uci set firewall.@forwarding[-1].dest='wan'

# Kids端末がIPアドレスを取得できるようDHCPを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Kids-DHCP'
uci set firewall.@rule[-1].src='kids'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='67-68'
uci set firewall.@rule[-1].target='ACCEPT'

# Kids端末が指定DNSを使えるようDNSを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Kids-DNS'
uci set firewall.@rule[-1].src='kids'
uci set firewall.@rule[-1].proto='tcpudp'
uci set firewall.@rule[-1].dest_port='53'
uci set firewall.@rule[-1].target='ACCEPT'

uci changes firewall
uci commit firewall
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
```

## IoT用ネットワークの作り方

IoT機器は、家庭内LANのNASやPCにアクセスする必要がないものが多いため、独立したネットワークへ分けてLAN通信を制限します。

特に見守りカメラやスマート家電は、最初からIoT用へまとめておくと整理しやすくなります。

### IoTネットワーク設定

1. **Network** → **Interfaces** → **Add new interface**
2. Name: `iot`、Protocol: `Static address`
3. IPv4 address: `192.168.4.1`、Netmask: `255.255.255.0`
4. DHCPサーバーを設定（Leasetime: `12h`）
5. **Save**

### IoT用SSIDを作る

1. **Network** → **Wireless** → 2.4GHz帯の **Add**
2. ESSID: `MyHome_IoT`
3. Network: `iot`
4. Encryption: `WPA2-PSK`
5. **Save** → **Apply**

IoT機器は2.4GHz帯しか対応していないことも多いため、IoT用SSIDは2.4GHz側で作るとつながりやすい場合があります。

### IoT用firewallゾーン（LAN通信を制限）

1. **Network** → **Firewall** → **Zones** → **Add**
2. Name: `iot`、Input: `REJECT`、Output: `ACCEPT`、Forward: `REJECT`
3. Covered networks: `iot`
4. Allow forward to destination zones: `wan`（クラウド通信のみ許可）
5. **Save & Apply**

### IoTからLANへの通信を明示的に遮断

```sh
echo "### backup firewall config"
cp /etc/config/firewall /etc/config/firewall.backup.$(date +%Y%m%d-%H%M)

echo "### block IoT to LAN"
uci add firewall rule
uci set firewall.@rule[-1].name='Block IoT to LAN'
uci set firewall.@rule[-1].src='iot'
uci set firewall.@rule[-1].dest='lan'
uci set firewall.@rule[-1].target='REJECT'
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

IoT機器はクラウド通信だけ必要なケースも多いため、「IoT → WANだけ許可」という構成にすると整理しやすくなります。

## DHCP予約で端末を固定する

特定の端末（NAS・プリンター・カメラなど）へ固定IPアドレスを割り当てます。

端末が増えてくると、「毎回IPが変わる」状態より、重要機器だけ固定しておいたほうが管理しやすくなります。

### LuCIでの設定手順

1. **Network** → **DHCP and DNS** → **Static Leases** タブを開く
2. **Add** をクリック
3. 以下を入力:
   - Hostname: 端末の名前（例: `nas-1`）
   - MAC address: 端末のMACアドレス（Wi-Fi設定やルーターのDHCPリストから確認）
   - IP address: 割り当てたい固定IP（例: `192.168.1.100`）
4. **Save & Apply**

端末のMACアドレスは `cat /tmp/dhcp.leases` で確認できます（現在接続中の端末一覧）。

固定IPを付ける時は、ネットワークごとにアドレス範囲を整理しておくと、あとから見返した時にも分かりやすくなります。

## フィルタリングDNSの選択肢

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-03.png)

最初は Cloudflare Family DNS くらいから試すと入りやすいです。

フィルタリング精度や誤判定はサービスごとに違うため、「どれが絶対によい」というより、家庭に合うものを試しながら決めるほうが現実的です。

## DNSだけでできること、できないこと

DNSフィルタリングで制御できるのは、名前解決が中心です。

期待しすぎないために、「DNSでできること」と「端末側でやるべきこと」を分けて考えると整理しやすくなります。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-04.png)

つまり、DNSフィルタリングは「全部を制御する機能」というより、「危険サイトや不要な入口を減らす仕組み」と考えるほうが分かりやすいです。

子ども用端末の管理は、ルーター側のネットワーク分離とDNSフィルタリングに加えて、端末側のペアレンタルコントロール機能や家庭内ルールを組み合わせるのが現実的です。

ルーターだけで全部を制御しようとすると複雑になりやすいため、「ルーター側は入口整理、細かな利用制限は端末側」と分けると運用しやすくなります。

## 設定後の確認方法

まずは次の3つが確認できれば、設定は大きく外していません。最初はここまで確認できれば十分です。

```sh
echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### kids/iot interfaces"
uci show network.kids 2>/dev/null || true
uci show network.iot 2>/dev/null || true
ip addr show | grep -A4 -E 'br-kids|kids|br-iot|iot'

echo "### DHCP DNS options"
uci show dhcp | grep -E 'kids|iot|dhcp_option'

echo "### firewall family zones"
uci show firewall | grep -E 'kids|iot|guest|Allow-Kids|Block IoT'

echo "### DHCP logs"
logread | grep -i dnsmasq | tail -n 80
```

`cat /tmp/dhcp.leases` は、どの端末がどのネットワークへ入っているかを見る時にかなり便利です。

CLIは、最初は設定変更より「今どうつながっているか」を確認する用途で使うくらいで十分です。

## まとめ

家族向けネットワーク分離は、次の順番で考えると整理しやすいです:

1. 子ども用・IoT用ネットワーク（インターフェース）を作成
2. それぞれのSSIDを作成してネットワークに紐付け
3. firewall zoneを設定してLAN間通信を制限
4. 子ども用にはDHCPでフィルタリングDNSを配布
5. 重要な機器はDHCP予約で固定IPを割り当て

制限より分類を先に行うことで、各グループに適したDNSとfirewall設定を整理しやすくなります。

端末の制御は端末側のペアレンタルコントロール機能と組み合わせると、ルーター側設定を過度に複雑にせずに済みます。

最初から完璧な制限を目指すより、「Kidsだけ分ける」「IoTだけ分ける」くらいから始めるほうが、家庭では運用しやすいです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-03.png)

## よくある質問

### 子ども用Wi-FiのフィルタリングはDNSだけで十分？

十分ではありません。

DNSは入口の整理には役立ちますが、利用時間の制限やアプリごとの制御は端末側機能も組み合わせるほうが現実的です。

### スマート家電やカメラは家族用Wi-Fiと分けたほうがいい？

分けたほうが管理しやすいです。IoT機器はLAN内のPCやNASへ触れさせない設計にすると整理しやすくなります。

### 家庭のWi-Fiは最初からKids用とIoT用に分けるべき？

最初はKidsとIoTを分けるくらいでも十分です。

必要になってからGuestや細かな制御を足すほうが無理がありません。最初から複雑にしすぎないほうが、家庭では運用しやすくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Cloudflare Family DNS: https://1.1.1.1/family/

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
