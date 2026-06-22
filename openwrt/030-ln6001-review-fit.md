<!-- mirror-source: articles/030-ln6001-review-fit.md -->

# LN6001-JPは誰に刺さる？｜Wi-Fi 7だけじゃない“触れるルーター”としての購入前レビュー【OpenWrt集中連載030】

Wi-Fi 7ルーターが欲しい。

でも、ただ速いだけだと、ちょっと物足りない。

ゲストWi-Fiをちゃんと分けたい。  
IoT機器も整理したい。  
NASやミニPCへ外から入りたい。  
TailscaleやWireGuardも気になる。  
IPoEの状態も見たい。  
Adblockも試したい。  
SSHでログも見たい。  
VLANも、いつか触ってみたい。

こういう人には、Linksys Velop WRT Pro 7（LN6001-JP）はかなり面白いルーターです。

一方で、こういう人には少し濃いかもしれません。

```txt
アプリで全部おまかせしたい
設定画面は見たくない
ログもCLIも興味ない
Wi-Fiが出ればそれでOK
```

LN6001-JPは、普通の家庭用Wi-Fiルーターと、自分でOpenWrtを入れて作るルーターの、ちょうど中間にいるような製品です。

買ってすぐ使い始められる。  
でも、中もちゃんと見られる。  
LuCIがある。  
SSHがある。  
opkgも使える。  
VPNやIPoE向けの公式導線もある。  
ゲストWi-Fi、VLAN、firewall zone、DHCP予約、DNS制御まで触れる。

つまり、これは「置いて終わり」のルーターというより、

```txt
自分のネットワークを少しずつ育てたい人向けのWi-Fi 7ルーター
```

です。

この記事では、LN6001-JPをレビュー的に見ながら、どんな人に向いているか、逆にどんな人には向きにくいか、買う前に何を確認すべきかを整理します。

スペック表だけでは分かりにくい“相性”の話をします。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、スペック表をなぞることではありません。

買う前に、

```txt
これは自分に合うルーターなのか？
```

を判断できるようにすることです。

この記事では、次を整理します。

- LN6001-JPが向いている人・向いていない人
- 一般的なWi-Fiルーターとの違い
- 自作OpenWrtルーターとの違い
- Wi-Fi 7 / 6GHz / MLOをどう見ればよいか
- 2.5GbE WANと1GbE LANの注意点
- IPoE、VPN、VLAN、Adblockとの相性
- 家庭・小さなオフィス・店舗での使いどころ
- 買う前に確認したいこと
- 買ったあと最初にやること

先に言うと、LN6001-JPは「全員におすすめ」タイプではありません。

でも、刺さる人にはかなり刺さります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/diagram-01.png)

## 先にざっくり結論

LN6001-JPは、次のような人に向いています。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-01.png)

逆に、次のような人には少し向きにくいです。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-02.png)

ひとことで言うなら、LN6001-JPはこれです。

```txt
Wi-Fi 7の速さだけでなく、
自宅や小さな拠点のネットワークを自分で整えたい人向け
```

ただ速いだけのルーターではありません。

“触れる”ルーターです。

## このレビューの前提

この記事では、次の前提で見ています。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-03.png)

このレビューでは、ベンチマークで何Gbps出るかだけを見ません。

もちろん速度は大事です。

でも、この連載で見たいのはそこだけではありません。

```txt
普通に使い始められるか
必要な時に深く触れるか
日本のIPoE環境で現実的に使えるか
ゲストWi-FiやVPNを作りやすいか
壊した時に戻せるか
家庭や小規模拠点で説明しやすいか
```

このあたりを重視します。

LN6001-JPは、スペック表だけ見ても魅力が伝わりにくいタイプです。

使い込むほど、「あ、ここ見られるの助かる」が増えるルーターです。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/diagram-02.png)

## LN6001-JPの主な特徴

ざっくり特徴をまとめると、こうです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-04.png)

スペックだけ見れば、Wi-Fi 7トライバンドルーターです。

でも、LN6001-JPの面白さは、そこにOpenWrtベースの運用性が乗っているところです。

Wi-Fi 7の速さ。  
OpenWrtの自由度。  
日本向け製品としての始めやすさ。

