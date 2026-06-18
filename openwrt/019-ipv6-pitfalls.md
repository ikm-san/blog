<!-- mirror-source: articles/019-ipv6-pitfalls.md -->

> Original note.com article: [IPv6の落とし穴: 家庭用ルーター感覚で見落としやすい点【OpenWrt集中連載019】](https://note.com/ikmsan/n/n61235a13b478)

# IPv6の落とし穴: 家庭用ルーター感覚で見落としやすい点【OpenWrt集中連載019】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

IPv6では、IPv4のNAT前提とはネットワークの見え方がかなり変わります。

特にIPoEやIPv4 over IPv6環境では、「IPv4は生きているがIPv6だけ不安定」「WAN6だけ落ちている」といった状態も起きます。

ただし、最初からIPv6を全部深く理解しようとしなくても大丈夫です。

まずは「IPv4とIPv6を分けて確認する」「WAN6を見る」「firewallをIPv6側まで確認する」だけでも、切り分けはかなりしやすくなります。

LN6001-JPでIPv6/IPoEを使う際に見落としやすいポイントと、まず覚えておきたい確認コマンドを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-01.png)

## この記事でわかること

- IPv4とIPv6を分けて確認する理由
- IPoE環境で見落としやすいIPv6側の確認ポイント
- prefix delegation や firewall をどう見ればいいか
- WAN6トラブル時の基本的な切り分け順序

## こんな人に向いています

- IPv4はつながるのに一部通信だけ不安定
- IPoE環境でIPv6側の見方が分からない
- WAN6やprefix delegationをどこまで見ればいいか迷っている

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-02.png)

## まずはここまでで十分

IPv6も、最初から全部を深く理解しなくても大丈夫です。

まずは次の3つだけ見れば十分です。

1. `ifstatus wan6` でIPv6アドレスを確認する
2. `ping6` でIPv6疎通を見る
3. `ip6tables -L -n -v` または `nft list ruleset` でIPv6 firewallを見る

IPv4とIPv6を分けて考えるだけでも、切り分けはかなりしやすくなります。

最初は「IPv6だけ生きているのか」「IPv4だけ落ちているのか」を分けられるだけでも十分です。

IPoE環境では、「インターネット」という1つの塊ではなく、「IPv4側」と「IPv6側」が別々に動いているイメージで見ると整理しやすくなります。

## IPv6とIPv4を分けて確認する

IPoE環境では、IPv4とIPv6は別々の経路で動いています。

「インターネットにつながらない」という症状も、IPv4側の問題かIPv6側の問題かで原因が変わります。

| 確認対象 | コマンド | 見るべき内容 |
|---|---|---|
| WAN（IPv4） | `ifstatus wan` | ipaddr（IPアドレスを取得しているか） |
| WAN6（IPv6） | `ifstatus wan6` | ip6addr（IPv6アドレスを取得しているか） |
| ルーティング | `ip route show` / `ip -6 route show` | デフォルトゲートウェイが存在するか |
| DNS | `nslookup www.google.co.jp` | 名前解決できるか |

最初は `ifstatus wan` と `ifstatus wan6` を分けて見るだけでもかなり意味があります。

特にIPv4 over IPv6環境では、「WAN6は正常だがIPv4 over IPv6側だけ落ちている」ケースもあります。

### 確認コマンドの実行例

```sh
# WAN（IPv4 over IPv6の場合はIPv4が割り当てられているか確認）
ifstatus wan | grep -E '"up"|"ipaddr"'

# WAN6（IPv6アドレスとprefix delegation確認）
ifstatus wan6 | grep -E '"up"|"ip6addr"|"ip6prefix"'

# IPv6アドレス一覧
ip -6 addr show

# IPv6ルーティングテーブル
ip -6 route show

# IPv6の疎通確認
ping6 -c 3 2001:4860:4860::8888

# IPv4の疎通確認
ping -c 3 8.8.8.8
```

`ifstatus wan6` と `ip -6 route show` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「今どうつながっているか」を確認する用途で使うくらいでも十分です。

IPv6トラブルでは、「IPv4感覚のまま見てしまう」ことが原因になるケースもかなり多くあります。

### 設定変更前に状態を保存する

IPv6/IPoEの切り分けでは、変更する前の状態が重要です。

まずは読み取り中心で確認し、設定変更に入る前だけバックアップを取ります。

