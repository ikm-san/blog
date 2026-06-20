<!-- mirror-source: articles/013-wireguard-tailscale-remote.md -->

# WireGuard/Tailscaleでリモート接続: Linksys公式モジュール前提の設計【OpenWrt集中連載013】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

外出先から自宅や小さなオフィスへ安全に接続するには、VPN設計が必要です。

LN6001-JPでは、Linksys公式サポート（https://support.linksys.com/kb/article/8723-jp/）でWireGuardおよびTailscaleの公式モジュールが案内されています。

ただし、最初から「どちらが最強か」を決める必要はありません。

固定IPやポート開放が使えるならWireGuard、自宅回線条件を気にせず始めたいならTailscale、くらいの分け方でも十分です。

また、VPNは「つながれば終わり」ではなく、「どこまでアクセスを許可するか」もかなり重要です。

この記事では、WireGuardとTailscaleそれぞれの特徴、Linksys公式モジュール前提での設定手順、アクセス範囲の考え方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-01.png)

## この記事でわかること

- LN6001-JPでWireGuardとTailscaleをどう選ぶか
- 固定IPやポート開放が必要かどうか
- Linksys公式モジュール前提での安全な進め方
- VPN接続後のアクセス範囲をどう絞るべきか

## こんな人に向いています

- 外出先から自宅や小さなオフィスへ安全に入りたい
- 固定IPがなくてもリモート接続したい
- WireGuardとTailscaleのどちらから始めるべきか迷っている

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-02.png)

## まずはここまでで十分

VPNも最初から大きく作り込まなくて大丈夫です。むしろ、最初は「必要最低限だけ入れる」ほうが安全です。

まずは次の3つを決めれば十分です。

1. 固定IPやポート開放が使えるか確認する
2. 条件に合うほうをWireGuardかTailscaleから1つ選ぶ
3. 接続後は必要最小限の端末だけアクセス可能にする

まずは「安全につながること」と「入れる範囲を広げすぎないこと」を優先すると整理しやすいです。

VPNは、LAN全体を無条件で公開するより、「必要な機器へだけ入れる」くらいから始めるほうが扱いやすくなります。

## 導入前に現在の状態を保存する

VPN設定は `network`、`firewall`、パッケージ状態、回線方式に影響します。VPN Assistantを入れる前に、現在の状態を控えておきます。

```sh
BACKUP_DIR="/root/vpn-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json"
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"

ls -l "$BACKUP_DIR"
```

VPN設定には鍵や接続先情報が関係するため、設定ファイルやQRコードを公開場所へ貼らないようにします。

## 用語ミニ解説

- VPN: 暗号化されたトンネルで外部から自宅・事務所LANに接続する仕組みです。
- WireGuard: シンプルで高速なVPNプロトコルです。自宅側でVPN入口を運用します。
- Tailscale: WireGuardをベースにしたVPNサービスです。NAT越えを自動処理します。
- NAT越え（NAT Traversal）: 固定IPやポート開放なしで接続を確立する仕組みです。

## VPN方式の選び方

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-01.png)

最初に迷った場合は、「固定IPやポート開放を自分で管理したいか」で考えると整理しやすいです。

自宅回線条件をあまり気にせず始めたいならTailscale、経路や構成を自分で把握したいならWireGuardが向いています。

まずは1つだけ選んで試すほうが分かりやすいです。

最初からWireGuardとTailscaleを同時運用すると、どちら経由で通信しているか分かりにくくなることがあります。

## WireGuardの設定手順

WireGuardは、自宅やオフィス側へVPN入口を作るイメージです。

そのため、まず「外部から自宅へ入れる条件があるか」を確認するところから始めます。

### 事前確認: 固定IPとポート開放

WireGuardはLN6001-JPをVPN入口（サーバー）として動かします。インターネット側からアクセスできる条件が必要です。

確認事項:
- プロバイダー契約で固定IPまたはDDNSサービスを使える状態か
- ホームゲートウェイのポート開放が可能か（IPv4接続の場合）
- IPv4 over IPv6（IPoE）の場合、ポート開放の条件はプロバイダーによって異なる

