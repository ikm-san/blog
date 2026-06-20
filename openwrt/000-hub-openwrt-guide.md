<!-- mirror-source: articles/000-hub-openwrt-guide.md -->

# LN6001-JPで始めるOpenWrtベースルーター実践ガイド【OpenWrt集中連載 目次】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## このページについて

OpenWrtベースのWi-Fiルーターは、できることが多いぶん、最初にどこから触ればいいのか少し分かりにくいところがあります。

このページは、Linksys Velop WRT Pro 7（LN6001-JP）を使って、家庭・小さなオフィス・店舗のネットワークを整えていくための連載インデックスです。単なる目次というより、「まず何を読めばいいか」「自分の使い方に合いそうか」「どこまでできるのか」をざっくりつかむための入口として用意しています。

LN6001-JPは、Wi-Fi 7世代のハードウェアに、OpenWrtベースならではの柔軟さを組み合わせた国内向け製品です。一般的な市販ルーターよりも少し踏み込んだ設定ができる一方で、工場出荷時点で初期設定を済ませてあるので、最初から全部をCLIで触るような製品でもありません。

この連載では、買う前に見ておきたいポイントから、初期設定、LuCI、Wi-Fi、VLAN、VPN、広告ブロック、トラブル対応まで、知りたい順に追いやすい形でまとめています。

対象はOpenWrtの上級者だけではありません。Wi-Fi 7ルーターを探している人、IPoE回線で悩んでいる人、ゲストWi-FiやVPNを整理したい家庭や小さなオフィスの人にも向いています。

一般的な市販ルーターでは少し物足りない。でも、いきなり全部を自分で作り込むほどではない。そんな人にとって、LN6001-JPはかなりちょうどいい立ち位置のルーターだと思います。YAMAHAやNECの業務用ルーターを自宅で使ってサーバーラックも構築してしまうような逸般の誤家庭の方でもWi-Fiは必要ですので、カスタムできるWiFi APとして遊んでほしいです。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/000/diagram-01.png)

## このページでわかること

- LN6001-JPの連載を、どの順番で読むと迷いにくいか
- 家庭、小さなオフィス、店舗それぞれで最初に読むべき記事
- IPoE、Wi-Fi、VLAN、VPNなど、目的別にどの記事を見ればよいか

## 最初に知っておくと楽な用語

この連載では専門用語も出てきますが、各記事の中でも必要なところで短く補足します。最初は次の理解で十分です。

- LuCI: ブラウザで開くOpenWrt系の管理画面です。
- SSH / CLI: 文字で状態を確認する操作方法です。最初は読み取り中心で使います。
- LAN: 家や事務所の内側ネットワークです。
- WAN: インターネット回線側のネットワークです。
- SSID: スマートフォンやPCに表示されるWi-Fi名です。
- DHCP: ルーターが端末へIPアドレスを自動で配る仕組みです。
- DNS: `example.com` のような名前を通信先の住所へ変換する仕組みです。
- VLAN: 1つの機器上で Staff、Guest、Camera のようにネットワークを部屋分けする仕組みです。
- firewall zone: その部屋同士が通信してよいかを決める境界線です。
- IPoE: 日本の光回線でよく使われるIPv6系の接続方式です。
- PPPoE: プロバイダのIDとパスワードで接続する方式です。
- VPN: 外出先から自宅や事務所へ安全に入るための通路です。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/000/diagram-02.png)

## まず読むならこの5本

