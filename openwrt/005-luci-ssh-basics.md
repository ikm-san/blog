<!-- mirror-source: articles/005-luci-ssh-basics.md -->

# LuCIとSSHは怖くない｜LN6001-JPで最初に見る画面とコマンド【OpenWrt集中連載005】

OpenWrtベースのルーターと聞くと、ちょっと身構える人もいると思います。

「黒い画面でコマンドを打たないと使えないのでは？」  
「LuCIって何？」  
「SSHで入ったら、何か壊しそう」  
「普通のWi-Fiルーターみたいに画面で設定できないの？」

大丈夫です。

LN6001-JPを最初に触る時、いきなりCLIから入らなくて大丈夫です。

基本は **LuCI** というブラウザ管理画面で見ます。  
そして、LuCIでは見えにくいログや細かい状態だけ、**SSH** で補います。

この使い分けが、いちばん無理なく始められます。

最初のゴールは、設定を書き換えることではありません。

```txt
今どうなっているかを見られるようになる
```

これだけです。

WANは取れている？  
端末にIPは配れている？  
DNSだけおかしい？  
Wi-Fi側で何かログが出ている？

こういう時に、LuCIとSSHの両方を少し使えると、急に見える景色が変わります。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で最初に見るLuCIの画面、SSHでの接続方法、そして設定変更前に覚えておきたい読み取りコマンドを整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、CLIで何でも設定できるようになることではありません。

まずは、ここまでできればOKです。

- LuCIでStatusやInterfacesを見られる
- WAN / WAN6 / LAN / Wi-Fiの状態をざっくり確認できる
- SSHでLN6001-JPへログインできる
- `ubus call system board` や `ifstatus wan` を実行できる
- `logread` で直近ログを見られる
- 設定変更前にバックアップを取る感覚が分かる

OpenWrtは、深く触ろうと思えばかなり深いです。

でも最初は、

```txt
LuCIで見る
SSHで照合する
変更前にバックアップする
```

これだけで十分です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-01.png)

## 先にざっくり結論

最初は次の3つができれば十分です。

1. LuCIの **Status → Overview** と **Network → Interfaces** を見られる
2. SSHで `root@192.168.1.1` にログインできる
3. `ubus call system board`、`ifstatus wan`、`logread | tail -n 80` を実行できる

OpenWrtベースのルーターは、LuCIとSSHが別々の世界にあるわけではありません。

LuCIは、ブラウザで見やすくした管理画面です。  
SSHは、同じルーターの中をコマンドで確認する入口です。

つまり、

```txt
LuCI = 画面で見る
SSH = 文字で見る
```

くらいで考えると分かりやすいです。

最初から `uci set` や `uci commit` をコピペして設定変更する必要はありません。

むしろ、最初はやらなくていいです。

まずは読む。  
見る。  
控える。  
バックアップする。

そこからで大丈夫です。

## こういう人向けです

この記事は、次のような人向けです。

- OpenWrtベース製品を初めて触る
- LuCIとSSHのどちらから始めればいいか迷っている
- まず状態確認だけできるようになりたい
- トラブル時にWAN、DNS、Wi-Fi、DHCPのどこで止まっているか見たい
- CLIには興味があるけど、設定を壊すのが怖い
- コマンドは使ってみたいが、最初は読み取りだけにしたい
- ゲストWi-Fi、VLAN、VPNへ進む前に基本画面を押さえたい

逆に、すでにOpenWrtのCLI運用に慣れている人には、かなり基本寄りです。

でも、慣れている人でも「最初はLuCIで見て、SSHで照合する」という考え方はかなり実用的だと思います。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-02.png)

## 最初に言葉だけそろえる

最初に出てくる用語だけ、軽く整理しておきます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-01.png)

最初は全部を覚えなくて大丈夫です。

まずは、

```txt
LuCI = 画面
SSH = 黒い画面
UCI = OpenWrtの設定管理
logread = ログを見る
```

くらいで十分です。

## LuCIとSSHの使い分け

最初は、LuCIを主役、SSHを補助として考えます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-02.png)

SSHが使えると、一気に“玄人っぽく”見えます。

でも最初の価値はそこではありません。

SSHの本当の便利さは、LuCIだけでは見えにくいログや状態を確認できることです。

つまり、SSHは最初から設定を壊すための道具ではありません。

```txt
状態を見るための道具
```

です。

## まずLuCIへログインする

LN6001-JPの管理画面は、ブラウザから開きます。

PCまたはスマートフォンを、LN6001-JPに接続します。

