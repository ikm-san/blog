<!-- mirror-source: articles/025-ssh-basic-commands.md -->

> Original note.com article: [SSH基本コマンド集: ルーターの状態確認と設定操作【OpenWrt集中連載025】](https://note.com/ikmsan/n/n99897dd3cae6)

# SSH基本コマンド集: ルーターの状態確認と設定操作【OpenWrt集中連載025】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPへSSH接続すると、LuCIだけでは見えにくい状態確認や詳細ログを確認できます。

特にOpenWrt系では、`ifstatus`、`logread`、`uci show` などを使えるようになるだけでも、トラブル時の切り分けがかなりしやすくなります。

ただし、最初からCLIで全部設定変更しなくても大丈夫です。

まずは「状態確認」「ログ確認」「バックアップ」だけ覚えるくらいでも、かなり運用しやすくなります。

この記事では、LN6001-JPでまず覚えておきたいSSHコマンド、LuCIとCLIの使い分け、設定変更前に意識したいポイントを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-01.png)

## この記事でわかること

- LN6001-JPでまず覚えたいSSHコマンド
- 状態確認と設定変更をどう分けるか
- LuCIとSSHをどう併用すると扱いやすいか
- CLIで触る前に確認したいポイント

OpenWrt系では、LuCIとSSHを両方使えるようになると、かなり状態把握しやすくなります。

最初は「設定変更」より、「今どう動いているか」を見る用途でSSHを使うくらいでも十分です。

## LN6001-JPでのCLI基本作法

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LN6001-JPでは、いきなり設定を書き換えるより、次の順番で進めると安全です。

1. 読み取りコマンドで現在状態を見る
2. 変更対象の設定ファイルをバックアップする
3. `uci set` で小さく変更する
4. `uci changes` で差分を確認する
5. `uci commit <config>` で対象設定だけ保存する
6. 必要なサービスだけ再起動する
7. `ifstatus`、`wifi status`、`logread` で変更後を確認する

特にLAN IP、VLAN、firewall、Wi-Fi設定は、間違えるとLuCIやSSHへ戻れなくなることがあります。変更前に有線LANで接続し、バックアップを取ってから進めます。

### まず取っておくベースライン

SSHへ入ったら、最初にこの程度を控えておくと、あとで状態比較しやすくなります。

```sh
date
ubus call system board
uname -a
uptime
free
df -h

# 主要パッケージの確認
opkg list-installed | grep -E 'base-files|busybox|dnsmasq|firewall|dropbear|uhttpd'
```

この出力にはファームウェア情報や内部IP、ホスト名が含まれることがあります。公開の場所にそのまま貼らず、必要な範囲だけ共有してください。

### 設定変更前のまとめバックアップ

CLIで設定を触る前に、最低限この5つを保存しておくと復旧しやすくなります。

```sh
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ls -l "$BACKUP_DIR"
```

このバックアップはルーター内に保存されます。大きな変更前は、LuCIの **System** → **Backup / Flash Firmware** からPCにもバックアップを保存しておくほうが安全です。

### 変更する時のUCIテンプレート

実際に設定を変更する時は、次の型を崩さないようにします。

```sh
# 1. まず対象設定を読む
uci show network

# 2. 対象ファイルをバックアップ
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)

# 3. 変更を1つだけ入れる（例: LAN IP変更。実行前に接続先が変わる点に注意）
uci set network.lan.ipaddr='192.168.10.1'

# 4. 未保存の変更を確認
uci changes

# 5. 問題なければ保存
uci commit network

# 6. 反映。SSH/LuCIが切れる可能性あり
/etc/init.d/network restart
```

上のLAN IP変更は例です。実際に実行すると管理画面のURLが変わります。普段はLuCIで変更し、CLIでは読み取りとバックアップを中心に使うほうが安全です。

### rollbackの基本

設定変更後におかしくなった場合は、バックアップしたファイルを戻して対象サービスを再起動します。

```sh
# 例: network設定を戻す
cp /etc/config/network.backup.YYYYMMDD-HHMM /etc/config/network
/etc/init.d/network restart

# 例: firewall設定を戻す
cp /etc/config/firewall.backup.YYYYMMDD-HHMM /etc/config/firewall
/etc/init.d/firewall restart

# 例: Wi-Fi設定を戻す
cp /etc/config/wireless.backup.YYYYMMDD-HHMM /etc/config/wireless
wifi reload
```

`network restart` や `wifi reload` は接続が切れることがあります。有線LAN接続と、必要なら本体リセットできる状態を用意してから実行します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-02.png)

## SSHログイン方法

**Mac/Linux（ターミナル）:**

```sh
ssh root@192.168.1.1
```

**Windows（PowerShellまたはコマンドプロンプト）:**

```powershell
ssh root@192.168.1.1
```

初回接続時は fingerprint の確認メッセージが出るので `yes` と入力します。

パスワードはLuCIのadmin（root）パスワードと同じです。

SSH接続そのものの流れは、033の記事でより詳しくまとめています。

