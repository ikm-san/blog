<!-- mirror-source: articles/027-home-configuration.md -->

# 自宅Wi-Fi、全部同じでいい？｜Main・Kids・IoT・ゲストWi-Fiをゆるく分ける【OpenWrt集中連載027】

自宅のWi-Fiって、気づくと“全部入り鍋”になります。

親のPC。  
スマホ。  
子どものタブレット。  
ゲーム機。  
テレビ。  
スマートスピーカー。  
ロボット掃除機。  
見守りカメラ。  
NAS。  
プリンター。  
そしてゲストWi-Fi。

最初は全部同じSSIDでいいんです。

むしろ、そのほうが楽です。  
家族にも説明しやすいし、端末もすぐつながります。

でも、ある日こう思います。

```txt
子ども用端末だけDNSフィルタリングしたい
スマート家電はNASやPCから分けたい
ゲストにはインターネットだけ使わせたい
プリンターのIPが変わってほしくない
スマホは速いWi-Fi、IoTは安定するWi-Fiにしたい
```

ここでいきなり全部をVLANでガチガチに分けると、だいたい家庭内サポートセンターが始まります。

```txt
プリンターが見えない
テレビにキャストできない
学校用PCだけ教材サイトへ入れない
スマート家電アプリが機器を見つけられない
結局どのSSIDにつなげばいいの？
```

つらいです。

なので、家庭向け構成で大事なのは、最初から完璧なネットワーク城を建てることではありません。

まずMainを安定させる。  
次にゲストWi-Fiを分ける。  
必要になったらKidsとIoTを足す。  
NASとプリンターだけIPを固定する。  
AdblockやVPNは、動く状態を作ってから追加する。

このくらいで十分です。

LN6001-JPはOpenWrtベースなので、SSID、network、DHCP、DNS、firewall zoneを組み合わせて、自宅ネットワークをかなり柔軟に整理できます。

でも家庭では、自由度よりも“続けられること”が大事です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）を自宅で使う場合の、Main / ゲストWi-Fi / Kids / IoTの分け方を、現実的な順番で整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、自宅をいきなり業務用ネットワークみたいにすることではありません。

まずは、ここまで分かればOKです。

- Main / ゲストWi-Fi / Kids / IoTをどう分けるか分かる
- 最初から全部作らず、段階的に増やす考え方が分かる
- 2.4GHz / 5GHz / 6GHzを家庭でどう使い分けるか分かる
- KidsにFamily DNSを配る時の注意点が分かる
- IoT用SSIDを作る時のハマりどころが分かる
- NASやプリンターをDHCP予約すべき理由が分かる
- Adblockを家庭で強くしすぎない理由が分かる
- 設定後に見るLuCI画面とCLIコマンドが分かる

家庭ネットワークでは、完璧な分離より、

```txt
家族が普通に使える
危ない混ざり方を避ける
困ったら戻せる
```

この3つのほうが大事です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-01.png)

## 先にざっくり結論

家庭では、最初から高度なVLAN設計を作らなくて大丈夫です。

おすすめの順番はこれです。

1. **Main Wi-Fiを安定させる**
2. **ゲストWi-Fiを追加する**
3. **必要ならKidsを追加する**
4. **スマート家電が増えたらIoTを追加する**
5. **NASやプリンターだけDHCP予約する**
6. **Adblockは軽めに入れて、壊れたら戻せるようにする**

最終形は、たとえばこうです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-01.png)

ただし、最初から4つ全部はいりません。

最初の成功条件は、これくらいで十分です。

```txt
Main Wi-Fiで家族が普通に使える
ゲストWi-Fiから家庭内LANへ入れない
NASとプリンターのIPが変わらない
```

ここまでできれば、自宅ネットワークとしてかなり扱いやすくなります。

家庭では“全部できる構成”より、“家族が困らない構成”のほうが強いです。

## この記事の前提

この記事では、次のような家庭構成を例にします。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-02.png)

HGW配下で使っている場合や、LN6001-JPのLAN IPを `192.168.10.1` などへ変更している場合は、自分の環境に合わせて読み替えてください。

