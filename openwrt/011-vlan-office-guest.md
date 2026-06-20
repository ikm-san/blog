<!-- mirror-source: articles/011-vlan-office-guest.md -->

# VLANで社内・ゲストネットワークを分ける【OpenWrt集中連載011】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

VLANは、1台のルーターやスイッチ上で複数の論理ネットワークを分ける仕組みです。

小さなオフィスや店舗では、スタッフ端末、ゲストWi‑Fi、POS、防犯カメラなどを分けるために役立ちます。

ただし、最初から全部をVLAN化する必要はありません。まずはGuest Wi‑Fiとfirewall zoneで無線分離を作り、「有線端末も分けたい」となった段階でVLANを追加していくほうが現実的です。

LN6001-JPはOpenWrtベースなので、SSID、DHCP、VLAN、firewall zone を組み合わせた柔軟な設計ができます。

この記事では、小さなオフィスや店舗向けに、「どこまで分けるべきか」「どの順番で進めると崩しにくいか」を中心に整理していきます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-01.png)

## この記事でわかること

- VLANとGuest Wi‑Fiの違い
- 小さなオフィスや店舗でVLANをどう分けると整理しやすいか
- firewall zone を含めた基本的な分離の考え方
- VLAN設定でつまずきやすいポイントと安全な進め方

## こんな人に向いています

- ゲストWi-Fiだけでなく、有線のPOSやカメラも分けたい
- 小さなオフィスや店舗で Staff、Guest、Camera を整理したい
- VLANが必要か、Guest Wi‑Fiだけで足りるか判断したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-02.png)

## まずはここまでで十分

VLANは最初から完成形を作ろうとすると崩しやすいです。むしろ、最初は「最低限の分離」だけ作るほうが安全です。

まずは次の順で進めると安全です。

1. Guest Wi‑Fiとfirewallで無線分離を作る
2. DHCP予約で主要機器のIPを整理する
3. 必要になったらCameraやDevice用VLANを追加する

最初の目的は「全部を分けること」ではなく、「分けたい通信だけ確実に分けること」です。

小さなオフィスでは、複雑な構成より「誰がどのネットワークへ入るか」が整理されているほうが、運用しやすくなります。

## 設定前に現在の状態を控える

VLANやfirewallを触る前に、現在の状態を保存します。特に有線ポートやVLANを変更する場合、管理画面へ戻れなくなる可能性があるため、有線LANで管理端末を接続し、復旧手段を用意してから進めます。

```sh
BACKUP_DIR="/root/vlan-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ip link show > "$BACKUP_DIR/ip-link.txt"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
bridge vlan show > "$BACKUP_DIR/bridge-vlan.txt" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"

ls -l "$BACKUP_DIR"
```

このバックアップにはネットワーク構成やSSID情報が含まれるため、公開場所へそのまま貼らないようにします。

## 用語ミニ解説

- VLAN: 1台の機器上に複数の仮想ネットワークを作る仕組みです。
- firewall zone: ネットワーク同士の通信可否を決める境界線です。
- DHCP: 各ネットワークの端末へIPアドレスを自動配布する仕組みです。
- tagged/untagged: スイッチポートのVLANタグ設定です。通常、taggedはスイッチ間接続、untaggedは端末接続で使います。

## VLANとGuest Wi-Fiの違い

Guest Wi‑Fi（SSID分離＋firewall）は、無線端末の分離に向きます。

VLANは、有線端末も含めてネットワークを論理的に分ける場合に必要です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-01.png)

つまり、「ゲストWi‑Fiだけ分けたい」ならGuest Wi‑Fiで十分なこともあります。

一方で、POS、防犯カメラ、業務PCなど、有線も含めて整理したい場合はVLANが必要になります。

まずゲストWi‑FiのSSID分離を行い、有線機器の分離が必要になった段階でVLANへ進む順番が現実的です。

## 小さなオフィスのVLAN設計例

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-02.png)

最初から細かく分けすぎる必要はありません。

例えば、StaffとGuestだけでもかなり意味があります。CameraやIoTは、必要になってから追加しても遅くありません。

POSや決済端末は、ベンダーのサポート条件を優先します。

勝手にVLANへ組み込まず、ベンダーへ確認してから対応を決めてください。業務系端末は、通信要件がかなり限定されている場合があります。

## firewall通信方針を先に決める

VLANを作る前に、「どのネットワークからどこへ通信してよいか」を先に決めておくと、あとから設定が崩れにくくなります。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/table-03.png)

特にGuestからStaffへ届かないことを最優先に確認します。

Cameraネットワークも、「録画機だけ許可」「クラウドだけ許可」のように、必要最小限へ寄せたほうが整理しやすいです。

OpenWrt系では、「VLAN」「ネットワーク」「SSID」「firewall zone」を別々に組み合わせて作ります。

最初は少し複雑に見えますが、役割ごとに分かれているぶん、あとから整理しやすいのが特徴です。

