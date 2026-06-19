<!-- mirror-source: articles/015-monitoring-logs.md -->

> Original note.com article: [監視とログ: 小規模オフィスで見るべき最低限の情報【OpenWrt集中連載015】](https://note.com/ikmsan/n/n8571cacdde40)

# 監視とログ: 小規模オフィスで見るべき最低限の情報【OpenWrt集中連載015】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

小規模オフィスや店舗では、ルーターが止まるだけで業務影響が出ることがあります。

ただし、最初から大規模な監視システムを導入しなくても大丈夫です。

まずは「WANが生きているか」「Wi‑Fi端末が見えているか」「DHCPが配れているか」「直近ログにエラーがないか」を確認できるだけでも、障害時の切り分けはかなり速くなります。

LN6001-JPは、LuCI管理画面とSSHの両方で状態確認できます。

この記事では、平常時に見ておくべき最低限の情報、障害時にどこから確認すると切り分けしやすいか、まず覚えておきたい確認コマンドを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-01.png)

## この記事でわかること

- 小規模オフィスや店舗で最低限見ておきたい項目
- 障害時にどの順番で確認すると切り分けしやすいか
- LuCIとSSHでまず覚えておきたい確認方法

## こんな人に向いています

- 小規模オフィスや店舗で、最低限の監視を運用に入れたい
- 障害時に何から見ればいいか決めておきたい
- 平常時の状態をざっくり把握しておきたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-02.png)

## まずはここまでで十分

高度な監視基盤を最初から入れなくても、まずは次の3つで十分です。

1. LuCI の Overview を定期的に見る
2. DHCPリースとWi‑Fi接続端末を確認できるようにする
3. `logread | tail -n 100` を見られるようにする

平常時の見え方を知っておくだけでも、「いつもと違う」をかなり見つけやすくなります。

最初は「ログを全部読む」より、「どこを見ると異常を見つけやすいか」を覚えるほうが重要です。

監視と言っても、最初からSNMPやGrafanaを入れる必要はありません。

まずはLuCIとSSHで「今どう動いているか」を確認できるだけでも十分役立ちます。

## 3分で取る状態スナップショット

障害時に毎回コマンドを思い出すのは大変です。まずは、以下をひとまとまりで保存できるようにしておくと便利です。

```sh
SNAP_DIR="/tmp/ln6001-snapshot-$(date +%Y%m%d-%H%M)"
mkdir -p "$SNAP_DIR"

date > "$SNAP_DIR/date.txt"
ubus call system board > "$SNAP_DIR/system-board.json"
uptime > "$SNAP_DIR/uptime.txt"
free > "$SNAP_DIR/free.txt"
df -h > "$SNAP_DIR/df-h.txt"
ifstatus wan > "$SNAP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$SNAP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$SNAP_DIR/ip-addr.txt"
ip route show > "$SNAP_DIR/route-v4.txt"
ip -6 route show > "$SNAP_DIR/route-v6.txt"
cat /tmp/dhcp.leases > "$SNAP_DIR/dhcp-leases.txt"
wifi status > "$SNAP_DIR/wifi-status.json"
logread | tail -n 200 > "$SNAP_DIR/logread-tail.txt"

ls -l "$SNAP_DIR"
```

このスナップショットには端末名、MACアドレス、SSID、回線情報が含まれます。サポートへ共有する場合は、必要に応じて伏せ字にします。

## LuCIで確認できる主な情報

### Status > Overview（概要画面）

最初に見る画面です。LN6001-JPへログイン（`https://192.168.1.1`）すると表示されます。

障害時も、まずここを見るだけで「WANなのかWi‑Fiなのか」をかなり切り分けしやすくなります。

確認ポイント:

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-01.png)

最初は「いつもの状態」を覚えておくことが大事です。

例えば、通常時のWi‑Fi接続台数やCPU負荷感を知っておくと、「今日は異常に重い」を見つけやすくなります。

### Network > Wireless > Associated Stations

Wi‑Fi接続中の端末一覧とMACアドレス、SSID、信号強度を確認できます。

「Wi‑Fiがつながらない」という報告があった際に、端末がアソシエート（接続）できているかをここで確認します。

SSIDへ見えていない場合は、認証や電波側の問題を疑いやすくなります。

### Network > DHCP and DNS > Active DHCP Leases

現在DHCPでIPアドレスを割り当て中の端末一覧です。

端末名（ホスト名）、MACアドレス、IPアドレス、リース期限が表示されます。

「Wi‑Fiにはつながっているが通信できない」時は、ここでIP取得できているかを見ると切り分けしやすくなります。

### System > System Log

LuCIからのログ確認です。テキスト形式で表示されます。