また、この記事は家庭向けの“構成例”です。

各機能の細かい手順は、それぞれの記事で詳しく扱っています。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-02.png)

## こういう家庭向けです

この記事は、次のような家庭に向いています。

- 家族のスマートフォン、PC、タブレットが多い
- 子ども用端末だけDNSフィルタリングしたい
- スマート家電やカメラを家庭内LANから分けたい
- ゲストWi-Fiを作りたい
- NASやプリンターを安定して使いたい
- Wi-Fi 7、6GHz、MLOに興味はあるが、まず安定運用したい
- AdblockやVPNもあとから足したい
- いきなり複雑なVLAN構成にはしたくない
- 家族に説明できるWi-Fi名にしたい

逆に、端末が少なく、ゲストWi-Fiも不要で、子ども用端末やIoTもほとんどない場合は、Mainだけでも大丈夫です。

家庭ネットワークは、必要になってから育てればOKです。

## 最初に言葉だけそろえる

家庭向け構成で出てくる言葉を、ざっくり整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-03.png)

最初は、次の理解で十分です。

```txt
Main = 家族用
ゲストWi-Fi = お客さん用
Kids = 子ども用
IoT = スマート家電用
```

ここまで分かれば、十分スタートできます。

## 家庭向け構成の考え方

家庭向けでは、セキュリティを高めたい一方で、家族が普通に使えることも大事です。

強く分けすぎると、こうなりがちです。

- プリンターが見えない
- テレビへキャストできない
- スマート家電アプリが動かない
- 子どもの学校用PCだけ教材サイトへ入れない
- ゲーム機だけ接続が不安定
- 家族にSSIDの使い分けを説明しきれない

なので、家庭では次の方針が現実的です。

```txt
Mainは安定優先
ゲストWi-FiはLAN隔離
KidsはDNS方針を分ける
IoTはLANから分ける
例外は必要になってから作る
```

最初から「すべての通信を厳密に制御する」より、「役割ごとにSSIDを分けて、危ない混ざり方を避ける」くらいから始めるほうが続きます。

家庭ネットワークは、攻めすぎると家族から怒られます。

そこそこ安全。  
そこそこ便利。  
困ったら戻せる。

このくらいが本当に強いです。

## まずは2つから始める

最初のおすすめは、MainとゲストWi-Fiの2つです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-04.png)

最初はこれだけで十分です。

```txt
HomeWifi:
  家族のPC、スマートフォン、NAS、プリンター

HomeWifi_Guest:
  ゲスト端末
  インターネットだけ
  NASやプリンターには入れない
```

KidsやIoTは、必要になってから追加します。

家族に説明できないSSIDを増やしすぎると、運用がかえって難しくなります。

```txt
このSSID、誰が使うの？
```

に答えられないSSIDは、まだ作らなくて大丈夫です。

## 最終的な自宅ネットワーク設計図

最終的には、次のような構成を目指せます。

```txt
インターネット
    │
ONU / HGW
    │
LN6001-JP
    │
    ├── Main: 192.168.1.0/24
    │     SSID: HomeWifi
    │     用途: 親のPC、スマートフォン、NAS、プリンター
    │
    ├── Guest: 192.168.2.0/24
    │     SSID: HomeWifi_Guest
    │     用途: ゲスト端末
    │     方針: インターネットのみ
    │
    ├── Kids: 192.168.3.0/24
    │     SSID: HomeWifi_Kids
    │     用途: 子ども用スマートフォン、タブレット、学校用PC
    │     DNS: Family DNS
    │
    └── IoT: 192.168.4.0/24
          SSID: HomeWifi_IoT
          用途: スマート家電、テレビ、カメラ
          方針: LANへは原則入れない
```

この設計は、027時点で全部作る必要はありません。

このあと必要に応じて、007のゲストWi-Fi、009の家族向けフィルタリング、010の家庭内ネットワーク分離、012のDHCP予約、016のfirewall zonesへ進めば大丈夫です。

## 2.4GHz / 5GHz / 6GHzの使い分け

