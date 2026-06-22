<!-- mirror-source: articles/026-hardware-overview.md -->

# 箱から出したらまずここ｜LN6001-JPのポート・LED・ボタンをざっくり見る【OpenWrt集中連載026】

LN6001-JPを箱から出した。

さて、最初にどこを見るか。

LuCI？  
SSH？  
VLAN？  
IPoE？  
VPN？  
MLO？

気持ちは分かります。

OpenWrtベースのWi-Fi 7ルーターなので、いろいろ触りたくなります。

でも、最初から管理画面やCLIへ突撃しなくて大丈夫です。

まず見るのは、もっと物理です。

```txt
WANポート
LANポート
LED
RESETボタン
底面ラベル
```

このあたりです。

地味に見えますが、ここを押さえるだけでかなり楽になります。

赤LEDが出た時に、まずWANケーブルを見られる。  
LuCIへ入れない時に、有線LANで試せる。  
IPoE設定の前に、ONU / HGW / WANポートのつなぎ方を確認できる。  
リセットボタンを押す前に、「まだ戻れるかも」と踏みとどまれる。

OpenWrtは画面やコマンドも大事です。

でも、最初のトラブルはだいたい物理です。

ケーブル。  
電源。  
WANとLANの挿し間違い。  
初期ラベルの見落とし。  
起動中なのに焦ってリセット。

このへんです。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）の本体まわり、ポート、LED、ボタン、初期設定前に見ておきたいポイントを、家庭・小さなオフィス・店舗向けに整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、ハードウェア仕様を全部暗記することではありません。

まずは、ここまで分かればOKです。

- WAN / InternetポートとLANポートの役割が分かる
- LEDの白点灯・赤点灯・青点滅をざっくり読める
- 初期設定前に底面ラベルと付属品を確認できる
- 2.5GbE WANと1GbE LANの違いを理解できる
- 有線LANでLuCIへ入る意味が分かる
- LuCI / SSHで本体状態を安全に読み取り確認できる
- RESETボタンを押す前に確認すべきことが分かる
- サポート相談時に控える情報が分かる

最初は、これくらいで十分です。

```txt
回線側はWAN
端末側はLAN
赤LEDはWAN側から見る
リセットは最後
```

この4つだけでも、かなり迷子になりにくくなります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-01.png)

## 先にざっくり結論

箱から出したら、まず見るのはこの5つです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-01.png)

初期設定で多いミスは、難しいOpenWrt設定ではありません。

**WANとLANを逆に挿すこと**です。

まず配線を見ます。

```txt
ONU / HGW / モデム
  ↓
LN6001-JPのWAN / Internetポート

PC / スイッチ / NAS
  ↓
LN6001-JPのLANポート
```

LEDは、最初はこれで十分です。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-02.png)

赤いから即故障。  
青点滅だから即リセット。  
消灯だから即故障。

そう決めつけないほうが安全です。

まず物理を見ます。

## こういう人向けです

この記事は、次のような人向けです。

- LN6001-JPを箱から出したばかり
- どのポートへ何を挿せばよいか確認したい
- LEDが赤い時に、まず何を見るか知りたい
- WANポートとLANポートの違いを整理したい
- 2.5GbE WANと1GbE LANの違いを知りたい
- リセットボタンを押す前の注意点を知りたい
- 6GHzやMLOへ進む前に、まず本体まわりを見たい
- サポートや詳しい人へ相談する時の情報を控えたい

逆に、すでに機器構成やポートマップを把握している人には基本寄りです。

ただ、トラブル時ほど基本が効きます。

赤LEDの時に、いきなりIPoE設定を疑うより、まずWANケーブルを見たほうが早いことがあります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-02.png)

## 最初に言葉だけそろえる

ハードウェアまわりで出てくる言葉を、ざっくり整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-03.png)

最初は、次だけ覚えれば十分です。

