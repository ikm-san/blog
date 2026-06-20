<!-- mirror-source: articles/021-troubleshooting-connectivity.md -->

# つながらない時の切り分け: WAN/LAN/DNS/Wi-Fiを分けて見る【OpenWrt集中連載021】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

「つながらない」は、WAN・LAN・DNS・Wi‑Fiのどこで止まっているかを分けて確認すると、かなり切り分けしやすくなります。

特にIPoE環境では、「IPv4だけ落ちている」「WAN6だけ不安定」「DNSだけ失敗している」など、原因が分かれることがあります。

ただし、最初から全部のログや設定を見始めなくても大丈夫です。

まずは「物理」「WAN」「DNS」のどこで止まっているかを分けるだけでも、かなり前に進めます。

LN6001-JPでは、LuCIとSSHの両方で状態確認できるため、勘に頼らず切り分けできます。

この記事では、症状別の確認順序、まず覚えておきたい確認コマンド、設定変更後に戻す時の考え方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-01.png)

## この記事でわかること

- 「つながらない」をWAN・LAN・DNS・Wi‑Fiへ分けて見る方法
- LN6001-JPで最初に確認したいコマンド
- 設定変更直後にどこから戻すべきか

## こんな時に向いています

- Wi-Fiにはつながるのにインターネットが使えない
- 設定変更のあと、どこが壊れたのか見当がつかない
- WAN、DNS、DHCPのどれを疑うべきか迷っている

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-02.png)

## まずはここまでで十分

トラブル時は、最初から全部の設定を見始めるより、まず次の3つから確認したほうが整理しやすくなります。

1. LED・ONU/HGW・ケーブルの物理状態
2. 有線でLuCIへ入れるか
3. `ifstatus wan` と `nslookup` でWANかDNSかを分ける

最初の数分で「物理」「WAN」「DNS」のどこかまで絞れれば、かなり前へ進めます。

最初は「全部を理解する」より、「どこで止まっているか」を分けることが大事です。

トラブル対応では、「全部おかしい」と考えるより、「どこまでは正常か」を確認するほうが切り分けしやすくなります。

## 3分で集める診断ログ

原因が分からない時は、設定を直し始める前に、まず読み取りコマンドで状態を残します。LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアでは、次のような順番で見ると、WAN、IPv6、DNS、DHCP、Wi-Fi、firewallを分けやすくなります。

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
cat /tmp/resolv.conf* 2>/dev/null
cat /tmp/dhcp.leases
uci show dhcp | grep -E 'dns|server|dhcp_option|ignore' || true

echo "### wifi"
wifi status
iw dev

echo "### firewall"
uci show firewall
command -v iptables >/dev/null && iptables -L -n -v | head -n 80

echo "### recent logs"
logread | tail -n 120
```

この出力にはMACアドレス、内部IP、SSID、端末名、プロバイダ情報が含まれることがあります。公開のコメント欄やSNSへそのまま貼らず、必要な範囲だけ共有してください。

読み方の目安は次の通りです。

- `ifstatus wan` / `ifstatus wan6`: WAN側がupか、IPv4/IPv6アドレスがあるか
- `ip route show`: デフォルトルートがあるか
- `cat /tmp/dhcp.leases`: 端末にIPアドレスが配られているか
- `wifi status`: SSIDが有効か、radioがdownになっていないか
- `iptables -L -n -v`: DROP/REJECTが想定外に増えていないか
- `logread | tail`: 直近に失敗ログが集中していないか

## まず症状を分類する

「つながらない」の前に、まず状況を分けます:

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-01.png)

最初は、「Wi‑Fi問題なのか」「DHCP問題なのか」「WAN問題なのか」を分けるだけでもかなり役立ちます。

実際には、「設定問題」より「ケーブル」「ONU/HGW」「再起動中」だった、というケースもかなり多くあります。

## ステップ1: 物理・LED状態の確認

```
電源 → LED → ケーブル → ONU/HGW
```

- **電源**: LN6001-JPの電源LEDが点灯しているか
- **LED状態**: 上部LED白点灯=オンライン、赤点灯=インターネット接続なし、青点滅=起動中
- **WANケーブル**: ONU/HGWとLN6001-JPのWANポート間のケーブルが抜けていないか
- **ONU/HGWの状態**: ONU/HGWの電源・LED状態も確認（LN6001-JPではなくONU/HGW側の問題の可能性）

最初は、ルーターだけでなくONU/HGW側LEDも確認するほうが切り分けしやすくなります。

## ステップ2: 有線でLuCIにアクセスできるか確認

PCとLN6001-JPのLANポートを有線接続して `https://192.168.1.1` にアクセスします。

