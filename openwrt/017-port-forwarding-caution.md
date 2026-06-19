<!-- mirror-source: articles/017-port-forwarding-caution.md -->

> Original note.com article: [ポート開放の注意点: 公開前に確認するリスク【OpenWrt集中連載017】](https://note.com/ikmsan/n/n765ffea6ea82)

# ポート開放の注意点: 公開前に確認するリスク【OpenWrt集中連載017】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

ポート開放は、インターネット側からLAN内機器へ入口を作る設定です。

NAS、監視カメラ、ゲームサーバーなどを外部公開する時に使われますが、設定を誤ると意図しない通信を許可するリスクがあります。

ただし、最初から「ポート開放は危険だから全部禁止」と考えなくても大丈夫です。

まずは「本当に公開が必要なのか」「VPNで代替できないか」を整理するだけでも、かなり安全性は変わります。

LN6001-JPでは、LuCIの **Network** → **Firewall** → **Port Forwards** からポートフォワーディングを設定できます。

この記事では、ポート開放が必要になるケース、VPNへ置き換えたほうがよいケース、公開前に最低限確認したいポイントを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-01.png)

## この記事でわかること

- ポート開放が本当に必要かどうかの考え方
- LN6001-JPで公開前に確認したいポイント
- VPNで置き換えたほうがよいケース
- 公開範囲を最小限へ絞る考え方

## こんな人に向いています

- NASやカメラを外から見たいが、公開してよいか不安
- ポート開放とVPNのどちらを選ぶべきか迷っている
- 公開前に最低限のリスクを把握したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-02.png)

## まずはここまでで十分

ポート開放も、最初から複雑に考えなくて大丈夫です。

まずは次の3つを確認するだけでも、かなり判断しやすくなります。

1. VPNで代替できないか考える
2. 公開先機器のIPを固定する
3. 開ける範囲を最小限へ絞る

「設定できるか」より、「本当に公開が必要か」を先に考えるほうが安全です。

特に「自分だけが使う」用途なら、VPNのほうが整理しやすいこともかなり多くあります。

## ポート開放が必要かどうかをまず判断する

ポートを開ける前に、まず代替手段がないか確認します。

最初は「インターネットへ直接公開しなくても済むか」を考えるだけでもかなり重要です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-01.png)

特に管理画面系（RDP、NAS管理UI、ルーター設定画面）は、直接公開しないほうが安全です。

「外から自分だけアクセスしたい」なら、まずVPNを優先したほうが安全です（Article 013参照）。

ポート開放は、「不特定端末から直接アクセスされる前提」になることを意識しておく必要があります。

それでもポート開放が必要な場合は、「公開範囲をできるだけ小さくする」方向で設計します。

## ポートフォワーディングの設定手順（LuCI）

### 事前準備: 公開先機器のIPを固定する

ポートフォワーディングの転送先IPが変わると動作しなくなります。

事前にDHCP予約で固定します（Article 012参照）。

```sh
# 公開先機器のIPを確認
cat /tmp/dhcp.leases
# 例: NAS → 192.168.1.10 に固定済みであることを確認
```

最初は「公開対象機器だけ固定IP化する」くらいで十分です。

### 事前準備: 回線と既存ルールを確認する

公開前に、WAN側の状態、WAN6側の状態、既存のポート転送を確認します。

```sh
# WAN / WAN6の状態確認
ifstatus wan | grep -E '"up"|"ipaddr"|"proto"'
ifstatus wan6 | grep -E '"up"|"ip6addr"|"ip6prefix"|"proto"'

# 既存のポート転送を確認
uci show firewall | grep redirect

# 念のため現在設定を保存
BACKUP_DIR="/root/port-forward-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/dhcp "$BACKUP_DIR/dhcp"
uci show firewall > "$BACKUP_DIR/uci-firewall.txt"
```

