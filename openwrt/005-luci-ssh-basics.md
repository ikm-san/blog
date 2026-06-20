<!-- mirror-source: articles/005-luci-ssh-basics.md -->

# LuCIとSSHの基本: LN6001-JPで最初に触る画面とコマンド【OpenWrt集中連載005】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPはOpenWrtベースの製品ですが、最初からCLIだけで操作する必要はありません。

基本はLuCIというブラウザ管理画面で確認・設定し、LuCIで見えにくい情報だけSSHで補うくらいが、いちばん無理のない使い方です。

OpenWrtというと、黒い画面で大量のコマンドを打つイメージを持つ人もいます。もちろんSSHで深く触ることもできますが、家庭や小さなオフィスなら、最初は「状態確認」に使うだけでも十分役に立ちます。

特にトラブル時は、LuCIだけでは見えない情報をSSHで確認できることが大きな強みになります。WANがつながっているか、IPアドレスが取れているか、DNSが動いているか、といった切り分けがしやすくなるためです。

この記事では、LuCIの主要な画面の見方と、SSHでの接続方法、最初に覚えておくと便利なコマンドをまとめます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-01.png)

## この記事でわかること

- LuCIとSSHをどう使い分けると分かりやすいか
- LN6001-JPで最初に見るべき画面とコマンド
- 設定変更前に確認しておきたいポイント
- SSHを「変更」ではなく「確認」に使う考え方

## こんな人に向いています

- OpenWrtベース製品を初めて触る
- LuCIとSSHのどちらから始めるべきか迷っている
- まずは状態確認だけできるようになりたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-02.png)

## まずはここまでで十分

この段階では、全部のコマンドを覚える必要はありません。むしろ、最初は「今どうなっているか」を確認できれば十分です。

まずは次の3つで十分です。

1. LuCIのOverviewとInterfacesを見られるようにする
2. SSHでログインできるようにする
3. `ubus call system board` と `ifstatus wan` を実行できるようにする

最初は「設定変更」より、「今の状態が分かること」を優先したほうが入りやすいです。

## LuCIで見た内容をSSHで照合する

LN6001-JPでは、LuCIを主役にして、SSHは「画面で見た内容をもう少し詳しく確認する道具」と考えると扱いやすくなります。

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。まずは次のように、LuCIの画面と読み取りコマンドを対応させて覚えるのがおすすめです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-01.png)

この表のコマンドは基本的に読み取り用です。最初は `uci set` や `uci commit` を使わなくても、状態を見られるだけで十分役に立ちます。

### 最初に実行する読み取りセット

SSHログインできたら、まずこのセットを実行して、LuCIのOverviewと見比べます。

```sh
echo "### system"
ubus call system board
uptime
free
df -h

echo "### wan"
ifstatus wan
ifstatus wan6

echo "### lan clients"
cat /tmp/dhcp.leases

echo "### recent logs"
logread | tail -n 80
```

この段階では設定は変わりません。トラブル時も、まずはこの出力で「本体」「WAN」「LAN端末」「ログ」を分けて見ます。

### 設定変更前に取る軽いバックアップ

LuCIでWi-Fi、DHCP、firewall、LAN IPなどを変更する前に、SSHでテキスト控えを取っておくと、何を変えたか比較しやすくなります。

```sh
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ls -l "$BACKUP_DIR"
```

本格的な復元用には、LuCIの **System** → **Backup / Flash Firmware** からPCへバックアップを保存しておきます。SSHのバックアップは「差分確認用の控え」と考えると分かりやすいです。

## 用語ミニ解説

- LuCI: ブラウザで開くOpenWrt系の管理画面です。
- SSH: ルーターへ文字コマンドでログインする方法です。状態確認や詳細ログ確認で役立ちます。
- CLI: コマンドライン操作のことです。
- UCI: OpenWrt系の設定管理の仕組みです。`uci show network` のように使います。
- logread: ルーターのログを見るコマンドです。トラブル時の手がかりになります。

## LuCIの主要メニューと使い方

LuCIは、OpenWrt系ルーターをブラウザから管理するための画面です。

最初は全部を理解しようとしなくて大丈夫です。まずは「どこを見るとWAN状態が分かるか」「どこでWi‑Fi設定を変えるか」だけ覚えるとかなり扱いやすくなります。

### Status（概要）

ログイン直後に表示されます。