この3つが同居しているのが特徴です。

## 一般的なWi-Fiルーターとの違い

一般的な家庭用Wi-Fiルーターは、かんたん設定を重視します。

これはこれで正しいです。

アプリで設定できる。  
SSIDとパスワードを決めれば終わる。  
ゲストWi-Fiもワンタップ。  
細かい項目は見えない。  
だから迷いにくい。

こういう製品が合う人もたくさんいます。

一方で、LN6001-JPは少し違います。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-05.png)

LN6001-JPは、全部を隠してくれるルーターではありません。

必要な時に中を見られるルーターです。

ここを魅力に感じるか、面倒に感じるか。

購入前の分かれ道は、かなりここです。

## 自分でOpenWrtを入れるルーターとの違い

OpenWrtを使う方法には、自分で対応機種を選んで、ファームウェアを書き込む方法もあります。

これはこれで楽しいです。

いわゆる“沼”です。

ただし、初めての人にはハードルがあります。

- 対応機種を調べる
- ファームウェアを選ぶ
- 書き込み手順を確認する
- 失敗時の復旧手段を用意する
- Wi-Fiドライバやハードウェア対応を確認する
- 日本国内の電波法・技適を確認する
- 初期状態から設定を作り込む

LN6001-JPは、このあたりを製品としてかなり吸収しています。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-06.png)

つまりLN6001-JPは、OpenWrtの入口としてかなり現実的です。

「ファームウェアを自分で焼くところから楽しい」という人には、少し物足りないかもしれません。

でも、

```txt
OpenWrtっぽい運用を、
ちゃんと製品として始めたい
```

という人にはかなりちょうどいいです。

## Wi-Fi 7ルーターとして見る

LN6001-JPは、2.4GHz / 5GHz / 6GHzのトライバンド構成です。

Wi-Fi 7対応ルーターなので、6GHzやMLOも気になります。

ただし、ここで大事なのは端末側です。

ルーターだけWi-Fi 7でも、スマホやPCが対応していなければ、Wi-Fi 7の機能は使い切れません。

## Wi-Fi 7の恩恵を受けやすい人

次のような人は、Wi-Fi 7ルーターとしての価値を感じやすいです。

- Wi-Fi 7対応スマートフォンやPCを持っている
- Wi-Fi 6E対応端末がある
- 6GHzを使える端末がある
- 近距離で高速通信したい
- 大容量ファイル転送やNASアクセスをよく使う
- 新しい端末へ買い替える予定がある
- 5GHzが混雑していて、6GHzを使いたい

## 恩恵が限定的な人

逆に、次のような人は、Wi-Fi 7の恩恵は限定的かもしれません。

- 端末がWi-Fi 5 / Wi-Fi 6中心
- 6GHz対応端末がない
- 速度よりカバー範囲を重視している
- 2.4GHz IoT機器が中心
- そもそも回線速度がそれほど速くない
- 家が広く、1台だけではカバーしきれない

ただし、Wi-Fi 7対応端末がまだ少なくても、今後の端末更新を見越して選ぶのはありです。

今は5GHz中心の高性能ルーターとして使い、対応端末が増えたら6GHzやMLOを活かす。

この見方もかなり現実的です。

## 2.4GHz / 5GHz / 6GHzの見方

家庭や小規模店舗では、帯域をこう見ると分かりやすいです。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-07.png)

最初からMLOを使い倒す必要はありません。

まず5GHzを安定させる。  
IoTは2.4GHzへ置く。  
対応端末があれば6GHzを試す。

このくらいの順番で十分です。

## 2.5GbE WANと1GbE LANの注意点

ここは購入前にかなり大事です。

LN6001-JPは、WAN側に2.5Gbpsポートがあります。

一方で、LANポートは1Gbpsが4つです。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-08.png)

つまり、LN6001-JPは、

```txt
2.5GbE WAN + Wi-Fi 7
```

を活かすルーターです。

有線LAN側も全部2.5GbEで固めたい人は、ここを必ず確認してください。

「Wi-Fiは速いけど、有線NASは1GbEで頭打ち」という構成になる場合があります。

ここを理解して買えば問題ありません。

理解せずに買うと、あとで「あれ？」となります。