最初は全部を理解しようとせず、「err」「fail」「warn」が出ていないかを見るくらいでも十分です。

SSHへ入れるようになると、LuCIより細かい状態確認ができます。

ただし、最初は設定変更より「今どう動いているか」を見る用途で使うくらいでも十分です。

## SSHで確認するコマンド

### システム全体の確認

```sh
echo "### system board"
ubus call system board

echo "### uptime"
uptime

echo "### memory"
free

echo "### storage"
df -h
```

`uptime` と `free` は、まず最初に確認したいコマンドです。

再起動ループやメモリ不足は、小規模オフィス環境でも比較的よく起きる切り分けポイントです。

### ネットワーク接続状態

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### addresses"
ip addr show

echo "### routes"
ip route show
ip -6 route show
```

`ifstatus wan` と `ifstatus wan6` は、まず最初に確認したいコマンドです。

特にIPoE環境では、「IPv6だけ生きている」「IPv4 over IPv6側だけ落ちている」ケースもあります。

### DHCPリース確認

```sh
echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP config"
uci show dhcp
```

`cat /tmp/dhcp.leases` は、「その端末が本当にIP取得できているか」を見る時にかなり便利です。

### Wi‑Fi状態確認

```sh
echo "### Wi-Fi status"
wifi status

echo "### wireless interfaces"
iw dev

echo "### wireless logs"
logread | grep -Ei 'wireless|wifi|wlan|hostapd' | tail -n 80
```

Wi‑Fi問題では、「端末がSSIDへ見えているか」と「IP取得できているか」を分けて確認すると切り分けしやすくなります。

### システムログの確認

```sh
# 直近100行のログ
logread | tail -n 100

# DNS/DHCPに関するログ（dnsmasq）
logread | grep -i dnsmasq | tail -n 50

# ファイアウォール関連のログ
logread | grep -i 'DROP\|REJECT\|firewall' | tail -n 50

# WAN/ネットワーク関連のログ
logread | grep -i 'wan\|pppoe\|ipoe\|ipip' | tail -n 30

# エラーや警告のログ
logread | grep -i 'err\|warn\|fail' | tail -n 50

# 特定時刻以降のログ（例: 午後2時以降）
logread | grep '14:' | tail -n 50
```

最初は「エラーが大量に出ていないか」を見るくらいでも十分です。

特定時刻で grep すると、「その時だけ何が起きたか」をかなり追いやすくなります。

### パッケージとリソース

```sh
# インストール済みパッケージ
opkg list-installed

# プロセス一覧（重いプロセスを確認）
top

# カーネルメッセージ
dmesg | tail -n 50
```

`top` は、「CPUが急に重くなっていないか」を確認する時に便利です。

SQMやAdblock、VPNなどを複数有効化している場合は、負荷確認で役立つことがあります。

障害対応では、「どこから壊れているか」を順番に切り分けることが重要です。

最初から全部を見るより、「WAN → DNS → DHCP → Wi‑Fi」のように順番を固定したほうが整理しやすくなります。

## 障害時の切り分け手順

「インターネットにつながらない」という報告があった場合の確認順序:

### ステップ1: 全体かどうかを確認

- 有線でも起きているか確認（有線端末をルーターのLANポートに直接接続してテスト）
- 複数の端末で起きているか確認

最初に「1台だけの問題なのか」「全体障害なのか」を分けるだけでも、かなり方向性が変わります。

### ステップ2: WANの状態を確認

```sh
# WANの接続状態
ifstatus wan | grep -E '"up"|"ipaddr"|"error"'
ifstatus wan6 | grep -E '"up"|"ip6addr"'
```

LuCIでは **Status** → **Overview** でWAN/WAN6のステータスを確認します。

WAN/WAN6が落ちている場合は、LANやWi‑Fiを見る前に回線側を疑ったほうが切り分けしやすくなります。

### ステップ3: DNSを確認

```sh
# DNS解決テスト
nslookup www.google.co.jp
# または
ping -c 3 8.8.8.8      # IPアドレスへのpingが通ればネット接続自体は問題ない
ping -c 3 google.co.jp  # これが失敗するならDNSの問題
```

「IPアドレスへのpingは通るが名前解決だけ失敗する」場合は、DNS問題の可能性が高くなります。

### ステップ4: DHCPを確認

```sh
# 端末がIPアドレスを取得しているか
cat /tmp/dhcp.leases