最初は、有線LAN接続でSSH確認するほうが切り分けしやすくなります。

---

## システム状態確認コマンド

SSHへ入れたら、まず「今の状態を見る」コマンドから覚えるほうが整理しやすくなります。

```sh
# ルーターのモデル・ファームウェアバージョン確認
ubus call system board

# 起動時間・ロードアベレージ確認
uptime

# メモリ使用量確認
free

# ストレージ使用量確認
df -h

# 実行中のプロセス一覧
ps

# カーネルバージョン確認
uname -r
```

`ubus call system board`、`uptime`、`free` は、まず最初に確認したいコマンドです。

最初は「CPUやメモリが異常に重くないか」を見られるだけでもかなり役立ちます。

---

## ネットワーク状態確認コマンド

OpenWrt系では、「WAN」「LAN」「Wi‑Fi」を分けて確認することがかなり重要です。

```sh
# IPアドレス一覧（全インターフェース）
ip addr show

# ルーティングテーブル
ip route show

# IPv6アドレス
ip -6 addr show

# IPv6ルーティングテーブル
ip -6 route show

# WANインターフェースの詳細状態（JSON形式）
ifstatus wan
ifstatus wan6

# WANのIPアドレスだけ抜き出す
ifstatus wan | grep '"ipaddr"'
ifstatus wan6 | grep '"ip6addr"'

# Wi-Fiインターフェース一覧と状態
wifi status

# 接続中の無線端末一覧
# 実際のwlan名は環境ごとに変わるため、先に iw dev で確認
iw dev
WLAN_IF="ath00"  # iw devで表示されたInterface名に置き換える
iw dev "$WLAN_IF" station dump

# ARPテーブル（LANに接続している端末のMACアドレス）
cat /proc/net/arp

# DHCPリース一覧
cat /tmp/dhcp.leases
```

`ifstatus wan` と `ifstatus wan6` は、まず最初に覚えておきたいコマンドです。

IPoE環境では、「WANは正常だがWAN6だけ落ちている」ケースもあります。

### LN6001-JP向けネットワーク診断パック

原因が分からない時は、次の順番で見るとWAN、IPv6、DNS、DHCPをまとめて確認できます。

```sh
echo "### interfaces"
ifstatus wan
ifstatus wan6

echo "### addresses"
ip addr show

echo "### routes"
ip route show
ip -6 route show

echo "### dns"
cat /tmp/resolv.conf* 2>/dev/null
uci show dhcp | grep -E 'dns|server|dhcp_option|ignore' || true

echo "### dhcp leases"
cat /tmp/dhcp.leases
```

この出力で、WANがupか、IPv4/IPv6アドレスがあるか、DNS配布が不自然でないかをまとめて確認できます。

---

## DNS確認コマンド

DNS問題は、「インターネット全部が落ちた」と誤解しやすいポイントです。

```sh
# DNS解決テスト
nslookup google.co.jp
nslookup google.co.jp 127.0.0.1   # ルーター自身のDNSサーバーで確認
nslookup google.co.jp 8.8.8.8     # GoogleのDNSで直接確認

# pingで疎通確認
ping -c 3 8.8.8.8         # IPv4
ping6 -c 3 2001:4860:4860::8888   # IPv6

# adblockのブロックリスト確認
grep example.com /tmp/adblock/*.txt 2>/dev/null
```

最初は「IPアドレスへpingできるか」と「名前解決だけ失敗しているか」を分けるだけでもかなり役立ちます。

---

## UCIコマンド（設定の確認と変更）

UCI（Unified Configuration Interface）は、OpenWrtの設定システムです。

CLI設定変更では、最初は `uci show` や `uci get` で「読む」ことを優先したほうが安全です。

```sh
# 設定ファイルの内容を表示
uci show network        # ネットワーク設定
uci show wireless       # Wi-Fi設定
uci show firewall       # ファイアウォール設定
uci show dhcp           # DHCP・DNS設定
uci show system         # システム設定

# 特定のパラメータを確認
uci get network.lan.ipaddr
uci show wireless | grep ssid

# 変更前にバックアップ
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d)
cp /etc/config/wireless /etc/config/wireless.backup.$(date +%Y%m%d)

# LAN IPのような基本項目を変更する場合は、LuCIでの操作を優先
# CLIで変更する場合も、実機のセクション名を uci show で確認してから行う
```

最初は「network」「wireless」「firewall」「dhcp」の4つだけ読めればかなり役立ちます。

> **注意**: `uci set` / `uci commit` は設定を実際に変更します。
>
> Wi‑Fi名やLAN IPは環境ごとにセクション名が異なるため、この記事では読み取り中心にしています。
>
> 最初はLuCI側で変更し、SSHでは状態確認中心にするほうが安全です。

---

## サービス管理コマンド

OpenWrt系では、「設定変更後にサービス再起動が必要」なケースがあります。