LN6001-JPは、2.4GHz / 5GHz / 6GHzのトライバンド構成です。

家庭では、次のように考えると分かりやすいです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-05.png)

最初は、5GHzをMainの主力にします。

IoTは2.4GHz中心にします。

6GHzやMLOは、対応端末がある場合に後から使えば十分です。

```txt
HomeWifi       → 5GHz中心
HomeWifi_IoT   → 2.4GHz中心
HomeWifi_6G    → 6GHz対応端末向け
```

無理に最初からMLOを使い倒す必要はありません。

家庭では「理論値」より「家族が普通に使えること」が最優先です。

## SSID名の考え方

SSID名は、短く、説明しやすく、家族が迷いにくい名前にします。

おすすめ例です。

```txt
HomeWifi
HomeWifi_Guest
HomeWifi_Kids
HomeWifi_IoT
HomeWifi_6G
```

避けたい例です。

```txt
住所が入っているSSID
本名が入っているSSID
部屋番号が入っているSSID
長すぎるSSID
記号が多すぎるSSID
日本語や絵文字を含むSSID
```

特にIoT機器では、日本語SSIDや記号の多いSSIDで相性が出ることがあります。

古いスマート家電やプリンターをつなぐSSIDは、英数字中心にしておくと切り分けしやすいです。

## 設定前にバックアップを取る

Main以外のSSIDやネットワークを作る前に、バックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-home-config-20260622.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/home-before-$(date +%Y%m%d-%H%M)"
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

バックアップにはSSID、Wi-Fiパスワード、内部IP、MACアドレスなどが含まれることがあります。

SNS、公開リポジトリ、記事スクリーンショットへそのまま出さないでください。

家庭ネットワークでも、バックアップは大事です。

むしろ家庭こそ、週末の深夜に壊すとつらいです。

## ステップ1: Main Wi-Fiを安定させる

最初に作るのは、家族が普段使うMain Wi-Fiです。

Mainには、次のような端末を置きます。

- 親のPC
- スマートフォン
- タブレット
- NAS
- プリンター
- 家族共用PC
- 必要に応じてテレビ
- 管理用PC

Mainは、家庭内で一番信頼するネットワークです。

ここを安定させることが最優先です。

## Main SSIDの考え方

SSID名は、分かりやすく短めにします。

例:

```txt
HomeWifi
```

MainのWi-Fiパスワードは、家族で管理しやすく、かつ十分強いものにします。

ゲストWi-FiやIoT用SSIDとは別のパスワードにします。

```txt
Mainのパスワード = 家族用
ゲストWi-Fiのパスワード = 来客用
IoTのパスワード = 機器用
```

全部同じパスワードにすると、あとで分けた意味が薄くなります。

## LuCIでMain SSIDを確認する

1. **Network** → **Wireless** を開く
2. 5GHz radioを確認する
   - Linksys公式手順上、`wifi1` が5GHzです
3. Main SSIDの **Edit** を開く
4. **General Setup** でESSIDを確認する
5. **Wireless Security** で暗号化とKeyを確認する
6. **Network** が `lan` になっていることを確認する

最初は、5GHzのMain SSIDが安定して使えることを確認します。

6GHzやMLOは後からでも大丈夫です。

## Mainの確認

スマートフォンやPCをMain SSIDへ接続し、次を確認します。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-06.png)

SSHで確認するなら次です。

```sh
echo "### wireless mapping"
uci show wireless | grep -E 'ssid|network|encryption'

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### WAN"
ifstatus wan
ifstatus wan6

echo "### DNS"
nslookup www.google.co.jp
```

Mainが不安定な状態でゲストWi-Fi、Kids、IoTを追加すると、切り分けが難しくなります。

まずMainを安定させます。

## ステップ2: ゲストWi-Fiを作る

次に追加するなら、ゲストWi-Fiです。

ゲストWi-Fiの目的は、単に別SSIDを作ることではありません。

**ゲスト端末を家庭内LANから分け、インターネットだけ使えるようにすること**です。

```txt
HomeWifi_Guest
  ↓
guest network: 192.168.2.0/24
  ↓
guest zone
  ↓
Internet only
```