特にIPoE（IPv4 over IPv6）環境では、プロバイダー側仕様によってポート開放可否が変わることがあります。

迷う場合は、まずTailscaleから試したほうが入りやすいです。

### Linksys公式VPNアシスタントモジュールのインストール

通常はLinksys公式サポートページ（https://support.linksys.com/kb/article/8723-jp/）からVPN Assistantモジュールをダウンロードし、LuCIからインストールします。

上級者向けにWireGuard関連パッケージを個別導入する方法も公式ページにありますが、順番指定があるため、まずVPN Assistantを使うほうが安全です。

1. 上記URLからLN6001-JP用のVPNアシスタントモジュール（`.ipk` ファイル）をPCにダウンロード
2. LuCIで **System** → **Software** を開く
3. **Update lists** をクリックしてパッケージリストを更新し、完了後 **Dismiss** で閉じる
4. **Upload Package** をクリックし、ダウンロードしたVPN Assistantの `.ipk` を選択して **Upload**
5. 確認画面で **Install** をクリック
6. `installed in root is up to date.` と表示されたら **Dismiss**
7. ルーターを再起動し、LuCIを開き直すと **Services** に **VPN Assistant** が追加される

最初はCLIで個別パッケージを組み合わせるより、公式モジュールで動く状態を先に作るほうが、トラブルを切り分けやすくなります。

インストール後は、パッケージとメニュー追加の状態を確認します。

```sh
echo "### VPN related packages"
opkg list-installed | grep -Ei 'vpn|wireguard|tailscale|luci-app-wireguard|kmod-wireguard' || true

echo "### services"
ls /etc/init.d | grep -Ei 'wireguard|tailscale|vpn' || true

echo "### recent install/service logs"
logread | grep -Ei 'vpn|wireguard|tailscale|opkg' | tail -n 80
```

### VPN AssistantでWireGuardを設定する

1. **Services** → **VPN Assistant** を開く
2. **WireGuard Server** タブを開く
3. **Enable WireGuard VPN** を有効にする
4. 必要な項目を入力し、**Save & Apply** をクリック
5. 設定反映後、安定運用のためルーターを一度再起動する
6. **Client Peers** セクションの **Show QR** から、スマートフォン用QRコードまたはテキスト設定を取得する

QRコードを使うと、スマートフォン側設定をかなり簡単に取り込めます。

最初はスマートフォン1台だけ接続し、正常動作を確認してからPCや他端末を追加するほうが安全です。

公式手順ではVPN Assistantを使う流れが主です。

`Network` → `Interfaces` でWireGuardインターフェースを手作業で作る方法は、個別パッケージ導入まで理解している場合の上級者向けとして扱います。

### ファイアウォールの設定

VPN Assistantを使う場合は、まずVPN Assistantの画面で保存した設定どおりに動くか確認します。

以下のUCI例は、手動でWireGuardインターフェースやポート開放を作る場合の参考です。

公式モジュールで設定した内容と重複しないよう、実行前に現在のfirewall設定を確認してください。

```sh
echo "### backup firewall config"
cp /etc/config/firewall /etc/config/firewall.backup.$(date +%Y%m%d-%H%M)

echo "### allow WireGuard UDP port 51820"
uci add firewall rule
uci set firewall.@rule[-1].name='WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

CLIは、最初は設定変更より「今どう設定されているか」を確認する用途で使うくらいでも十分です。

### クライアント側の設定（Windows/Mac/iPhone）

WireGuard公式クライアントアプリを使います（iOS/Android/Windows/Mac対応）:

最初はスマートフォン側から試すほうが、外出先接続を確認しやすくなります。

1. WireGuardアプリをインストール
2. **Add Tunnel** → **Create from scratch** または QRコードを使用
3. クライアント設定の例:

```ini
[Interface]
PrivateKey = <クライアントの秘密鍵>
Address = 10.0.0.2/24
DNS = 192.168.1.1

