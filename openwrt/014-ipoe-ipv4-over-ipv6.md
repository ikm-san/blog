<!-- mirror-source: articles/014-ipoe-ipv4-over-ipv6.md -->

> Original note.com article: [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか【OpenWrt集中連載014】](https://note.com/ikmsan/n/n97ddc6c12ca8)

# NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか【OpenWrt集中連載014】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPを日本の光回線で使う場合、IPoE / IPv4 over IPv6 の設定が必要になるケースがかなり多くあります。

ただし、最初から MAP-E、DS-Lite、IPIP の違いを全部理解しようとしなくても大丈夫です。

まずは「HGW配下なのか」「ONU直下なのか」を見分けるだけでも、設定方針はかなり整理しやすくなります。

LN6001-JPにはLinksys公式の「オートIPoEモジュール」があり、OCNバーチャルコネクト・transix・v6プラス・クロスパスなど主要方式へ対応しています。

もちろん全部手動設定も可能ですが、特にMAP-EやIPIPは設定項目が多いため、最初は公式モジュールを使ったほうが安全です。

この記事では、日本の回線でよくある接続パターン、HGW配下とONU直下の違い、Linksys公式IPoEモジュールを使った設定の流れを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-01.png)

## この記事でわかること

- 自分の回線がHGW配下かONU直下かの見分け方
- LN6001-JPでIPoEとIPv4 over IPv6をどう設定するか
- OCNバーチャルコネクトやtransixで確認したいポイント
- PPPoEとIPoEをどう使い分けるべきか

## こんな人に向いています

- ONU直下かHGW配下かで設定方針が分からない
- IPoE、IPv4 over IPv6、PPPoEの違いで混乱している
- 日本の光回線でLN6001-JPを自然に動かしたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-02.png)

## まずはここまでで十分

IPoE設定も、最初から全部を深く理解しなくても大丈夫です。

まずは次の3つを押さえれば十分です。

1. 自分の回線がHGW配下かONU直下かを見分ける
2. HGW配下ならDHCP、自前担当なら公式IPoEモジュールを使う
3. 接続後に `ifstatus wan` と `ifstatus wan6` を確認する

回線方式の見極めができるだけでも、設定の迷いはかなり減ります。

最初は「IPv4とIPv6の両方でインターネットへ出られる」ことを確認できれば十分です。

最初にSSHで状態を見るなら、次の読み取り専用コマンドから始めます。

```sh
echo "### system"
date
ubus call system board

echo "### wan status"
ifstatus wan
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show
```

この段階では設定を変えません。HGW配下でDHCP自動接続しているだけなのか、LN6001-JP自身がIPoE/IPv4 over IPv6を担当しているのかを切り分ける入口にします。

## 用語ミニ解説

- IPoE: IPv6を使ってインターネットへ接続する方式。PPPoEより混雑しにくいことが多い
- PPPoE: プロバイダのID・パスワードを使ってインターネットへ接続する従来の方式
- IPv4 over IPv6: IPv6/IPoE回線上でIPv4通信も使えるようにする仕組み（OCNバーチャルコネクト、transix、v6プラスなど）
- VNE: IPv4 over IPv6の変換サービスを提供する事業者
- HGW（ホームゲートウェイ）: ひかり電話接続のために設置するNTTのルーター機器
- ONU: 光回線をLANケーブルで使える形に変換する機器（ルーター機能なし）

最初は「IPoE = IPv6側の接続方式」「IPv4 over IPv6 = IPv4も通す仕組み」くらいで整理すると分かりやすいです。

## 自分の回線パターンを確認する

まず、自分の回線がどのパターンかを確認します。

ここを先に整理しておくと、「なぜつながらないのか」をかなり切り分けしやすくなります。

![IPoE設定の判断フロー](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/ipoe-decision-flow.png)

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-01.png)

最初に迷いやすいのは、「HGWがすでにIPoEを担当しているのか」「LN6001-JP側でIPoE設定が必要なのか」の違いです。

ひかり電話契約がある場合は、HGW側がすでにIPoEを担当しているケースもかなり多くあります。

SSHで見る場合は、WAN側のIPアドレスとIPv6側の状態を確認します。