## ゲストWi-Fiの基本方針

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-07.png)

007の記事で詳しく扱った通り、guest zoneのInputを `REJECT` にする場合は、DHCPとDNSを明示的に許可します。

```txt
Allow-Guest-DHCP
Allow-Guest-DNS
```

これがないと、ゲスト端末がIPを取れなかったり、DNS名前解決できなかったりします。

ゲストWi-Fiの定番ハマりポイントです。

## ゲストWi-Fiを5GHzに出すか、2.4GHzに出すか

家庭の来客端末は、スマートフォンやノートPCが中心です。

そのため、まず5GHzで作るのが扱いやすいです。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-08.png)

ゲストWi-Fiは、必要以上に強く速くするより、家庭内LANから分けることのほうが大事です。

## ゲストWi-Fiの確認

ゲストWi-Fiへ接続した端末で、次を確認します。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-09.png)

SSHで確認するなら次です。

```sh
echo "### guest interface"
ifstatus guest 2>/dev/null || true

echo "### guest DHCP"
uci show dhcp.guest 2>/dev/null || true

echo "### guest firewall"
uci show firewall | grep -E 'guest|Allow-Guest|forwarding'

echo "### wireless guest mapping"
uci show wireless | grep -E 'guest|HomeWifi_Guest|ssid|network'
```

ゲストWi-FiからLANへ届かないことを必ず確認します。

「インターネットへ出られる」だけでは不十分です。

## ステップ3: Kidsを追加する

子ども用端末を分けたい場合は、Kidsネットワークを作ります。

ただし、Kidsは「子ども端末を完全に制御する魔法」ではありません。

最初は、次のように考えると現実的です。

```txt
Kids SSIDを分ける
Family DNSを配る
端末側のファミリー機能も併用する
必要なら時間ルールを後から足す
```

## Kidsの基本方針

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-10.png)

Cloudflare Family DNSで、マルウェアと成人向けコンテンツを減らす場合は、IPv4では次を使います。

```txt
1.1.1.3
1.0.0.3
```

DHCP Optionで配る例です。

```txt
6,1.1.1.3,1.0.0.3
```

IPv6もKidsへ配る場合は、IPv6側のDNS、RA / DHCPv6、firewallも合わせて見ます。

最初はIPv4で構成を安定させてから、IPv6を考えるくらいでも大丈夫です。

## LuCIでKidsへFamily DNSを配る考え方

1. **Network** → **Interfaces** を開く
2. `kids` interfaceを作る
3. `192.168.3.1/24` を設定する
4. DHCP Serverを有効にする
5. **DHCP Server** → **Advanced Settings** を開く
6. **DHCP Options** に次を入れる

```txt
6,1.1.1.3,1.0.0.3
```

7. Kids用SSIDを作り、Networkを `kids` にする
8. kids zoneを作り、kids → wanのみ許可する

CLIで状態確認するなら次です。

```sh
echo "### kids interface"
ifstatus kids 2>/dev/null || true

echo "### kids DHCP"
uci show dhcp.kids 2>/dev/null || true

echo "### kids DNS option"
uci show dhcp | grep -E 'kids|dhcp_option'

echo "### kids firewall"
uci show firewall | grep -E 'kids|Allow-Kids|forwarding'
```

## Kidsで注意すること

Family DNSだけで、すべての不適切コンテンツやアプリ利用を完全に制御できるわけではありません。

次のような抜け道や限界があります。

- DoHを使うブラウザ
- VPNアプリ
- モバイル回線
- アプリ内通信
- 同一ドメインからの広告や動画
- 学校管理端末の独自設定
- ゲーム機や動画サービス側の年齢設定

そのため、Kidsではルーター側DNSだけでなく、端末側のファミリー機能も使います。

家庭では、技術だけで全部を封じるより、家庭内ルールと組み合わせるほうが長続きします。

ネットワーク設定は、親子バトルの代わりにはなりません。

補助輪くらいのつもりで使うのが良いです。

## ステップ4: IoTを追加する