## OpenWrtベースであるメリット

LN6001-JPの魅力は、Wi-Fi 7だけではありません。

OpenWrtベースであることがかなり大きいです。

## LuCIで細かく見られる

ブラウザからLuCIへ入り、次のような設定を確認できます。

- Network
- Wireless
- Interfaces
- DHCP and DNS
- Firewall
- Port Forwards
- Static Leases
- Software
- System Log
- Backup / Flash Firmware

一般的なルーターでは隠れている部分が、かなり見えます。

```txt
今どの端末がIPを取っているか
WAN6は生きているか
guest zoneはどうなっているか
Adblockの状態はどうか
```

こういう確認ができます。

## SSHで状態確認できる

SSHへ入ると、状態確認がしやすくなります。

最初は読み取りだけで十分です。

```sh
ubus call system board
ifstatus wan
ifstatus wan6
cat /tmp/dhcp.leases
logread | tail -n 80
wifi status
uci show firewall
```

これにより、「なんかつながらない」から一歩進めます。

```txt
WANが落ちているのか
DNSだけおかしいのか
DHCPでIPを取れていないのか
Wi-Fiにはつながっているのか
firewallで止めているのか
```

を分けて見られます。

これは本当に大きいです。

## opkgで拡張できる

必要に応じて、パッケージを追加できます。

例:

- `tcpdump`
- `curl`
- `adblock`
- `luci-app-adblock`
- `luci-app-ddns`
- banIP系
- 診断ツール

ただし、何でも入れるのはおすすめしません。

`opkg` は楽しいですが、入れすぎると管理が大変になります。

特に、`opkg upgrade` による一括更新は基本的に避けます。

ファームウェア更新と必要パッケージの再導入で管理するほうが安全です。

## VLAN / firewall zoneが使える

家庭でも小さな店舗でも、次のような分離を作れます。

```txt
Main / Staff
Guest
Kids
IoT / Device
VPN
```

SSID、network、DHCP、firewall zoneを組み合わせることで、ゲストWi-FiをLANから分離したり、IoT機器をMainから分けたりできます。

このあたりは、一般的な家庭用ルーターでは見えにくい部分です。

LN6001-JPでは、ちゃんと設計できます。

## VPNへ進める

Linksys公式のVPN Assistantを使うことで、WireGuard / Tailscaleの導線があります。

外出先からNAS、LuCI、ミニPC、店舗の録画機へ入りたい場合、ポート開放よりVPN経由にしやすいのは大きな利点です。

特にTailscaleは、固定IPなし・ポート開放なしでも始めやすいので、家庭や小規模店舗と相性が良いです。

## IPoE / IPv4 over IPv6を確認しやすい

日本の光回線では、IPoE / IPv4 over IPv6がややこしいです。

OCNバーチャルコネクト。  
transix。  
v6プラス。  
クロスパス。  
MAP-E。  
DS-Lite。  
IPIP。

名前だけで疲れます。

LN6001-JPでは、公式のオートIPoE / Internet Assistantの導線があり、WAN / WAN6の状態も確認できます。

```sh
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
logread | grep -Ei 'ipoe|map|dslite|ipip|wan6'
```

「IPv6は生きているがIPv4だけだめ」みたいな切り分けができます。

ここも、OpenWrtベースの良さです。

## OpenWrtベースである注意点

自由度が高いということは、注意点もあります。

## 設定ミスでつながらなくなることがある

たとえば、次を間違えると通信できなくなることがあります。

- LAN IP
- VLAN
- firewall zone
- WAN / WAN6
- DHCP
- DNS
- Wi-Fi暗号化
- VPN route

ただし、これは「危険だから触れない」という意味ではありません。

バックアップを取り、1つずつ変更すればかなり安全に運用できます。

```txt
変更前にバックアップ
1つ変える
確認する
動いたらまたバックアップ
```

このリズムが大事です。

## 追加パッケージは更新後に再導入が必要な場合がある

Adblock、VPN Assistant、オートIPoE、Tailscale、WireGuard関連など、あとから入れたものは、ファームウェア更新後に状態確認が必要です。

更新前には次を保存します。

```sh
opkg list-installed > /root/opkg-list-installed-before-update.txt
```