[Peer]
PublicKey = <ルーター（LN6001-JP）の公開鍵>
Endpoint = <ルーターの固定IPまたはDDNSホスト名>:51820
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 25
```

`AllowedIPs = 192.168.1.0/24` とすることで、自宅LANへのトラフィックのみがVPNを通ります（スプリットトンネル）。`0.0.0.0/0` にすると全トラフィックがVPN経由になります。

最初は `192.168.1.1/32` のように管理画面だけ許可し、必要に応じて範囲を広げる方法も安全です。

### 動作確認

```sh
echo "### WireGuard status"
if command -v wg >/dev/null; then
  wg show
else
  echo "wg command is not installed"
fi

echo "### WireGuard-related firewall and interfaces"
uci show firewall | grep -Ei 'wireguard|51820|wg' || true
ip addr show | grep -A4 -Ei 'wg|wireguard' || true

echo "### ping from router to LAN gateway"
ping 192.168.1.1

echo "### WireGuard logs"
logread | grep -Ei 'wireguard|wg|51820' | tail -n 80
```

`wg show` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「接続できているか」を確認する用途で使うくらいで十分です。

## Tailscaleの設定手順

Tailscaleは、固定IPやポート開放をあまり意識せず始めやすいところが特徴です。

特にIPoEやIPv4 over IPv6環境では、まずTailscaleから始めるほうが入りやすい場合があります。

### Linksys公式VPNアシスタントモジュールのインストール

TailscaleもWireGuardと同じVPN Assistantモジュールから有効化します。Linksys公式サポートページ（https://support.linksys.com/kb/article/8723-jp/）からVPNアシスタントモジュールをダウンロードしてインストールします。

1. 上記URLからLN6001-JP用のVPNアシスタントモジュール（`.ipk` ファイル）をPCにダウンロード
2. LuCIで **System** → **Software** を開き、**Update lists** を実行
3. **Upload Package** からダウンロードしたファイルを選択して **Upload**
4. 確認画面で **Install** をクリック
5. 完了後にルーターを再起動し、LuCIを開き直す（**Services** に **VPN Assistant** が追加される）

### VPN AssistantでTailscaleを有効化する

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブへ移動する
3. **Enable Tailscale Service** を有効にする
4. 必要に応じて **Advertise as Exit Node** を選ぶ
5. 認証用URLが表示されたらブラウザで開き、Tailscaleアカウントで登録する
6. 最後に **OK** をクリックし、ルーターの再起動を待つ

Tailscaleは、複数端末をあとから追加しやすいところも扱いやすいポイントです。

まずはスマートフォン1台だけ接続確認してから、必要な端末を増やしていくほうが整理しやすくなります。

Tailscaleアカウントでログインすると、ルーターがTailscaleネットワーク（Tailnet）へ参加します。

CLIの `tailscale status` は、設定後の状態確認に使います。

### LAN全体へのアクセスを許可する（サブネットルーター設定）

外出先端末からLN6001-JP配下のLAN全体へアクセスするには、サブネットルーターとして設定します:

ただし、最初からLAN全体を許可しなくても大丈夫です。

VPN Assistantでサブネットルーターとして使う設定をした場合は、Tailscale管理コンソール（https://login.tailscale.com/admin/routes）で広告されたルートを承認します。

CLIで追加設定する場合は、公式UIでの状態と衝突しないよう、変更前に現在のTailscale設定を確認してください。

### Tailscaleでのアクセス確認

```sh
echo "### Tailscale status"
if command -v tailscale >/dev/null; then
  if tailscale status; then
    echo "### Tailscale IP"
    tailscale ip

    echo "### ping over Tailscale"
    TAILSCALE_NODE="example-node"
    tailscale ping "$TAILSCALE_NODE"
  else
    echo "tailscaled is not running or not authenticated yet"
  fi
else
  echo "tailscale command is not installed"
fi

echo "### Tailscale logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 80
```

`tailscale status` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「どの端末が見えているか」を確認する用途で使うくらいで十分です。

### 起動を自動化する

```sh
echo "### enable and start tailscale"
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale enable
  /etc/init.d/tailscale start

  echo "### verify tailscale service"
  /etc/init.d/tailscale status
else
  echo "tailscale service is not installed"