# dnsmasqのログを確認
logread | grep -i dnsmasq | tail -n 30
```

Wi‑Fiへ接続できていても、DHCPでIP取得できていなければ通信できません。

### ステップ5: ファイアウォールを確認

最近firewallの設定を変更した場合:

```sh
uci show firewall
logread | grep -i 'DROP\|REJECT' | tail -n 30
```

Guest Wi‑FiやVLANを追加した直後は、firewall設定ミスで通信を止めているケースもあります。

### ステップ6: ログでエラーを確認

```sh
logread | grep -i 'err\|fail' | tail -n 50
```

まずは「fail」「error」「warn」が大量に出ていないかを見るだけでも十分役立ちます。

## 平常時スナップショットを残す

障害時に「いつもと違う」を判断できるよう、平常時の状態をメモしておきます。

小規模オフィスでは、この“平常時メモ”がかなり役立ちます。

### 記録しておくべき項目

```sh
# 以下の出力結果をテキストファイルに保存する
ubus call system board       # ファームウェアバージョン確認
uptime                       # 稼働時間
ifstatus wan                 # WANの状態
ifstatus wan6                # WAN6の状態
ip addr show                 # IPアドレス一覧
cat /tmp/dhcp.leases         # 主要機器のIPアドレス
```

最初は全部を記録しなくても、「WAN状態」「主要機器IP」「通常の接続台数」くらいを残すだけでも十分です。

### 平常時メモの例

```

## LN6001-JP 平常時メモ（作成: 2026-05-07）
- ファームウェア: 1.2.0.15
- LAN IP: 192.168.1.1
- WANプロトコル: IPoE（OCNバーチャルコネクト / MAP-E）
- WAN IPv6: 2404:xxxx:xxxx:xxx::/56（DHCPv6-PD）
- 主要機器:
  - NAS: 192.168.1.10（DHCP予約）
  - プリンター: 192.168.1.20（DHCP予約）
  - カメラ: 192.168.1.31（DHCP予約）
- SSID: MyHome（5GHz/6GHz）、MyHome_2G（2.4GHz）、MyHome_Guest（ゲスト）
- 通常のWi-Fi接続台数: 約10〜15台
```

Markdownやテキストファイルで残しておくだけでも、障害時の比較がかなり楽になります。

## 変更履歴を残す

設定変更後に問題が起きた場合、変更履歴があると原因をかなり絞りやすくなります。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/table-02.png)

```sh
BACKUP_DIR="/root/change-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ls -l "$BACKUP_DIR"
```

最初は「何を変更したか」を1行だけ残す程度でも十分役立ちます。

特にGuest Wi‑Fi、VLAN、SQM、VPN設定は、あとから影響範囲を追いやすくなります。

## サポートへ伝える情報

トラブル時は、「今どうなっているか」を整理して伝えるだけでも、かなりサポートを受けやすくなります。

トラブル時にLinksysサポートや詳しい人に相談する際に必要な情報:

- 製品名: Linksys Velop WRT Pro 7 (LN6001-JP)
- ファームウェアバージョン（System > Software または `ubus call system board`）
- 回線事業者・プロバイダ・IPoE/PPPoE方式
- HGW（ひかり電話ゲートウェイ）の有無
- 問題が始まった時刻・直前の設定変更
- 有線でも起きているか、Wi-Fiだけか
- `logread | tail -n 100` の出力

## まとめ

最初の成功条件としては、次の3つが見えれば十分です。最初はここまで把握できれば大きく外していません。

- WAN、Wi-Fi、DHCP の平常時の状態が把握できている
- 障害時にまず見る画面とコマンドが決まっている
- 設定変更前後の記録を簡単に残せている

最初から高度な監視基盤を入れなくても、「平常時を知っている」だけで障害対応はかなり速くなります。

特に小規模オフィスでは、「どこを見るか」を決めておくだけでも運用負荷をかなり減らせます。

日常的なルーター監視のポイント:

1. **LuCI Status > Overview** でWAN・Wi-Fi・メモリ状態を定期確認
2. `logread | tail -n 100` で直近ログにエラーがないか確認
3. 平常時の状態（WAN IP・主要機器IP・接続台数）をメモとして残す
4. 設定変更前は必ずバックアップを取り、変更履歴を記録する
5. 障害時はWAN → DNS → DHCP → firewall の順で切り分ける

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/015/diagram-03.png)

## よくある質問

### ルーター監視は何から見ればいい？

まずは LuCI の Status > Overview で WAN、Wi‑Fi、メモリ状態を見るだけでも十分です。

そこから必要に応じてSSHでログ確認へ進むと分かりやすくなります。

### 毎日ログを全部見る必要はある？

そこまでは不要です。

平常時の状態を把握しておき、障害時に `logread | tail -n 100` で直近ログを見るだけでもかなり役立ちます。

### 小規模オフィスで最低限残すべきメモは？

WAN IP、主要機器の固定IP、接続台数の目安、最後に設定変更した内容を残しておくと切り分けが速くなります。

特に「最後に何を変えたか」が分かるだけでも、原因特定しやすくなります。

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