**確認できる主な情報:**
- ルーター名・ファームウェアバージョン
- CPU負荷・メモリ使用量
- 稼働時間
- WAN/LAN/WAN6の接続状態・IPアドレス
- 接続中の端末数

インターネットにつながらない時は、まずここのWAN/WAN6の状態を確認します。

「WANにIPアドレスが付いているか」を見るだけでも、回線側なのかWi‑Fi側なのかの切り分けがかなり楽になります。

### Network > Interfaces（インターフェース状態）

**確認・設定できること:**
- WAN / WAN6 / LAN の状態と詳細
- IPアドレス、GW、DNS
- PPPoE / DHCP / 静的IP の切り替え

OpenWrt系では、ネットワーク設定の中心になる画面です。回線がつながらない時も、まずここを見ることが多くなります。

**PPPoE設定の手順（PPPoE回線の場合）:**
1. **Network** → **Interfaces** を開く
2. **WAN** の **Edit** をクリック
3. **Protocol** を `PPPoE` に変更
4. **PAP/CHAP username** にプロバイダのID、**Password** にパスワードを入力
5. **Save & Apply** をクリック
6. WAN statusが `Connected` になることを確認

PPPoE利用時は、契約書やプロバイダメールに書かれている接続IDとパスワードが必要です。

### Network > Wireless（Wi-Fi設定）

**確認・設定できること:**
- 2.4GHz / 5GHz / 6GHz の各SSIDの状態
- SSID名・暗号化方式・パスワードの変更
- 無線の有効/無効の切り替え

家庭で触る機会が最も多いのがこの画面です。SSID変更、Wi‑Fiパスワード変更、ゲストWi‑Fi追加などもここから行います。

**SSID設定の手順:**
1. **Network** → **Wireless** を開く
2. 変更したいSSIDの **Edit** をクリック
3. **Interface Configuration** タブ:
   - **ESSID**: Wi-Fi名を入力
   - **Mode**: `Access Point`（通常はそのまま）
4. **Wireless Security** タブ:
   - **Encryption**: `WPA2-PSK` または `WPA3-SAE`
   - **Key**: Wi-Fiパスワード（8文字以上）
5. **Save** → **Apply**

### Network > Firewall（ファイアウォール）

**確認・設定できること:**
- zone（LAN/WAN/Guest等）の設定
- zone間の転送ルール
- ポートフォワーディング

Guest Wi‑FiやVLANを使い始めると、この画面を見る機会が増えます。最初は「LANとWANを分けている設定」くらいの理解でも十分です。

### System > Backup / Flash Firmware（バックアップ・更新）

設定変更前は、できるだけここからバックアップを取るようにします。

**バックアップ手順:**
1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** ボタンをクリック
3. `.tar.gz` ファイルをPCに保存

**ファームウェア更新手順:**
1. Linksys公式からファームウェアをダウンロード
2. **Flash new firmware image** セクションで **Choose File** をクリック
3. ダウンロードしたファイルを選択
4. **Flash image** をクリック
5. 確認画面で **Proceed** をクリック（接続が切れる）

ファームウェア更新中は電源を切らず、有線LAN接続で行うほうが安全です。

### System > Administration（管理設定）

**設定できること:**
- 管理パスワード（Router Password セクション）
- SSH アクセス設定（SSH Access セクション）
  - **Interface**: どのインターフェースからSSHを受け付けるか（初期はLANのみを推奨）
  - **Port**: SSHポート番号（デフォルト22）

SSHを有効にしたまま使う場合でも、WAN側からSSHを開けない構成にしておくほうが安心です。

## SSHでLN6001-JPに接続する

SSHは、ルーターの中を少し詳しく見るための入口です。

最初は「CLIで全部設定する」ためではなく、「LuCIで見えない状態を確認する」くらいの感覚で十分です。

### 接続前の確認

SSHは初期状態で有効になっています。LuCIの **System** → **Administration** → **SSH Access** セクションで接続許可インターフェースを確認できます。

セキュリティ上、SSHアクセスはLANポートのみに限定しておくことをおすすめします（WANからのSSHは無効のまま）。

普段使いでは、家やオフィスの内部ネットワークからだけ接続できれば十分なケースがほとんどです。

### Macからの接続

```sh
ssh root@192.168.1.1
# パスワードを入力（管理パスワードと同じ）
```

### Windowsからの接続

