<!-- mirror-source: articles/028-vpn-configuration.md -->

> Original note.com article: [VPN設定まとめ: WireGuard・Tailscaleの選び方と導入手順【OpenWrt集中連載028】](https://note.com/ikmsan/n/nb0b4b7e63309)

# VPN設定まとめ: WireGuard・Tailscaleの選び方と導入手順【OpenWrt集中連載028】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPには、Linksys公式のVPN Assistantモジュールがあり、WireGuardとTailscaleをサポートしています。

「自前で入口を管理したい（WireGuard）」か、「NAT越えや接続管理をできるだけ任せたい（Tailscale）」かで選び方が変わります。

ただし、最初から完璧にVPN設計しなくても大丈夫です。

まずは「自分の回線でポート開放できるか」「必要な機器だけへ安全に入れるか」を確認するだけでも、かなり整理しやすくなります。

この記事では、WireGuardとTailscaleの選び方、LN6001-JPでの導入の流れ、LAN全体を広げすぎない考え方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-01.png)

## この記事でわかること

- WireGuardとTailscaleをどう選べばよいか
- 固定IPやIPoE環境で判断するポイント
- LN6001-JPでVPN Assistantを使う流れ
- VPN接続後にアクセス範囲をどう絞るか

## こんな人に向いています

- 自宅や小規模拠点へ外出先から安全に入りたい
- IPoE環境でWireGuardが使えるか迷っている
- VPNを最小構成から安全に始めたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-02.png)

## まずはここまでで十分

VPNも、最初から全部の端末やLAN全体へ広げる必要はありません。

まずは次の3つで十分です。

1. 自分の回線でポート開放できるか確認する
2. WireGuardかTailscaleのどちらか1つを選ぶ
3. 必要な機器だけへ接続できる状態を作る

最初は「つながること」と「公開範囲を広げすぎないこと」を優先するほうが安全です。

VPNは、「どちらが最強か」より、「自分の回線条件で扱いやすいか」で選ぶほうが整理しやすくなります。

## VPN方式の選択基準

![表 01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-01.png)

特にIPoE環境では、WireGuard側のポート開放条件が難しくなるケースもあります。

そのため、「まずTailscaleで安全に外から入れる状態を作る」という進め方はかなり現実的です。

まずは、自分の回線でポート開放できるかを確認してから選ぶと整理しやすくなります（014の記事も参照してください）。

最初はCLIで個別パッケージを組み合わせるより、Linksys公式VPN Assistantから始めるほうが切り分けしやすくなります。

---

## Linksys公式VPNアシスタントモジュールの取得

WireGuard・Tailscaleは、まずLinksys公式サポートページからVPN Assistantモジュールを入手して設定します。

WireGuardについては上級者向けに関連パッケージを個別導入する公式手順もありますが、順番指定があるため、通常はVPN Assistantを使うほうが安全です。

1. https://support.linksys.com/kb/article/8723-jp/ にアクセス
2. LN6001-JP向けのVPNアシスタントモジュール（`.ipk` ファイル）をPCにダウンロード
3. LuCI: **System** → **Software** で **Update lists** を実行
4. **Upload Package** から `.ipk` をアップロードして **Install**
5. 完了後にルーターを再起動し、LuCIを開き直す
6. **Services** メニューに **VPN Assistant** が追加される

最初は「LuCIから動く状態を作る」ことを優先するほうが整理しやすくなります。

インストール前後の状態は、SSHから次のように確認できます。

```sh
BACKUP_DIR="/root/vpn-assistant-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
opkg list-installed | grep -E 'vpn|wireguard|tailscale' > "$BACKUP_DIR/opkg-before.txt" 2>/dev/null || true

# VPN Assistant導入後の確認
opkg list-installed | grep -E 'vpn|wireguard|tailscale'
ls /etc/init.d | grep -E 'tailscale|wireguard|vpn' || true
logread | grep -i 'vpn\|wireguard\|tailscale' | tail -n 80
```