1. [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)
2. [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)
3. [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)
4. [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
5. [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3)

この5本を読めば、LN6001-JPがどんな製品か、自分の回線や使い方に合いそうかを判断しやすくなります。細かい設定に入る前の地図として、最初にざっと見ておくと楽です。

## 用途別の読み方

### 家庭で使いたい人

- 003 まず製品の立ち位置をつかむ
- 004 初期設定の流れに目を通す
- 006 Wi-Fi帯域の使い分けを押さえる
- 007 Guest Wi-Fiの作り方を見る
- 010 家族・IoT・来客の分け方を考える

### 小さなオフィスで使いたい人

- 003 製品の立ち位置を先に読む
- 014 IPoEと回線方式を整理する
- 011 VLANの考え方を押さえる
- 012 DHCP予約の設計を見ておく
- 013 WireGuard/Tailscaleの使いどころを比べる

### 店舗で使いたい人

- 029 店舗構成の全体像を見る
- 007 ゲストWi-Fiの作り方を読む
- 011 Staff/Guest/Camera分離の考え方を押さえる
- 017 ポート開放よりVPNを検討
- 028 VPN構成の例を参考にする

### OpenWrtが気になる人

- 001 OpenWrtの考え方から入る
- 002 日本国内でのWi-Fi機器の前提を押さえる
- 005 LuCIとSSHの入口を眺める
- 024 パッケージ追加の考え方を知る
- 025 読み取りCLIで何が見えるかをつかむ

## 構成例記事の選び方

家庭、小規模オフィス、店舗のどれに近いか迷うなら、まずは次の3本から一番近いものを選ぶと入りやすいです。ネットワーク設計は、最初に「誰の端末を分けたいのか」を決めると考えやすくなります。

- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751): 家族用、来客用、IoT用のWi-Fiを分けたい家庭向けです。Main、Kids、IoT、Guest をどう分けるかを最初に押さえられます。
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309): 外出先から自宅や小規模拠点へ安全に入りたい人向けです。WireGuard と Tailscale をどう選ぶかが入口になります。
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680): Staff、Guest、POS、カメラを整理したい店舗向けです。Staff、Guest、Device をどう分けるかを先に確認できます。

家庭で「まずWi-Fiを整理したい」なら 027、リモートアクセスを先に考えたいなら 028、店舗で業務端末と来客用を分けたいなら 029 から入ると、必要な関連記事にもつながりやすくなります。

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

