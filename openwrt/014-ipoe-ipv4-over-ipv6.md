<!-- mirror-source: articles/014-ipoe-ipv4-over-ipv6.md -->

# IPv6はつながるのにIPv4が開かない？｜LN6001-JPでIPoE回線をまず見分ける【OpenWrt集中連載014】

日本の光回線でルーターを設定する時、いちばん分かりにくいのがIPoEまわりです。

IPv6はつながる。  
でもIPv4サイトが開かない。  
OCNバーチャルコネクトとtransixの違いが分からない。  
v6プラス、クロスパス、DS-Lite、MAP-E、IPIP。  
名前だけでもうお腹いっぱい。

さらに、家の中にはONU、ホームゲートウェイ、既存ルーターがいて、

```txt
結局、誰がインターネット接続を担当しているの？
```

となりがちです。

ここで焦ってMAP-EやDS-Liteを手動で組み始めると、だいたい沼ります。

なので、最初にやることは設定ではありません。

まず、自分の回線構成を見分けます。

```txt
ひかり電話HGWがあるのか
ONU直下なのか
LN6001-JPをメインルーターにするのか
HGW配下に置くのか
プロバイダの方式は何なのか
```

ここが分かると、かなり楽になります。

LN6001-JPには、Linksys公式の **オートIPoE** モジュールがあります。

NTTフレッツ系のONU直下構成では、このモジュールを使うのが基本です。  
一方で、ひかり電話ホームゲートウェイがある場合は、HGW側がIPoEを担当していて、LN6001-JPはDHCP自動で十分なこともあります。

この記事では、LN6001-JPで日本の光回線を使う時に、HGW配下、ONU直下、PPPoE、固定IP・IPIPをどう見分けるかを整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、MAP-EやDS-Liteを手動で書けるようになることではありません。

まずは、ここまで分かればOKです。

- HGW配下とONU直下で設定方針が変わると分かる
- LN6001-JPでオートIPoEモジュールを使う場面が分かる
- OCNバーチャルコネクト、transix、v6プラス、クロスパスなどを「方式名」として見られる
- PPPoEとIPoEの違いをざっくり理解できる
- `ifstatus wan` と `ifstatus wan6` で状態確認できる
- IPv4、IPv6、DNSを分けて切り分けられる
- 設定前にバックアップを取る理由が分かる

IPoEまわりは、細かい方式名より先に、

```txt
いま誰が回線接続を担当しているのか
```

を見るのが大事です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-01.png)

## 先にざっくり結論

最初に見るべきなのは、MAP-EやDS-Liteの細かい違いではありません。

まずこの3つです。

1. **ひかり電話ホームゲートウェイがあるか**
2. **LN6001-JPをHGW配下に置くのか、ONU直下に置くのか**
3. **契約しているIPv4 over IPv6方式が何か**

判断はざっくりこうです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-01.png)

最初の確認コマンドは、この2つです。

```sh
ifstatus wan
ifstatus wan6
```

これで、IPv4側とIPv6側を分けて見られます。

IPoE設定は、焦って手動で作り込むより、まず「自分の回線構成を正しく見分ける」ほうが大事です。

## こういう人向けです

この記事は、次のような人向けです。

- LN6001-JPを日本の光回線で使いたい
- OCNバーチャルコネクト、transix、v6プラス、クロスパスで混乱している
- HGW配下とONU直下の違いが分からない
- IPv6はつながるのにIPv4サイトが開けず困っている
- PPPoEのIDとパスワードはあるが、IPoEとどう使い分けるか分からない
- 小さなオフィスで固定IPやPPPoE併用を検討している
- CLIでWAN/WAN6の状態を安全に読み取りたい
- VPNやポート開放の前に、回線方式を整理したい

逆に、すでにMAP-E、DS-Lite、IPIP、DHCPv6-PD、RA、policy routingまで分かっている人には基本寄りです。

この記事では、まず家庭・小さなオフィスで迷子にならないことを優先します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-02.png)

## 最初に言葉だけそろえる

IPoEまわりの用語は、名前が似ていてややこしいです。

ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-02.png)

最初は、次の理解で十分です。

```txt
IPoE = IPv6側の接続方式
IPv4 over IPv6 = IPv6の上でIPv4も使う仕組み
PPPoE = ID/パスワードで接続する方式
```