```sh
echo "### wan json"
ifstatus wan
ifstatus wan6

echo "### interface addresses"
ip addr show

echo "### default routes"
ip route show default
ip -6 route show default
```

WAN側に `192.168.x.x` や `10.x.x.x` のようなプライベートIPv4アドレスが見えている場合、HGWや親ルーター配下で動いている可能性があります。この場合、LN6001-JP側でIPv4 over IPv6の詳細設定を作るより、まず上流機器とのIPアドレス重複を避けることが重要です。

一方、ONU直下でIPv6は来ているのにIPv4がうまく出ない場合は、VNE方式に合わせて公式モジュールを使う判断になります。

## パターンA・B: DHCP自動で接続する場合

HGW（ホームゲートウェイ）がある場合や、既存ルーター配下へ置く場合は、LN6001-JPのWAN設定をDHCP自動にするだけで接続できることが多いです。

このケースでは、「LN6001-JP自身がIPoEを処理しているわけではない」点がポイントです。

### 設定確認手順

1. **Network** → **Interfaces** → `wan` の **Edit**
2. Protocol: **DHCP client** になっていることを確認
3. **Save & Apply**
4. **Status** → **Overview** でWANのIPアドレスが取得されていることを確認

### LANのIPアドレス重複を回避する

HGWとLN6001-JPが両方 `192.168.1.1` を使っている場合、通信が不安定になったり、管理画面へ戻れなくなることがあります。

LN6001-JP側のLAN IPアドレスを変更します:

1. **Network** → **Interfaces** → `lan` の **Edit**
2. IPv4 address: `192.168.2.1`（または別のセグメントに変更）
3. **Save & Apply**

```sh
echo "### current LAN address"
uci get network.lan.ipaddr

echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)
ls -l /etc/config/network.backup.*

echo "### change LAN address"
uci set network.lan.ipaddr='192.168.2.1'
uci changes network
uci commit network
/etc/init.d/network restart
```

最初は `192.168.2.1` のように、大きく別セグメントへ逃がすほうが分かりやすいです。

変更後はブラウザで `https://192.168.2.1` にアクセスして管理画面に戻ります。

## パターンC: ONU直下でLinksys公式IPoEモジュールを使う

ひかり電話HGWがなく、LN6001-JPをONU直下へ置く場合、IPv4 over IPv6設定をLN6001-JP側で行います。

このケースでは、LN6001-JP自身がMAP-EやDS-Liteを処理します。

### 対応VNE・方式

Linksys公式（https://support.linksys.com/kb/article/6902-jp/）に記載のある対応方式:

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-02.png)

最初は「自分のプロバイダがどの方式を使っているか」だけ分かれば十分です。

MAP-E、DS-Lite、IPIPの内部動作まで、最初から全部覚える必要はありません。

自分のプロバイダがどのVNE・方式を使っているかは、プロバイダの契約情報やFAQページで確認します。

「IPv6オプション」「IPoEサービス名」で記載されていることもあります。

### オートIPoEモジュールの導入手順

1. Linksys公式サポートページ（https://support.linksys.com/kb/article/6902-jp/）からオートIPoEモジュール（`velop-auto-ipoe_1.x_all.ipk`）をダウンロード
2. **System** → **Software** を開き、まず **Update lists...** をクリックしてパッケージリストを更新
3. 更新結果を確認して **Dismiss** で閉じる
4. **Upload Package** → **Browse...** でダウンロード済みの `.ipk` ファイルを選び、**Upload** をクリック
5. 確認画面で **Install** をクリック
6. `installed in root is up to date.` と表示されたら **Dismiss** で閉じる
7. **Services** → **Internet Assistant** を開く（表示されない場合はブラウザをリロード）
8. **オートIPoEを実行** をクリックし、回線の自動判定と設定完了を待つ
9. サービス開始ダイアログ後にルーターが再起動したら、数分待って **Status** → **Overview** でIPv4/IPv6アドレスを確認

最初はCLIでMAP-EやDS-Liteを手動構築するより、公式モジュールで動く状態を先に作るほうが、切り分けしやすくなります。

### 作業前にバックアップを取る

IPoE設定は `network`、`firewall`、`dhcp` などに影響します。モジュールを実行する前に、現在の設定を保存しておきます。