```sh
BACKUP_DIR="/root/ipv6-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
cp /etc/config/dhcp "$BACKUP_DIR/dhcp"

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip route show > "$BACKUP_DIR/ip-route-v4.txt"
ip -6 route show > "$BACKUP_DIR/ip-route-v6.txt"
ip -6 addr show > "$BACKUP_DIR/ip-addr-v6.txt"
logread | grep -i 'ipv6\|dhcpv6\|odhcp6c\|radvd\|ip6' | tail -n 120 > "$BACKUP_DIR/ipv6-log.txt"

echo "backup: $BACKUP_DIR"
```

IPoEやIPv4 over IPv6の不具合は、設定変更後に元の状態が分からなくなると切り分けが難しくなります。

「変更する前に保存する」だけで、戻しやすさがかなり変わります。

### 落とし穴1: IPv4のみ確認してIPv6側の問題を見落とす

症状: 一部のサイトが遅い、特定のアプリが不安定、IPv4では表示されるがIPv6では見えない

確認方法:
```sh
# IPv4とIPv6を分けて疎通確認
ping -c 3 8.8.8.8           # IPv4 OK?
ping6 -c 3 2001:4860:4860::8888  # IPv6 OK?
nslookup www.google.co.jp   # DNS OK?

# IPv6側のDNS確認
nslookup -type=AAAA www.google.co.jp

# 実際にIPv6経路を使っているか確認
traceroute6 www.google.co.jp 2>/dev/null || true
```

最初は「IPv4だけ正常なのか」「IPv6だけ正常なのか」を分けるだけでも、かなり切り分けしやすくなります。

### 落とし穴2: IPv6でNATがないためfirewallが重要になる

IPv4では、NATで内部端末が外から見えにくい構造があります。

IPv6では端末へグローバルIPv6アドレスが割り当てられる場合があるため、firewall確認がかなり重要になります。

確認:
```sh
# firewallのIPv6設定を確認
uci show firewall | grep -E 'wan|zone'

# ip6tables互換ルール確認
ip6tables -L -n -v 2>/dev/null || true

# nftables側で見える環境ではこちらも確認
nft list ruleset 2>/dev/null | grep -E 'ip6|icmpv6|reject|drop|wan' | head -n 100
```

最初は「WAN側InputがREJECTになっているか」を確認するだけでも十分です。

LN6001-JPのOpenWrtは標準でip6tablesルールを持っています。

WANゾーンからの入力はREJECTが基本です。

### 落とし穴3: Prefix Delegationが取得できないとLAN側のIPv6が配布されない

IPoE環境では、ルーターがプロバイダから「/56」や「/48」などのIPv6プレフィックス（アドレスブロック）を取得し、LAN端末へ配布します（DHCPv6-PD）。

```sh
# プレフィックス委任（PD）の状態確認
ifstatus wan6 | grep -i 'prefix\|ip6prefix'

# odhcp6cのログ確認
logread | grep -i 'odhcp6c\|dhcpv6\|prefix' | tail -n 80

# LAN側のIPv6設定確認
uci show network.lan6 2>/dev/null || uci show network | grep -i 'ip6'

# LAN端末へのIPv6アドレス配布状況
ip -6 addr show br-lan
```

WAN6がupでも、prefix delegationが崩れているとLAN側だけIPv6不安定になることがあります。

プレフィックスが取得されていない場合は、まずWAN6状態を確認します。

### 落とし穴4: IPoEとIPv4 over IPv6の言葉を混同する

| 用語 | 意味 |
|---|---|
| IPoE | IPv6を使ってインターネットへ接続する方式 |
| IPv4 over IPv6 | IPv6網の上でIPv4通信を扱う仕組み（MAP-E/DS-Liteなど） |
| OCNバーチャルコネクト | NTTコム系のIPv4 over IPv6サービス（MAP-E方式） |
| transix | インターネットマルチフィードのIPv4 over IPv6サービス（DS-Lite方式） |

最初は、「IPoE = IPv6側の接続方式」「IPv4 over IPv6 = IPv4も通す仕組み」くらいで整理すると分かりやすくなります。

IPoEとIPv4 over IPv6は別の話です。

IPoEだけではIPv4サイトへアクセスできない場合があり、IPv4 over IPv6設定が別途必要になります。

IPv6トラブルでは、「WAN6」「DHCPv6」「DNS」「firewall」を順番に見ると切り分けしやすくなります。