MAP-EやDS-Liteの違いは、必要になった時に見れば大丈夫です。

## まず回線構成を見る

IPoEで最初にやるべきことは、LuCIの設定変更ではありません。

まず配線と契約を見ます。

![IPoE設定の判断フロー](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/ipoe-decision-flow.png)

## 代表的な接続パターン

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-03.png)

ひかり電話契約がある場合、HGWがIPoEを担当していることがあります。

この場合、LN6001-JP側でIPv4 over IPv6を作り直すのではなく、HGW配下のルーターとしてDHCP自動接続する構成が自然なことがあります。

一方で、ひかり電話契約なし、ONUのみ設置、LN6001-JPをONU直下に置く構成では、LN6001-JP側でオートIPoEモジュールを使う流れになります。

## 物理構成を見る

まず、実際の配線を見ます。

```txt
ONU → LN6001-JP
```

この場合、LN6001-JPが回線接続を直接担当している可能性があります。

```txt
ONU → HGW → LN6001-JP
```

この場合、HGWがIPoEやルーティングを担当していて、LN6001-JPはHGW配下で動いている可能性があります。

```txt
ONU → 既存ルーター → LN6001-JP
```

この場合、既存ルーター配下の二重ルーター構成になっている可能性があります。

まずは、上流にルーター機能を持つ機器がいるかを見ます。

ここを見ずに設定を触ると、

```txt
LN6001-JPで頑張ってIPoEを作ろうとしていたけど、
実は上流のHGWがもう担当していた
```

みたいなことが起きます。

## LuCIでWAN状態を見る

LuCIでは、まずここを見ます。

1. **Status** → **Overview** を開く
2. **Network** セクションを見る
3. IPv4 WAN Statusを確認する
4. IPv6 WAN Statusを確認する
5. WAN側IPアドレスが何になっているかを見る

WAN側IPv4アドレスが次のような範囲なら、上流ルーターやHGW配下の可能性があります。

```txt
192.168.x.x
10.x.x.x
172.16.x.x - 172.31.x.x
```

これは悪い状態とは限りません。

HGW配下でDHCP自動接続しているなら、普通に使える構成です。

ただし、LN6001-JPのLAN側IPとHGW側LAN IPが重複すると問題が起きます。

## CLIでWAN状態を見る

SSHで入れる場合は、まず読み取りだけ行います。

```sh
echo "### system"
date
ubus call system board

echo "### wan status"
ifstatus wan
ifstatus wan6

echo "### addresses"
ip addr show

echo "### default routes"
ip route show default
ip -6 route show default

echo "### recent wan logs"
logread | grep -Ei 'wan|wan6|dhcp|dhcpv6|odhcp6c|netifd' | tail -n 100
```

この段階では設定を変更しません。

見るポイントは次です。

- `wan` にIPv4アドレスが付いているか
- `wan6` にIPv6アドレスやprefixが来ているか
- IPv4 default routeがあるか
- IPv6 default routeがあるか
- ログにDHCP、DHCPv6、netifd関連のエラーが出ていないか

最初はこの読み取りだけで十分です。

## HGW配下ならまずDHCP自動

ひかり電話HGWがある場合、HGW側がIPoEを担当していることがあります。

この構成では、LN6001-JP側はWANをDHCP自動にして、HGW配下のルーターとして使うのが自然です。

## LuCIでWANをDHCP clientにする

1. **Network** → **Interfaces** を開く
2. `wan` の **Edit** をクリックする
3. **Protocol** が `DHCP client` になっているか確認する
4. 違う場合は `DHCP client` を選ぶ
5. **Save** → **Save & Apply** をクリックする

CLIで確認するだけなら次です。

```sh
echo "### WAN protocol"
uci show network.wan
```

HGW配下では、無理にオートIPoEを入れなくても動くことがあります。

ここで大事なのは、LN6001-JPが何を担当するかです。

```txt
HGWが回線接続を担当
LN6001-JPはその下でWi-Fiや家庭内LANを担当
```

こういう構成です。

## HGW配下で注意するのはLAN IP重複

HGWのLAN IPアドレスが `192.168.1.1` で、LN6001-JPのLAN IPも `192.168.1.1` の場合、重複します。

これはよくあります。

この状態では、管理画面へ入れなかったり、通信が不安定になったりします。

例:

```txt
HGW:        192.168.1.1
LN6001-JP: 192.168.1.1
```

この場合は、LN6001-JP側のLAN IPを変更します。

例:

```txt
HGW:        192.168.1.1
LN6001-JP: 192.168.10.1
```

## LuCIでLAN IPを変更する

1. **Network** → **Interfaces** を開く
2. `lan` の **Edit** をクリックする
3. **General Settings** の IPv4 addressを変更する
4. 例: `192.168.10.1`
5. **Save** → **Save & Apply** をクリックする
6. PCやスマートフォンを再接続する
7. 新しいURLでLuCIへ入る

```txt
https://192.168.10.1
```

管理画面の住所が変わるので、ここで迷子になりがちです。

変更後は、必ずメモしておきます。

## CLIでLAN IPを変更する場合

CLIで変更する場合は、必ずバックアップを取ってから行います。

```sh
echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.before-lan-ip-change.$(date +%Y%m%d-%H%M)

echo "### current LAN address"
uci get network.lan.ipaddr

echo "### change LAN address"
uci set network.lan.ipaddr='192.168.10.1'
uci changes network
uci commit network
/etc/init.d/network restart
```

ネットワーク再起動後、SSHやLuCIは一度切れます。

新しい管理画面URLは次です。

```txt
https://192.168.10.1
```

PC側が古いIPを持ったままの場合は、Wi-Fiや有線LANを一度切断して再接続します。

## ONU直下ならオートIPoEモジュール

ひかり電話契約なし、ONUのみ、LN6001-JPをONU直下へ接続する場合は、LN6001-JP側がIPoE / IPv4 over IPv6を担当します。

この場合、Linksys公式のオートIPoEモジュールを導入します。

## 対応サービスの見方

Linksys公式サポートでは、オートIPoEモジュールについて、MAP-E、DS-Lite、IPIP方式の対応状況が案内されています。

代表的には、次のようなサービスがあります。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/table-04.png)

ここで大事なのは、自分のプロバイダが何を使っているかです。

同じ「IPv4 over IPv6」でも、OCNバーチャルコネクト、transix、v6プラス、クロスパスでは方式が違います。

契約中のプロバイダのマイページ、開通案内、FAQで、サービス名や方式を確認してください。

分からない場合は、プロバイダ名と契約名を控えておくだけでも、かなり切り分けしやすくなります。

## オートIPoE導入前に確認すること

導入前に、次を確認します。

```txt
□ LN6001-JPがONUに接続されている
□ PCまたはスマートフォンからLuCIへ入れる
□ IPv6通信がある程度使えている
□ System → Software の Update lists が通る
□ プロバイダのIPoE / IPv4 over IPv6契約が有効
□ ひかり電話HGW配下ではなく、ONU直下で使う構成
□ 設定バックアップを取っている
```

IPv4 over IPv6設定前でも、NTT IPoE回線ではIPv6通信が先に使えることがあります。

Linksys公式手順でも、パッケージリスト更新ができる状態がひとつの目安として案内されています。

ここでUpdate listsがまったく通らない場合は、先に配線、IPv6、回線契約を確認します。

## 設定前にバックアップを取る

オートIPoEモジュールは、`network`、`firewall`、`dhcp` などに影響します。

まずLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-auto-ipoe-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/ipoe-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

設定ファイルには回線情報やネットワーク情報が含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

回線設定で一番強い人は、方式名を全部暗記している人ではありません。

変更前に戻れる人です。

## オートIPoEモジュールを導入する

LuCIで導入します。

1. Linksys公式サポートページから、オートIPoEモジュールをPCへダウンロードする
2. LuCIへログインする
3. **System** → **Software** を開く
4. **Update lists...** をクリックする
5. 正しく更新できたら **Dismiss** で閉じる
6. **Upload Package** をクリックする
7. **Browse...** から `velop-auto-ipoe_x.x_all.ipk` を選ぶ
8. **Upload** をクリックする
9. 確認画面で **Install** をクリックする
10. `installed in root is up to date.` のような表示を確認する
11. **Dismiss** で閉じる
12. **Services** → **Internet Assistant** を開く
13. メニューが出ない場合は、ブラウザをリロード、再ログイン、または再起動する
14. **オートIPoEを実行** をクリックする
15. 回線の自動判定と設定が終わるまで待つ
16. サービス開始ダイアログ後、ルーター再起動を待つ
17. 数分待ってから接続状態を確認する