- 有線LAN: LN6001-JPのLANポートとPCをLANケーブルで接続
- Wi-Fi: 本体底面ラベルに記載されたデフォルトSSIDへ接続

初期設定や設定変更を行う時は、有線LAN接続がおすすめです。

Wi-Fi設定を変更すると、無線接続が一度切れることがあります。

有線のほうが落ち着いて作業できます。

ブラウザのアドレスバーに次を入力します。

```txt
https://192.168.1.1
```

ユーザー名は次の通りです。

```txt
root
```

パスワードは、本体底面ラベルに記載された初期パスワード、または初期設定後に変更した管理者パスワードです。

初回アクセス時に証明書警告が出ることがあります。

LN6001-JPへ直接つないだ状態で、アクセス先が `https://192.168.1.1` であることを確認したうえで、ブラウザの詳細設定から続行します。

## LuCIで最初に見る画面

LuCIにはたくさんのメニューがあります。

最初から全部を理解しようとしなくて大丈夫です。

まずは、次の2種類に分けて覚えると楽です。

```txt
状態を見る画面
設定を変える時に触る画面
```

最初は「見る画面」を中心にします。

## Status → Overview

ログイン直後にまず見る画面です。

ここは、ルーターの健康診断みたいな場所です。

確認するポイントは次の通りです。

- ホスト名
- ファームウェアバージョン
- 稼働時間
- CPU負荷
- メモリ使用量
- WAN / WAN6 / LANの状態
- WAN側IPアドレス
- 接続中の端末数

インターネットにつながらない時は、まずここでWAN側にIPアドレスが付いているかを見ます。

WANにIPアドレスが付いていなければ、回線機器、WANケーブル、回線方式、PPPoE/IPoE設定のどこかを見ます。

WANにIPアドレスが付いているのにWebサイトが開けない場合は、DNSやルーティングの可能性を見ます。

「Wi-Fiが悪い」と思っていたら、実はWANやDNSだった、ということはよくあります。

## Network → Interfaces

WAN、WAN6、LANなどの状態を確認する画面です。

確認するポイントは次の通りです。

- WANのProtocol
- WAN側IPアドレス
- Gateway
- DNS
- WAN6の状態
- LAN側IPアドレス
- DHCPで端末にIPアドレスを配っているか

PPPoE回線を使う場合は、この画面からWANのProtocolをPPPoEへ変更し、プロバイダのIDとパスワードを入力します。

ただし、日本のNTTフレッツ系IPoEやIPv4 over IPv6を使う場合は、契約方式やホームゲートウェイの有無によって手順が変わります。

IPoEまわりは別記事で扱います。

この005では、まず

```txt
WANとWAN6の状態を見られる
```

ところまでをゴールにします。

## Network → Wireless

Wi-FiのSSID、暗号化方式、パスワード、radioの状態を確認・変更する画面です。

LN6001-JPでは、2.4GHz / 5GHz / 6GHzのradioを扱います。

この画面で見るポイントは次の通りです。

- どのSSIDが有効か
- どのradioにSSIDが紐づいているか
- 暗号化方式
- Wi-Fiパスワード
- 接続中のクライアント
- 2.4GHz / 5GHz / 6GHzの状態

SSIDやWi-Fiパスワードを変更する場合は、LuCIで対象のSSIDを選び、**Edit** から変更します。

Wi-Fi設定を反映すると、無線接続は一度切れます。

これは正常です。

新しいSSIDとWi-Fiパスワードで再接続してください。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

## Network → DHCP and DNS

DHCPとDNSまわりを見る画面です。

ここは、端末にIPを配ったり、名前解決を扱ったりする場所です。

確認するポイントは次の通りです。

- 端末へIPアドレスを配っているか
- DHCPリースの一覧
- 固定DHCP割り当て
- DNS転送設定
- ローカル名解決

家庭や小さなオフィスでは、NAS、プリンター、監視カメラ、ミニPCなどに固定DHCP割り当てを使うことがあります。

ただし、初日は無理に触らなくて大丈夫です。

まずは、接続中の端末にIPアドレスが配られていることを確認できれば十分です。

## Network → Firewall

firewall zoneや転送ルールを見る画面です。

名前がちょっと怖いですが、最初はこう考えれば大丈夫です。

```txt
LAN = 家や事務所の内側
WAN = インターネット側
zone = 部屋
forwarding = 部屋から部屋へ行ってよいか
```

ゲストWi-Fi、VLAN、VPNを使い始めると、この画面を見る機会が増えます。