スマート家電、テレビ、カメラ、スマートスピーカーが増えてきたら、IoT用SSIDを作ると整理しやすくなります。

IoTの目的は、スマート家電を家庭内LANの中心から少し離すことです。

```txt
HomeWifi_IoT
  ↓
iot network: 192.168.4.0/24
  ↓
iot zone
  ↓
Internet / 必要な通信だけ
```

## IoTの基本方針

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-11.png)

IoT機器は2.4GHzしか対応していないものが多いです。

最初は、2.4GHzにIoT専用SSIDを作るのが扱いやすいです。

## IoTに置く端末例

- スマートスピーカー
- ロボット掃除機
- スマート照明
- スマートコンセント
- テレビ
- ストリーミング端末
- 見守りカメラ
- 空気清浄機
- 温湿度センサー

ただし、プリンターやNASはMainに置くほうが便利な場合もあります。

プリンターをIoTへ置くと、家族のPCから見えなくなることがあります。

家庭では、厳密な分離よりも「実際に困らない構成」を優先します。

## IoTで注意すること

IoTを分けると、次のような問題が出ることがあります。

- 初期設定アプリが機器を見つけられない
- スマートテレビへキャストできない
- カメラアプリから見えない
- mDNSやローカル探索が必要
- クラウド接続先がDNSブロックで止まる
- 2.4GHzしか対応していない
- WPA3に対応していない

最初から厳しく閉じすぎないほうが安全です。

まずはIoT用SSIDへつながること、インターネットへ出られること、アプリが動くことを確認します。

その後で、LANへの通信を必要最小限にします。

## ステップ5: NASとプリンターをDHCP予約する

家庭で「急に使えない」と言われやすいのが、NASとプリンターです。

これらはIPアドレスを固定しておくと楽です。

端末側へ固定IPを手入力するより、まずLN6001-JP側のDHCP予約を使います。

## 予約する機器の例

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-12.png)

最初は、NASとプリンターだけで十分です。

「IPが変わると困る機器」だけ予約します。

スマホや一時利用PCまで全部予約し始めると、管理表がすぐに筋トレメニューみたいになります。

## LuCIでDHCP予約する

1. **Network** → **DHCP and DNS** を開く
2. **Static Leases** タブを開く
3. **Add** をクリックする
4. Hostname、MAC address、IPv4 addressを入力する
5. **Save & Apply** をクリックする

例:

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-13.png)

CLIで確認するなら次です。

```sh
echo "### active leases"
cat /tmp/dhcp.leases

echo "### static leases"
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='
```

DHCP予約は012の記事で詳しく扱っています。

## ステップ6: Adblockを軽めに入れる

家庭では、Adblockを入れると、広告・トラッキング系ドメインの一部を減らせます。

ただし、Adblockは「広告を全部消す魔法」ではありません。

YouTubeやSNSの広告には効きにくいことがあります。  
アプリ内広告が残ることもあります。  
DoHを使う端末には迂回されることもあります。

さらに、強くしすぎると、ログイン、決済、動画、スマート家電、ゲーム、学校用サービスが壊れることがあります。

## 家庭でのおすすめ方針

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-14.png)

最初は、Mainだけ軽めに使うくらいで十分です。

KidsはFamily DNSを主役にし、Adblockは必要に応じて考えます。

IoTは誤ブロックでアプリが不安定になることがあるため、慎重にします。

```txt
広告ゼロを目指す
```

より、

```txt
壊さず少し減らす
```

くらいが家庭ではちょうどいいです。

## Adblock導入前にバックアップする

AdblockはDNSに関係するため、導入前にバックアップします。

```sh
BACKUP_DIR="/root/home-before-adblock-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in dhcp firewall network; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

導入スクリプトを使う場合は、内容を確認してから実行します。

```sh
curl -sS -o /tmp/adb_setup.sh https://raw.githubusercontent.com/ikm-san/velop/main/adb_setup.sh
sed -n "1,160p" /tmp/adb_setup.sh
sh /tmp/adb_setup.sh -v
```

手動で進める場合は次です。

```sh
opkg update
opkg install adblock luci-app-adblock luci-i18n-adblock-ja