オートIPoE実行後は、すぐに焦って何度も押さないほうがいいです。

再起動後、回線側の状態が落ち着くまで少し待ちます。

## オートIPoE実行後に確認する

LuCIでは、まず次を見ます。

1. **Status** → **Overview**
2. IPv4 WAN Status
3. IPv6 WAN Status
4. **Network** → **Interfaces**
5. `wan` / `wan6` の状態

CLIでは次を確認します。

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
opkg list-installed | grep -Ei 'map|dslite|ipoe|ipip|mape|464xlat|internet|assistant' || true

echo "### recent IPoE-related logs"
logread | grep -Ei 'ipoe|map-e|mape|dslite|ipip|wan6|odhcp6c|netifd' | tail -n 100
```

この段階では、IPv4とIPv6の両方で通信できるかを見ます。

細かいトンネル名やインターフェース名は、モジュールやファームウェアのバージョンで変わる可能性があります。

名前だけで成功・失敗を判断せず、WAN/WAN6、経路、ログ、実通信をまとめて見ます。

## 接続確認

ルーター側で疎通確認します。

```sh
echo "### IPv4 reachability"
ping -c 4 8.8.8.8

echo "### IPv6 reachability"
ping6 -c 4 2001:4860:4860::8888

echo "### DNS"
nslookup www.google.co.jp
nslookup ipv6.google.com

echo "### default routes"
ip route show default
ip -6 route show default
```

端末側でも、普段使うWebサイトやアプリを確認します。

- Google検索
- YouTube
- SNS
- 銀行・決済
- 業務アプリ
- VPN
- ゲーム機
- スマート家電アプリ

IPv4サイトだけ見られない。  
IPv6サイトだけ見られない。  
DNSだけおかしい。

このように分けて見ます。

「インターネットが全部だめ」ではなく、IPv4、IPv6、DNSを分けるのがコツです。

## PPPoE接続を使う場合

固定IP契約や、プロバイダの指定でPPPoEが必要な場合は、WANをPPPoEに設定します。

PPPoEは、プロバイダからもらったIDとパスワードで接続する方式です。

## LuCIでPPPoEを設定する

1. **Network** → **Interfaces** を開く
2. `wan` の **Edit** をクリックする
3. **Protocol** で `PPPoE` を選ぶ
4. 確認画面で **Switch protocol** をクリックする
5. **PAP/CHAP username** に接続IDを入力する
6. **PAP/CHAP password** に接続パスワードを入力する
7. **Save** をクリックする
8. 保留中の変更を確認し、**Save & Apply** をクリックする

CLIで設定する場合は、必ずバックアップを取ります。

```sh
echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.before-pppoe.$(date +%Y%m%d-%H%M)

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
logread | grep -Ei 'ppp|pppoe|wan' | tail -n 80
```

記事やSNSにPPPoE接続IDやパスワードを貼らないでください。

見落としがちですが、接続IDもかなり重要な情報です。

## IPoEとPPPoEを併用する

メイン接続をIPoEにしつつ、特定用途だけPPPoEを使いたい場合があります。

たとえば、

```txt
普段の通信はIPoE
固定IPが必要な業務システムだけPPPoE
```

という構成です。

![IPoEとPPPoEの併用イメージ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/ipoe-pppoe-coexistence.png)

ただし、併用構成は少し難しくなります。

次が関係します。

- 回線契約
- プロバイダのPPPoE接続情報
- IPoE / IPv4 over IPv6方式
- WAN側インターフェース名
- firewall zone
- default route
- policy routing
- DNSの向き先

最初から併用を作るより、まずIPoE単体、またはPPPoE単体で安定させるほうが安全です。

現在の状態確認は次の通りです。

```sh
echo "### WAN/WAN6"
ifstatus wan
ifstatus wan6

echo "### interfaces"
ip link show

echo "### network ppp hints"
uci show network | grep -Ei 'ppp|pppoe|wan|wan6'