**Windows 10/11の場合（OpenSSH使用）:**
1. PowerShellまたはコマンドプロンプトを開く
2. 以下を実行:

```powershell
ssh root@192.168.1.1
```

**PuTTY使用の場合:**
- Hostname: `192.168.1.1`
- Port: `22`
- Connection type: `SSH`
- Login as: `root`

詳細は別記事「SSH接続: WindowsとMacからLN6001-JPへログインする」を参照してください。

### 接続確認

ログイン成功後、プロンプトが表示されます:

```

     MM           NM                    MMMMMMM          M       M
   $MMMMM        MMMMM                MMMMMMMMMMM      MMM     MMM
  MMMMMMMM     MM MMMMM.              MMMMM:MMMMMM:   MMMM   MMMMM
MMMM= MMMMMM  MMM   MMMM       MMMMM   MMMM  MMMMMM   MMMM  MMMMM'
MMMM=  MMMMM MMMM    MM       MMMMM    MMMM    MMMM   MMMMNMMMMM
MMMM=   MMMM  MMMMM          MMMMM     MMMM    MMMM   MMMMMMMM
MMMM=   MMMM   MMMMMM       MMMMM      MMMM    MMMM   MMMMMMMMM
MMMM=   MMMM     MMMMM,    NMMMMMMMM   MMMM    MMMM   MMMMMMMMMMM
MMMM=   MMMM      MMMMMM   MMMMMMMM    MMMM    MMMM   MMMM  MMMMMM
MMMM=   MMMM   MM    MMMM    MMMM      MMMM    MMMM   MMMM    MMMM
MMMM$ ,MMMMM  MMMMM  MMMM    MMM       MMMM   MMMMM   MMMM    MMMM
  MMMMMMM:      MMMMMMM     M         MMMMMMMMMMMM  MMMMMMM MMMMMMM
    MMMMMM       MMMMN     M           MMMMMMMMM      MMMM    MMMM
     MMMM          M                    MMMMMMM        M       M
       M
 ...

root@LN6001-JP:~#
```

この画面が出ればSSHログイン成功です。最初は怖く見えるかもしれませんが、ここから状態確認コマンドを少しずつ覚えていけば十分です。

## SSHでよく使うコマンド

最初は全部覚える必要はありません。

まずは「WANがつながっているか」「IPアドレスが付いているか」「ログにエラーが出ていないか」を確認できるだけでも、LuCIだけの時よりかなり切り分けしやすくなります。

### システム情報の確認

```sh
# ファームウェア・ハードウェア情報
ubus call system board

# CPU・メモリ・稼働時間
cat /proc/uptime
free

# ストレージ使用量
df -h
```

### ネットワーク状態の確認

```sh
# WAN状態を詳細確認（JSONで出力）
ifstatus wan
ifstatus wan6

# IPアドレス一覧
ip addr show

# ルーティングテーブル
ip route show

# DNS設定確認
cat /etc/resolv.conf

# 接続中のDHCPリース（端末一覧）
cat /tmp/dhcp.leases
```

`ifstatus wan` と `ifstatus wan6` は特に重要です。IPアドレス取得状況、GW、DNS、IPv6状態などをまとめて確認できます。

### 設定内容の確認（UCI）

```sh
# ネットワーク設定全体
uci show network

# 無線設定全体
uci show wireless

# ファイアウォール設定
uci show firewall

# DHCP設定
uci show dhcp

# 特定の設定だけ確認
uci get network.lan.ipaddr
uci show wireless | grep ssid
```

### ログの確認

```sh
# 最近のシステムログ（100行）
logread | tail -n 100

# Wi-Fi関連ログのみ
logread | grep -i wireless | tail -n 50

# DHCP関連ログのみ
logread | grep -i dnsmasq | tail -n 50

# WAN接続関連ログ
logread | grep -i netifd | tail -n 50

# エラーのみ抽出
logread | grep -i error | tail -n 50
```

困った時は、まず `logread | tail -n 100` を見るだけでもかなり情報が拾えます。WAN接続、DNS、Wi‑Fi、DHCP関連のエラーが見つかることも多いです。

### パッケージ管理（opkg）

```sh
# インストール済みパッケージ一覧
opkg list-installed

# パッケージを検索
opkg list | grep -i adblock

# パッケージをインストール（例）
opkg update
opkg install adblock luci-app-adblock

# パッケージをアンインストール
opkg remove adblock
```