```sh
# サービスの再起動
/etc/init.d/network restart
/etc/init.d/firewall restart
/etc/init.d/dnsmasq restart    # DNS/DHCPサーバー
/etc/init.d/uhttpd restart     # LuCI (Webサーバー)
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock restart
# TailscaleはLinksys公式モジュール導入後に存在する場合のみ
[ -x /etc/init.d/tailscale ] && /etc/init.d/tailscale restart

# サービスの起動・停止
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock start
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock stop

# 起動時の自動起動を有効化・無効化
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock enable
[ -x /etc/init.d/adblock ] && /etc/init.d/adblock disable

# Wi-Fiの再読み込み
wifi reload

# ネットワーク全体の再起動（切断注意）
/etc/init.d/network restart
```

最初は `dnsmasq`、`network`、`uhttpd` の3つだけ覚えておけばかなり役立ちます。

特に `/etc/init.d/network restart` は通信断が発生するため注意します。

---

## ログ確認コマンド

SSHの強みは、「LuCIでは見えにくいログを直接見られること」です。

```sh
# システムログ（全件）
logread

# 末尾50行
logread | tail -n 50

# リアルタイムでログを表示
logread -f

# dnsmasq（DNS/DHCP）のログ
logread | grep dnsmasq

# ファイアウォール（DROP/REJECT）のログ
logread | grep -E 'DROP|REJECT'

# WAN接続のログ
logread | grep -i wan

# Wi-Fiのログ
logread | grep -i wifi
```

`logread | tail -n 50` は、まず最初に覚えておきたいログ確認方法です。

最初は「error」「fail」「reject」が大量に出ていないかを見るくらいでも十分役立ちます。

### firewallの実適用状態を見る

LuCIやUCIで見えるfirewall設定と、実際に適用されているルールは分けて確認します。

```sh
# 設定として保存されているfirewall
uci show firewall

# 実際に適用されているルール（LN6001-JPではまずiptablesを見る）
command -v iptables >/dev/null && iptables -L -n -v | head -n 80
command -v iptables-save >/dev/null && iptables-save | head -n 120

# nftが入っている環境なら参考確認
command -v nft >/dev/null && nft list ruleset | head -n 120
```

一般的な最新OpenWrt記事では `nft` / fw4 前提の説明を見かけますが、LN6001-JPの製品ファームウェアで確認する時は、まず `uci show firewall` と `iptables` 系の出力を見るほうが現実的です。

### サポート相談前に集めるログ

サポートや詳しい人へ相談する前に、次の読み取りコマンドを取っておくと説明しやすくなります。

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

ただし、ログにはMACアドレス、内部IP、端末名、SSID、プロバイダ情報が含まれることがあります。noteのコメント欄やSNSなど、公開の場所へそのまま貼らないでください。

---

## ファイル操作

OpenWrt系では、設定ファイルを直接読むことで「LuCIで何が設定されているか」を理解しやすくなります。

```sh
# 設定ファイルの確認
cat /etc/config/network
cat /etc/config/wireless
cat /etc/config/firewall

# 設定ファイルをバックアップ
cp /etc/config/network /etc/config/network.bak

# テキストエディタで編集（viが標準搭載）
vi /etc/config/network
```

最初は `cat` で読むだけでもかなり役立ちます。

`vi` 編集は、慣れてから少しずつ触るくらいでも十分です。

> viの基本操作: `i` で挿入モード、`ESC` でコマンドモード、`:wq` で保存・終了、`:q!` で保存せず終了
>
> 最初は「保存せず閉じる（:q!）」を覚えておくだけでも安心感がかなり変わります。

---

## よく使うコマンドの組み合わせ

慣れてくると、「複数確認を1行でまとめる」と状態把握しやすくなります。

```sh
# WANのIPとDNSを一気に確認
ifstatus wan | grep -E '"ipaddr"|"dns"'

# 接続中の全端末（MACアドレス・IP・ホスト名）
cat /tmp/dhcp.leases

# ルーターの基本情報を一画面で確認
ubus call system board && uptime && free && df -h

# 現在の設定を変更する前にバックアップ
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d)
```

最初は `ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` の3つを中心に見るだけでもかなり役立ちます。

---

## まとめ

最初は全部を覚える必要はありません。

まずは `ubus call system board`、`ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` の4つを触れるようになるだけでも、トラブル時の見え方がかなり変わります。

設定変更はLuCI、細かな確認はSSH、という分け方で使うと整理しやすくなります。

最初は「状態確認できること」を優先するだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/025/diagram-03.png)

## よくある質問

### LN6001-JPのSSHで最初に覚えるべきコマンドは？

`ubus call system board`、`ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` の4つからで十分です。

最初は「WAN状態」「接続端末」「直近ログ」が見えるだけでもかなり役立ちます。

### LN6001-JPはSSHで設定変更までやるべき？

慣れるまでは状態確認とバックアップ中心で十分です。

日常的な設定変更はLuCIのほうが安全に進めやすくなります。

### LN6001-JPでSSHが使えると何が便利？

LuCIでは見えにくいログやインターフェース状態を直接確認できます。

特に `ifstatus` や `logread` を見られるようになると、トラブル時の切り分けがかなりしやすくなります。

## 参考リンク

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
