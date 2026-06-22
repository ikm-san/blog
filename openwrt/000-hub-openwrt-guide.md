<!-- mirror-source: articles/000-hub-openwrt-guide.md -->

# OpenWrtは怖くない｜LN6001-JPで家庭・店舗ネットワークを少しずつ育てる実践ガイド【連載目次】

OpenWrt系ルーターって、ちょっと気になりますよね。

普通のWi-Fiルーターより細かく触れそう。  
ゲストWi-Fiも、VPNも、VLANも、広告ブロックもできそう。  
NASやミニPC、店舗のカメラ管理にも使えそう。

でも同時に、こういう不安もあると思います。

「日本のIPoE回線でちゃんと使えるの？」  
「設定を触りすぎて戻せなくならない？」  
「SSHとかVLANとか、急に難しくならない？」  
「家庭で使うにはやりすぎでは？」

この連載は、そこをひとつずつほどいていくための入口です。

使うのは、OpenWrtベースのWi-Fi 7ルーター **Linksys Velop WRT Pro 7（LN6001-JP）**。

LN6001-JPは、日本向けに技適や法令へ対応したモデルです。一般的なWi-Fiルーターのように初期状態から使い始められる一方で、LuCI、SSH、VLAN、VPN、IPoE/IPv6など、OpenWrtベースらしい深い設定にも進めます。

実機開発にも関わった立場から、この連載では「全部をいじり倒す」よりも、家庭・小さなオフィス・店舗で **現実的に使えるOpenWrt運用** を優先してまとめていきます。

最初から完璧な構成を作らなくて大丈夫です。

まず普通に使える状態を作る。  
次にゲストWi-Fiを分ける。  
必要になったらVPNやVLANを足す。  
触る前にバックアップを取る。  
困ったらログを見る。

このくらいの距離感が、OpenWrtベースルーターとは長く付き合いやすいと思っています。

## このページの使い方

このページは、連載全体の地図です。

上から順番に全部読む必要はありません。

「回線設定が不安」ならIPoEの記事へ。  
「ゲストWi-Fiを作りたい」ならゲストWi-Fiの記事へ。  
「外出先からNASへ入りたい」ならVPNの記事へ。  
「店舗でStaffとGuestを分けたい」なら店舗向け構成例へ。

自分の困りごとに近いところから読んで大丈夫です。

探し方は、大きく3つです。

- まず読んでおきたい5本から探す
- 家庭・小さなオフィス・店舗など、用途別に探す
- IPoE、Wi-Fi、VLAN、VPN、トラブル対応など、困りごとから探す

LN6001-JPはできることが多いルーターです。

でも、最初から全部を使う必要はありません。

むしろおすすめは逆です。

```txt
まず普通に使う
  ↓
必要になったところだけ足す
  ↓
触る前にバックアップを取る
  ↓
うまくいったらまたバックアップする
```

これだけで、かなり安心して遊べます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/000/diagram-01.png)

## このページでわかること

このページでは、連載全体の読み方をまとめています。

- LN6001-JPの連載を、どの順番で読むと迷いにくいか
- 家庭、小さなオフィス、店舗で最初に読むべき記事
- IPoE、Wi-Fi、VLAN、VPNなど、目的別にどの記事を見ればよいか
- OpenWrtベースルーターで、最初に触るべきところ
- 逆に、最初から触りすぎなくてよいところ

OpenWrtは自由度が高いぶん、最初の地図があるとかなり楽です。

## まず読むならこの5本

LN6001-JPやOpenWrt系ルーターが初めてなら、まずは次の5本から読むと全体像をつかみやすくなります。