WireGuard/TailscaleはLinksys公式VPNアシスタントモジュールを使います。`opkg install luci-app-wireguard` を前提にしないでください。

OpenWrt系では `opkg install` で色々追加できますが、最初から大量にパッケージを入れないほうが安定して使いやすいです。

### 設定のバックアップと復元

```sh
# 設定ファイルをバックアップ
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d)
cp /etc/config/wireless /etc/config/wireless.backup.$(date +%Y%m%d)
cp /etc/config/firewall /etc/config/firewall.backup.$(date +%Y%m%d)
cp /etc/config/dhcp /etc/config/dhcp.backup.$(date +%Y%m%d)

# バックアップから復元
cp /etc/config/network.backup.20260507 /etc/config/network
/etc/init.d/network restart
```

## UCIで設定を変更する

UCIはOpenWrt系の設定管理の中心です。

ただし、最初からCLIだけで設定を書き換える必要はありません。まずは `uci show` や `uci get` で現在設定を読むことから始めると、安全に慣れていけます。

### 基本的な変更の流れ

```sh
# 1. 変更前に現在の設定を確認
uci show wireless | grep ssid

# 2. バックアップを取る
cp /etc/config/wireless /etc/config/wireless.backup.$(date +%Y%m%d)

# 3. 実際のSSID変更はLuCIのWireless画面で行う
```

Wi‑FiのUCIセクション名は環境で変わることがあります。

SSHに慣れるまでは、CLIでは確認とバックアップまでに留め、変更はLuCIで行うほうが安全です。

### よく使う変更コマンド

```sh
# タイムゾーン変更
uci set system.@system[0].timezone='JST-9'
uci set system.@system[0].zonename='Asia/Tokyo'
uci commit system
/etc/init.d/system restart

# ホスト名変更
uci set system.@system[0].hostname='LN6001-home'
uci commit system

# LAN IP変更
uci set network.lan.ipaddr='192.168.10.1'
uci commit network
/etc/init.d/network restart
```

## LuCIとSSHの使い分けの目安

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-02.png)

最初は「LuCI中心、SSHは補助」くらいで考えると入りやすいです。

SSHが役立つのは、LuCIで見えない詳細ログや状態確認をしたい時です。特に回線トラブル時は、SSHが使えるだけで切り分けしやすさがかなり変わります。

## OpenWrtの設定構造を理解する

OpenWrtの設定は `/etc/config/` 配下のテキストファイルで管理されています。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/table-03.png)

LuCIで変更した内容も、裏側ではこれらのファイルに反映されます。`uci show <config>` で現在の設定を確認できます。

つまり、LuCIとSSHは別物ではなく、同じ設定を違う見え方で触っているイメージです。

## まとめ

最初の成功条件としては、次の3つが見えれば十分です。最初からCLIを全部覚える必要はありません。

- LuCI の主要画面で WAN、LAN、Wi-Fi の状態が追える
- SSH でログインして基本コマンドを打てる
- 設定変更前にバックアップを取る流れが分かっている

LuCIとSSHの使い方の要点:

1. **LuCI**: 日常的な設定変更・状態確認に使う
2. **SSH**: 詳細ログ確認・状態確認・トラブル対応に使う
3. **UCI**: 設定の確認・変更に使う（変更後は `uci commit` が必要）
4. **設定変更前**: 必ずバックアップを取る

最初はLuCIで状態を確認し、必要な場面だけSSHを使う流れで十分です。

CLIは「全部を難しくするもの」ではなく、「見えなかった情報を確認できる道具」と考えると入りやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/005/diagram-03.png)

## よくある質問

### LN6001-JPは最初からSSHで触るべき？

最初はLuCI中心で十分です。SSHは詳細な状態確認やトラブル時の切り分けに使う、と分けるほうが入りやすくなります。

### LuCIとSSHはどちらが安全？

日常的な設定変更はLuCIのほうが安全です。SSHは便利ですが、設定ファイルを直接触る分だけミスの影響も大きくなります。

最初は「SSHで読む、変更はLuCI」という分け方がおすすめです。

### LN6001-JPでまず覚えるべきSSHコマンドは？

`ubus call system board`、`ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` あたりからで十分です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT web interface login: https://support.linksys.com/kb/article/223-en/?section_id=175
- Linksys OpenWRT SSH setup: https://support.linksys.com/kb/article/226-en/?section_id=175

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
