<!-- mirror-source: articles/018-tailscale-zerotier-use.md -->

> Original note.com article: [Tailscaleの使いどころ: VPNを自分で持つか任せるか【OpenWrt集中連載018】](https://note.com/ikmsan/n/n8d21425036a9)

# Tailscaleの使いどころ: VPNを自分で持つか任せるか【OpenWrt集中連載018】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

VPNには、「自分で入口を持つ方式（WireGuardなど）」と、「管理をサービスへ任せる方式（Tailscaleなど）」があります。

LN6001-JPでは、Linksys公式モジュールによるWireGuard/Tailscaleが案内されています。

ただし、最初から「どちらが上か」を決める必要はありません。

固定IPやポート開放を管理できるならWireGuard、IPoE環境や固定IPなしで始めやすくしたいならTailscale、くらいの整理でも十分です。

特にTailscaleは、NAT越えを自動処理できるため、「とにかく安全に外から入りたい」用途と相性があります。

この記事では、Tailscaleを中心に、WireGuardとの使い分け、Linksys公式VPN Assistantを使った導入方法、アクセス範囲を広げすぎない考え方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-01.png)

## この記事でわかること

- Tailscale が向く回線や運用条件
- LN6001-JPでTailscaleを有効化する流れ
- WireGuardと迷った時の考え方
- LAN全体を公開しすぎない設計

## こんな人に向いています

- 固定IPやポート開放なしで外部接続したい
- IPoE環境で自前VPNが難しい
- Tailscaleを安全に最小構成から試したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-02.png)

## まずはここまでで十分

Tailscaleも、最初からLAN全体を広告する必要はありません。

まずは次の3つで十分です。

1. VPN Assistantで有効化する
2. スマートフォン1台だけ接続確認する
3. 必要になったらあとでサブネットルーターを有効化する

最初は「安全につながること」を確認してから、アクセス範囲を広げるほうが整理しやすくなります。

LAN全体を最初から広告するより、「必要な機器へだけ届く」くらいから始めるほうが安全です。

VPNは、「どちらが最強か」より、「自分の回線条件で扱いやすいか」で選ぶほうが現実的です。

## 方式の使い分け

| 方式 | 向く状況 | 主な特徴 |
|---|---|---|
| WireGuard（自前） | 固定IP・ポート開放が可能、経路を自分で管理したい | 自前でVPN入口を運用する |
| Tailscale | 固定IPなし、複数端末管理を楽にしたい | NAT越えを自動処理、外部サービスを使う |

特にIPoE環境では、WireGuard側のポート開放条件が難しくなるケースもあります。

そのため、「まずTailscaleで安全に外から入れる状態を作る」という進め方はかなり現実的です。

技術名の優劣ではなく、回線条件・管理者・接続範囲で選ぶほうが整理しやすくなります。

最初はCLIで個別パッケージを組み合わせるより、Linksys公式VPN Assistantで動く状態を先に作るほうが切り分けしやすくなります。

## Tailscaleの設定手順（LN6001-JP）

Linksys公式モジュール（https://support.linksys.com/kb/article/8723-jp/）を使います。

### Linksys公式VPNアシスタントモジュールのインストール

Tailscaleは `opkg install tailscale` ではインストールできません。

必ずLinksys公式サポートページ（https://support.linksys.com/kb/article/8723-jp/）からVPNアシスタントモジュールを入手してください。

1. 上記URLからLN6001-JP用のVPNアシスタントモジュール（`.ipk` ファイル）をPCにダウンロード
2. LuCIで **System** → **Software** を開き、**Update lists** を実行
3. **Upload Package** からダウンロードしたファイルを選択して **Upload**
4. 確認画面で **Install** をクリック
5. 完了後にルーターを再起動し、LuCIを開き直す（**Services** に **VPN Assistant** が追加される）

最初はLuCI上で動く状態を作ってからCLI確認へ進むほうが、構成を理解しやすくなります。

インストール前に、既存のnetwork/firewall設定とパッケージ状態を保存しておくと、後から切り戻しや確認がしやすくなります。