ただし、firewallは設定ミスの影響が大きい場所です。

最初は、

```txt
LANとWANが分かれている
ゲストWi-Fiを作る時にzoneを意識する
```

くらいで十分です。

いきなりCLIからfirewallを書き換えなくて大丈夫です。

## System → Administration

管理者パスワードとSSHアクセスを確認する画面です。

見るポイントは次の通りです。

- Router Password
- SSH Access
- SSHの待ち受けポート
- SSHを受け付けるインターフェース
- SSH鍵の設定

管理者パスワードは初期設定で変更しておきます。

SSHはLAN側からだけ入れる状態にしておくのがおすすめです。

WAN側からSSHを開ける必要は、家庭や小さなオフィスではほとんどありません。

外出先から管理したい場合は、SSHをインターネットへ直接開けるより、TailscaleやWireGuardなどのVPNを検討したほうが安全に運用しやすいです。

## System → Backup / Flash Firmware

設定バックアップとファームウェア更新の画面です。

ここはかなり大事です。

設定を変更する前は、ここからバックアップを取ります。

手順はシンプルです。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れて保管する

例:

```txt
backup-LN6001-before-wifi-change-20260621.tar.gz
```

バックアップファイルには重要な設定が含まれます。

SNS、公開リポジトリ、ブログ記事の添付などへそのままアップロードしないでください。

OpenWrt系ルーターで一番強い人は、全部の設定を暗記している人ではありません。

変更前にバックアップを取っている人です。

## System → Software

パッケージを確認・追加する画面です。

OpenWrt系では、opkgを使って機能を追加できます。

ただし、最初から大量にパッケージを追加しないほうが安全です。

特にVPNまわりは、LN6001-JPではLinksys公式のVPN Assistantモジュールを使う前提の構成があります。

いきなり一般的なOpenWrt手順をそのまま実行するのではなく、まず公式手順とこの連載の該当記事を確認してください。

パッケージ追加は楽しいですが、ルーターはPCではありません。

必要なものだけ、1つずつ入れるのがおすすめです。

## LuCIで見た内容をSSHで照合する

LuCIで画面を見たあと、SSHでは同じ状態をもう少し詳しく確認します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-03.png)

この表のコマンドは、基本的に読み取り用です。

最初は、設定を変えずに状態を見られるだけでかなり役に立ちます。

## SSHでLN6001-JPに接続する

SSHは、ルーターの中をコマンドで確認するための入口です。

最初は「LuCIで見えない情報を見る道具」と考えます。

## SSH接続前に確認すること

SSH接続前に、次の点を確認します。

- PCがLN6001-JPのLAN側に接続されている
- 管理者パスワードを控えている
- LuCIへログインできる
- SSH AccessがLAN側で有効になっている
- WAN側からSSHを開けていない

SSHの設定は、LuCIの **System → Administration** で確認できます。

家庭や小さなオフィスでは、SSHはLAN側からだけ使えれば十分です。

## Macから接続する

Macでは、ターミナルを開いて次を実行します。

```sh
ssh root@192.168.1.1
```

初回接続時は、接続先を信頼するか聞かれます。

接続先がLN6001-JPであることを確認して、`yes` と入力します。

パスワードを求められたら、管理者パスワードを入力します。

入力中は画面に文字が表示されません。

これは正常です。

## Windowsから接続する

Windows 10/11では、PowerShellまたはWindows TerminalからSSHを使えます。

```powershell
ssh root@192.168.1.1
```

PuTTYを使う場合は、次のように設定します。

- Host Name: `192.168.1.1`
- Port: `22`
- Connection type: `SSH`
- Login as: `root`

Windows / MacのSSHログイン手順は、別記事でもう少し丁寧に扱います。

## ホスト鍵の警告が出た時

ルーターを初期化した後や、同じIPアドレスに別の機器をつないだ後は、SSHのホスト鍵警告が出ることがあります。

自分がLN6001-JPへ接続していることを確認したうえで、PC側の古いホスト鍵を削除します。

MacやWindowsのOpenSSHなら、PC側で次を実行します。

```sh
ssh-keygen -R 192.168.1.1
```

その後、もう一度SSH接続します。

```sh
ssh root@192.168.1.1
```

警告が出た時は、反射的に無視しないでください。

「本当に自分のLN6001-JPへつないでいるか」を確認してから進めます。

## 最初に実行する読み取りセット

SSHでログインできたら、まず次のセットを実行します。

設定は変更しません。