設定バックアップとパッケージ一覧は、分けて管理します。

```txt
設定バックアップ = /etc/configを戻す
パッケージ一覧 = 何を入れていたか思い出す
```

この2つを混同しないほうが安全です。

## バックアップ運用が重要

OpenWrtベースでは、バックアップがかなり大事です。

LuCIでは次です。

```txt
System → Backup / Flash Firmware → Generate archive
```

SSHでは次です。

```sh
sysupgrade -b /tmp/backup-ln6001-$(date +%Y%m%d-%H%M).tar.gz
```

バックアップはルーター内だけでなく、PCへ保存します。

バックアップにはWi-Fiパスワードや内部設定が含まれることがあるため、公開場所には置かないでください。

バックアップは地味ですが、最強です。

## 全部を自動でやってくれる製品ではない

LN6001-JPは、設定を細かく見られることが価値です。

逆に言えば、

```txt
細かいことは一切見たくない
ログも見たくない
バックアップも面倒
全部アプリでやりたい
```

という人には少し向きにくいです。

最低限、次の気持ちがあると相性がよいです。

```txt
必要な時はLuCIを開いてもいい
変更前にバックアップを取れる
ログを少し見ることに抵抗がない
```

それだけで、かなり楽しめます。

## 向いている人

## 1. 家庭内ネットワークを整理したい人

家庭では、次のような構成を作れます。

```txt
HomeWifi
HomeWifi_Guest
HomeWifi_Kids
HomeWifi_IoT
```

ゲストWi-FiをLANから分離する。  
KidsにFamily DNSを配る。  
IoTをMainから分ける。  
NASやプリンターをDHCP予約する。

家庭内の端末が増えてきた人には、かなり相性が良いです。

## 2. NASやミニPCを使っている人

NASやミニPCを使っている人には、OpenWrtベースの柔軟性がかなり役に立ちます。

- DHCP予約
- ローカルDNS
- VPN接続
- Tailscale
- WireGuard
- Port Forwardの管理
- firewall zone
- ログ確認

NASの管理画面をインターネットへ直接出すより、TailscaleやWireGuard経由で入る構成を作りやすいです。

NAS持ちの人には、わりと刺さると思います。

## 3. 小さなオフィスや店舗でネットワークを分けたい人

小さなオフィスや店舗では、次のような分離ができます。

```txt
Staff
Guest
Device
VPN
```

GuestからStaffへ届かないようにする。  
カメラや録画機をDeviceへ置く。  
Staffから必要な機器だけ管理する。  
外からはVPNで入る。

こうした設計へ進めるのは、LN6001-JPの大きな魅力です。

もちろん、本格的な法人向け集中管理機器とは違います。

でも、小さな拠点で“ちゃんと分けたい”人にはちょうどいいです。

## 4. IPoE / IPv4 over IPv6を確認しながら使いたい人

日本の回線では、IPoE / IPv4 over IPv6が分かりにくいです。

LN6001-JPでは、WAN / WAN6の状態確認や、オートIPoEモジュールを使った設定へ進めます。

```sh
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
logread | grep -Ei 'ipoe|map|dslite|ipip|wan6'
```

「IPv6は生きているがIPv4だけだめ」のような状況を、自分で切り分けたい人にはかなり向いています。

## 5. SSHやCLIで状態確認したい人

CLIで全部を設定する必要はありません。

でも、読み取りコマンドが使えるだけで、かなり安心感があります。

```sh
ubus call system board
cat /tmp/dhcp.leases
logread | tail -n 80
wifi status
uci show firewall
```

「壊れたら初期化」ではなく、「どこが壊れているか見る」運用へ進めます。

この違いは、使っているとじわじわ効いてきます。

## 6. 将来的にホームラボや小規模ネットワークを育てたい人

最初は普通のWi-Fiルーターとして使い、あとから次を追加できます。

- ゲストWi-Fi
- Adblock
- VPN
- VLAN
- DHCP予約
- Tailscale
- WireGuard
- 監視とログ
- IPoE詳細設定

一度に全部やる必要はありません。

少しずつ育てられる余地があることが魅力です。

## 向いていない人

## 1. 設定画面をまったく見たくない人