- [VLANで社内・来客ネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
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

### トラブル対応

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
- [Windows/MacのSSHログイン方法](https://note.com/ikmsan/n/na751d7336b87)

## 読む順番の決め方

この連載は、上から全部を順番に読む前提ではありません。むしろ、自分の困りごとに近い記事と、その前後だけ拾っていくほうが入りやすいです。

たとえば「回線設定が不安」ならIPoEの記事から、「ゲストWi-Fiを分けたい」ならGuest Wi-Fiとfirewall zoneの記事から入ると、やりたいことと設定項目がつながりやすくなります。

OpenWrtベースと聞くと、最初から難しいCLI操作が必要そうに見えるかもしれません。ですが、最初に見ておきたいのは、直感的に触れるLuCIの管理画面、回線方式、Wi-Fi名、バックアップの有無です。

SSHやCLIも、いきなり設定を書き換えるためというより、まず今の状態を落ち着いて確かめるための道具として使えます。

買う前に特に見ておきたいのは、次の3点です。

- 自分の回線方式がIPoE、PPPoE、IPv4 over IPv6、固定IPのどれか
- 家庭、店舗、小規模オフィスなど、どんな用途で使うのか
- Guest Wi-Fi、VPN、VLAN、DNS設定など、追加で使いたい機能は何か

この3点が決まると、読む記事も設定の優先順位もかなり絞れます。LN6001-JPはできることが多い製品ですが、最初から全部を漏れなく使う必要はありません。

まずは普通に使える状態を作って、必要な機能をあとから一つずつ足していくくらいがちょうどいいです。正直なところ、筆者自身も普段はオートIPoEモジュールでインターネット接続を済ませて、あとは工場出荷時の初期設定をベースに使うことが多いです。全部をいじり倒すより、安定して使えることのほうが大事な場面もかなりあります。

## 困りごとから探す

- 回線まわりがよく分からない: [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8) から読むと、IPoE、IPv4 over IPv6、HGWまわりの整理がしやすいです。
- 買ってすぐ何を確認すべきか知りたい: [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0) と [LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3) を先に見ると、配線と初期設定で迷いにくくなります。
- Wi-Fiを家族用、来客用、IoT用に分けたい: [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)、[家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)、[家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751) が入口です。
- 小さなオフィスや店舗で分離したい: [VLANで社内・来客ネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)、[店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)、[firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4) をまとめて見るとつながります。
- 外出先から自宅や店舗へ安全に入りたい: [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)、[Tailscaleの使いどころ](https://note.com/ikmsan/n/n8d21425036a9)、[VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309) が近いです。
- つながらない時の見方を知りたい: [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)、[監視とログ](https://note.com/ikmsan/n/n8571cacdde40)、[SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6) を先に押さえると原因を絞りやすくなります。
- SSHやCLIはどこから始めればいいか知りたい: [LuCIとSSHの基本](https://note.com/ikmsan/n/n4c7478f4efe2)、[SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)、[Windows/MacのSSHログイン方法](https://note.com/ikmsan/n/na751d7336b87) から入ると無理がありません。

## 設定前に毎回残すメモ

どの記事から始めるにしても、設定前に残しておきたいメモは共通です。回線方式、現在のLAN側IP、Wi-Fi名、管理画面URL、バックアップの有無を書いてから作業に入ると、途中で迷っても戻りやすくなります。

特にIPoE、VLAN、VPN、firewallは、1つの変更が複数の場所に響きます。作業前に「いま何ができているか」をひとこと残しておくと、作業後に「何が変わったか」を比べやすくなります。

中上級者の方が設定を深く触る場合は、`uci show` で設定内容を一気に出力して、テキストファイルとして保存しておくのもおすすめです。

## よくある質問

### LN6001-JPの連載はどの記事から読めばいい？

まずは [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](https://note.com/ikmsan/n/n1e25c8075cc2)、[日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](https://note.com/ikmsan/n/n9fcf33e2b050)、[NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)、[LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)、[LN6001-JPのハードウェアを見る](https://note.com/ikmsan/n/nf5e2df270ea3) の5本から入ると、製品の立ち位置、国内利用の前提、回線との相性、初期設定まで一通りつかみやすくなります。迷ったら、この5本だけ先に読めば大丈夫です。

### OpenWrt初心者でもこの連載は読める？

はい、大丈夫です。最初からCLI中心で進める前提だと大変なので、Web管理画面のLuCI、回線方式、Wi-Fi、バックアップのような入りやすいテーマから順に追える構成にしています。

SSHやCLIも出てきますが、最初は設定を書き換えるためというより、「今どうなっているか」を確認するための道具として使うくらいで十分です。

### 家庭向けと小さなオフィス向けでは読む記事は違う？

家庭なら [LN6001-JPのWi-Fi設定](https://note.com/ikmsan/n/ndd6f12876bc8)、[ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)、[家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3) 寄り、小さなオフィスなら [NTT IPoEとIPv4 over IPv6](https://note.com/ikmsan/n/n97ddc6c12ca8)、[VLANで社内・来客ネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)、[固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)、[WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c) 寄りから読むと必要な情報へ早くたどり着けます。

## この連載で扱わないこと

ここでは、一般的な家庭用Wi-Fiルーターに非公式ファームウェアをインストールする手順は扱いません。日本国内でWi-Fi機器を使う前提で、国内向けOpenWrtベースのOSを搭載したLN6001-JPをベースに進めます。

また、技適や国内利用の前提に関わる無線設定を回避・変更するような内容も扱いません。

## この連載の方針

この連載では、Linksys公式サポートで確認できる情報を優先します。無線の国設定・送信出力・DFSなど、国内使用の前提に関わる値は変更しません。

IPoE、VPN、MLOなどは公式手順を確認しながら進め、設定変更の前後にはバックアップと復旧を重視します。OpenWrtらしい自由度は活かしつつ、家庭や小規模オフィスで現実的に安定して使うことを優先します。

## CLI例の前提

この連載のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。技適に関わる無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報もあわせて確認してください。

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