1. [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
2. [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)
3. [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)
4. [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
5. [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3)

この5本を読むと、だいたい次が見えてきます。

- LN6001-JPがどんな製品なのか
- 日本国内でOpenWrt系Wi-Fiルーターを使う時に何を気にすべきか
- 自分の回線で使えそうか
- 買ったあと、最初に何を確認すればよいか
- どのポートに何を挿せばよいか

細かい設定に入る前の地図として、まずここだけ見ておくと楽です。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/000/diagram-02.png)

## 用途別の読み方

### 家庭で使いたい人

家庭で使うなら、最初に考えることはシンプルです。

```txt
家族が普通に使える
ゲストWi-Fiは家庭内LANから分ける
IoTや子ども用は必要になったら足す
```

最初から全部を分ける必要はありません。

まずはMain Wi-Fiを安定させて、次にゲストWi-Fi。  
KidsやIoTは、必要になってからで大丈夫です。

おすすめの入口はこちらです。

- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
- [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
- [LN6001-JPのWi-Fi設定](https://note.com/ikmsan/n/ndd6f12876bc8)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)

家庭ネットワークでは、完璧な設計より「家族に説明できる設計」のほうが強いです。

普段は `HomeWifi`。  
ゲストには `HomeWifi_Guest`。  
スマート家電が増えたら `HomeWifi_IoT`。

このくらいから始めるのが、かなり現実的です。

### 小さなオフィスで使いたい人

小さなオフィスでは、家庭より少しだけ考えることが増えます。

- 回線方式
- 固定IPやDHCP予約
- StaffとゲストWi-Fiの分離
- NASやプリンターのIP管理
- VPNでのリモート接続
- firewall zone

おすすめの入口はこちらです。

- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
- [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)

小さなオフィスでは、まず「業務端末」と「ゲストWi-Fi」を混ぜないことが大事です。

難しいことをする前に、

```txt
StaffはStaff
ゲストWi-Fiはインターネットだけ
NASやプリンターはIP固定
```

ここまでできるだけでも、かなり運用しやすくなります。

### 店舗で使いたい人

店舗では、Staff、Guest、POS、カメラ、スマートロックなど、役割の違う端末が混ざりがちです。

まずは、誰の端末をどこに置くかを分けます。

```txt
Staff:
  スタッフPC、管理端末、業務タブレット

Guest:
  ゲストWi-Fi

Device:
  カメラ、録画機、設備機器

POS / Payment:
  ベンダー要件を優先
```

おすすめの入口はこちらです。

- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)

店舗では、ポート開放よりVPNを優先したほうが安全に運用しやすい場面が多いです。

録画機や管理PCへ外から入りたい場合も、まずはWireGuardやTailscaleを検討します。

### OpenWrtが気になる人

OpenWrtそのものに興味がある人は、最初にここを押さえると入りやすいです。

- 普通のWi-Fiルーターと何が違うのか
- LuCIでは何が見えるのか
- SSHでは何が見えるのか
- opkgで何を追加できるのか
- どこは触りすぎないほうがよいのか

おすすめの入口はこちらです。

- [OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](https://note.com/ikmsan/n/n8d8823719244)
- [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)
- [LuCIとSSHの基本](https://note.com/ikmsan/n/n4c7478f4efe2)
- [パッケージ追加で壊さないための考え方](https://note.com/ikmsan/n/n2bc5e8447e77)
- [SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)

OpenWrtベースと聞くと、CLI中心の世界に見えるかもしれません。

でも最初はLuCIだけで十分です。

SSHも、最初は設定変更ではなく「今どうなっているかを見る道具」として使えば大丈夫です。

## 困りごとから探す

### 回線まわりがよく分からない

日本の光回線まわりは、正直ちょっと分かりにくいです。

IPoE、IPv4 over IPv6、MAP-E、DS-Lite、HGW、ONU。  
似たような言葉が並びます。

まずはこの記事からどうぞ。

- [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)

ここでは、HGW配下なのか、ONU直下なのか、LN6001-JP側で何を担当するのかを整理しています。

### 買ってすぐ何を確認すべきか知りたい

箱から出した直後は、難しい設定より先に見るものがあります。

- WANポート
- LANポート
- LED
- 底面ラベル
- 初期SSID
- 管理画面URL
- バックアップ

入口はこちらです。

- [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
- [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3)

まずは、普通に管理画面へ入れて、インターネットへ出られるところまででOKです。

### Wi-Fiを家族用、ゲスト用、IoT用に分けたい

家庭でよく出てくるテーマです。

最初は、MainとゲストWi-Fiだけで十分です。

そのあと、必要になったらKidsやIoTを足します。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)

SSIDを増やすことが目的ではありません。

ゲスト端末からNASやプリンターへ入れない。  
子ども用端末にはDNS方針を分ける。  
IoTはMainから少し離す。

このくらいからで十分です。

### 小さなオフィスや店舗でネットワークを分けたい

業務端末とゲストWi-Fiが同じLANにいると、あとからかなり困ります。

まずはStaff、Guest、Deviceを分けるところからです。

- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)

firewall zoneは、最初は怖い名前に見えます。

でも考え方はシンプルです。

```txt
Staffの部屋
Guestの部屋
Deviceの部屋
VPNの入口
```

どの部屋からどの部屋へ行ってよいかを決めるだけです。

### 外出先から自宅や店舗へ安全に入りたい

NAS、ミニPC、店舗の録画機、LuCI管理画面。

外から見たいものはいろいろあります。

でも、いきなりポート開放する前に、まずVPNを考えたほうが安全です。

- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [Tailscaleの使いどころ](https://note.com/ikmsan/n/n8d21425036a9)
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)

固定IPやポート開放を管理できるならWireGuard。  
固定IPなし、IPoE環境で始めやすさを重視するならTailscale。

この分け方で考えると迷いにくいです。

### つながらない時の見方を知りたい

「Wi-FiにはつながるのにWebが開けない」

これ、よくあります。

でも中身は分けられます。

```txt
WANなのか
DNSなのか
DHCPなのか
Wi-Fiなのか
firewallなのか
```

入口はこちらです。

- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)

困った時に最初に見るのは、だいたいこのあたりです。

```sh
ifstatus wan
ifstatus wan6
cat /tmp/dhcp.leases
nslookup www.google.co.jp
logread | tail -n 50
```

最初は読むだけで大丈夫です。

### SSHやCLIはどこから始めればいいか知りたい

SSHは、最初から設定変更に使わなくて大丈夫です。

まずは状態確認だけで十分です。

- [LuCIとSSHの基本](https://note.com/ikmsan/n/n4c7478f4efe2)
- [Windows/MacのSSHログイン方法](https://note.com/ikmsan/n/na751d7336b87)
- [SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)

SSHは黒い画面なので少し怖く見えます。

でも、最初にやるのはこれくらいです。

```sh
ubus call system board
ifstatus wan
cat /tmp/dhcp.leases
logread | tail -n 50
```

設定を変えずに、ただ見るだけ。

それだけでもかなり役に立ちます。

## 構成例記事の選び方

「結局、自分はどの記事を読めばいいの？」となったら、まず構成例から入るのもおすすめです。

ネットワーク設計は、最初に「誰の端末を分けたいのか」を決めると考えやすくなります。

- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)  
  家族用、ゲストWi-Fi、Kids、IoTをどう分けるかを整理しています。

- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)  
  外出先から自宅や小規模拠点へ安全に入りたい人向けです。WireGuardとTailscaleの使い分けが分かります。

- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)  
  Staff、Guest、Deviceを分けたい店舗向けです。POSやカメラをどう考えるかの入口になります。

迷ったら、まず自分に一番近い構成例を読んでください。

そこから必要な関連記事へ飛ぶほうが、いきなり全部を読むより楽です。

## 記事一覧

### 入門・導入

- [OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](https://note.com/ikmsan/n/n8d8823719244)
- [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)
- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
- [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
- [LuCIとSSHの基本](https://note.com/ikmsan/n/n4c7478f4efe2)

### 家庭ネットワーク改善

- [LN6001-JPのWi-Fi設定](https://note.com/ikmsan/n/ndd6f12876bc8)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [DNS広告ブロック](https://note.com/ikmsan/n/n4759cf81d0e1)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)

### 小さなオフィス

- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)

### セキュリティ

- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [ポート開放の注意点](https://note.com/ikmsan/n/n765ffea6ea82)
- [Tailscaleの使いどころ](https://note.com/ikmsan/n/n8d21425036a9)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)
- [ファームウェア更新運用](https://note.com/ikmsan/n/nff6b598da354)

### トラブル対応・CLI

- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [リセットと復旧](https://note.com/ikmsan/n/n5088d68a2205)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [パッケージ追加で壊さないための考え方](https://note.com/ikmsan/n/n2bc5e8447e77)
- [SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)
- [Windows/MacのSSHログイン方法](https://note.com/ikmsan/n/na751d7336b87)

### 製品・構成例

- [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)
- [LN6001-JP導入レビュー](https://note.com/ikmsan/n/n3ae0dd924a6d)

### 追加トピック

- [MLOを有効化する前に確認すること](https://note.com/ikmsan/n/n00750e23f49c)
- [AP/ブリッジ/WDS子機として使う](https://note.com/ikmsan/n/n40b623436544)

## 最初に知っておくと楽な用語

OpenWrt系の記事は、どうしても専門用語が出てきます。

でも最初は、ざっくりで大丈夫です。

- LuCI: ブラウザで開くOpenWrt系の管理画面です。
- SSH / CLI: 文字で状態を確認する操作方法です。最初は読み取り中心で使います。
- LAN: 家や事務所の内側ネットワークです。
- WAN: インターネット回線側のネットワークです。
- SSID: スマートフォンやPCに表示されるWi-Fi名です。
- DHCP: ルーターが端末へIPアドレスを自動で配る仕組みです。
- DNS: `example.com` のような名前を通信先の住所へ変換する仕組みです。
- VLAN: 1つの機器上でStaff、Guest、Cameraのようにネットワークを部屋分けする仕組みです。
- firewall zone: その部屋同士が通信してよいかを決める境界線です。
- IPoE: 日本の光回線でよく使われるIPv6系の接続方式です。
- PPPoE: プロバイダのIDとパスワードで接続する方式です。
- VPN: 外出先から自宅や事務所へ安全に入るための通路です。

たとえばfirewall zoneは、最初は「部屋分け」くらいで考えれば十分です。

Guestの部屋からインターネットへは行っていい。  
でもStaffやNASの部屋には入れない。

まずはその感覚で大丈夫です。

## 読む順番の決め方

この連載は、上から全部を順番に読むより、自分の困りごとに近い記事と、その前後だけ拾っていくほうが入りやすいです。

たとえば、

```txt
回線設定が不安
  → IPoEの記事

ゲストWi-Fiを分けたい
  → ゲストWi-Fi + firewall zone

外出先からNASへ入りたい
  → VPN + DHCP予約

店舗でStaff / Guestを分けたい
  → 店舗向け構成例 + VLAN + firewall
```

という感じです。

OpenWrtベースと聞くと、最初から難しいCLI操作が必要そうに見えるかもしれません。

でも、最初に見るべきなのはここです。

- LuCIの管理画面
- 回線方式
- Wi-Fi名
- バックアップの有無
- いま通信できていること

SSHやCLIは、いきなり設定を書き換えるためではなく、まず今の状態を落ち着いて確かめるための道具として使えます。

## 買う前に見ておきたいこと

買う前に特に見ておきたいのは、次の3点です。

- 自分の回線方式がIPoE、PPPoE、IPv4 over IPv6、固定IPのどれか
- 家庭、店舗、小規模オフィスなど、どんな用途で使うのか
- Guest Wi-Fi、VPN、VLAN、DNS設定など、追加で使いたい機能は何か

この3点が決まると、読む記事も設定の優先順位もかなり絞れます。

個人的には、まず「自分の回線がどうなっているか」を先に見るのがおすすめです。

Wi-FiやVPNを触る前に、WAN / WAN6がどう見えているかを分かっていると、あとでかなり楽です。

## 実際どこまで触るべきか

正直なところ、全部をいじり倒す必要はありません。

僕自身も、普段はオートIPoEモジュールでインターネット接続を済ませて、あとは工場出荷時の初期設定をベースに使うことが多いです。

全部カスタムするより、安定して使えることのほうが大事な場面はかなりあります。

一方で、YAMAHAやNECの業務用ルーターを自宅で使い、サーバーラックまで組んでしまうような“逸般の誤家庭”の方にとっても、LN6001-JPはカスタムできるWi-Fi APとして遊べる余地があります。

つまり、LN6001-JPはかなり幅があります。

```txt
普通のWi-Fiルーターとして使う
少しだけゲストWi-Fiを分ける
家庭内をKids / IoTまで整理する
店舗でStaff / Guest / Deviceを分ける
VPNやTailscaleで外から入る
ホームラボの一部として使う
```

どこまで触るかは、自分の使い方に合わせれば大丈夫です。

## 設定前に毎回残すメモ

どの記事から始めるにしても、設定前に残しておきたいメモは共通です。

最低限、次を書いてから作業に入ると、途中で迷っても戻りやすくなります。

- 回線方式
- 現在のLAN側IPアドレス
- 管理画面のURL
- Wi-Fi名
- バックアップの有無
- 変更前に通信できていたこと

特にIPoE、VLAN、VPN、firewallは、1つの変更が複数の場所に響きます。

作業前に「いま何ができているか」をひとこと残しておくと、作業後に「何が変わったか」を比べやすくなります。

中上級者の方が設定を深く触る場合は、`uci show` で設定内容を一気に出力して、テキストファイルとして保存しておくのもおすすめです。

```sh
uci show network > network-before.txt
uci show wireless > wireless-before.txt
uci show firewall > firewall-before.txt
uci show dhcp > dhcp-before.txt
```

設定で一番強い人は、全部暗記している人ではありません。

変更前に戻れる状態を作っている人です。

## よくある質問

### LN6001-JPの連載はどの記事から読めばいい？

まずは次の5本から入ると、全体像をつかみやすいです。

- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
- [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)
- [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
- [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3)

迷ったら、この5本だけ先に読めば大丈夫です。

### OpenWrt初心者でもこの連載は読める？

大丈夫です。

最初からCLI中心で進める前提ではありません。

Web管理画面のLuCI、回線方式、Wi-Fi、バックアップのような入りやすいテーマから順に追える構成にしています。

SSHやCLIも出てきますが、最初は設定を書き換えるためではなく、「今どうなっているか」を確認するための道具として使うくらいで十分です。

### 家庭向けと小さなオフィス向けでは読む記事は違う？

少し違います。

家庭なら、次の記事が入口になります。

- [LN6001-JPのWi-Fi設定](https://note.com/ikmsan/n/ndd6f12876bc8)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)

小さなオフィスなら、次の記事が近いです。

- [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)

自分の用途に近いほうから読めば大丈夫です。

### LN6001-JPは全部を細かく設定しないと使えない？

いいえ。

LN6001-JPは、一般的なWi-Fiルーターのように初期状態から使い始められる製品です。

そのうえで、必要に応じてOpenWrtベースならではの設定へ踏み込めます。

最初からVLAN、VPN、パッケージ追加まで全部を設定する必要はありません。

まずは安定してインターネットにつながる状態を作る。  
そのあと、必要な機能だけ足していく。

この進め方がおすすめです。

### SSHやCLIは必須ですか？

必須ではありません。

LuCIだけでもかなり操作できます。

ただ、トラブル時にSSHで次のような読み取りコマンドを使えると、かなり楽になります。

```sh
ifstatus wan
ifstatus wan6
cat /tmp/dhcp.leases
logread | tail -n 50
```

最初は見るだけでOKです。

### ゲストWi-FiはSSIDを分けるだけでいい？

SSIDを分けるだけでは不十分です。

ゲストWi-Fiとして使うなら、network、DHCP、firewall zoneまで分けるのがおすすめです。

大事なのは、ゲスト端末から家庭内LANや業務LANへ入れないことです。

### 外出先からNASや店舗機器へ入りたい時は？

まずVPNを考えるのがおすすめです。

NAS、カメラ、LuCI、SSHをインターネットへ直接公開するより、WireGuardやTailscale経由にするほうが安全に運用しやすいです。

固定IPやポート開放を管理できるならWireGuard。  
固定IPなし、IPoE環境で始めやすさを重視するならTailscale。

この分け方で見ると選びやすいです。

## この連載で扱わないこと

この連載では、一般的な家庭用Wi-Fiルーターに非公式ファームウェアをインストールする手順は扱いません。

日本国内でWi-Fi機器を使う前提で、国内向けOpenWrtベースのOSを搭載したLN6001-JPをベースに進めます。

また、技適や国内利用の前提に関わる無線設定を回避・変更するような内容も扱いません。

## この連載の方針

この連載では、Linksys公式サポートで確認できる情報を優先します。

無線の国設定、送信出力、DFSなど、国内使用の前提に関わる値は変更しません。

IPoE、VPN、MLOなどは公式手順を確認しながら進め、設定変更の前後にはバックアップと復旧を重視します。

OpenWrtらしい自由度は活かしつつ、家庭や小規模オフィスで現実的に安定して使うことを優先します。

## CLI例の前提

この連載のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

技適に関わる無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報もあわせて確認してください。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/000/diagram-03.png)

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