LN6001-JPは初期設定済みで使い始めやすいです。

ただし、魅力を活かすにはLuCIを開く場面が出てきます。

```txt
SSID変更
ゲストWi-Fi
DHCP予約
Adblock
VPN
IPoE確認
バックアップ
```

こうした画面を一切見たくない人には、少し合わないかもしれません。

## 2. 専用メッシュ製品の手軽さだけが欲しい人

広い家を、アプリで簡単にメッシュ化したいだけなら、専用メッシュ製品のほうが楽な場合があります。

LN6001-JP同士のAP / ブリッジ / WDS構成は選択肢になります。

ただし、一般的な「アプリでノード追加して終わり」のメッシュ製品とは方向性が違います。

手軽さ最優先なら、そこは比較したほうがよいです。

## 3. 有線LANも2.5GbEで固めたい人

LN6001-JPの2.5GbEはWAN側です。

LANポートは1Gbpsが4つです。

有線NAS、デスクトップPC、スイッチまで2.5GbEで組みたい人は、この点を確認してから選んでください。

ここは購入前チェックポイントです。

## 4. Wi-Fi 7対応端末がまったくない人

Wi-Fi 7対応端末がない場合、6GHzやMLOの恩恵は限定的です。

もちろん、将来の端末更新を見越して選ぶのはありです。

でも、今すぐWi-Fi 7の性能を使い切りたいなら、端末側の対応も確認します。

## 5. 法人向け集中管理やSLAが必要な人

LN6001-JPは、小さなオフィスやSOHO、店舗でも活用できます。

ただし、次のような環境では業務用ネットワーク機器を検討したほうがよい場合があります。

- 多拠点集中管理
- 専用サポート契約
- 監査ログ管理
- 法人向けクラウド管理
- SLA
- 厳密なアクセス制御
- 大人数の端末管理

LN6001-JPは、小さな拠点で柔軟に使うには良いです。

大規模法人の管理基盤そのものではありません。

## 用途別の相性

用途ごとに見ると、こんな感じです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-09.png)

LN6001-JPは、家庭用としても使えます。

でも、特に面白くなるのは、

```txt
ただのWi-Fiではなく、
ネットワークを整理したい
```

と思い始めたタイミングです。

## 買う前チェックリスト

購入前に、次を確認してみてください。

```txt
□ Wi-Fi 7 / 6GHz対応端末がある、または今後増える予定がある
□ ゲストWi-Fiを家庭内LANや業務LANから分けたい
□ NAS、ミニPC、ホームラボ、カメラなどを使っている
□ IPoE / IPv4 over IPv6の設定や状態確認に興味がある
□ TailscaleやWireGuardを使いたい
□ AdblockやDNS制御に興味がある
□ LuCIやSSHを少し触ることに抵抗がない
□ 変更前にバックアップを取る運用ができそう
□ 2.5GbE LANではなく、2.5GbE WANでよい
□ 専用メッシュの手軽さだけを求めているわけではない
```

このうち半分以上にチェックが入るなら、LN6001-JPはかなり相性がよいと思います。

逆に、ほとんどチェックが入らず、

```txt
ただ置いて、Wi-Fiが出ればOK
```

という場合は、もっとシンプルな製品でもよいかもしれません。

## 購入後に最初にやること

買った直後は、いきなりVLANやVPNから始めなくて大丈夫です。

おすすめ順は次です。

1. 箱から出して、本体底面ラベルを確認する
2. ONU / HGWからWANへ接続する
3. PCまたはスマートフォンを初期SSIDへ接続する
4. `https://192.168.1.1` へアクセスする
5. `root` でログインする
6. 管理パスワードを変更する
7. Main Wi-FiのSSIDとパスワードを整える
8. WAN / WAN6を確認する
9. LuCIバックアップを取る
10. 必要ならゲストWi-FiやIPoE設定へ進む

最初のバックアップは早めに取ります。

LuCIでは次です。

```txt
System → Backup / Flash Firmware → Generate archive
```

この1本があるだけで、かなり安心して触れます。

## 導入後に試したい読み取りコマンド

SSHに抵抗がなければ、まず読み取りだけ試します。