echo "### routes"
ip route show
ip -6 route show
```

PPPoE併用が必要な場合は、プロバイダの契約情報と、どの通信をどちらへ流すかを先に決めてから進めます。

## 固定IP / IPIPの場合

固定IP系サービスでは、IPIP方式や国内標準プロビジョニング方式が関係することがあります。

Linksys公式サポートでは、オートIPoEモジュールにより標準的なIPIP設定や国内標準プロビジョニング方式のサポートが案内されています。

ただし、固定IPやIPIPは、通常の家庭向け動的IPより確認項目が増えます。

- 契約している固定IP方式
- プロバイダの対応方式
- ONU直下かHGW配下か
- HGW側のIPv6配信方式
- RA / DHCPv6-PDの扱い
- ベンダーやプロバイダの設定条件

固定IPサービスを使う場合は、プロバイダの公式情報を優先してください。

手動設定を試す前に、オートIPoEモジュールの高度な設定やLinksys公式サポート情報を確認するのがおすすめです。

## AP / ブリッジモードで使う場合

手前にHGWや既存ルーターがあり、LN6001-JPをルーターではなくアクセスポイントとして使いたい場合があります。

この場合、IPoEやPPPoEは上流ルーターが担当します。

LN6001-JPは、Wi-Fi APやブリッジとして動作する形になります。

この構成では、次を確認します。

- DHCPをどの機器が担当するか
- firewallをどの機器が担当するか
- LN6001-JPの管理IPをどうするか
- ゲストWi-FiやVLAN分離をどこで行うか
- 二重ルーターを避けたいのか、あえてルーター配下で使うのか

AP / ブリッジモードは別記事で扱います。

IPoE記事では、「回線接続は上流機器が担当する構成」として理解しておけば大丈夫です。

## よくある問題と対処

## パッケージリスト更新でErrorが出る

オートIPoEモジュール導入時に、**Update lists...** でErrorが出ることがあります。

考えられる原因です。

- IPv6通信がまだ確立していない
- ONU / HGWとのケーブル接続が正しくない
- IPoE契約がまだ有効になっていない
- 回線側の配信情報がまだ反映されていない
- WAN/WAN6の状態が想定と違う

確認します。

```sh
ifstatus wan
ifstatus wan6
ip -6 route show
logread | grep -Ei 'opkg|wget|uclient|wan6|dhcpv6|odhcp6c|netifd' | tail -n 100
```

NTT IPoE回線では、IPv4 over IPv6の設定前でもIPv6通信は利用できることがあります。

そのため、パッケージリスト更新が成功するかどうかは、IPv6側の状態を見る目安になります。

## Internet Assistantメニューが表示されない

オートIPoEモジュールのインストール直後は、LuCIメニューに反映されないことがあります。

対処:

- ブラウザをリロードする
- LuCIからログアウトして再ログインする
- ルーターを再起動する
- `opkg list-installed` でモジュールが入っているか確認する

```sh
opkg list-installed | grep -Ei 'ipoe|internet|assistant|velop' || true
logread | grep -Ei 'opkg|ipoe|internet|assistant' | tail -n 100
```

メニューが出ない時は、まずリロードです。

ここで慌てて再インストールを繰り返さなくて大丈夫です。

## IPv6はつながるがIPv4サイトが開けない

よくある状態です。

IPv6/IPoEは来ているけれど、IPv4 over IPv6側がまだ設定できていない可能性があります。

確認します。

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
ping6 -c 4 2001:4860:4860::8888
ping -c 4 8.8.8.8
logread | grep -Ei 'ipoe|map|mape|dslite|ipip|wan|wan6' | tail -n 100
```

対処:

- オートIPoEを再実行する
- 契約中のVNE方式を確認する
- プロバイダのIPv4 over IPv6契約が有効か確認する
- ONU直下かHGW配下かを再確認する
- 必要ならPPPoE接続情報の有無も確認する

「IPv6は生きているのにIPv4だけだめ」は、IPoEまわりではかなり典型的です。

ここでWi-Fi設定を触らないでください。

まずWAN/WAN6とIPv4 over IPv6側を見ます。

## HGW配下で管理画面に入れない

HGWとLN6001-JPのLAN IPアドレスが重複している可能性があります。

特に、どちらも `192.168.1.1` の場合は注意です。

対処:

- LN6001-JPのLAN IPを `192.168.10.1` などへ変更する
- PCを再接続して新しいIPを取得する
- 新しい管理URLへアクセスする

```txt
https://192.168.10.1
```

「LuCIに入れない」のではなく、「同じ住所の機器が2台いる」だけのことがあります。

## IPv4もIPv6もつながらない

まず物理接続から見ます。

