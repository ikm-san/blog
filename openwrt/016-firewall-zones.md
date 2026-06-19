<!-- mirror-source: articles/016-firewall-zones.md -->

> Original note.com article: [firewall zonesの考え方: LAN/Guest/VPNを分ける基本【OpenWrt集中連載016】](https://note.com/ikmsan/n/nfe0609ff7bd4)

# firewall zonesの考え方: LAN/Guest/VPNを分ける基本【OpenWrt集中連載016】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

OpenWrt系のfirewall zoneは、「どこからどこへ通信してよいか」を整理するための仕組みです。

LAN・Guest・IoT・VPNなどを別々の「ゾーン」として扱うことで、来客端末から社内LANへのアクセスを制限したり、IoT機器をNASやPCから切り離したりできます。

ただし、最初から細かなTraffic Rulesを大量に作る必要はありません。

まずは「GuestからLANへ届かない」「IoTから社内端末へ届かない」くらいを整理できれば、家庭や小さなオフィスではかなり十分です。

LN6001-JPでは、LuCIの **Network** → **Firewall** から、ゾーンと転送ルールを視覚的に管理できます。

この記事では、firewall zone の考え方、LAN/Guest/IoT/VPNの分け方、壊しにくい進め方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-01.png)

## この記事でわかること

- firewall zone が何を分ける仕組みなのか
- LAN、Guest、IoT、VPN の通信方針をどう考えるか
- ゾーン設定で壊しにくく進める考え方
- Traffic Rules をどこまで細かく作るべきか

## こんな人に向いています

- Guest、IoT、VPN の通信をLANと分けたい
- firewall zone の考え方が抽象的でつかみにくい
- ルールを増やす前に通信方針を整理したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-02.png)

## まずはここまでで十分

firewallも、最初から細かなTraffic Rulesを増やす必要はありません。

まずは次の3つで十分です。

1. LAN、Guest、IoT の通信方針を決める
2. ゾーンを分ける
3. GuestやIoTからLANへ行けないことを確認する

最初は「全部を制御する」より、「不要な方向へ届かない」ことを確認するほうが大事です。

先に通信方針を決めてから設定するだけでも、かなり壊しにくくなります。

## 用語ミニ解説

- firewall zone: 通信ルールを管理するためのグループです。LAN、WAN、Guest などのネットワークを区分けします。
- Forwarding: ゾーン間の通信の許可・拒否を設定するルールです。
- Traffic Rules: 特定のポートや送信元/宛先を条件にした通信ルールです。
- Input: そのゾーンからルーター自身への通信（ルーター管理画面へのアクセスなど）。
- Output: ルーターからそのゾーンへの通信。
- Forward: そのゾーンを経由して他のゾーンへ転送される通信。

最初は「zone = 通信グループ」くらいで整理すると分かりやすいです。

LAN、Guest、IoTごとに「どこへ行ってよいか」を決めるイメージです。

## デフォルトのゾーン設定

LN6001-JPの工場出荷時のファイアウォール設定:

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-01.png)

LAN→WANへの転送（インターネット接続）はForwardingルールで許可されています。

最初は、この「lan → wan」だけでも理解できれば十分です。

## ゾーン設計の考え方

ネットワーク分離を設計する前に、まず通信方針を決めます。

ここを先に整理しておくと、「なぜそのルールを作ったか」を後から見返しやすくなります。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-02.png)

最初は、「Guest → LAN拒否」だけでもかなり意味があります。

IoTやVPNは、必要になってから段階的に追加していくほうが整理しやすくなります。

方針を決めてから設定を進めると、後からルールの意図が分かりやすくなります。

最初からTraffic Rulesを大量追加するより、「どのゾーン間を許可するか」を先に決めるほうが壊しにくいです。

OpenWrt系では、「ネットワーク」「SSID」「firewall zone」を組み合わせて分離を作ります。

最初は少し複雑に見えますが、役割ごとに分かれているぶん、あとから整理しやすい構成になっています。

## 新しいゾーンを作る手順（LuCI）

Guest・IoT・VPN用のゾーンを追加する手順です。

### ステップ1: ゾーンを追加する

1. **Network** → **Firewall** → **Zones** タブを開く
2. **Add** をクリック
3. 以下を設定:

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-03.png)

4. **Save & Apply**

ここで作っているのは、「guestネットワーク専用の通信ルール」です。

guest側だけInputやForwardを制限することで、LANとは違う扱いにできます。