### ステップ1: 各ネットワークインターフェースを作成する

**Guestネットワーク（VLAN ID: 10）:**

1. **Network** → **Interfaces** → **Add new interface**
2. Name: `guest`
3. Protocol: `Static address`
4. IPv4 address: `192.168.2.1`、Netmask: `255.255.255.0`
5. **DHCP Server** タブ → Setup DHCP Server
   - Start: `100`、Limit: `50`、Leasetime: `1h`
6. **Save**

ここで作っているのは、Guest端末専用のIPアドレス空間です。

Staff側とは別の `192.168.2.x` を使うことで、あとからfirewallで分離しやすくなります。

**Cameraネットワーク（VLAN ID: 20）:**

1. **Network** → **Interfaces** → **Add new interface**
2. Name: `camera`
3. Protocol: `Static address`
4. IPv4 address: `192.168.3.1`、Netmask: `255.255.255.0`
5. **DHCP Server** タブ → Setup DHCP Server
   - Start: `100`、Limit: `20`、Leasetime: `12h`
6. **Save**

Cameraネットワークは、防犯カメラや録画機などをまとめる用途を想定しています。

「Camera → Staffは拒否」「Staff → Cameraだけ許可」のような構成にすると整理しやすくなります。

### ステップ2: firewall zonesを設定する

ここが実際にネットワーク間の通信を制御する部分です。

VLANやSSIDを分けても、firewall zone を分けなければ通信できてしまう場合があります。

1. **Network** → **Firewall** → **Zones** タブ
2. **Add** をクリック（guestゾーン）:
   - Name: `guest`
   - Input: `REJECT`、Output: `ACCEPT`、Forward: `REJECT`
   - Covered networks: `guest`
   - Allow forward to: `wan`
3. **Add** をクリック（cameraゾーン）:
   - Name: `camera`
   - Input: `REJECT`、Output: `ACCEPT`、Forward: `REJECT`
   - Covered networks: `camera`
   - Allow forward to: （空白）※インターネット接続不要ならwanも外す
4. **Save & Apply**

この設定では、「Guest → WAN」は許可し、「Guest → Staff」は拒否されます。

Camera側も、必要な通信だけ許可する方向で作ると、あとから見返した時に分かりやすくなります。

### ステップ3: StaffからCameraへのアクセスを許可する

1. **Network** → **Firewall** → **Traffic Rules** → **Add**
2. Name: `Staff to Camera`
3. Source zone: `lan`（Staffネットワーク）
4. Destination zone: `camera`
5. Action: `ACCEPT`
6. **Save & Apply**

監視画面確認や録画管理など、必要な方向だけ許可するイメージです。

最初から双方向通信を全部許可しないほうが、構成がシンプルになります。

### ステップ4: SSIDをネットワークに紐付ける

各SSIDを対応するネットワークに紐付けます（Guest Wi-Fi設定の手順と同じ）:

1. **Network** → **Wireless** → 任意帯域の **Add**
2. ESSID: `Staff_WiFi` / `Guest_WiFi` など
3. Network: 対応するネットワーク（`lan` / `guest` / `camera`）を選択
4. **Save** → **Apply**

Staff用とGuest用SSIDは、名前を明確に分けたほうが運用しやすいです。

特に店舗では、ゲスト側が迷わないSSID名にしておくとトラブルを減らしやすくなります。

## 有線ポートのVLAN割り当て（高度な設定）

LN6001-JPの物理的なLANポートを特定のVLANへ割り当てる場合、ネットワークスイッチ設定も関係してきます。

ここは設定を間違えると管理画面へ戻れなくなることもあるため、最初は無理に触らなくても大丈夫です。

**注意事項:**
- 有線ポートのVLAN設定を誤ると、管理画面へのアクセスが失われる場合があります
- 設定前に必ずバックアップを取り、有線でLANポートに接続した状態で作業してください

まずはSSID分離とfirewallだけで運用し、必要になった時に有線VLANへ進むほうが安全です。

### 現在のスイッチ構成を確認する

```sh
echo "### Linux interfaces"
ip link show

echo "### bridge VLAN table, if available"
bridge vlan show 2>/dev/null || true

echo "### swconfig, if available"
swconfig list 2>/dev/null || true
swconfig dev switch0 show 2>/dev/null || true

echo "### network UCI hints"
uci show network | grep -E 'device|bridge|ports|vlan|switch|ifname|proto'
```

まずは「今どう構成されているか」を確認するところから始めます。

CLIは、最初は変更より状態確認に使うくらいで十分です。

### UCIで有線ポートにVLANを割り当てる

LN6001-JPのポートマップを確認した上で設定します（ポート番号はファームウェアで異なる場合があります）:

```sh
echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)

echo "### current VLAN-related config"
uci show network | grep -E 'device|bridge|ports|vlan|switch|ifname'
```

有線ポートVLANは、実機ポート番号を勘違いすると管理アクセスを失いやすい部分です。