IPoEやIPv4 over IPv6の環境では、WAN6が正常でもIPv4の外部着信条件が別に制限される場合があります。

「LAN内から見える」ことと「インターネット側から着信できる」ことは別なので、公開前に分けて確認します。

### ステップ1: Port Forwardsを開く

1. **Network** → **Firewall** → **Port Forwards** タブを開く
2. **Add** をクリック

### ステップ2: 転送ルールを設定する

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-02.png)

最初は「必要なポート1つだけ」を開けるほうが安全です。

広いポート範囲をまとめて開放しないほうが整理しやすくなります。

外部ポートを443以外（例: 8443）へ変えると、一般的なスキャンへ少し引っかかりにくくなります（完全な対策ではありません）。

3. **Save & Apply**

最初は、公開後すぐに外部接続確認できる環境を用意しておくと切り分けしやすくなります。

### ステップ3: 設定を確認する

```sh
# ポートフォワーディングの設定を確認
uci show firewall | grep redirect

# 外部からのアクセステスト（別ネットワーク端末から）
# https://ルーターのWAN-IP:8443 でNASにアクセスできるか確認
```
外部確認は、同じWi‑Fi内ではなく「モバイル回線」など別ネットワーク側から試す必要があります。

WAN IPアドレスは **Status** → **Overview** → IPv4 WAN Status で確認します。

IPoE環境では、そもそも外部着信が制限されるケースもあります。

ポートフォワーディングはCLIでも設定できますが、最初はLuCIから設定したほうが通信方向を理解しやすいです。

## CLIでポートフォワーディングを設定する

```sh
# バックアップ
BACKUP_DIR="/root/port-forward-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/firewall "$BACKUP_DIR/firewall"

# ポートフォワーディングを追加（NASのHTTPS 8443→443）
uci add firewall redirect
uci set firewall.@redirect[-1].name='NAS-HTTPS'
uci set firewall.@redirect[-1].src='wan'
uci set firewall.@redirect[-1].proto='tcp'
uci set firewall.@redirect[-1].src_dport='8443'
uci set firewall.@redirect[-1].dest='lan'
uci set firewall.@redirect[-1].dest_ip='192.168.1.10'
uci set firewall.@redirect[-1].dest_port='443'
uci set firewall.@redirect[-1].target='DNAT'

# 反映前に差分を確認
uci changes firewall

uci commit firewall
/etc/init.d/firewall restart

# 設定確認
uci show firewall | grep -A 8 redirect
logread | grep -i 'firewall\|redirect\|nat' | tail -n 50
```

`uci show firewall | grep redirect` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「今どのポートが公開されているか」を確認する用途で使うくらいでも十分です。

## アクセス元IPを制限する（推奨）

固定IPや特定拠点からのみアクセスする場合は、送信元IPを制限したほうが安全です。

```sh
# 変更前に対象ルールを確認
uci show firewall | grep -A 10 "name='NAS-HTTPS'"

# 特定IPからのみ許可する場合（例: 203.0.113.1からのみ）
uci set firewall.@redirect[-1].src_ip='203.0.113.1'
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart

# 反映後の確認
uci show firewall | grep -A 10 "name='NAS-HTTPS'"
```

LuCIでは **Source IP address** フィールドに送信元IPを入力します。

「誰からでもアクセス可能」より、「自分の拠点だけ許可」のほうがかなり安全性を上げやすくなります。

## IPoE環境でのポート開放の注意点

MAP-E・DS-Lite環境では、外部からの着信ポートが制限される場合があります。

特にIPv4 over IPv6環境では、「ポート開放できる前提ではない」ケースもあります。

```sh
# 回線方式を確認
ifstatus wan | grep -E '"up"|"ipaddr"|"proto"'
ifstatus wan6 | grep -E '"up"|"ip6addr"|"ip6prefix"|"proto"'

# ルーティングも確認
ip route show
ip -6 route show
```

`ifstatus wan` と `ifstatus wan6` は、まず最初に確認したいコマンドです。