```txt
WAN = 回線側
LAN = 家や事務所の内側
LuCI = 管理画面
RESET = 最後の手段
```

## 本体で最初に見る場所

![本体まわりの確認ポイント](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/hardware-map.png)

LN6001-JPで最初に見る場所は、次の5つです。

1. 底面ラベル
2. WAN / Internetポート
3. LAN 1〜4ポート
4. 電源ポートと電源スイッチ
5. ステータスLEDとRESETボタン

底面ラベルには、初期SSID、初期Wi-Fiパスワード、管理画面ログインに使う初期パスワードなど、初期設定に必要な情報が記載されています。

このラベルはかなり大事です。

ただし、写真をSNSや記事へ出さないでください。

初期Wi-Fiパスワードや管理用情報が含まれます。

```txt
底面ラベル = 初期設定の鍵
```

くらいに考えてください。

## 主要スペック

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-04.png)

最初に覚えるべきなのは、細かなチップ名や速度表ではありません。

まずはこれです。

```txt
WANは2.5GbE
LANは1GbE × 4
Wi-Fiは2.4GHz / 5GHz / 6GHz
管理画面は https://192.168.1.1
```

ここが分かっていれば、初期設定やトラブル時の見通しがかなり良くなります。

## 付属品

パッケージには、主に次が含まれます。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-05.png)

最初の動作確認では、できれば付属の電源アダプターとイーサネットケーブルを使います。

ケーブルや電源を別のものに変えると、切り分けが少し難しくなります。

```txt
最初は付属品で確認
動いてから必要に応じて差し替える
```

この順番が安全です。

## 背面ポートの見方

背面には、主に次のポートがあります。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-06.png)

有線LANの基本は、次です。

```txt
回線側機器 → WAN / Internet
PCやスイッチ → LAN 1〜4
```

最初の設定時は、PCをLANポートへ有線接続してLuCIを開くのがおすすめです。

Wi-Fi経由でも設定できます。

ただ、Wi-Fi名やWi-Fiパスワードを変更すると、作業中の無線接続が一度切れることがあります。

有線なら、かなり落ち着いて作業できます。

## WANポートの役割

WAN / Internetポートは、回線側の入口です。

接続例です。

```txt
ONU → LN6001-JP WAN
HGW → LN6001-JP WAN
既存ルーター → LN6001-JP WAN
```

NTTフレッツ系でひかり電話HGWがある場合は、HGWがIPoEを担当し、LN6001-JPはHGW配下でDHCP自動接続することがあります。

その場合でも、LN6001-JP側の接続先はWAN / Internetポートです。

ただし、HGWとLN6001-JPのLAN IPがどちらも `192.168.1.1` の場合は、IPアドレス重複に注意します。

例:

```txt
HGW:        192.168.1.1
LN6001-JP: 192.168.1.1
```

この状態だと、管理画面へ入れなかったり、通信が不安定に見えたりします。

必要に応じて、LN6001-JP側のLAN IPを `192.168.10.1` などへ変更します。

## LANポートの役割

LAN 1〜4は、家庭内・社内側の有線機器を接続するポートです。

接続例です。

```txt
PC → LAN 1
NAS → LAN 2
スイッチ → LAN 3
プリンター → LAN 4
```

初期設定や復旧作業では、管理用PCをLANポートへ直接つなぐと安全です。

特に次を触る時は、有線管理PCがあるとかなり安心です。

- LAN IP
- VLAN
- firewall
- ゲストWi-Fi
- DHCP
- Wi-Fi設定
- VPN
- ファームウェア更新

ゲストWi-FiやVLAN、firewallを触っていても、有線LANで管理画面へ戻れる状態を残しておくと復旧しやすくなります。

```txt
困ったら有線LAN
```

これは本当に強いです。

## 2.5GbE WANと1GbE LANの見方

LN6001-JPは、WAN側に2.5GbEポートを持っています。

一方で、LAN側は1GbEポートが4つです。