`Input: REJECT` にすることで、来客端末からルーター自身（管理画面443、80番ポート）へのアクセスも拒否されます。

最初は「Guestから管理画面へ入れない」だけでもかなり安全性が上がります。

### ステップ2: ゾーン間のForwardingを確認する

1. **Network** → **Firewall** → **Forwarding** タブ
2. 許可されているゾーン間転送が表示される
3. 不要な転送が許可されていないか確認

最初は、「guest → lan」が許可されていないことだけ確認できれば十分です。

### ステップ3: 追加のTraffic Rulesを設定する

Traffic Rulesは、「特定通信だけ例外的に許可・拒否したい」時に使います。

最初から大量に作りすぎると、あとから何を許可したか分からなくなりやすいです。

1. **Network** → **Firewall** → **Traffic Rules** → **Add**
2. Name: ルールの説明
3. Source zone / Destination zone を選択
4. Protocol / Destination port を設定
5. Action: `ACCEPT` または `REJECT`

例: Guestからルーター管理画面（443番）へのアクセスを拒否:

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/table-04.png)

最初は、Traffic Rulesよりzone設計を優先したほうが整理しやすくなります。

## UCIコマンドでゾーンを設定する

firewall zone はCLIでも設定できますが、最初はLuCIから設定したほうが構成を理解しやすいです。

CLIを使う場合も、まず現在設定を確認してから変更するほうが安全です。

### 変更前に現在の状態を保存する

firewallはネットワーク全体へ影響するため、変更前に設定ファイルと現在の適用状態をまとめて保存しておくと戻しやすくなります。

```sh
BACKUP_DIR="/root/firewall-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/firewall "$BACKUP_DIR/firewall"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/dhcp "$BACKUP_DIR/dhcp"

uci show firewall > "$BACKUP_DIR/uci-firewall.txt"
uci show network > "$BACKUP_DIR/uci-network.txt"
iptables-save > "$BACKUP_DIR/iptables-save.txt" 2>/dev/null || true
nft list ruleset > "$BACKUP_DIR/nft-ruleset.txt" 2>/dev/null || true
logread | tail -n 120 > "$BACKUP_DIR/logread-before.txt"

echo "backup: $BACKUP_DIR"
```

LN6001-JPのOpenWrt/QSDK系環境では、iptables互換コマンドで見える場合とnft側で見える場合があります。

どちらか一方だけに決め打ちせず、確認できるほうを残しておくと後から比較しやすくなります。

```sh
# 現在の設定を確認
uci show firewall

# 新しいゾーン（guest）を追加
uci add firewall zone
uci set firewall.@zone[-1].name='guest'
uci set firewall.@zone[-1].input='REJECT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='REJECT'
uci add_list firewall.@zone[-1].network='guest'

# guestからwanへの転送を許可
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='guest'
uci set firewall.@forwarding[-1].dest='wan'

# guest端末がIPアドレスを取得できるようDHCPを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Guest-DHCP'
uci set firewall.@rule[-1].src='guest'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].dest_port='67-68'
uci set firewall.@rule[-1].target='ACCEPT'

# guest端末がルーターのDNSを使えるようDNSを許可
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Guest-DNS'
uci set firewall.@rule[-1].src='guest'
uci set firewall.@rule[-1].proto='tcpudp'
uci set firewall.@rule[-1].dest_port='53'
uci set firewall.@rule[-1].target='ACCEPT'

# guestからlanへの転送を明示的に拒否するルールを追加
uci add firewall rule
uci set firewall.@rule[-1].name='Block Guest to LAN'
uci set firewall.@rule[-1].src='guest'
uci set firewall.@rule[-1].dest='lan'
uci set firewall.@rule[-1].target='REJECT'

# 反映前に差分を確認
uci changes firewall

uci commit firewall
/etc/init.d/firewall restart

# 反映後の確認
uci show firewall | grep -E "guest|Allow-Guest|Block Guest"
logread | grep -i 'firewall\|reject\|drop' | tail -n 50
```

`uci show firewall` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「今どう分離されているか」を確認する用途で使うくらいでも十分です。

## 現在のファイアウォール状態を確認する

設定後は、「本当に想定どおりに通信が止まっているか」を確認することが大事です。