/etc/init.d/adblock status 2>/dev/null || true
logread | grep -Ei 'adblock|dnsmasq' | tail -n 80
```

日本語パッケージが見つからない場合は、`adblock` と `luci-app-adblock` だけでも構いません。

## VPNは必要になってからでいい

家庭でも、外出先からNASやミニPCへ入りたいことがあります。

その場合は、ポート開放よりVPNを優先します。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-15.png)

ただし、家庭向け構成の最初からVPNまで入れる必要はありません。

まずMain。  
次にゲストWi-Fi。  
必要ならKids / IoT。  
その後でVPN。

この順番のほうが切り分けしやすいです。

## IPv6もあとで確認する

IPoE環境では、IPv6が有効になっていることがあります。

Mainでは問題なくても、ゲストWi-Fi、Kids、IoTでIPv6側の分離を見落とすことがあります。

最初はIPv4で構成を安定させ、その後でIPv6側も確認します。

確認コマンドです。

```sh
echo "### IPv6 WAN"
ifstatus wan6

echo "### IPv6 routes"
ip -6 route show

echo "### IPv6 firewall hints"
uci show firewall | grep -Ei 'ip6|ipv6|family|guest|kids|iot'
```

ゲストWi-FiからLANへ入れない構成にしたつもりでも、IPv6側で抜け道がないか確認します。

IPv6は019の記事で詳しく扱っています。

## 家庭での確認チェックリスト

設定後は、実際の端末で確認します。

## Main

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-16.png)

## ゲストWi-Fi

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-17.png)

## Kids

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-18.png)

## IoT

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/table-19.png)

ここで大事なのは、画面上の設定だけで満足しないことです。

最後は実端末で確認します。

## CLIで棚卸しする

家庭向け構成の状態を一気に確認するなら、次を使います。

```sh
echo "### SSID and network mapping"
uci show wireless | grep -E 'ssid|network|encryption|disabled'

echo "### network interfaces"
uci show network | grep -E 'lan|guest|kids|iot|ipaddr|proto'

echo "### DHCP settings"
uci show dhcp | grep -E 'guest|kids|iot|dhcp_option|@host|\\.name=|\\.mac=|\\.ip='

echo "### firewall zones"
uci show firewall | grep -E 'guest|kids|iot|zone|forwarding|Allow-'

echo "### active leases"
cat /tmp/dhcp.leases

echo "### WAN/WAN6"
ifstatus wan
ifstatus wan6