```sh
BACKUP_DIR="/root/ipoe-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json"
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"

ls -l "$BACKUP_DIR"
```

このバックアップは、設定を戻すためだけでなく、サポートへ相談するときの状況整理にも役立ちます。

### CLIで接続状態を確認する

```sh
echo "### IPv4/IPv6 interface status"
ifstatus wan
ifstatus wan6

echo "### addresses and routing"
ip addr show
ip route show
ip -6 addr show
ip -6 route show

echo "### installed IPoE-related packages"
opkg list-installed | grep -Ei 'map|dslite|ipoe|ipip|mape|464xlat' || true

echo "### recent IPoE-related logs"
logread | grep -Ei 'ipoe|map-e|mape|dslite|ipip|wan6' | tail -n 80
```

`ifstatus wan` と `ifstatus wan6` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「今どうつながっているか」を確認する用途で使うくらいで十分です。

方式ごとに見るポイントは少し違います。

```sh
echo "### MAP-E-like interfaces"
ip link show | grep -Ei 'map|mape|v4|tun' || true

echo "### DS-Lite-like interfaces"
ip link show | grep -Ei 'dslite|ds|tun' || true

echo "### IPIP-like interfaces"
ip link show | grep -Ei 'ipip|tun' || true

echo "### firewall masq hints"
uci show firewall | grep -E 'zone|network|masq|mtu_fix'
```

名前は環境やモジュールのバージョンで変わることがあります。ここでは「この名前でなければ失敗」と決め打ちせず、WAN/WAN6、経路、ログをまとめて確認します。

## パターンD: PPPoE接続

固定IP契約や、特殊な業務要件でPPPoEが必要な場合の設定です。

現在でも、固定IPサービスや一部業務用途ではPPPoEを使うケースがあります。

### LuCIでPPPoEを設定する

1. **Network** → **Interfaces** → `wan` の **Edit**
2. Protocol: **PPPoE** を選択
3. **General Settings** タブ:
   - PAP/CHAP username: プロバイダから提供されたID
   - PAP/CHAP password: プロバイダから提供されたパスワード
4. **Save & Apply**

```sh
echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)

echo "### current WAN config"
uci show network.wan

echo "### switch WAN to PPPoE"
uci set network.wan.proto='pppoe'
uci set network.wan.username='接続ID@プロバイダドメイン'
uci set network.wan.password='接続パスワード'
uci changes network
uci commit network
/etc/init.d/network restart

echo "### verify"
ifstatus wan
logread | grep -Ei 'ppp|pppoe|wan' | tail -n 50
```

最初は「IPoEが使えるか」を確認し、それでも必要な場合だけPPPoEを追加するほうが整理しやすいです。

## IPoEとPPPoEを併用する

メイン接続をIPoEにしながら、特定用途（固定IPが必要な業務システムなど）だけPPPoEを使いたい場合があります。

小さなオフィスでは、「通常通信はIPoE」「特定業務だけPPPoE」という構成もあります。

![IPoEとPPPoEの併用イメージ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/ipoe-pppoe-coexistence.png)

ただし、併用構成は回線契約、VNE方式、PPPoE接続情報、WAN側インターフェース名、経路制御の設計によって変わります。ここではコピペ用の追加インターフェース作成コマンドは扱わず、まず現在の状態確認に留めます。

```sh
# WAN/WAN6の状態を確認
ifstatus wan
ifstatus wan6

# 実際のインターフェース名を確認
ip link show

# 既存のPPPoE設定があるか確認
uci show network | grep -i ppp
```

PPPoEを併用する場合は、LuCIの **Network** → **Interfaces** で実際のWANデバイス名を確認し、プロバイダの接続条件に合わせて設定します。Internet Assistantの設定と衝突しないよう、変更前に必ずバックアップを取ります。

最初は「今どのインターフェースが動いているか」を整理するだけでも十分役立ちます。

## 接続後の確認

```sh
echo "### IPv4 reachability"
ping -c 3 8.8.8.8

echo "### IPv6 reachability"
ping6 -c 3 2001:4860:4860::8888

echo "### DNS"
nslookup www.google.co.jp
nslookup ipv6.google.com

echo "### final route check"
ip route show default
ip -6 route show default
```