つまり、有線構成だけで見ると次のようになります。

```txt
WAN側: 最大2.5Gbpsリンク
LAN側: 各ポート最大1Gbpsリンク
```

これは、次のように理解すると分かりやすいです。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-07.png)

有線NASを2.5GbEで使いたい場合、LN6001-JPのLANポートだけでは2.5GbE接続にはなりません。

この製品は、

```txt
2.5GbE WAN + Wi-Fi 7
```

を活かす構成で考えると分かりやすいです。

## ポート状態をSSHで確認する

SSHでインターフェース状態を見る場合は、まず読み取りだけ行います。

```sh
echo "### interfaces"
ip link show

echo "### IPv4/IPv6 addresses"
ip addr show

echo "### network config hints"
uci show network | grep -E 'device|bridge|ports|vlan|ifname|proto|ipaddr'
```

内部インターフェース名は、背面ラベルの `WAN`、`LAN 1` と完全に同じ名前とは限りません。

最初は、LuCIの **Network → Interfaces** とSSHの出力を照らし合わせて読みます。

ここでいきなり設定を書き換える必要はありません。

まず読むだけで十分です。

## リンク速度を確認する

環境によっては、`/sys/class/net/*/speed` でリンク速度を読めます。

```sh
for dev in /sys/class/net/*; do
  name="$(basename "$dev")"
  [ -r "$dev/speed" ] || continue
  printf "%s speed: " "$name"
  cat "$dev/speed" 2>/dev/null || true
done
```

`2500` と出れば2.5Gbps、`1000` と出れば1Gbpsリンクです。

ただし、仮想インターフェース、bridge、無線インターフェースでは表示されないこともあります。

出ないから異常、とは限りません。

リンク速度を見る時は、次も合わせて確認します。

- 接続先機器が2.5GbE対応か
- LANケーブルが問題ないか
- 上流ONU / HGWのポート仕様
- スイッチのポート仕様
- 対象が物理ポートかbridgeか

## LEDの見方

LN6001-JPは、本体上部にステータスLEDがあります。

最初は、次だけ覚えれば十分です。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-08.png)

LEDを見るだけで、ざっくり次の切り分けができます。

```txt
白点灯:
  まずオンライン状態。端末側やDNS、Wi-Fi設定を確認する。

赤点灯:
  WANケーブル、ONU/HGW、回線設定、IPoE/PPPoEを確認する。

青点滅:
  起動中。すぐ操作せず、しばらく待つ。

消灯:
  電源、電源スイッチ、ACアダプター、コンセントを確認する。
```

LEDは、細かな原因までは教えてくれません。

でも、「まずどこを見るべきか」の方向は教えてくれます。

## LEDが赤い時に見ること

赤点灯が続く場合、すぐ故障と判断しないでください。

まず次を見ます。

1. WANケーブルがWAN / Internetポートに挿さっているか
2. ONU / HGWの電源が入っているか
3. ONU / HGWのLANポートが生きているか
4. 上流機器を再起動した直後ではないか
5. WANがDHCP clientになっているか
6. IPoE / PPPoE設定が必要な回線ではないか
7. HGWとLN6001-JPのLAN IPが重複していないか