公式IPoEモジュールやWAN設定を触る前に、まず現在の回線方式を確認します。

```sh
# 現在のWAN/WAN6プロトコルを確認
uci show network.wan
uci show network.wan6 2>/dev/null || true

# Linksys公式IPoEモジュール導入済みか確認する目安
opkg list-installed | grep -Ei 'ipoe|map|dslite|ipip' || true
```

Linksys公式サポートでは、多くの回線はIPv4/IPv6ともDHCP自動で利用でき、PPPoEや固定IP/IPIPなどは方式に応じて設定します。

IPv6だけを疑う前に、「自分の回線がDHCP自動でよいのか」「PPPoEや固定IP/IPIPの設定が必要なのか」を整理すると、不要な変更を減らせます。

## IPv6関連のログ確認

```sh
# IPv6関連のログを確認
logread | grep -i 'ipv6\|dhcpv6\|radvd\|ip6' | tail -n 50

# DNS関連のログ（dnsmasq）
logread | grep -i dnsmasq | tail -n 30

# ファイアウォールのIPv6ログ
logread | grep -i 'DROP\|REJECT' | grep -i 'ipv6\|ip6' | tail -n 30
```

最初は「error」「fail」「reject」が大量に出ていないかを見るくらいでも十分役立ちます。

IPv6問題では、「WAN6」「prefix delegation」「DNS」のどこで止まっているかを分けて考えると整理しやすくなります。

## IPv6のよくある症状と対処

| 症状 | 確認ポイント | 対処 |
|---|---|---|
| IPv6アドレスが取れない | `ifstatus wan6` でupになっているか | IPoE設定を確認、公式IPoEモジュールを再確認 |
| LAN端末にIPv6が配布されない | prefix delegationが取れているか | `uci show network`でipv6設定確認 |
| 一部サイトだけ遅い | IPv6/IPv4の経路を分けて確認 | IPv6を使っているサービスかどうか確認 |
| DNSでAAAAレコードが引けない | DNS設定 | `uci show dhcp`でDNSサーバー設定確認 |
| VPN経由でIPv6が使えない | VPNのIP設定 | WireGuardのAllowedIPsにIPv6範囲を追加 |

最初は、「WAN6が正常か」「LAN側へIPv6が配られているか」を分けて確認するだけでもかなり役立ちます。

## 買う前チェックリスト（IPoE環境用）

LN6001-JPをIPoE環境で使う前に、次を確認しておくと切り分けしやすくなります:

- プロバイダのIPv4 over IPv6方式（MAP-E/DS-Lite/IPIP）
- ひかり電話ホームゲートウェイの有無
- HGWがある場合: LN6001-JPとHGWのLAN IPアドレス重複確認
- 既存ルーターのLAN側IPアドレス
- LinksysオートIPoEモジュールの対応方式確認（https://support.linksys.com/kb/article/6902-jp/）

## まとめ

IPv6/IPoE環境では、次の順番で確認すると整理しやすくなります:

1. `ifstatus wan` と `ifstatus wan6` を分けて確認する
2. IPv4とIPv6の疎通テストを分けて実行する（`ping` / `ping6`）
3. `ip6tables -L -n -v` または `nft list ruleset` でIPv6 firewallを確認する
4. Prefix Delegation取得状態を確認する
5. WAN6 → DNS → firewall → LAN端末側の順に切り分ける

IPv6を特別に怖いものとして扱う必要はありません。

「IPv4とIPv6で見る場所が違う」「firewallもIPv6側まで確認する」という2点を押さえるだけでも、トラブル時の見通しはかなり改善します。

最初は「WAN6が正常か」「IPv6疎通できるか」を確認できれば十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/019/diagram-03.png)

## よくある質問

### IPv6だけつながらない時は何を見ればいい？

`ifstatus wan6`、`ping6`、`ip6tables -L -n -v` の3つを分けて見ると切り分けしやすくなります。

最初は「WAN6がupか」「IPv6 pingが通るか」だけでもかなり役立ちます。

### IPv4がつながっていればIPv6は気にしなくていい？

IPoE環境ではIPv4とIPv6が別経路で動くため、片方だけ不調ということがあります。

両方を分けて確認することが大事です。

### prefix delegation は何のために見る？

LAN側端末へIPv6プレフィックスが正しく配られているかを見るためです。

ここが崩れると、WAN6はつながっていても端末側が不安定になることがあります。

## 参考リンク

- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/

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