```sh
BACKUP_DIR="/root/vpn-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
cp /etc/config/dhcp "$BACKUP_DIR/dhcp"
uci show network > "$BACKUP_DIR/uci-network.txt"
uci show firewall > "$BACKUP_DIR/uci-firewall.txt"
opkg list-installed | grep -E 'tailscale|wireguard|vpn' > "$BACKUP_DIR/opkg-vpn.txt" 2>/dev/null || true

echo "backup: $BACKUP_DIR"
```

### Tailscaleへの接続

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブへ移動する
3. **Enable Tailscale Service** を有効にする
4. 必要に応じて **Advertise as Exit Node** を選ぶ
5. 認証用URLが表示されたらブラウザで開き、Tailscaleアカウントで登録する
6. 最後に **OK** をクリックし、ルーターの再起動を待つ
7. 再起動後、必要に応じてSSHから `tailscale status` で状態を確認する

最初はスマートフォン1台だけ接続確認するほうが切り分けしやすくなります。

問題なくつながることを確認してから、PCや他端末を追加するほうが安全です。

### LAN全体をTailscaleから見えるようにする

VPN Assistantでサブネットルーターとして使う設定をした場合は、Tailscale管理コンソール（https://login.tailscale.com/admin/routes）で広告されたルートを承認します。

CLIで `tailscale up --advertise-routes=...` を実行する場合は、公式UI設定と衝突しないよう、現在設定を確認してから行います。

最初はLAN全体ではなく、「NASだけ」「管理セグメントだけ」くらいから始めるほうが整理しやすくなります。

### 接続確認

```sh
# Tailscaleの接続状態と参加端末を確認
if command -v tailscale >/dev/null; then
  if tailscale status; then
    # LN6001-JP自身のTailscale IPアドレスを確認
    tailscale ip -4
    tailscale ip -6

    # 特定のノードへのpingテスト
    TAILSCALE_NODE="example-node"
    tailscale ping "$TAILSCALE_NODE"
  else
    echo "tailscaled is not running or not authenticated yet"
  fi
else
  echo "tailscale command is not installed"
fi

# サービス状態とログ確認
[ -x /etc/init.d/tailscale ] && /etc/init.d/tailscale status
logread | grep -i tailscale | tail -n 80
```

`tailscale status` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「どの端末が見えているか」を確認する用途で使うくらいでも十分です。

### Tailscaleの無効化・削除

```sh
# Tailscaleネットワークから抜ける
if command -v tailscale >/dev/null; then
  tailscale logout
fi

# 自動起動を無効化
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale disable
  /etc/init.d/tailscale stop
fi
```

不要になったVPN設定を残しっぱなしにしないことも、安全運用ではかなり重要です。

## WireGuardとの組み合わせ

Tailscaleがサービス依存になることを避けたい場合はWireGuardを使います。

WireGuardの設定は Article 013 で詳しく解説しています。

2つの方式を使い分ける設計例:

- **メインのリモートアクセス**: Tailscale（固定IPが不要、端末管理が楽）
- **拠点間の常時接続**: WireGuard（安定した専用トンネル）

最初から両方を同時に大規模運用するより、「まず1つを安定させる」ほうが切り分けしやすくなります。

VPNは、「つながること」だけでなく、「どこまでアクセスできるか」もかなり重要です。

## ファイアウォールとアクセス範囲の設計

### Tailscaleのアクセス範囲制限

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:admin"],
      "dst": ["192.168.1.0/24:*"]
    },
    {
      "action": "accept",
      "src": ["tag:viewer"],
      "dst": ["192.168.1.31:80,443"]
    }
  ]
}
```

ACLを使うことで、「管理者だけLAN全体」「閲覧用ユーザーはNASだけ」のように整理しやすくなります。

### OpenWrt側のfirewallでも制限する

```sh
# Tailscaleインターフェースを確認
ip addr show tailscale0

# 変更前に設定を保存
BACKUP_DIR="/root/tailscale-firewall-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"

# firewallで扱うためのネットワーク項目を作成
uci set network.tailscale=interface
uci set network.tailscale.proto='none'
uci set network.tailscale.ifname='tailscale0'
uci changes network
uci commit network