```sh
echo "### system"
date
ubus call system board
uptime
free
df -h

echo "### wan"
ifstatus wan
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### lan clients"
cat /tmp/dhcp.leases

echo "### recent logs"
logread | tail -n 80
```

この出力を見るだけで、かなり多くのことが分かります。

- ルーターがどのファームウェアで動いているか
- 起動してからどのくらい経っているか
- メモリやストレージに余裕があるか
- WAN / WAN6がどう見えているか
- デフォルトルートがあるか
- LAN側端末にIPアドレスが配られているか
- 直近ログにエラーが出ていないか

トラブル時は、まずこの読み取りセットから始めると、闇雲に設定を触らずに済みます。

## よく使う読み取りコマンド

ここからは、場面別によく使うコマンドを整理します。

すべて最初は読み取り用として使います。

## システム情報を見る

```sh
ubus call system board
uptime
free
df -h
```

`ubus call system board` では、ファームウェア情報や機種情報を確認できます。

`uptime` では稼働時間、`free` ではメモリ、`df -h` ではストレージ使用量を見ます。

特に `df -h` は、パッケージ追加前にも見ておくと安心です。

## WAN接続を見る

```sh
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
```

`ifstatus wan` と `ifstatus wan6` は、回線トラブル時にかなり使います。

WAN側IPアドレス、Gateway、DNS、IPv6アドレス、接続状態などを確認できます。

`ip route show` でIPv4の経路、`ip -6 route show` でIPv6の経路を見ます。

## DNSと疎通を見る

```sh
ping -c 4 8.8.8.8
ping -c 4 example.com
nslookup example.com
```

`8.8.8.8` へpingが通るのに `example.com` が引けない場合は、DNSまわりの問題が疑われます。

どちらも通らない場合は、WAN接続、デフォルトルート、回線機器側を見ます。

「インターネットが死んだ」と見えて、実はDNSだけだった、ということはかなりあります。

## LAN側端末を見る

```sh
cat /tmp/dhcp.leases
ip neigh show
```

`cat /tmp/dhcp.leases` では、DHCPでIPアドレスを取得した端末を確認できます。

`ip neigh show` では、近隣端末のIPアドレスとMACアドレスの対応を確認できます。

スマートフォン、PC、プリンター、NAS、カメラなどが想定通り見えているかを確認する時に使います。

## Wi-Fi状態を見る

```sh
uci show wireless | grep -E 'ssid|encryption|disabled|network'
wifi status
iw dev
```

SSID、暗号化方式、有効/無効状態、radioの状態、無線インターフェースを確認します。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

Wi-Fiの変更は、最初はLuCIの **Network → Wireless** から行うのがおすすめです。

CLIでは、まず読むだけで十分です。

## DHCP / DNS設定を見る

```sh
uci show dhcp
cat /tmp/resolv.conf.d/resolv.conf.auto 2>/dev/null
cat /etc/resolv.conf
logread | grep -i dnsmasq | tail -n 50
```

DHCPやDNSまわりの設定、上流DNS、dnsmasq関連ログを確認します。

DNS広告ブロックや固定DHCP割り当てを使うようになると、このあたりを見る機会が増えます。

## firewall設定を見る

```sh
uci show firewall
iptables-save 2>/dev/null | head -n 120 || true
ip6tables-save 2>/dev/null | head -n 120 || true
```

OpenWrt系では、ファームウェア世代によってfirewallの実装や表示方法が変わることがあります。

最初は `uci show firewall` で設定を見るだけで十分です。

`iptables-save` や `ip6tables-save` は、実際に展開されたルールを確認したい中上級者向けです。

この段階でfirewallをCLIから書き換える必要はありません。

## ログを見る

```sh
logread | tail -n 100
logread | grep -i netifd | tail -n 50
logread | grep -i dnsmasq | tail -n 50
logread | grep -i wireless | tail -n 50
logread | grep -iE 'error|fail|warn' | tail -n 50
```

ログを見ると、LuCIだけでは見えにくいヒントが出てきます。

- WAN接続が上がっているか
- DHCPが動いているか
- DNSでエラーが出ていないか
- Wi-Fi radioが再起動していないか
- firewallやパッケージでエラーが出ていないか

困った時は、まず `logread | tail -n 100` だけでも見てみる価値があります。

## 設定変更前に取る軽い控え

LuCIでWi-Fi、DHCP、firewall、LAN IPなどを変更する前に、SSHでテキスト控えを取っておくと比較しやすくなります。