echo "### DNS/Adblock logs"
logread | grep -Ei 'dnsmasq|adblock|firewall' | tail -n 100
```

この出力で、SSIDとnetworkの紐づき、DHCP、firewall zone、端末リースがざっくり確認できます。

## よくある失敗

## SSIDを増やしすぎる

家庭では、SSIDを増やしすぎると家族に説明しにくくなります。

最初は次くらいで十分です。

```txt
HomeWifi
HomeWifi_Guest
HomeWifi_IoT
HomeWifi_Kids
```

役割を説明できないSSIDは作らないほうが管理しやすいです。

## ゲストWi-FiがLANへ届いてしまう

ゲストWi-Fiとしては失敗です。

確認します。

```sh
uci show wireless | grep -E 'guest|ssid|network'
uci show firewall | grep -E 'guest|lan|forwarding'
```

見るポイントです。

- ゲストSSIDのNetworkが `guest` か
- guest → lan forwardingを作っていないか
- guest zoneのForwardが `REJECT` か
- guest zoneからwanだけ許可しているか

## ゲストWi-FiでIPが取れない

guest zoneのInputを `REJECT` にしている場合、DHCP許可が必要です。

確認します。

```sh
uci show dhcp.guest
uci show firewall | grep -E 'Allow-Guest-DHCP|guest'
logread | grep -i dnsmasq | tail -n 80
```

`Allow-Guest-DHCP` と `Allow-Guest-DNS` を確認します。

## Kidsで学校用サービスが動かない

Family DNSやAdblockで誤ブロックされている可能性があります。

確認します。

- Mainでは動くか
- Kidsだけで壊れるか
- Family DNSを一時的に通常DNSへ戻すと動くか
- Adblockを止めると動くか
- 学校管理端末側のVPNやDNS設定があるか

学校用PCは、学校側の管理ポリシーを使っていることがあります。

家庭側だけで原因を決めつけないようにします。

## IoT機器が見つからない

IoTを分けると、アプリの初期設定で見つからないことがあります。

よくある原因です。

- 初期設定時だけ同じLANにいる必要がある
- mDNSやローカル探索が必要
- 2.4GHzしか対応していない
- WPA3に対応していない
- AdblockやDNS制御でクラウド接続が止まっている

最初はIoT用SSIDへ接続できることを優先します。

LAN隔離やDNS制御は、そのあと少しずつ整えます。

## プリンターが見えない

プリンターをIoTやゲストWi-Fiへ置くと、Mainから見えないことがあります。

家庭では、プリンターはMainに置き、DHCP予約するほうが楽な場合が多いです。

どうしても分けたい場合は、必要な通信だけfirewallで例外許可します。

ただし、AirPrint、IPP、mDNS、メーカー独自アプリが絡むことがあります。

無理に分けすぎないほうが家庭では楽です。

## Adblockで一部サイトが壊れる

Adblockを一時停止して確認します。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

これで直るなら、Adblockやブロックリストが原因の可能性があります。

必要なドメインをAllowlistへ追加するか、ブロックリストを減らします。

確認後、戻す場合です。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

## 家族に説明しやすい運用メモ

家庭内ネットワークでも、簡単なメモがあると便利です。

```txt
自宅ネットワーク:

HomeWifi:
  家族用。PC、スマートフォン、NAS、プリンター。
  IP: 192.168.1.0/24

HomeWifi_Guest:
  ゲストWi-Fi用。
  IP: 192.168.2.0/24
  家庭内LANへは入れない。

HomeWifi_Kids:
  子ども用端末。
  IP: 192.168.3.0/24
  DNS: 1.1.1.3 / 1.0.0.3

HomeWifi_IoT:
  スマート家電用。
  IP: 192.168.4.0/24
  2.4GHz中心。

固定IP:
  NAS: 192.168.1.10
  プリンター: 192.168.1.20

バックアップ:
  backup-LN6001-home-ok-20260622.tar.gz
```

家族には、全部を説明する必要はありません。

最低限、次だけ伝われば十分です。

```txt
普段はHomeWifi
ゲストにはHomeWifi_Guest
スマート家電はHomeWifi_IoT
子ども用端末はHomeWifi_Kids
```

## 変更する時の順番

家庭では、変更を一気にやらないほうが安全です。

おすすめ順です。

1. Main Wi-Fiを安定させる
2. バックアップを取る
3. ゲストWi-Fiを作る
4. ゲストWi-FiからLANへ届かないことを確認する
5. バックアップを取る
6. Kidsを作る
7. Family DNSと学校用サービスを確認する
8. バックアップを取る
9. IoTを作る
10. スマート家電アプリが動くか確認する
11. バックアップを取る
12. AdblockやVPNを必要に応じて追加する

1つ作って、確認して、バックアップ。

これが一番安全です。

OpenWrtベースルーターは、自由に触れるからこそ、戻れる状態を作ってから進めます。

## まとめ

家庭向け構成では、最初から全部を分ける必要はありません。

まずMainを安定させます。

次にゲストWi-Fiを作ります。

必要になったらKidsとIoTを足します。

おすすめの考え方はこれです。

```txt
HomeWifi:
  家族の主力ネットワーク

HomeWifi_Guest:
  ゲストWi-Fi。インターネットだけ。

HomeWifi_Kids:
  子ども用。Family DNSを配る。

HomeWifi_IoT:
  スマート家電用。2.4GHz中心。