- WANケーブルがInternet/WANポートに入っているか
- ONU / HGWが正常に動いているか
- 回線機器を再起動したか
- LuCIでWAN/WAN6がupになっているか
- WAN設定を触りすぎていないか

CLIでは次を見ます。

```sh
ifstatus wan
ifstatus wan6
ip link show
logread | grep -Ei 'link|wan|wan6|netifd|dhcp|ppp|ipoe' | tail -n 120
```

いきなりMAP-EやDS-Liteの設定を疑うより、まず物理接続、WAN protocol、WAN/WAN6のup/downを確認します。

## PPPoEがつながらない

確認するポイントです。

- PPPoE接続IDが正しいか
- PPPoEパスワードが正しいか
- プロバイダの接続IDにドメイン部分が必要か
- 回線契約でPPPoEが有効か
- 同時接続制限に引っかかっていないか
- IPoE設定や既存WAN設定と衝突していないか

CLIでは次を見ます。

```sh
ifstatus wan
logread | grep -Ei 'ppp|pppoe|pap|chap|wan' | tail -n 120
```

PAP/CHAP認証エラーが出ている場合は、IDやパスワードを見直します。

## DNSだけおかしい

IPv4/IPv6のpingは通るのに、Webサイト名が引けない場合はDNSを確認します。

```sh
ping -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
nslookup www.google.co.jp
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null || true
cat /etc/resolv.conf
logread | grep -Ei 'dnsmasq|dns' | tail -n 100
```

Adblock、Family DNS、DoH対策、VPN、Tailscaleなどを入れている場合は、DNSの経路が複雑になっていることがあります。

まず、端末がどのDNSを使っているかを確認します。

## MTUっぽい問題がある

一部サイトだけ開けない。  
VPNやゲームだけ不安定。  
画像や動画だけ止まる。

この場合、MTUや経路の問題が関係することがあります。

ただし、最初からMTUを変更する必要はありません。

まずは、通常のIPv4/IPv6、DNS、回線方式が正しいかを確認します。

```sh
ping -c 4 8.8.8.8
ping -M do -s 1472 -c 4 8.8.8.8
ping6 -c 4 2001:4860:4860::8888
```

MTU変更は影響が大きいので、公式手順やプロバイダ情報を確認してから行います。

## 買う前・設定前チェックリスト

購入前、または設定前に、次の情報を整理しておくとかなり楽です。

```txt
回線事業者:
  NTT東日本 / NTT西日本 / その他

プロバイダ:
  OCN / IIJ / BIGLOBE / GMO / ASAHIネット / その他

ひかり電話:
  あり / なし

HGW:
  あり / なし

ONU直下:
  はい / いいえ

IPv4 over IPv6サービス:
  OCNバーチャルコネクト / transix / v6プラス / クロスパス / v6コネクト / 不明

方式:
  MAP-E / DS-Lite / IPIP / 不明

PPPoE接続情報:
  あり / なし

固定IP契約:
  あり / なし

既存ルーターのLAN IP:
  例: 192.168.1.1

LN6001-JPのLAN IP予定:
  例: 192.168.10.1
```

分からない項目があっても大丈夫です。

「HGWありか」「ONU直下か」「プロバイダ名」だけでも、かなり切り分けしやすくなります。

## 運用メモのテンプレート

設定が終わったら、次のようなメモを残しておくと便利です。

```txt
回線:
  NTTフレッツ 光クロス / 光ネクスト など

プロバイダ:
  OCN / IIJ / BIGLOBE など

ひかり電話:
  あり / なし

構成:
  ONU -> LN6001-JP
  または
  ONU -> HGW -> LN6001-JP

IPv4 over IPv6:
  OCNバーチャルコネクト / transix / v6プラス / クロスパス

方式:
  MAP-E / DS-Lite / IPIP

LN6001-JP LAN IP:
  192.168.10.1

WAN:
  DHCP / オートIPoE / PPPoE

オートIPoEモジュール:
  導入日: 2026-06-21
  バージョン: 確認したバージョンを記録

確認:
  ifstatus wan OK
  ifstatus wan6 OK
  IPv4 ping OK
  IPv6 ping OK
  DNS OK

バックアップ:
  backup-LN6001-before-auto-ipoe-20260621.tar.gz
```

回線設定は、半年後に見返すとかなり忘れています。