SSHで見られる場合は、次を確認します。

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### WAN logs"
logread | grep -Ei 'wan|wan6|netifd|dhcp|dhcpv6|odhcp6c|ppp|pppoe|ipoe|map|dslite|ipip' | tail -n 120
```

赤点灯は、WAN側の問題であることが多いです。

Wi-Fi名や暗号化方式を触る前に、まず回線側を見ます。

## LEDが青点滅の時

青点滅は、起動中や再起動中の状態です。

ファームウェア更新後、リセット後、電源投入直後は、しばらく青点滅することがあります。

この時にやらないほうがよいことです。

- すぐ電源を抜く
- 何度もリセットボタンを押す
- WANケーブルを抜き差しし続ける
- スマートフォンのWi-Fi一覧だけで失敗判断する
- ブラウザを何度も更新して焦る

まずは数分待ちます。

長時間戻らない場合は、有線LANでPCをつなぎ、LuCIへ入れるか確認します。

ここで焦ってリセットボタンを押したくなります。

分かります。

でも、まず待ちます。

## LEDが消灯している時

LEDが消えている場合は、設定より先に電源です。

確認するものです。

- 付属ACアダプターが挿さっているか
- 電源スイッチがオンか
- 電源タップのスイッチが入っているか
- コンセントが生きているか
- ACアダプターが正しいものか
- 本体が過度に熱くなっていないか

設定ミスでLEDが完全に消えることは、通常あまり考えにくいです。

まず物理電源を見ます。

## リセットボタンの見方

RESETボタンは、本体底面にあります。

操作はシンプルです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-09.png)

リセットを開始すると、LEDが赤から青点滅へ変わり、再起動します。

再起動後、インターネット接続を検知すると白点灯になります。

赤点灯のままなら、WAN配線や回線設定を確認します。

## リセット前に確認すること

リセットは最後の手段です。

実行前に、次を確認します。

```txt
□ 有線LANでPCをつないだ
□ 192.168.1.1以外のLAN IPへ変更していないか確認した
□ PC側のDefault Gatewayを確認した
□ LuCIへ入れるか確認した
□ SSHへ入れるか確認した
□ 直前に触った設定を思い出した
□ LuCIバックアップがあるか確認した
□ 初期化後に必要なIPoE / PPPoE / VPN情報があるか確認した
```

設定ミスの多くは、初期化しなくても戻せます。

たとえば、firewallだけ戻す。  
wirelessだけ戻す。  
networkだけ戻す。  
dhcpだけ戻す。

こういう方法があります。

リセットと復旧は、022の記事で詳しく扱っています。

## 電源スイッチと電源ポート

LN6001-JPには、電源ポートとスライド式電源スイッチがあります。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-10.png)

LEDが完全に消灯している時は、設定より先に電源を確認します。

トラブル時に意外と多いのが、電源タップ側のスイッチが切れているケースです。

小さなオフィスや店舗では、清掃中や配線整理中に電源が抜けることもあります。

## 予備ボタンについて

公式仕様では、ボタンとスイッチとして次が案内されています。

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-11.png)

予備ボタンは、通常運用で使うものとして考えなくて大丈夫です。

初期設定やトラブル時に見るべきなのは、RESETボタンと電源スイッチです。

## 初期ログイン情報

初期状態では、次のようにアクセスします。

```txt
URL: https://192.168.1.1
ユーザー名: root
パスワード: 本体底面ラベルのデフォルトパスワード
```

初期SSIDとWi-Fiパスワードも、本体底面ラベルに記載されています。

初回ログイン後は、管理者パスワードを変更します。

パスワード変更後は、パスワードマネージャーなどへ安全に保存してください。

底面ラベルの写真や管理画面のスクリーンショットを公開しないように注意します。

## 初期配線パターン

## ONU直下で使う

```txt
ONU
  ↓
LN6001-JP WAN
  ↓
LAN / Wi-Fi端末
```

ひかり電話なしのNTTフレッツ系IPoE環境では、LN6001-JP側でオートIPoE / Internet Assistantが必要になる場合があります。

詳細は014の記事で扱っています。

## HGW配下で使う

```txt
ONU
  ↓
HGW
  ↓
LN6001-JP WAN
  ↓
LAN / Wi-Fi端末
```

ひかり電話HGWがある場合、HGWがIPoEを担当し、LN6001-JPはDHCP自動で使えることがあります。

ただし、HGWとLN6001-JPのLAN IPが重複しないよう注意します。

## 既存ルーター配下で使う

```txt
既存ルーター
  ↓
