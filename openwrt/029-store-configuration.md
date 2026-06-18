<!-- mirror-source: articles/029-store-configuration.md -->

> Original note.com article: [小規模店舗での設定例: スタッフ・お客様・機器を分けて管理する【OpenWrt集中連載029】](https://note.com/ikmsan/n/n506a774c0680)

# 小規模店舗での設定例: スタッフ・お客様・機器を分けて管理する【OpenWrt集中連載029】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

小規模店舗では、「スタッフ用」「お客様用Guest」「POSレジ・監視カメラなどの業務端末」を分けるだけでも、セキュリティと運用管理がかなり整理しやすくなります。

特に店舗では、「お客様Wi‑Fiへ業務端末を混ぜない」ことがかなり重要です。

ただし、最初からVLANやVPNまで全部細かく作り込まなくても大丈夫です。

まずは「Staff」「Guest」「Device」を分けるだけでも、トラブル切り分けや管理はかなり楽になります。

この記事では、小規模店舗で無理なく運用しやすいネットワークの分け方、固定IPやVPNを追加する順番、あとから拡張しやすい考え方をまとめます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-01.png)

## この記事でわかること

- 店舗で Staff / Guest / Device をどう分けるか
- 小規模店舗で無理のないネットワーク設計
- 固定IP管理やVPNをどの順番で足すべきか
- 店舗向けで後から拡張しやすい構成の考え方

## こんな店舗に向いています

- スタッフ用、来客用、業務機器用の3系統を整理したい
- POSや監視カメラを来客Wi‑Fiと分けたい
- 小規模店舗でも後からVPNや固定IP管理を足したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-02.png)

## まずはここまでで十分

店舗構成も、最初から全部を細かく作らなくて大丈夫です。

まずは次の3つで十分です。

1. Staff、Guest、Device の3系統を分ける
2. 業務端末をGuestへ混ぜない
3. 重要機器だけDHCP予約で固定する

最初は「来客用と業務用が混ざらない」状態を作るだけでも、運用はかなり安定します。

最初は、「全部を高度に分離する」より、「用途ごとにざっくり分ける」くらいで十分役立ちます。

## 設定前に店舗の現状を保存する

店舗構成では `network`、`wireless`、`firewall`、`dhcp` をまとめて変更します。POSやカメラがある場合、設定ミスの業務影響が大きいため、作業前に必ず状態を保存します。

```sh
BACKUP_DIR="/root/store-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
ip addr show > "$BACKUP_DIR/ip-addr.txt"

ls -l "$BACKUP_DIR"
```

バックアップにはSSID、MACアドレス、店舗機器名などが含まれるため、そのまま公開場所へ貼らないようにします。

## 店舗ネットワーク設計図

```
インターネット
    │
    ONU/HGW
    │
    LN6001-JP（192.168.1.1）
    ├── Staff LAN（192.168.1.0/24）  ← スタッフPC・iPadレジ・NAS
    │     SSID: Staff-Wifi（WPA3-SAE, 非公開推奨）
    ├── Guest（192.168.2.0/24）      ← お客様向けゲストWi-Fi
    │     SSID: Guest-Wifi（LAN隔離）
    └── Device（192.168.3.0/24）     ← 監視カメラ・POSレジ・スマートロック
          SSID: Device-Wifi（LAN隔離、外部からのアクセス制限）
```

最初は、「Staff」「Guest」「Device」の3つを安定運用できればかなり十分です。

---

スタッフ用ネットワークは、「業務で一番安定させたいネットワーク」として考えるほうが整理しやすくなります。

## ステップ1: スタッフ用ネットワーク（LAN）の設定

デフォルトのLAN（192.168.1.0/24）をスタッフ用として使います。

1. LuCI: **Network** → **Wireless** → `default_radio1`（5GHz）の **Edit**
2. SSID: `Staff-Wifi`、Encryption: WPA3-SAE、強いパスワードを設定（12文字以上）
3. **Network**: `lan`

スタッフSSIDは店内掲示せず、入社時に個別共有する運用のほうが安全です。

スタッフ用IPアドレス設計（DHCP Static Lease推奨）:

| 端末 | 固定IP | 用途 |
|---|---|---|
| スタッフPC | 192.168.1.100〜109 | 業務PC |
| iPadレジ | 192.168.1.110〜119 | POSアプリ |
| NAS | 192.168.1.200 | データ共有 |
| プリンター | 192.168.1.201 | レシートプリンター等 |

最初は「NAS」「POS」「プリンター」だけ固定IP化するくらいでもかなり役立ちます。

固定IP設定は **Network** → **DHCP and DNS** → **Static Leases**（012の記事参照）から行います。

---

Guest Wi‑Fiは、「インターネットだけ使えるネットワーク」として考えると整理しやすくなります。

## ステップ2: お客様向けゲストWi-Fiの設定

詳細は 007 の記事を参照してください。

1. **Network** → **Interfaces** → **Add new interface**:
   - Name: `guest`, Protocol: Static Address, IPv4 address: `192.168.2.1`, Netmask: `255.255.255.0`
   - **DHCP Server** タブ: **Enable DHCP** → Start: `100`, Limit: `100`, Leasetime: `2h`
2. **Network** → **Wireless** → 新規SSID:
   - SSID: `Guest-Wifi`（お客様に見せる名前）
   - Encryption: WPA2-PSK（定期的にパスワード変更する）
   - Network: `guest`
3. **Network** → **Firewall** → **Zones** → `guest`ゾーン追加:
   - Input: REJECT, Output: ACCEPT, Forward: REJECT
   - Allow forward to `wan`: チェック
4. Traffic Rules: Guest → LAN方向のForwardをREJECT