最初は「IPv4サイトもIPv6サイトも普通に見られるか」を確認できれば十分です。

速度最適化や高度なチューニングは、そのあとでも遅くありません。

## 設定前のバックアップ

```sh
BACKUP_DIR="/root/network-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json"
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json"

ls -l "$BACKUP_DIR"
```

IPoE設定は network / firewall / routing に影響するため、変更前バックアップを残しておくと安心です。

## よくある問題と対処

IPoE設定で困る時は、「HGW配下なのか」「ONU直下なのか」の認識違いが原因になっていることがかなり多いです。

### IPv4アドレスが取得できない（MAP-E/DS-Lite環境）

- **確認:** `ifstatus wan6` でIPv6アドレスが取得されているか確認
- **対処:** IPv6が取得されていない場合はHGW設定またはプロバイダのIPv6対応を確認する

```sh
ifstatus wan6
ip -6 addr show
ip -6 route show
logread | grep -Ei 'wan6|dhcpv6|odhcp6c|ipoe|map|dslite|ipip' | tail -n 80
```

### LAN内端末はインターネットに接続できるがルーターにpingが届かない

- **原因:** ルーターのWAN側IPがプライベートアドレス（HGW配下のため）
- **対処:** 通常動作。HGW配下ではルーターのWAN側はプライベートアドレスになる

### PPPoEの接続IDが分からない

**対処:** プロバイダのマイページ・契約書・開通通知書で確認する。問い合わせ時は「PPPoE接続情報を確認したい」と伝える

### 設定変更後に管理画面へ戻れない

- **確認:** LAN側IPアドレスを変更していないか、PC側が新しいセグメントのIPを取得しているか確認
- **対処:** 有線LANで接続し直し、PCのIPアドレスを取り直してから `https://192.168.2.1` など変更後のアドレスへアクセスする

```sh
uci get network.lan.ipaddr
cat /tmp/dhcp.leases
logread | grep -Ei 'netifd|dnsmasq|dhcp' | tail -n 50
```

## 買う前確認チェックリスト

IPoE設定は、購入前に回線情報を整理しておくだけでもかなり進めやすくなります。

- プロバイダと回線事業者（NTT東/西など）
- ひかり電話契約の有無
- ホームゲートウェイの有無
- IPv4 over IPv6方式（OCNバーチャルコネクト/transix/v6プラスなど）
- 固定IP契約の有無
- 既存ルーターのLAN側IPアドレス

## まとめ

最初の成功条件としては、次の3つが確認できれば十分です。最初はここまで確認できれば大きく外していません。

- WAN と WAN6 の両方で想定どおりの状態が見える
- HGW との LAN IP 重複が起きていない
- ふだん使う端末からインターネットへ出られる

LN6001-JPのIPoE設定は:

1. 自分の回線パターン（HGW有無・ONU直下かどうか）を確認する
2. HGW配下ならDHCP自動で接続できることが多い。LAN IP重複に注意する
3. ONU直下ならLinksys公式IPoEモジュールを使う（https://support.linksys.com/kb/article/6902-jp/）
4. PPPoEが必要な場合は、WAN設定でID/パスワードを入力する
5. 接続後は `ifstatus wan` / `ifstatus wan6` で状態確認する

最初からMAP-EやDS-Liteを手動設定するより、公式IPoEモジュールでまず接続確認するほうが、家庭や小さなオフィスでは進めやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-03.png)

## よくある質問

### LN6001-JPはIPoEに対応している？

はい。Linksys公式のオートIPoEモジュールが案内されており、OCNバーチャルコネクト、transix、v6プラス、クロスパスなど主要方式へ対応しています。

### HGWがある場合もIPoE設定は必要？

HGWがすでにIPoEを担当しているなら、LN6001-JP側はWANをDHCP自動にするだけでつながることが多いです。

その代わり、LAN IP重複には注意が必要です。

### ONU直下ならPPPoEではなくIPoEにしたほうがいい？

契約している回線とプロバイダーがIPoE対応なら、まずはIPoEを優先して確認するのが自然です。

ただし、固定IP契約や特殊な要件がある場合はPPPoEが必要になることもあります。

## 参考リンク

- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/

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