```sh
# 全体の設定確認
uci show firewall

# iptables互換ルールを確認（実際に適用されているルール）
iptables -L -n -v 2>/dev/null || true

# nftables側で見える環境ではこちらも確認
nft list ruleset 2>/dev/null | grep -E 'guest|lan|wan|reject|drop' | head -n 80

# 特定ゾーンのforward chain確認
iptables -L FORWARD -n -v 2>/dev/null || true

# ファイアウォール関連のログ
logread | grep -i 'DROP\|REJECT\|firewall' | tail -n 50
```

最初は「REJECTやDROPが大量に出ていないか」を見るくらいでも十分役立ちます。

## ゾーン設定のよくある失敗

firewall設定で困る時は、「止めすぎた」か「分離できていない」かのどちらかが多いです。

最初は、Guest → LANが届かないことを確認できればかなり十分です。

### 設定後にインターネットにつながらない

**確認:** lan→wan のForwardingが存在するか確認

```sh
uci show firewall | grep forwarding
```

`src='lan' dest='wan'` のエントリがなければ追加する:

```sh
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='lan'
uci set firewall.@forwarding[-1].dest='wan'
uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

LAN→WAN forwarding が抜けると、LAN端末はインターネットへ出られなくなります。

最初は「lan → wan」が残っているかをまず確認すると切り分けしやすくなります。

### LuCI管理画面にアクセスできなくなった

- **原因:** LANゾーンのInputをREJECTに変更してしまった
- **対処:** 有線LAN接続でSSHアクセスしてfirewallをリセット:

```sh
cp /etc/config/firewall.backup.YYYYMMDD /etc/config/firewall
/etc/init.d/firewall restart
```

最初は、有線LAN接続を1本残した状態で作業するほうが安全です。

### Guestから社内LANにアクセスできてしまう

**確認:** guestゾーンのForwardがREJECTになっているか確認

```sh
uci show firewall | grep -A 5 "name='guest'"
```

ForwardがACCEPTになっていると、GuestからLANへそのまま届いてしまう場合があります。

### プリンターやNASにゲスト端末からアクセスできる

Guestゾーンの転送ルールで lan への転送が REJECT になっていても、特定ポートへのアクセスを許可するルールが残っている可能性があります:

```sh
# ゲストから社内へのルールを確認
iptables -L FORWARD -n -v | grep -i drop
```

Traffic Rulesで例外許可を作りすぎると、「なぜ通るのか」が分かりにくくなることがあります。

## 段階的に進める推奨手順

1. **まず**デフォルトの lan/wan ゾーン設定を確認して理解する
2. Guest Wi-FiのSSIDを作り、guestゾーンを追加する（Article 007参照）
3. 動作確認（Guest端末からLANへのpingが届かないことを確認）
4. 問題なければ IoT・VPN など追加のゾーンを検討する

一気に完成形を作るより、段階的に進めることで問題発生時の原因特定がかなり容易になります。

特に家庭や小さなオフィスでは、「Guestだけ分ける」くらいから始めるほうが運用しやすくなります。

## まとめ

firewall zone の設定は、次の順番で進めると整理しやすいです:

1. 通信方針（どのゾーンからどこへの転送を許可/拒否するか）を先に決める
2. **Network** → **Firewall** → **Zones** で新しいゾーンを追加する
3. Forwarding でゾーン間の転送を設定する
4. 必要に応じて Traffic Rules で細かいルールを追加する
5. `iptables -L -n -v` でルールが正しく適用されているか確認する

最初から細かなTraffic Rulesを大量追加するより、「GuestからLANへ届かない」ことを先に確認するほうが重要です。

設定変更前のバックアップと、有線接続での作業が安全対策の基本です。

設定後は、次の3つが確認できれば十分です。最初はここまで確認できれば大きく外していません。

- LAN からは必要な通信ができる
- Guest や IoT から LAN へ不用意に届かない
- 管理画面へ戻れなくなる変更をしていない

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/016/diagram-03.png)

## よくある質問

### firewall zone は何のために使う？

ネットワーク同士の通信をどこまで許可するかを整理するために使います。

GuestやIoTをLANから分けたい時に特に重要です。

### Guest と IoT は同じゾーンでもいい？

用途が違うなら分けたほうが管理しやすいです。

来客端末とIoT機器では、許可したい通信が違うことが多いためです。

### firewall 設定で一番気をつけることは？

管理画面へ戻れなくならないよう、有線接続で作業し、変更前にバックアップを取ってから進めることです。

特にLAN側Input設定を変更する時は注意します。

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