# firewallでTailscaleゾーンを作成（必要な場合）
uci add firewall zone
uci set firewall.@zone[-1].name='vpn'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci add_list firewall.@zone[-1].network='tailscale'
uci changes firewall
uci commit firewall
/etc/init.d/network reload
/etc/init.d/firewall restart

# 反映後の確認
uci show network.tailscale
uci show firewall | grep -A 8 "name='vpn'"
```

最初は「VPNからLAN全体へ全部許可」より、「必要な方向だけ許可」するほうが安全です。

VPNは、「どの端末が参加しているか」を継続管理することも重要です。

## アカウント・端末管理の注意点

### Tailscaleの場合

- 管理アカウント: 組織用メールアドレスで登録する（個人アドレスを避ける）
- 端末を失った・不要になった場合: 管理コンソール（https://login.tailscale.com/admin/machines）でデバイスを無効化・削除する
- 二要素認証: 管理アカウントへ必ず設定する

設定後は、「本当に想定どおり動いているか」を確認することが大事です。

## 現在の状態を確認する

```sh
# インストール済みのVPNパッケージを確認（公式モジュール経由でインストール済みの場合）
opkg list-installed | grep -E 'tailscale|wireguard'

# Tailscaleサービスとログを確認
[ -x /etc/init.d/tailscale ] && /etc/init.d/tailscale status
command -v tailscale >/dev/null && tailscale status
logread | grep -i tailscale | tail -n 80

# インターフェース一覧を確認
ip addr show

# VPN経由の経路確認
ip route show
ip -6 route show
```

`ip route show` は、「VPN経由でどこへルーティングされているか」を確認する時に便利です。

## バックアップ

```sh
BACKUP_DIR="/root/vpn-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
opkg list-installed | grep -E 'tailscale|wireguard|vpn' > "$BACKUP_DIR/opkg-vpn.txt" 2>/dev/null || true
tailscale status > "$BACKUP_DIR/tailscale-status.txt" 2>/dev/null || true
```

VPN設定は network / firewall と関係するため、変更前バックアップを残しておくと安心です。

Tailscaleの認証情報やWireGuardの秘密鍵に相当する情報が含まれる場合があるため、バックアップファイルは外部にそのまま共有しないようにします。

## まとめ

Tailscaleは、「固定IPやポート開放なしで安全に入りたい」用途とかなり相性があります。

1. Linksys公式VPNアシスタントモジュール（https://support.linksys.com/kb/article/8723-jp/）をLuCI経由でインストール
2. **Services** → **VPN Assistant** → **Tailscale** で有効化
3. 表示された認証URLからTailscaleアカウントへ登録
4. まずスマートフォン1台だけ接続確認する
5. 必要に応じて管理コンソールでルートやExit Nodeを承認する
6. SSHの `tailscale status` で状態確認する

固定IPやポート開放が難しいIPoE環境では、Tailscaleはかなり有力な選択肢です。

一方で、「自前で入口を管理したい」「外部サービス依存を減らしたい」場合はWireGuardが向いています。

最初からLAN全体を広告するより、「NASだけ」「管理画面だけ」のように必要な範囲へ絞るほうが安全です。

最初の成功条件としては、次の3つが見えれば十分です。最初はここまで確認できれば大きく外していません。

- Tailscale ネットワークへ正常参加できる
- 目的の機器へ届く
- 不要に広いルート広告をしていない

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-03.png)

## よくある質問

### Tailscaleは固定IPがなくても使える？

使えます。

固定IPやポート開放が難しい環境でも始めやすいところがTailscaleの強みです。

### TailscaleとWireGuardはどちらが簡単？

回線条件が厳しいならTailscaleのほうが始めやすいです。

自前で入口を管理したいならWireGuardのほうが合います。

### TailscaleでLAN全体へ入れるようにできる？

できます。

サブネットルーター設定を有効にし、管理コンソール側でルート承認を行う流れになります。

ただし、最初からLAN全体を広告するより、必要な範囲へ絞るほうが安全です。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Tailscale管理コンソール: https://login.tailscale.com/admin/

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