```sh
BACKUP_DIR="/root/store-guest-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
for cfg in network dhcp firewall wireless; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

echo "### create guest network"
uci set network.guest=interface
uci set network.guest.proto='static'
uci set network.guest.ipaddr='192.168.2.1'
uci set network.guest.netmask='255.255.255.0'
uci changes network
uci commit network

echo "### configure guest DHCP"
uci set dhcp.guest=dhcp
uci set dhcp.guest.interface='guest'
uci set dhcp.guest.start='100'
uci set dhcp.guest.limit='100'
uci set dhcp.guest.leasetime='2h'
uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart
```

最初は、「Guestからインターネットだけ使える」状態を作れれば十分です。

最初は「GuestからLANへ届かない」ことを確認できればかなり役立ちます。

---

POSや監視カメラは、「来客端末と分ける」だけでもかなり安全性が変わります。

## ステップ3: 業務端末専用ネットワークの設定（POSレジ・監視カメラ）

POSレジや監視カメラは、Guestネットワークから分離します。

1. **Network** → **Interfaces** → **Add new interface**:
   - Name: `device`, Protocol: Static Address, IPv4 address: `192.168.3.1`
2. **Network** → **Wireless** → 新規SSID:
   - SSID: `Device-Wifi`（非公開SSID推奨: Hidden SSIDへチェック）
   - Encryption: WPA2-PSK または WPA3-SAE
   - Network: `device`
3. **Network** → **Firewall** → `device`ゾーン作成（wan forward: ACCEPT, lan forward: REJECT）

業務端末に固定IPを設定（DHCP Static Lease）:

| 端末 | 固定IP |
|---|---|
| 監視カメラ1 | 192.168.3.10 |
| 監視カメラ2 | 192.168.3.11 |
| POSレジ端末 | 192.168.3.20 |
| スマートロック | 192.168.3.30 |

最初は「監視カメラ」と「POSレジ」だけ固定IP管理するくらいでも十分です。

---

店舗では、「外から安全に確認できる」だけでも運用がかなり楽になります。

## ステップ4: スタッフが外出先からアクセスできるVPNの設定

- **固定IPあり・ポート開放可能**: WireGuard（028の記事参照）
- **IPoE・固定IPなし**: Tailscale（Linksys公式VPN Assistant経由）

Tailscaleを使う場合は、スタッフデバイスをTailscaleアカウントへ参加させ、LN6001-JPをサブネットルーターとして設定します。

スタッフ退職時は、管理コンソールからデバイスを削除します。

---

## 設定完了後の確認チェックリスト

最初の成功条件としては、次の3つを見れば十分です。最初はここまで確認できれば大きく外していません。

- Staffから業務機器へ正常アクセスできる
- Guestから業務LANへ届かない
- Deviceネットワーク機器が想定どおり動く

| 確認事項 | 方法 |
|---|---|
| スタッフSSIDでインターネット・NASへ接続できる | PCから確認 |
| ゲストSSIDでインターネットへ接続できる | スマートフォンから確認 |
| ゲストSSIDからスタッフLANのNASへ接続できない | `ping 192.168.1.200` がタイムアウト |
| Device SSIDの監視カメラがクラウド接続できる | カメラアプリから確認 |
| Device SSIDからスタッフPCへ接続できない | `ping 192.168.1.100` がタイムアウト |

最初は、「GuestがStaffへ届かない」「POSやカメラが普通に動く」の2つを確認できればかなり十分です。

SSHでは、次のように確認できます。

```sh
echo "### network interfaces"
uci show network | grep -E 'lan|guest|device|proto|ipaddr'

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### firewall zones"
uci show firewall | grep -E 'zone|forwarding|guest|device|lan|wan'

echo "### Wi-Fi status"
wifi status

echo "### logs"
logread | grep -Ei 'dnsmasq|firewall|guest|device|reject|drop' | tail -n 100
```

---

## 店舗運用のポイント

設定後は、次の3つが確認できれば最初の段階として十分です。

- GuestからStaffやDeviceへ届かない
- POSや監視カメラが想定どおり動く
- スタッフ用Wi‑Fiだけで管理画面や業務機器へ触れられる

| 場面 | 推奨対応 |
|---|---|
| お客様Wi‑Fiパスワード変更 | 月1回程度。LuCI **Network** → **Wireless** → **Edit** → Key変更 |
| スタッフ退職時 | スタッフSSIDパスワード変更、VPN端末無効化 |
| 機器追加時 | `device`ネットワークへStatic Lease追加して固定IP管理 |
| 障害発生時 | 021 のトラブルシューティング記事を参照 |

最初は、「Guest隔離」と「業務機器安定動作」を優先するだけでもかなり運用しやすくなります。

---

## まとめ

店舗では、まず Staff、Guest、Device の3つへ分けるだけでも運用がかなり安定します。

そこへ固定IP管理やVPNを必要な範囲だけ追加していくと、日々の管理とトラブル切り分けがかなりしやすくなります。

最初は、「Guestを業務LANへ入れない」「POSやカメラを安定動作させる」の2つを優先するだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/029/diagram-03.png)

## よくある質問

### 小規模店舗ネットワークはいくつへ分けるべき？

まずは Staff、Guest、Device の3つで十分です。

これだけでも管理と切り分けがかなりしやすくなります。

### 店舗のPOSや監視カメラはGuest Wi‑Fiと同じでもいい？

避けたほうが安全です。

業務端末は来客ネットワークと分けたほうが管理しやすく、リスクも下げやすくなります。

### 店舗ネットワークをリモート確認したい時は何を使う？

固定IPやポート開放条件が合うならWireGuard、難しいならTailscaleを検討すると整理しやすくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
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