公式手順では、LuCIの **System** → **Software** からVPN AssistantのIPKをアップロードし、完了後に再起動して **Services** メニューに追加されたことを確認します。

CLI確認は、公式UI手順が終わった後の状態確認として使う位置づけです。

---

## WireGuardの導入手順（概要）

WireGuardは、「自分でVPN入口を管理したい」場合にかなり向いています。

### 事前条件の確認

```sh
# WANのIPアドレスを確認（固定IPかどうか）
ifstatus wan | grep '"ipaddr"'

# IPoE / IPv6側も確認
ifstatus wan6 | grep -E '"up"|"ip6addr"|"ip6prefix"|"proto"'

# 既存のVPN/ポート公開ルールを確認
uci show firewall | grep -E 'wireguard|tailscale|51820|redirect'
```

`ifstatus wan` は、まず最初に確認したいコマンドです。

最初は「固定IPなのか」「動的IPなのか」を分けるだけでもかなり役立ちます。

- 固定IPがある場合: Endpointとして直接使用
- 動的IPの場合: DynDNSサービス（ddns-scripts等）を使うか、Tailscaleへ切り替える

### VPN AssistantでのWireGuard設定（モジュールインストール後）

1. **Services** → **VPN Assistant** を開く
2. **WireGuard Server** タブを開く
3. **Enable WireGuard VPN** を有効にする
4. 必要なサーバー設定を入力し、**Save & Apply**
5. 設定反映後、ルーターを一度再起動
6. **Client Peers** の **Show QR** からクライアント用QRコードまたはテキスト設定を取得

最初はスマートフォン1台だけ接続確認するほうが切り分けしやすくなります。

### ファイアウォール設定（手動構成する場合）

VPN Assistantを使う場合は、まず公式UIで反映された設定を確認します。以下はWireGuardを手動構成する場合の参考例です。

```sh
BACKUP_DIR="/root/wireguard-firewall-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/firewall "$BACKUP_DIR/firewall"

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart

uci show firewall | grep -A 8 "name='Allow-WireGuard'"
logread | grep -i 'wireguard\|firewall' | tail -n 80
```

最初は「WireGuardポートだけ開いているか」を確認するくらいでも十分です。

### クライアント設定（スマートフォン・PC）

WireGuardアプリのトンネル設定例:

```ini
[Interface]
PrivateKey = <クライアントの秘密鍵>
Address = 10.0.0.2/24
DNS = 192.168.1.1

[Peer]
PublicKey = <LN6001-JPの公開鍵>
Endpoint = <WAN固定IPまたはDDNSホスト名>:51820
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 25
```

最初は「NASだけ」「管理画面だけ」のように、AllowedIPsを必要最小限へ絞るほうが安全です。

### 動作確認

```sh
# WireGuardの状態確認
wg show

# ルーター側の設定とログ確認
uci show network | grep -i wireguard
uci show firewall | grep -i wireguard
logread | grep -i wireguard | tail -n 80
```

`wg show` は、WireGuard状態確認でまず最初に覚えておきたいコマンドです。

---

## Tailscaleの導入手順（概要）

Tailscaleは、「固定IPなし」「IPoE環境」「ポート開放が難しい」ケースとかなり相性があります。

### インストール（公式モジュール経由）

1. https://support.linksys.com/kb/article/8723-jp/ からモジュールを取得
2. LuCI: **System** → **Software** → **Upload Package** からインストール

最初は、VPN Assistant経由でLuCIから動かすほうが切り分けしやすくなります。

### 初期設定

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブで **Enable Tailscale Service** を有効化
3. ルーターをExit Nodeにしたい場合は **Advertise as Exit Node** を選択
4. 表示される認証URLをブラウザで開いてTailscaleアカウントに登録
5. 必要に応じてTailscale管理コンソール（https://login.tailscale.com/admin/routes）でルートやExit Nodeを承認