```

大事なのは、SSIDを増やすことではありません。

用途ごとに分ける。  
必要な通信だけ許可する。  
壊れたら戻せる状態にしておく。

LN6001-JPはOpenWrtベースなので、あとからゲストWi-Fi、Kids、IoT、Adblock、VPN、VLANを足していけます。

家庭では、最初から完璧な構成を目指すより、動く状態から少しずつ整えるほうが長続きします。

家庭内ネットワークは、育てるものです。

最初から城を建てなくて大丈夫です。

まずは、家族が普通に使えるWi-Fiから始めましょう。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-03.png)

## 次に読むなら

家庭向け構成を作るなら、次の記事も合わせて読むと整理しやすいです。

- [Wi-Fi設定](https://note.com/ikmsan/n/n4dbf8f72744c)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [DNS広告ブロックの始め方](https://note.com/ikmsan/n/n4759cf81d0e1)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)

まずMainとゲストWi-Fiを作りたい人は、Wi-Fi設定とゲストWi-Fiの記事へ。

子ども用やIoTを分けたい人は、家族向けフィルタリングとfirewall zonesの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、Wi-Fi radio名、`ifstatus`、`wifi status`、`logread`、AdblockやFamily DNS関連の設定は、ファームウェアや追加モジュール更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートや各サービスの最新情報も確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### 自宅ではSSIDをいくつ作ればいい？

最初はMainとゲストWi-Fiの2つで十分です。

スマート家電が多いならIoT、子ども用端末を分けたいならKidsを後から追加します。

### KidsとIoTは最初から作るべき？

必要になってからで大丈夫です。

最初から増やしすぎると、家族に説明しにくくなります。

MainとゲストWi-Fiを安定させてから追加するのがおすすめです。

### ゲストWi-FiはSSIDを分けるだけでいい？

十分ではありません。

SSIDだけでなく、guest network、DHCP、firewall zoneを分け、guest → wanだけ許可し、guest → lanを拒否します。

### KidsにはAdblockとFamily DNSのどちらがいい？

最初はFamily DNSが分かりやすいです。

広告やトラッキングを減らしたい場合はAdblock、成人向けコンテンツやマルウェア系カテゴリを減らしたい場合はFamily DNSが向いています。

端末側のファミリー機能も併用してください。

### IoT用SSIDは2.4GHzで作るべき？

多くの場合、最初は2.4GHzで作るとつながりやすいです。

古いスマート家電やカメラは、5GHzや6GHzに対応していないことがあります。

### プリンターはIoTに入れるべき？

家庭ではMainに置くほうが楽なことが多いです。

IoTに分けると、PCやスマートフォンから見えにくくなる場合があります。

まずMainに置いてDHCP予約するのがおすすめです。

### Adblockは家族全員に入れていい？

軽めなら便利ですが、誤ブロックが起きることがあります。

ログイン、決済、動画、学校用サービス、スマート家電アプリが壊れたら、一時停止して切り分けます。

### 家庭でもVLANは必要？

最初は不要です。

SSID、network、firewall zoneで分けるだけでも十分なことが多いです。

有線ポートや管理スイッチまで分けたくなったら、VLAN記事へ進むとよいです。

### 6GHzやMLOは最初から使うべき？

必須ではありません。

まず5GHzをMainの主力にし、IoTは2.4GHz、対応端末がある場合だけ6GHzを使うくらいで十分です。

MLOはWi-Fi 7対応端末が増えてから試すくらいで大丈夫です。

### ゲストWi-FiにAdblockを入れるべき？

入れても構いませんが、最初は軽めで十分です。

ゲストWi-Fiで一番大事なのは、広告を消すことより、家庭内LANへ入れないことです。

### IPv6もGuest / Kids / IoTで分ける必要がありますか？

IPv6を各ネットワークへ配る場合は、IPv6側でもfirewall方針を確認してください。

IPv4では分離できていても、IPv6側の設計を見落とすことがあります。

迷う場合は、まずIPv4で構成を安定させ、その後でIPv6を確認します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- Cloudflare 1.1.1.1 for Families: https://developers.cloudflare.com/1.1.1.1/setup/
- Cloudflare DNS resolver IP addresses: https://developers.cloudflare.com/1.1.1.1/ip-addresses/
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