変更前バックアップと、有線LAN接続での作業を強くおすすめします。

有線ポートのVLAN割り当ては、LN6001-JPの実機ポートマップを確認してから設定します。ポート番号はLuCIの **Network** → **Switch** ページでも確認できます。

最初は「どのポートがどこにつながっているか」を整理するだけでも十分役立ちます。

## 設定確認コマンド

```sh
echo "### network"
uci show network

echo "### firewall"
uci show firewall

echo "### addresses"
ip addr show

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### routes"
ip route show

echo "### communication tests from staff network"
ping -c 3 192.168.2.1  # guestルーターへのping
ping -c 3 192.168.3.1  # cameraルーターへのping
```

`uci show network` と `uci show firewall` は、ネットワーク分離がどう組まれているかを見る時にかなり便利です。

CLIは、最初は設定変更より「今どうつながっているか」を確認する用途で使うくらいで十分です。

## 設定変更前のバックアップ

```sh
BACKUP_DIR="/root/vlan-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ip link show > "$BACKUP_DIR/ip-link.txt"
bridge vlan show > "$BACKUP_DIR/bridge-vlan.txt" 2>/dev/null || true

ls -l "$BACKUP_DIR"
```

VLAN設定は、network / firewall / wireless / dhcp がまとめて関係してきます。

変更前バックアップを残しておくと、問題が起きた時の切り分けがかなり楽になります。

## よくある失敗と対処

VLAN設定で困る時は、「通信を止めすぎた」か「分離できていない」かのどちらかが多いです。

最初は、Guest → Staff遮断だけでも確認できれば十分です。

### 管理画面にアクセスできなくなった

firewallやVLAN設定では、自分自身の管理アクセスまで止めてしまうことがあります。

最初は有線LANを1本つないだ状態で作業すると復旧しやすくなります。

**原因:** firewallの設定を誤ってLAN→ルーター間の通信を遮断した
**対処:** 有線LAN接続でアクセスを試みる。アクセスできない場合はバックアップから復元する:
```sh
cp /etc/config/firewall.backup.YYYYMMDD /etc/config/firewall
/etc/init.d/firewall restart
```

### GuestからLANに通信できてしまう

**原因:** firewall forwardingルールでguestからlanへの転送が許可されている
**対処:** **Network** → **Firewall** → **Traffic Rules** で `Block Guest to LAN` ルールを追加する

### カメラがインターネットに接続できない

**原因:** cameraゾーンのfirewall forwardingでwanへの転送が許可されていない
**対処:** cameraゾーンの「Allow forward to destination zones」に `wan` を追加する

## 段階的に進める推奨手順

ここまでで、次の3つが確認できれば大きく外していません。最初はここまで確認できれば十分です。

- Guest から Staff や Camera へ直接届かない
- Staff から必要な機器だけにアクセスできる
- 管理画面へ戻れなくなるような変更をしていない

1. **まずGuest Wi-Fiを分ける**（SSIDとfirewall zoneのみ）
2. **主要機器のDHCP予約を行う**（IPアドレスの把握）
3. **Cameraネットワークを追加する**
4. **必要なら有線ポートのVLAN割り当てを検討する**
5. **設定のたびにバックアップを取る**

一気に完成形を作るより、段階的に進めると問題発生時の原因特定が容易になります。

特に小さなオフィスでは、「運用しながら少しずつ整理する」くらいの進め方のほうが現実的です。

## まとめ

VLANによるオフィス向けネットワーク分離は、次の順番で考えると整理しやすいです:

1. ネットワーク（インターフェース）をIPアドレス範囲ごとに作成
2. SSIDをそれぞれのネットワークに紐付け
3. firewall zoneで通信の許可/拒否を設定
4. 必要なら有線ポートのVLAN割り当てを追加

Guest Wi‑FiのSSID分離で始め、有線機器の分離が必要になった段階でVLANを導入する順番が安全です。

最初から全部をVLAN化するより、「GuestをStaffへ入れない」「Cameraを必要最小限だけ許可する」といった整理から始めるほうが、運用しやすくなります。

設定変更前のバックアップと、有線接続での作業を習慣にしてください。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/011/diagram-03.png)

## よくある質問

### Guest Wi-FiがあればVLANは不要？

無線だけの分離ならGuest Wi‑Fiで足りることもあります。

有線端末も含めてStaff、POS、Cameraを分けたいならVLANが必要になります。

### VLANは家庭でも必要？

必須ではありません。

小さなオフィスや店舗のように、用途ごとに端末を明確に分けたい環境で特に効果を発揮します。

### VLAN設定はどこから始めるのが安全？

まず Guest Wi-Fi と DHCP予約で構成を整理し、そのあと Camera や Device 用ネットワークを足す流れが安全です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt network configuration: https://openwrt.org/docs/guide-user/network/network_configuration
- OpenWrt firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration

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