最初は「スマートフォン1台だけ接続確認する」くらいでも十分役立ちます。

CLIの `tailscale status` は、設定後の状態確認でまず最初に確認したいコマンドです。

```sh
# Tailscale関連パッケージとサービス状態
opkg list-installed | grep -E 'tailscale|vpn'
/etc/init.d/tailscale status

# 認証後の状態確認
tailscale status
tailscale ip -4
tailscale ip -6
logread | grep -i tailscale | tail -n 80
```

公式手順では、Tailscale有効化後に認証URLを開き、最後にOKを押すとログアウトしてルーターが再起動します。

数分待ってから `tailscale status` を確認すると、認証や接続の切り分けがしやすくなります。

---

## アクセス範囲の設計

VPNは、「つながること」だけでなく、「どこまでアクセスできるか」を整理することもかなり重要です。

![表 02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-02.png)

最初は「LAN全体」より、「NASだけ」「管理画面だけ」くらいから始めるほうが安全です。

最小権限の原則で、必要な範囲だけへ絞るほうが安全です。

設定後は、実際の経路が広がりすぎていないか確認します。

```sh
# IPv4/IPv6の経路確認
ip route show
ip -6 route show

# Tailscale利用時は広告ルートと端末一覧を確認
tailscale status 2>/dev/null || true

# WireGuard利用時はpeerとallowed ipsを確認
wg show 2>/dev/null || true
```

`0.0.0.0/0` や `::/0` は全通信をVPNへ流す設定です。

必要な場面では便利ですが、最初から使うと切り分けが難しくなるため、まずはLAN内の必要な範囲だけを指定するほうが安全です。

---

## 運用管理のポイント

VPNは、「どの端末が接続できるか」を継続管理することも重要です。

最初の成功条件としては、次の3つが見えれば十分です。最初はここまで確認できれば大きく外していません。

- スマートフォンやPCから接続できる
- 必要なLAN機器だけへ届く
- 使っていない経路や端末まで広く開いていない

### WireGuardの場合

- 秘密鍵（Private Key）は設定バックアップへ含まれるため、バックアップは暗号化して保管する
- 端末を紛失した場合はLuCIの **Peers** タブからその端末公開鍵を削除する

```sh
# WireGuard peerの棚卸し
wg show

# firewall上でWireGuardに関係するルールを確認
uci show firewall | grep -i wireguard
```

### Tailscaleの場合

- Tailscale管理コンソール（https://login.tailscale.com/admin/machines）で端末を管理する
- 退職・紛失時はコンソールからデバイスを削除・無効化する
- 管理アカウントへ二要素認証を設定する

```sh
# Tailscale参加端末と自分のIPを確認
tailscale status
tailscale ip -4

# Tailscale関連ログ
logread | grep -i tailscale | tail -n 80
```

---

## まとめ

迷ったときは、「ポート開放できるならWireGuard、難しいならTailscale」くらいの整理で十分です。

どちらも、まずはLinksys公式VPN Assistantを入口にして、アクセス範囲は必要最小限から始めるほうが安全です。

特にIPoE環境では、Tailscaleはかなり現実的な選択肢になります。

最初は「スマートフォン1台から安全に入れる」状態を作れるだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-03.png)

## よくある質問

### LN6001-JPのVPNはWireGuardとTailscaleのどちらが向いている？

固定IPとポート開放が使えるならWireGuard、IPoE環境や固定IPなしならTailscaleが始めやすくなります。

### LN6001-JPのVPNは最初からLAN全体へ入れるべき？

必要最小限から始めるほうが安全です。

まずはNASや管理画面など、必要な範囲だけへ絞るほうが整理しやすくなります。

### LN6001-JPではVPNは公式モジュールから始めたほうがいい？

はい。

この連載ではVPN Assistantを入口にする前提です。手動設定よりも手順を揃えやすくなります。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
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