LN6001-JP WAN
```

この場合、二重ルーター構成になります。

そのままでも動くことがあります。

ただし、ポート開放、VPN、IPoE、ゲストWi-Fi分離で挙動が分かりにくくなることがあります。

既存ルーターを残すか、LN6001-JPをメインルーターにするか、AP / ブリッジとして使うかを整理します。

## Wi-Fi 7と3つの帯域

LN6001-JPは、2.4GHz、5GHz、6GHzのトライバンド構成です。

最初は次の理解で十分です。

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-12.png)

Wi-Fi 7だからといって、すべての端末が6GHzへつながるわけではありません。

古い端末やIoT機器では、2.4GHzしか使えないものもあります。

最初は、次のような分け方が扱いやすいです。

```txt
MyHome       → 5GHz中心のメインSSID
MyHome_IoT   → 2.4GHz中心のIoT用SSID
MyHome_6G    → 6GHz対応端末用SSID
```

MLOや6GHz活用は、006のWi-Fi設定記事で詳しく扱います。

## radio名の見方

Linksys公式手順では、Wi-Fi radioは次のように扱われます。

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-13.png)

SSHで確認する場合は、次を使います。

```sh
echo "### wireless devices"
uci show wireless | grep '=wifi-device'

echo "### SSID and radio mapping"
uci show wireless | grep -E 'wifi-iface|device|ssid|network|encryption|disabled'

echo "### wifi status"
wifi status
```

実際の設定名は、ファームウェアや構成で変わる可能性があります。

CLIで変更する前に、LuCIの **Network → Wireless** と照らし合わせて確認してください。

## 本体情報をSSHで確認する

サポート相談やトラブル切り分けでは、本体情報を控えておくと便利です。

SSHで読み取り確認するなら、次のコマンドで十分です。

```sh
echo "### date"
date

echo "### system board"
ubus call system board

echo "### release"
cat /etc/openwrt_release 2>/dev/null || true

echo "### kernel"
uname -a

echo "### uptime"
uptime

echo "### memory"
free

echo "### storage"
df -h
```

この出力で、ファームウェア、カーネル、稼働時間、メモリ、ストレージを確認できます。

出力にはホスト名や内部情報が含まれることがあるため、公開の場所へそのまま貼らないでください。

## WAN / WAN6を確認する

ハードウェア記事でも、WAN状態だけは見られるようにしておくと便利です。

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### recent WAN logs"
logread | grep -Ei 'wan|wan6|netifd|dhcp|dhcpv6|odhcp6c|ppp|pppoe|ipoe|map|dslite|ipip' | tail -n 100
```

LEDが赤い時、まずこのあたりを見ます。

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-14.png)

## Wi-Fi端末を確認する

Wi-Fiに接続している端末は、LuCIの **Network → Wireless → Associated Stations** で見られます。

SSHで見るなら次です。

```sh
echo "### associated stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done
```

端末がここに出ていれば、少なくともWi-Fi接続まではできています。

その上でインターネットへ出られない場合は、DHCP、DNS、WAN、firewallを見ます。

## DHCPリースを確認する

端末がIPアドレスを取れているか確認します。

```sh
echo "### active DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP summary"
uci show dhcp | grep -E 'interface|start|limit|leasetime|@host|\\.mac|\\.ip|\\.name'
```

見るポイントです。

- PCやスマートフォンがリース一覧にいるか
- ゲストWi-Fi端末がguest側IPを取っているか
- NASやプリンターがDHCP予約どおりのIPを取っているか
- `169.254.x.x` のような自己割り当てになっていないか

DHCPが見えると、端末側なのかルーター側なのかを分けやすくなります。

## 設置場所の考え方

Wi-Fiルーターは、置き場所でもかなり変わります。

おすすめは次です。

- 床に直置きしない
- 金属棚や電子レンジの近くを避ける
- 壁や家具に囲まれすぎない場所に置く
- できれば家やオフィスの中心寄りに置く
- 通気をふさがない
- 電源アダプターに無理な力をかけない
- WANケーブルが抜けにくい場所に置く
- 店舗ではスタッフや来客が触りにくい場所へ置く