```sh
BACKUP_DIR="/root/config-check-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json"
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

これは復元用の正式バックアップではなく、変更前の控えです。

復元用には、LuCIの **System → Backup / Flash Firmware** から `Generate archive` を使ってPCへ保存します。

必要なら、PC側から次のように取得します。

```sh
scp -r root@192.168.1.1:/root/config-check-YYYYMMDD-HHMM ./
```

この控えにもSSID、パスワード、ネットワーク設定などが含まれることがあります。

公開しないように注意してください。

## UCIは「読む」から始める

UCIはOpenWrt系の設定管理の中心です。

ただし、最初からUCIで設定を書き換える必要はありません。

まずは読むところから始めます。

```sh
uci show network
uci show wireless
uci show firewall
uci show dhcp
uci show system
```

特定の値だけ見ることもできます。

```sh
uci get network.lan.ipaddr
uci show wireless | grep ssid
```

UCIで設定を変更する時は、基本的に次の流れになります。

```sh
uci set ...
uci changes
uci commit ...
/etc/init.d/... restart
```

ただし、この流れは慣れてからで十分です。

慣れないうちは、CLIでは `uci show` と `uci get` だけを使い、設定変更はLuCIで行うほうが安全です。

ここで無理にCLI職人を目指さなくて大丈夫です。

## `Save` と `Save & Apply` の違い

LuCIを使う時に少し混乱しやすいのが、`Save` と `Save & Apply` の違いです。

ざっくり言うと、次のイメージです。

- Save: 画面上で変更を保存候補にする
- Save & Apply: 実際の設定として反映する

SSIDやWi-Fiパスワードを変更しただけでは、まだ反映されていないことがあります。

画面上部に保留中の変更が出ている場合は、内容を確認してから **Save & Apply** します。

反映後、Wi-Fiやネットワークが一時的に切れることがあります。

特にWireless、Network、Firewallの変更では、切断が発生しても慌てず、少し待ってから再接続します。

ここで焦ってリセットボタンを押したくなります。

分かります。

でも、まだ押さなくて大丈夫です。

まずは再接続先のSSIDや、新しいLAN IPを確認します。

## 最初にやらないほうがよいこと

LuCIとSSHに慣れてくると、いろいろ触りたくなります。

ただ、最初の段階では次の操作は避けたほうが安全です。

- WAN側からSSHを開ける
- 無線の国設定を変更する
- 送信出力を変更する
- DFS関連の値を変更する
- firewallをCLIで一括変更する
- VLANを一気に作る
- 長いUCIスクリプトを中身を読まずに貼り付ける
- opkgで大量のパッケージを入れる
- VPNを複数方式まとめて試す
- バックアップなしでLAN IPアドレスを変更する

OpenWrtベースのルーターは自由度が高いぶん、1つの変更が複数の場所に影響します。

最初は、

```txt
読む
控える
1つ変える
確認する
バックアップする
```

の順番で進めるのが安全です。

## トラブル時の見方

LuCIとSSHを使えるようになると、トラブルの切り分けがかなり楽になります。

## インターネットにつながらない

まずはLuCIで **Status → Overview** と **Network → Interfaces** を見ます。

SSHでは次を確認します。

```sh
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
ping -c 4 8.8.8.8
ping -c 4 example.com
logread | grep -i netifd | tail -n 50
```

WANにIPアドレスがない場合は、WANケーブル、回線機器、回線方式を見ます。

IPアドレスがあるのに名前解決できない場合は、DNSを見ます。

## Wi-Fiにはつながるが通信できない

まず、端末がIPアドレスを取れているかを見ます。

```sh
cat /tmp/dhcp.leases
ip neigh show
logread | grep -i dnsmasq | tail -n 50
```

端末がDHCPリースに出てこない場合は、Wi-Fi接続、DHCP、SSIDの紐づくネットワークを見ます。

端末にIPアドレスはあるのに通信できない場合は、DNS、gateway、firewallを見ます。

## SSHでログインできない

確認する順番は次の通りです。

1. PCがLN6001-JPのLAN側に接続されているか
2. ルーターのIPアドレスが `192.168.1.1` のままか
3. 管理者パスワードが合っているか
4. LuCIの **System → Administration** でSSH Accessが有効か
5. ホスト鍵警告で止まっていないか
6. 既存ルーターやHGWとIPアドレスが重複していないか

IPアドレスを変更した場合は、新しいアドレスで接続します。

```sh
ssh root@192.168.10.1
```

「SSHできない」のではなく、接続先IPが変わっているだけのこともあります。

## LuCIで変更を反映したら接続が切れた

Wi-Fi名、Wi-Fiパスワード、LAN IPアドレス、firewall、VLANを変更すると、接続が切れることがあります。

これは必ずしも失敗ではありません。

変更内容に応じて、次のように対応します。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-04.png)

変更前にバックアップを取っておくと、落ち着いて戻しやすくなります。

## OpenWrtの設定ファイルをざっくり理解する

OpenWrt系の設定は、主に `/etc/config/` 配下にあります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-05.png)

LuCIで変更した内容も、裏側ではこれらの設定に反映されます。

つまり、LuCIとSSHは別世界ではありません。

同じ設定を、画面で見るか、テキストで見るかの違いです。

ここが分かると、OpenWrtが急に怖くなくなります。

## まとめ

LN6001-JPでLuCIとSSHを使い始める時は、次の順番で考えると迷いにくくなります。

1. LuCIで全体を見る
2. SSHで詳しい状態を読む
3. 設定変更前にバックアップを取る
4. 変更はまずLuCIで行う
5. CLIでの変更は慣れてからにする

最初の成功条件は、次の3つです。

- LuCIのOverviewとInterfacesでWAN/LAN/Wi-Fiの状態を見られる
- SSHでログインして読み取りコマンドを実行できる
- 設定変更前にLuCIバックアップとテキスト控えを残せる

CLIは、難しくするためのものではありません。

見えなかった情報を見えるようにするための道具です。

この感覚を持っておくと、IPoE、ゲストWi-Fi、VLAN、VPN、トラブル対応の記事へ進んだ時にもかなり楽になります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-03.png)

## 次に読むなら

LuCIとSSHの基本が見えてきたら、次は目的に合わせて進みます。

- [SSH接続: WindowsとMacからLN6001-JPへログインする](https://note.com/ikmsan/n/na751d7336b87)
- [SSHで見る基本情報](https://note.com/ikmsan/n/n99897dd3cae6)
- [LN6001-JP初期設定チェックリスト](https://note.com/ikmsan/n/n50a565a008a0)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

CLIに慣れたい人は、Windows/MacのSSHログイン記事へ。

トラブル対応の見方を深めたい人は、SSHで見る基本情報と、つながらない時の切り分けへ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### LN6001-JPは最初からSSHで触るべき？

いいえ。

最初はLuCI中心で十分です。

SSHは、LuCIだけでは見えにくいログや詳細状態を確認するための補助として使うのがおすすめです。

### LuCIとSSHはどちらが安全？

日常的な設定変更はLuCIのほうが安全に進めやすいです。

SSHは便利ですが、設定ファイルを直接触れるぶん、ミスの影響も大きくなります。

最初は「SSHで読む、変更はLuCI」という分け方が分かりやすいです。

### 最初に覚えるSSHコマンドは？

まずは次の4つで十分です。

```sh
ubus call system board
ifstatus wan
cat /tmp/dhcp.leases
logread | tail -n 80
```

この4つで、本体情報、WAN状態、LAN側端末、直近ログを確認できます。

### `uci set` は使わないほうがいい？

慣れるまでは、設定変更には使わなくて大丈夫です。

`uci show` や `uci get` で読むことから始め、変更はLuCIで行うほうが安全です。

UCIで変更する場合は、必ずバックアップを取り、`uci changes` で差分を確認してから反映します。

### SSHをWAN側から開けてもいい？

おすすめしません。

家庭や小さなオフィスでは、SSHはLAN側からだけ使えれば十分です。

外出先から管理したい場合は、SSHをインターネットへ直接開けるより、VPN経由で入る構成を検討してください。

### パッケージはopkgで自由に入れていい？

OpenWrt系なのでopkgで機能追加できますが、最初から大量に入れるのはおすすめしません。

特にVPNやIPoEまわりは、LN6001-JP向けの公式手順や追加モジュールがある場合があります。

まずは必要な機能を1つずつ追加し、変更前後でバックアップを取ります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へログインする方法: https://support.linksys.com/kb/article/7038-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041/
- Linksys MBE70 WRT Pro 7 FAQs: https://support.linksys.com/kb/article/217-en/?section_id=175
- OpenWrt User Guide: https://openwrt.org/docs/guide-user/start
- OpenWrt Wiki - Opkg package manager: https://openwrt.org/docs/guide-user/additional-software/opkg
- OpenWrt Wiki - Logging messages: https://openwrt.org/docs/guide-user/base-system/log.essentials

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