fi
```

## アクセス範囲の設計

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-02.png)

VPNは「LAN全体へ全部入れる」より、「必要な機器へだけ入れる」ほうが安全です。

特にNASや管理画面だけ使いたい場合は、AllowedIPsを狭くしたほうが整理しやすくなります。

LAN全体へ広く入れるより、必要な範囲に絞る設計のほうが安全です。

最初は「NASだけ」「管理画面だけ」くらいから始めると分かりやすくなります。

## 運用上の注意点

VPNは「設定したら終わり」ではなく、「どの端末が入れるか」を継続管理することも重要です。

### 鍵・アカウント管理

- WireGuard: 秘密鍵はバックアップしておく。端末を紛失した場合は、ルーター側のPeer設定から該当端末の公開鍵を削除する
- Tailscale: 管理コンソールから端末を無効化できる。不要端末や紛失端末は早めに無効化する

### バックアップ

```sh
BACKUP_DIR="/root/vpn-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

opkg list-installed | grep -Ei 'vpn|wireguard|tailscale' > "$BACKUP_DIR/vpn-packages.txt" || true

ls -l "$BACKUP_DIR"
```

VPN設定は network / firewall と関係するため、変更前バックアップを残しておくと安心です。

## よくある問題と対処

設定後は、次の3つが確認できれば最初の段階として十分です。最初はここまで確認できれば大きく外していません。

- スマートフォンやPCからVPN接続できる
- 目的のLAN機器へだけ届く
- 使っていない範囲まで広く公開していない

### WireGuardで接続できない

1. ルーターのfirewallでUDPポート51820が開いているか確認
2. ホームゲートウェイ側のポート開放設定を確認
3. `wg show` で接続状態を確認

特にIPoE環境では、「ポート開放できていない」ケースがかなり多いです。

迷う場合は、一度Tailscaleで接続できるか確認すると切り分けしやすくなります。

### Tailscaleでサブネットへアクセスできない

1. Tailscale管理コンソールでルートの承認を確認
2. `tailscale status` でルーターがネットワークに参加しているか確認

Tailnetへ参加できていても、「ルート承認」が完了していないとLANへ届かないことがあります。

### VPN接続は成功するがLANにpingが届かない

- **WireGuardの場合:** AllowedIPsにLANの範囲が含まれているか確認
- **Tailscaleの場合:** `--advertise-routes` の設定と管理コンソールの承認状態を確認

## まとめ

VPN設定は、次の順番で進めると整理しやすいです:

1. 回線条件（固定IP・ポート開放可否）を確認してWireGuard / Tailscaleを選ぶ
2. Linksys公式サポート（https://support.linksys.com/kb/article/8723-jp/）の手順でパッケージをインストール
3. WireGuardは **Services** → **VPN Assistant** → **WireGuard Server** で設定し、Client PeersのQRコードを使う
4. Tailscaleは **Services** → **VPN Assistant** → **Tailscale** で有効化し、認証URLから登録する
5. 接続後は `wg show` / `tailscale status` で状態を確認する

固定IPやポート開放を管理できるならWireGuard、回線条件をあまり気にせず始めたいならTailscale、という考え方が分かりやすいです。

最初からLAN全体をVPNへ公開するより、「NASだけ」「管理画面だけ」のように必要な範囲へ絞るほうが安全です。

公式VPN Assistantモジュールを使って、まずは動く状態を作ってから理解を深めていくほうが、家庭や小さなオフィスでは進めやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-03.png)

## よくある質問

### LN6001-JPでWireGuardとTailscaleはどちらが向いている？

固定IPがあり、ポート開放もできるならWireGuardが選びやすいです。

固定IPなし、IPoE環境、複数端末の管理を楽にしたいならTailscaleのほうが始めやすいです。

### 固定IPがなくてもLN6001-JPでVPNは使える？

使えます。その場合はTailscaleのほうが現実的です。WireGuardを使うなら、固定IPかDDNS、そしてポート開放の条件を事前に確認する必要があります。

### IPoE環境ではWireGuardは使えない？

必ずしも使えないわけではありません。

ただし、IPv4 over IPv6の方式やプロバイダー条件によってはポート開放が難しいことがあります。迷う場合は、まずTailscaleを候補にすると判断しやすいです。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/

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