```sh
echo "### system"
ubus call system board
uptime
free
df -h

echo "### WAN/WAN6"
ifstatus wan
ifstatus wan6

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### Wi-Fi"
wifi status

echo "### firewall"
uci show firewall | grep -E 'zone|forwarding'

echo "### recent logs"
logread | tail -n 80
```

これらは基本的に状態確認です。

購入前の人は、

```txt
こういう確認ができるルーターなんだ
```

くらいで見ればOKです。

購入後の人は、現在の状態を知る最初の入り口になります。

## 競合選択肢との比較

## 一般的な家庭用Wi-Fiルーター

向いている人:

```txt
アプリで簡単に設定したい
細かい設定は見たくない
ゲストWi-Fi程度で十分
ログやSSHは不要
```

LN6001-JPとの差:

```txt
LN6001-JPは自由度が高い
そのぶん設定項目も多い
必要な人には便利だが、不要な人には過剰
```

## 専用メッシュWi-Fi製品

向いている人:

```txt
広い家を手軽にカバーしたい
アプリでノード追加したい
細かいVLANやfirewallは不要
```

LN6001-JPとの差:

```txt
LN6001-JPはネットワーク設計の自由度が高い
専用メッシュ製品は展開の手軽さが強い
```

## 業務用ネットワーク機器

向いている人:

```txt
多拠点管理
集中管理
法人向けサポート
監査やポリシー管理
SLA
```

LN6001-JPとの差:

```txt
LN6001-JPは家庭・SOHO・小規模店舗にちょうどよい
本格的な法人管理基盤とは役割が違う
```

## 自作OpenWrtルーター

向いている人:

```txt
自分で機種選定したい
自分でファームウェアを書き込みたい
細部まで完全に自分で管理したい
```

LN6001-JPとの差:

```txt
LN6001-JPは製品として始めやすい
自作OpenWrtは自由度がさらに高いが、導入と復旧も自己責任が大きい
```

## この連載でのおすすめ導入順

LN6001-JPを買ったら、次の順で読むと迷いにくいです。

```txt
000: 連載の入口
001: 初回セットアップ
002: LuCIとSSH
006: Wi-Fi設定
007: ゲストWi-Fi
014: IPoE
023: バックアップ
021: つながらない時の切り分け
```

そのあと、必要に応じて進みます。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/table-10.png)

LN6001-JPは、最初から全部を使い切る必要はありません。

必要になったところから足していくのが、いちばん現実的です。

## 私ならこう始める

個人的におすすめの始め方は、かなりシンプルです。

```txt
1日目:
  初回セットアップ
  Main Wi-Fiだけ整える
  WAN/WAN6確認
  バックアップ

2日目:
  ゲストWi-Fiを作る
  GuestからLANへ入れないことを確認
  バックアップ

3日目以降:
  NASやプリンターをDHCP予約
  必要ならAdblock
  必要ならTailscale
  必要ならKids / IoT
```

いきなり全部やらない。

これがコツです。

OpenWrtベースルーターは、できることが多いです。

だからこそ、

```txt
1つ作る
確認する
バックアップする
次へ行く
```

このリズムがかなり大事です。

## まとめ

LN6001-JPは、単なるWi-Fi 7ルーターというより、

```txt
OpenWrtベースの自由度を、
製品として扱いやすくしたルーター
```

です。

向いているのは、次のような人です。

- Wi-Fi 7や6GHzを使いたい
- ゲストWi-FiをLANから分けたい
- NASやホームラボを使っている
- TailscaleやWireGuardを使いたい
- IPoE / IPv4 over IPv6を自分で確認したい
- AdblockやDNS制御に興味がある
- 小さなオフィスや店舗でStaff / Guest / Deviceを分けたい
- LuCIやSSHで状態確認できると安心する

向いていないのは、次のような人です。

- 設定画面を一切見たくない
- アプリだけで完結する専用メッシュが欲しい
- 有線LANも2.5GbEで固めたい
- Wi-Fi 7対応端末がなく、将来性にもあまり興味がない
- 法人向け集中管理やSLAが必要

この製品は、速度だけで判断しないほうがよいです。

むしろ、次の問いで考えると分かりやすくなります。

```txt
自分のネットワークを、少しずつ整理したいか？
```