Wi‑Fi設定変更直後は、有線で確認したほうがかなり安全です。

- **アクセスできる**: LN6001-JPは動作中。Wi-Fi設定やWAN接続の問題を確認
- **アクセスできない**: LAN側IPアドレスの変更、PCのIP設定、PCのブラウザキャッシュを確認

```powershell
# PCから確認（Windowsの場合）
ipconfig
```

```sh
# PCから確認（MacまたはLinuxの場合）
ip addr show
```

最初は「PC側が正しいIPを取れているか」を見るだけでもかなり意味があります。

LN6001-JPのLAN IPを変更している場合は、変更後IPアドレスでアクセスします。

有線でLuCIへ入れるなら、LAN側はかなり正常に近い状態です。

次は「WAN側へ出られているか」を確認します。

## ステップ3: WANの接続状態を確認

LuCI: **Status** → **Overview** でWAN/WAN6のIPアドレスが表示されているか確認。

```sh
# WAN（IPv4）の状態確認
ifstatus wan

# WAN6（IPv6）の状態確認
ifstatus wan6

# 出力の中で "up": true, "ipaddr": があるか確認
ifstatus wan | grep -E '"up"|"ipaddr"'
ifstatus wan6 | grep -E '"up"|"ip6addr"'
```

`ifstatus wan` と `ifstatus wan6` は、まず最初に確認したいコマンドです。

IPoE環境では、「WANは正常だがWAN6だけ落ちている」ケースもあります。

**WANがupでIPアドレスがない場合:**

```sh
# PPPoEの場合: 設定を確認
uci show network.wan

# IPoEの場合: オートIPoEモジュールが正常か確認
opkg list-installed | grep -i ipoe

# ログで接続エラーを確認
logread | grep -i 'wan\|pppoe\|ipoe' | tail -n 30
```

最初は「IPアドレスが付いているか」を見るだけでもかなり切り分けしやすくなります。

次は、「WANは上がっているがDNSだけ壊れている」のかを分けます。

## ステップ4: 疎通確認（IP直接 vs ドメイン名）

```sh
# インターネット疎通確認（IPアドレス直接）
ping -c 3 8.8.8.8       # IPv4
ping6 -c 3 2001:4860:4860::8888  # IPv6

# DNS解決確認（ドメイン名）
ping -c 3 google.co.jp
nslookup google.co.jp
```

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-02.png)

最初は「IP直打ちで通るか」を確認するだけでも、DNS問題をかなり見分けやすくなります。

Wi‑Fiへ接続できていても、IP取得できていなければ通信はできません。

## ステップ5: DHCPとIPアドレスの確認

```sh
# 現在のDHCPリース一覧
cat /tmp/dhcp.leases

# dnsmasqのログ（DHCP配布の状況）
logread | grep -i dnsmasq | tail -n 30
```

端末側での確認:
- **Windows**: `ipconfig` で IPアドレスが `169.254.x.x` の場合はDHCP取得失敗
- **Mac**: システム設定 → ネットワーク → IPアドレスを確認
- **スマートフォン**: Wi-Fi設定 → 接続済みSSIDの詳細

`cat /tmp/dhcp.leases` は、「その端末が本当にIP取得できているか」を確認する時にかなり便利です。

DHCPで問題がある場合は、次を確認します:
```sh
# DHCPサーバーの設定を確認
uci show dhcp.lan

# DHCPサーバーの再起動
/etc/init.d/dnsmasq restart
```

最初は「dnsmasqが動いているか」を見るだけでもかなり役立ちます。

DNS問題は、「インターネット全部が落ちた」と誤解しやすいポイントです。

## ステップ6: DNS設定の確認

IPアドレスへの疎通はできるが、ドメイン名でアクセスできない場合はDNSの問題です。

```sh
# ルーター自体でDNS解決テスト
nslookup google.co.jp 127.0.0.1
nslookup google.co.jp 8.8.8.8  # GoogleのDNSで直接テスト

# DNS設定の確認
uci show dhcp | grep -E 'server|dns'

# adblock（広告ブロック）が有効な場合
# adblockサービスのステータス確認
/etc/init.d/adblock status
```

最初は「IPアドレスへpingできるか」と「名前解決だけ失敗しているか」を分けることが大事です。

adblockが特定サイトをブロックしている場合は、adblockホワイトリストへ追加します（Article 008参照）。

最近Guest Wi‑FiやVLAN、VPN設定を変更した場合は、firewall設定も確認します。

## ステップ7: ファイアウォールの確認

```sh
# firewall設定の確認
uci show firewall

# 実際に適用されているiptablesルール
iptables -L -n -v | head -n 50

# ファイアウォールのドロップログ
logread | grep -i 'DROP\|REJECT' | tail -n 30
```