6GHzは近距離高速に向きますが、壁や床を越える用途では5GHzや2.4GHzのほうが安定することがあります。

「数字が大きい帯域ほど、遠くまで強い」わけではありません。

ここ、地味に大事です。

## 小さなオフィス・店舗での見方

小さなオフィスや店舗では、最初に本体まわりを次のように整理します。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-15.png)

店舗では、POSや決済端末を安易にゲストWi-FiやIoT用SSIDへ混ぜないようにします。

ベンダー指定のネットワーク要件がある場合は、それを優先してください。

## トラブル時の最初の切り分け

ハードウェアまわりで見る順番です。

```txt
LED
  ↓
電源
  ↓
WANケーブル
  ↓
ONU / HGW
  ↓
LANケーブル
  ↓
有線PCでLuCI
  ↓
WAN / WAN6状態
  ↓
Wi-Fi / DHCP / DNS
```

症状別に見ると、次です。

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/table-16.png)

より詳しい切り分けは、021の「つながらない時の切り分け」で扱っています。

## サポート相談時に控える情報

サポートや詳しい人へ相談する時は、次を控えておくと話が早くなります。

```txt
製品:
  Linksys Velop WRT Pro 7 / LN6001-JP

ファームウェア:
  1.2.0.15

LED状態:
  白点灯 / 赤点灯 / 青点滅 / 消灯

接続構成:
  ONU -> LN6001-JP
  ONU -> HGW -> LN6001-JP
  既存ルーター -> LN6001-JP

WAN:
  ifstatus wan の状態

WAN6:
  ifstatus wan6 の状態

LuCI:
  https://192.168.1.1 へ入れる / 入れない

有線LAN:
  PCをLANポートへ接続して確認済み / 未確認

直前の変更:
  なし / Wi-Fi設定 / IPoE / firewall / VLAN / VPN / Adblock
```

SSHでまとめて確認するなら次です。

```sh
echo "### quick hardware/support info"
date
ubus call system board
uptime
df -h
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
logread | tail -n 80
```

共有する時は、SSID、Wi-Fiパスワード、MACアドレス、IPv6 prefix、PPPoE ID、VPN情報などを伏せてください。

## 初回チェックリスト

箱から出した直後は、これだけ見ればOKです。

```txt
□ 付属品を確認した
□ 底面ラベルを確認した
□ ラベル写真を公開しないと決めた
□ WAN / Internetポートの位置を確認した
□ LAN 1〜4の位置を確認した
□ 電源アダプターを確認した
□ 電源スイッチをオンにした
□ LED状態を確認した
□ ONU / HGWからWANへ接続した
□ 管理PCをLANへ有線接続した
□ https://192.168.1.1 へアクセスした
□ rootでログインした
□ 管理者パスワードを変更した
□ 初期バックアップを取った
```

ここまでできれば、ハードウェアまわりの初期確認としては十分です。

細かなOpenWrt運用は、このあとで大丈夫です。

## まとめ

LN6001-JPを初めて触る時に、細かな仕様を全部覚える必要はありません。

まず見るのは、これだけです。

```txt
底面ラベル
WAN / Internetポート
LAN 1〜4
ステータスLED
RESETボタン
```

WAN / Internetポートは回線機器へ。  
LAN 1〜4はPC、スイッチ、NASなど内側の機器へ。  
LEDは、白点灯、赤点灯、青点滅で大まかな状態を見ます。  
RESETボタンは最後の手段です。

LN6001-JPには、2.5GbE WAN、1GbE LAN、2.4GHz / 5GHz / 6GHzのトライバンド、LuCI、SSH、opkgなど、多くの機能があります。

でも最初は、次で十分です。

```txt
どこへ何をつなぐか
LEDが何色か
LuCIへ入れるか
```