MAP-E環境では、使用可能ポート番号が制限されています（方式によって異なる）。

まずはプロバイダ仕様を確認するほうが切り分けしやすくなります。

固定IP（固定IPサービス + PPPoE）であれば、通常どおりポート開放しやすくなります。

## 公開後の運用チェックリスト

ポートを開放したら、「公開しっぱなし」にしないことが重要です。

以下を定期的に確認します:

- 公開先機器のファームウェアが最新か
- 認証が強いパスワード/証明書を使っているか
- アクセスログを定期的に確認しているか
- 不要になったらすぐに削除する（放置しない）

```sh
# ファイアウォールのログ確認
logread | grep -i 'FORWARD\|DROP\|REJECT' | tail -n 50

# NATのログ確認
logread | grep -i 'nat\|redirect' | tail -n 30

# 現在の公開ルール棚卸し
uci show firewall | grep -A 10 redirect
```

最初は「大量アクセスが来ていないか」「不要ルールを残していないか」を見るくらいでも十分役立ちます。

## 不要になったルールを削除する

ポートフォワーディングは「作って終わり」にしないことが大事です。

使わなくなった公開設定を放置しないだけでも、安全性はかなり変わります。

LuCIでの削除:
1. **Network** → **Firewall** → **Port Forwards**
2. 削除したいルールの **Delete** をクリック
3. **Save & Apply**

```sh
# CLI での削除（対象のインデックスを確認してから）
uci show firewall | grep -n redirect
# インデックス確認後
uci delete firewall.@redirect[N]  # N = 対象のインデックス番号
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart

# 削除されたことを確認
uci show firewall | grep redirect
```

最初は「今どのポートを開けているか」を定期的に棚卸しするだけでも十分役立ちます。

## VPN代替を使うべきケース

これらの用途は、ポート開放よりVPN（Article 013参照）を使ったほうが安全に整理しやすくなります:

- **ルーター管理画面への外部アクセス**: 絶対にポート開放しない
- **自分・家族だけが使うNAS**: TailscaleまたはWireGuardで入る
- **店舗カメラの遠隔監視**: VPN経由でアクセスする
- **リモートデスクトップ（RDP）**: VPN必須

特に「自分しか使わない」用途は、VPNへ寄せたほうが公開範囲をかなり狭くできます。

## まとめ

ポート開放は、次の順番で進めると整理しやすいです:

1. まずVPNで代替できないか検討する
2. DHCP予約で公開先機器のIPアドレスを固定する
3. **Network** → **Firewall** → **Port Forwards** → **Add** で転送ルールを設定する
4. できれば送信元IPを制限する
5. 公開後は機器の更新・認証・ログを定期確認する
6. 不要になったルールはすぐに削除する

「開けられる」より、「本当に開ける必要があるか」を先に考えることが、安全なポート開放の基本姿勢です。

特にNASや管理画面用途は、VPNへ置き換えたほうが安全に運用しやすいケースもかなり多くあります。

最初の成功条件としては、次の3つが見えれば十分です。最初はここまで確認できれば大きく外していません。

- 公開先機器が固定IPで安定している
- 不要なポートや広すぎる公開をしていない
- 代替として VPN を検討したうえで判断している

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-03.png)

## よくある質問

### NASやカメラを見るならポート開放が必要？

必ずしも必要ではありません。

自分や家族だけが使うなら、まずはVPNで代替できないかを考えるほうが安全です。

### ポート開放は一度設定したら放置でいい？

よくありません。

公開先機器の更新、認証、ログ確認、不用になったルールの削除まで含めて管理が必要です。

### ポート開放と VPN はどちらが安全？

一般的にはVPNのほうが安全に設計しやすいです。

ポート開放は、公開範囲と管理責任が大きくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT port forwarding: https://support.linksys.com/kb/article/224-en/?section_id=175
- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/

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