メモを残しておくと、ファームウェア更新、ルーター入れ替え、プロバイダ変更の時に助かります。

## まとめ

LN6001-JPで日本のIPoE / IPv4 over IPv6回線を使う時は、最初に回線構成を見分けます。

基本方針は次の通りです。

1. HGW配下なら、LN6001-JPはWAN=DHCP自動で使えることが多い
2. HGWとLN6001-JPのLAN IP重複に注意する
3. ひかり電話なし / ONU直下なら、Linksys公式オートIPoEモジュールを使う
4. OCNバーチャルコネクト、transix、v6プラス、クロスパスなどの方式は契約情報で確認する
5. PPPoEや固定IPは、プロバイダ情報を見て慎重に設定する
6. 接続後は `ifstatus wan` / `ifstatus wan6` で状態を確認する
7. IPv4、IPv6、DNSを分けて切り分ける

最初からMAP-E、DS-Lite、IPIPを手動で組む必要はありません。

まずは、回線構成を確認する。  
公式モジュールでつながる状態を作る。  
それからPPPoE併用、固定IP、IPIP、VPN、VLANへ進む。

この順番が安全です。

IPoEは名前が難しいですが、最初の見方はシンプルです。

```txt
HGWが担当しているのか
LN6001-JPが担当するのか
IPv4とIPv6のどちらが止まっているのか
```

まずここから見れば大丈夫です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/014/diagram-03.png)

## 次に読むなら

IPoEまわりが見えてきたら、次は目的に合わせて進みます。

- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)

IPv6で詰まりやすいポイントを知りたい人は、IPv6の落とし穴へ。

VPNを使う人は、IPoE環境でポート開放できるか、Tailscaleのほうが楽かをVPN記事で確認すると読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

オートIPoEモジュール、Internet Assistant、LuCI画面名、インターフェース名、ログ出力は、ファームウェアやモジュール更新で変わることがあります。

また、IPoE / IPv4 over IPv6、PPPoE、固定IP、IPIPの条件は、契約している回線、プロバイダ、ホームゲートウェイの有無によって変わります。

記事の内容と実際の画面や挙動が違う場合は、まずバックアップを取り、Linksys公式サポート、プロバイダの契約情報、最新のサポート記事を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPはIPoE / IPv4 over IPv6に対応している？

はい。

Linksys公式のオートIPoEモジュールが案内されており、MAP-E、DS-Lite、IPIPなどのIPv4 over IPv6方式に対応する構成が用意されています。

### HGWがある場合もオートIPoEモジュールは必要？

ひかり電話HGWがIPoEを担当している場合、LN6001-JP側はWAN=DHCP自動で使えることが多いです。

この場合、オートIPoEモジュールよりも、まずHGWとLN6001-JPのLAN IPアドレス重複を避けることが重要です。

### ONU直下なら何をすればいい？

ひかり電話なし / ONUのみのNTT IPoE回線では、Linksys公式のオートIPoEモジュールを導入し、Internet AssistantからオートIPoEを実行します。

### IPv6はつながるのにIPv4サイトが開けないのはなぜ？

IPv6/IPoEは来ているが、IPv4 over IPv6側の設定が完了していない可能性があります。

オートIPoEを再実行し、契約中のVNE方式、WAN/WAN6、ログを確認してください。

### PPPoEとIPoEはどちらを使えばいい？

通常利用では、プロバイダが対応しているならIPoE / IPv4 over IPv6を優先することが多いです。

ただし、固定IP契約や業務要件がある場合はPPPoEが必要になることがあります。

### 固定IPやIPIPは使える？

利用可能な場合がありますが、契約内容とプロバイダの方式確認が必要です。

固定IPやIPIPは通常の家庭向け動的IPより確認項目が多いため、Linksys公式サポートとプロバイダ情報を確認してから設定してください。

### IPoE環境でWireGuardは使える？

使える場合もありますが、MAP-E、DS-Lite、IPIPなど方式によって、外から任意ポートを届けられるかが変わります。

固定IPやポート開放が難しい場合は、Tailscaleのほうが始めやすいことがあります。

## 参考リンク

- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/
- WireGuard/Tailscaleでリモート接続: https://note.com/ikmsan/n/n1a83908a226c
- IPv6の落とし穴: https://note.com/ikmsan/n/n61235a13b478

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