本体まわりを理解しておくと、IPoE、Wi-Fi、ゲストWi-Fi、VLAN、VPNの記事を読む時にも迷いにくくなります。

ネットワークの第一歩は、だいたいケーブルです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-03.png)

## 次に読むなら

ハードウェアまわりを確認したら、次は目的に合わせて進みます。

- [初回セットアップ](https://note.com/ikmsan/n/n2e9666e3a1a5)
- [LuCIとSSHの基本](https://note.com/ikmsan/n/n3e9b348d4c95)
- [Wi-Fi設定](https://note.com/ikmsan/n/n4dbf8f72744c)
- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)

これから設置する人は初回セットアップへ。

LEDが赤い、WANが不安定、IPoE環境でつまずいている人は、IPoE記事と切り分け記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

内部インターフェース名、Wi-Fi radio名、LuCIの画面名、`ifstatus`、`wifi status`、`logread` の出力は、ファームウェア更新や設定状態によって変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートや最新ドキュメントも確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPのWANポートとLANポートはどう見分ければいい？

WAN / Internetポートは、ONU、HGW、モデムなど回線側へつなぐ入口です。

LAN 1〜4は、PC、スイッチ、NAS、プリンターなど家庭内・社内側の機器をつなぐポートです。

最初は「回線側はWAN、端末側はLAN」と覚えれば十分です。

### LEDが赤点灯の時は故障？

すぐ故障とは限りません。

赤点灯は、モデム側からインターネット接続がない状態を示す場合があります。

まずWANケーブル、ONU / HGW、回線設定、IPoE / PPPoE設定を確認してください。

### LEDが青点滅のままです。どうすればいい？

青点滅は起動中や再起動中の状態です。

ファームウェア更新後やリセット後は、しばらく待ちます。

長時間戻らない場合は、有線LANでPCをつなぎ、LuCIへ入れるか確認してください。

### RESETボタンはいつ使う？

LuCIやSSHで戻せない時の最後の手段です。

押す前に、有線LANでLuCIへ入れるか、SSHへ入れるか、バックアップがあるかを確認してください。

### 初期ログイン情報はどこにありますか？

本体底面ラベルに、初期SSID、Wi-Fiパスワード、管理用パスワードなどが記載されています。

LuCIは `https://192.168.1.1`、ユーザー名は `root` です。

### 2.5GbEはLANでも使えますか？

LN6001-JPの2.5GbEはWAN / Internetポートです。

LANポートは1GbE × 4です。

有線NASなどを2.5GbEでLAN接続したい場合は、この点に注意してください。

### 6GHzのSSIDが見えません

端末がWi-Fi 6EまたはWi-Fi 7に対応している必要があります。

古いスマートフォンやPCでは、6GHzのSSIDは見えません。

まず対応端末かどうか、ルーターの6GHz radioが有効かを確認してください。

### 本体底面ラベルを写真で保存してもいい？

個人用の安全な保管なら便利です。

ただし、SNS、記事、チャット、公開リポジトリへ出さないでください。

初期SSID、Wi-Fiパスワード、管理情報が含まれるためです。

### 店舗で置く時に注意することは？

RESETボタンや電源スイッチを来客やスタッフが誤って触らない場所へ置くのがおすすめです。

また、POSや決済端末を接続する場合は、ベンダー指定のネットワーク要件を優先してください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 light behavior: https://support.linksys.com/kb/article/216-en/?section_id=175
- Linksys MBE70 reset: https://support.linksys.com/kb/article/220-en/?section_id=175
- Linksys Velop WRT Pro 7 初期化方法: https://support.linksys.com/kb/article/7031-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へログインする方法: https://support.linksys.com/kb/article/7038-jp/
- 株式会社アスク Velop WRT Pro 7 製品ページ: https://www.ask-corp.jp/products/linksys/router/velop-wrt-pro-7.html

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