誤って必要な通信をブロックしている場合は、バックアップから設定を復元します:

```sh
cp /etc/config/firewall.backup.YYYYMMDD /etc/config/firewall
/etc/init.d/firewall restart
```

最初は「DROP」「REJECT」が大量に出ていないかを見るくらいでも十分役立ちます。

最初は、「最後に変更した設定」を疑うほうが切り分けしやすくなります。

トラブル時は、「最近何を変えたか」を思い出すだけでもかなり役立ちます。

## 変更直後に戻すための安全パターン

設定変更後におかしくなった場合は、やみくもに再起動するより、最後に触った範囲だけ戻すほうが安全です。

### 変更前バックアップを作る

大きめの変更前は、LuCIのバックアップに加えて、SSHでも設定ファイルを控えておきます。

```sh
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ls -l "$BACKUP_DIR"
```

### DNS/DHCPだけ戻す

IPアドレスは取れているが名前解決だけおかしい場合は、まずdnsmasqだけ再起動します。

```sh
/etc/init.d/dnsmasq restart
logread | grep -i dnsmasq | tail -n 50
nslookup google.co.jp 127.0.0.1
```

### Wi-Fiだけ反映し直す

有線は正常でWi-Fiだけおかしい場合は、ネットワーク全体ではなくWi-Fiだけを反映し直します。

```sh
wifi status
wifi reload
sleep 5
wifi status
```

`wifi reload` 中は無線端末が一時的に切断されます。有線でLuCIやSSHへ入れる状態で実行するほうが安全です。

### firewallだけ戻す

Guest Wi-Fi、VLAN、ポート開放、VPN設定のあとに通信できなくなった場合は、firewallだけ戻します。

```sh
cp /etc/config/firewall.backup.YYYYMMDD-HHMM /etc/config/firewall
/etc/init.d/firewall restart
uci show firewall
command -v iptables >/dev/null && iptables -L -n -v | head -n 80
```

### network設定を戻す

LAN IPやWAN設定を触った場合は、network設定を戻します。これはLuCI/SSHが切れる可能性があるため、有線LAN接続で実行します。

```sh
cp /etc/config/network.backup.YYYYMMDD-HHMM /etc/config/network
/etc/init.d/network restart
```

`/etc/init.d/network restart` 後は、PC側のIP再取得が必要になることがあります。LuCIへ入れない場合は、PCの有線LANを抜き差しするか、PC側でDHCPを更新します。

## 症状別クイック対処表

いったん次の3つが分かれば、切り分けはかなり進みます。

- 全端末に起きているのか、一部端末だけなのか
- IPアドレスは取れているのか
- IP直打ちでは通るのか、ドメイン名だけ失敗するのか

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-03.png)

LuCIだけでも、かなり多くの切り分けができます。

最初は「Overview」「DHCP Leases」「System Log」を見るだけでも十分役立ちます。

## LuCIの主要な状態確認画面

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/table-04.png)

## まとめ

接続トラブルは、次の順番で切り分けると整理しやすくなります:

1. LED・ケーブル・ONU/HGWの物理状態を確認する
2. 有線LAN接続でLuCIへ入れるか確認する
3. `ifstatus wan` / `ifstatus wan6` でWAN状態を確認する
4. `ping` と `nslookup` でDNS問題かを切り分ける
5. `cat /tmp/dhcp.leases` でDHCP状態を確認する
6. adblock・firewall・VLANなど最近変更した設定を確認する
7. 必要ならバックアップから復元する

「全部壊れている」と考えるより、「どこまでは正常か」を分けるほうが、かなり切り分けしやすくなります。

特にIPoE環境では、「IPv4だけ落ちている」「WAN6だけ不安定」など、片側だけの問題もあります。

最初は「物理」「WAN」「DNS」のどこで止まっているかを分けられるだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/021/diagram-03.png)

## よくある質問

### LN6001-JPがつながらない時は何から確認すればいい？

まずはLED、ケーブル、ONU/HGWの物理状態です。

そのあと有線でLuCIへ入れるかを見ると切り分けしやすくなります。

### Wi-Fiにはつながるのにネットが使えない時は何を見る？

`ifstatus wan`、`ping`、`nslookup` を使って、WANなのかDNSなのかを分けて見るのが近道です。

最初は「IP直打ちで通るか」を見るだけでもかなり役立ちます。

### 設定変更後におかしくなった時はどう戻せばいい？

最近変えた設定を先に疑い、adblock、firewall、VLANなどを順に確認します。

必要ならバックアップから戻すほうが早いこともあります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 light behavior: https://support.linksys.com/kb/article/216-en/?section_id=175
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/

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