答えが「はい」なら、LN6001-JPはかなり面白い選択肢になります。

買って終わりではなく、育てるルーターです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/030/diagram-03.png)

## 次に読むなら

購入判断のあとに読み進めるなら、次の記事がおすすめです。

- [LN6001-JPとは何か](https://note.com/ikmsan/n/n1e25c8075cc2)
- [初回セットアップ](https://note.com/ikmsan/n/n2e9666e3a1a5)
- [LuCIとSSHの基本](https://note.com/ikmsan/n/n3e9b348d4c95)
- [Wi-Fi設定](https://note.com/ikmsan/n/n4dbf8f72744c)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)

購入前の人は、まず000と030。

買った直後の人は、001、002、006、023へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、SSHコマンドの出力、VPN Assistant、Internet Assistant、Adblock、opkg、firewall、Wi-Fi radio名は、ファームウェアや追加モジュール更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずLinksys公式サポートの最新情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPは初心者にも使えますか？

初期設定済みで出荷されるため、入口はあります。

ただし、魅力を活かすには、LuCIの画面を見たり、必要に応じてSSHで状態確認したりする姿勢があるとより楽しめます。

完全に設定画面を見たくない人には、少し機能が多いかもしれません。

### 普通のWi-Fiルーターとしても使えますか？

使えます。

DHCP自動で使える回線環境なら、初期設定に近い状態でも利用できます。

ただし、LN6001-JPの価値は、あとからゲストWi-Fi、VPN、VLAN、Adblock、IPoE確認などへ進める余地があることです。

### Wi-Fi 7対応端末がないと意味がありませんか？

意味がないわけではありません。

5GHz中心の高性能ルーターとして使いながら、将来のWi-Fi 7端末に備える考え方もできます。

ただし、6GHzやMLOの恩恵をすぐ受けたいなら、端末側の対応も必要です。

### 2.5GbE LANでNASをつなげますか？

LN6001-JPの2.5GbEポートはWAN側です。

LANポートは1Gbps ×4です。

有線NASやデスクトップPCを2.5GbEでつなぎたい場合は、この点に注意してください。

### ゲストWi-FiやVLANは難しいですか？

最初はLuCIで作るのがおすすめです。

この連載では、ゲストWi-Fi、VLAN、firewall zoneを段階的に扱っています。

最初から全部作る必要はなく、まずMain Wi-FiとゲストWi-Fiだけでも十分です。

### VPNは使えますか？

Linksys公式のVPN AssistantでWireGuard / Tailscaleの導線があります。

固定IPやポート開放を管理できるならWireGuard、固定IPなしやIPoE環境で始めやすさを重視するならTailscaleが選びやすいです。

### IPoE / IPv4 over IPv6は使えますか？

NTTフレッツ系のIPv4 over IPv6向けに、オートIPoE / Internet Assistantモジュールが案内されています。

HGW配下かONU直下か、プロバイダの方式が何かによって設定方針が変わります。

### opkgで好きなパッケージを入れていいですか？

使えますが、何でも入れるのはおすすめしません。

空き容量、依存関係、ファームウェアとの相性を見ながら、必要なものだけ入れます。

`opkg upgrade` による一括更新は基本的に避けます。

### メッシュWi-Fi目的だけで選ぶのはありですか？

手軽に家全体をカバーしたいだけなら、専用メッシュ製品のほうが合う場合があります。

LN6001-JP同士のWDSやAP / ブリッジ構成は選択肢になりますが、アプリ中心の専用メッシュとは方向性が違います。

### 小さな店舗でも使えますか？

使えます。

Staff、Guest、Deviceを分ける構成と相性があります。

ただし、POSや決済端末はベンダー要件を優先し、雑にDeviceへ混ぜないほうが安全です。

### 買ったあと、最初にやるべきことは？

管理パスワード変更、Main Wi-Fi設定、WAN / WAN6確認、LuCIバックアップです。

いきなりVLANやVPNへ行かず、まず安定した基本状態を作るのがおすすめです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/
- 株式会社アスク Velop WRT Pro 7 製品ページ: https://www.ask-corp.jp/products/linksys/router/velop-wrt-pro-7.html
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/

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
