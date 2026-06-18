<!-- mirror-source: articles/027-home-configuration.md -->

> Original note.com article: [自宅での設定例: 家族全員が使いやすいネットワークを組む【OpenWrt集中連載027】](https://note.com/ikmsan/n/n73a7fed30751)

# 自宅での設定例: 家族全員が使いやすいネットワークを組む【OpenWrt集中連載027】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPは、自宅用途でも家族ごとに使い分けたネットワーク設計ができます。

「大人のメインWi‑Fi」「子ども向けフィルタリング付きWi‑Fi」「来客用Guest Wi‑Fi」「スマート家電専用Wi‑Fi」を分けることで、セキュリティと快適さを両立しやすくなります。

ただし、最初から全部を分離しなくても大丈夫です。

まずは「メインWi‑Fiを安定させる」「Guestを追加する」くらいから始めるだけでも、かなり整理しやすくなります。

この記事では、自宅向けとして現実的なネットワーク分けの考え方、Kids/IoT/Guestの追加順序、あとから拡張しやすい構成を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-01.png)

## この記事でわかること

- 自宅で Main / Guest / Kids / IoT をどう分けるか
- 家族向けに無理のないネットワーク設計の進め方
- 自宅用途で後から機能追加しやすい構成の考え方
- KidsやIoTを分ける時の最小構成

## こんな家庭に向いています

- 来客用Wi‑Fi、子ども用Wi‑Fi、IoT用Wi‑Fiを整理したい
- 家族の使い方に合わせて少しずつ分離を進めたい
- 家庭向けでも将来の追加設定を見越して整えておきたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-02.png)

## まずはここまでで十分

自宅向け構成も、最初から全部を分けなくて大丈夫です。

まずは次の順番で十分です。

1. メインSSIDを安定して使える状態にする
2. Guestを追加する
3. 必要になったらKidsやIoTを足す

家庭では、「あとから増やせる形」で始めるほうが、運用もトラブル対応もかなり楽になります。

最初は、「全部を高度に分離する」より、「用途ごとにざっくり分ける」くらいで十分役立ちます。

## 自宅ネットワーク設計図

```
インターネット
    │
    ONU/HGW
    │
    LN6001-JP（192.168.1.1）
    ├── LAN（192.168.1.0/24）  ← 大人・メインPC・NAS
    │     SSID: HomeWifi-Main（WPA3-SAE）
    ├── Kids（192.168.3.0/24） ← 子どものスマホ・タブレット
    │     SSID: HomeWifi-Kids（DNS: Cloudflare Family）
    ├── IoT（192.168.4.0/24）  ← スマート家電・監視カメラ
    │     SSID: HomeWifi-IoT（外部に出るのみ）
    └── Guest（192.168.2.0/24）← 来客用
          SSID: HomeWifi-Guest（LAN隔離）
```

最初は、「大人メイン」「Guest」「IoT」の3つくらいから始めると整理しやすくなります。

---

メインSSIDは、「一番安定させたいネットワーク」として考えるほうが整理しやすくなります。

設定を増やす前に、現在の状態を保存します。

```sh
BACKUP_DIR="/root/home-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status.json" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
```

## ステップ1: メインSSIDの設定（大人・家族メイン）

設定詳細は 006 の記事を参照してください。

1. LuCI: **Network** → **Wireless** → `default_radio1`（5GHz）の **Edit** をクリック
2. **Interface Configuration** タブ:

| 設定項目 | 値 |
|---|---|
| SSID | HomeWifi-Main |
| Encryption | WPA3-SAE（または WPA2-PSK/WPA3-SAE mixed） |
| Key | 任意の強いパスワード（12文字以上推奨） |
| Network | lan |

3. 6GHzラジオ（`default_radio2`）にも同じSSID・パスワードを設定すると、MLO対応端末が自動的に最適バンドを使用します（031の記事参照）

最初は「5GHzを安定利用できる」だけでもかなり快適さが変わります。

---

Guest Wi‑Fiは、「家庭内LANへ入れないWi‑Fi」として考えると分かりやすくなります。

## ステップ2: ゲストWi-Fiの設定（来客用）

設定詳細は 007 の記事を参照してください。

1. 専用ネットワーク（192.168.2.0/24）を作成
2. ゲスト専用SSIDを作成しゲストネットワークに接続
3. firewallでguestネットワーク→LAN間通信をブロック

LuCI手順の概要:
1. **Network** → **Interfaces** → **Add new interface** → Name: `guest`, IP: `192.168.2.1/24`, DHCP有効
2. **Network** → **Wireless** → 新規SSID追加 → Network: `guest`
3. **Network** → **Firewall** → **Zones** → guestゾーン追加（Input: REJECT, Forward to wan: ACCEPT, Forward to lan: REJECT）

最初は、「Guestからインターネットだけ使える」状態を作れれば十分です。

---

Kidsネットワークは、「子ども端末を全部制御する」より、「まず危険サイトを減らす」くらいから始めるほうが整理しやすくなります。

## ステップ3: 子ども向けWi-Fiの設定（フィルタリング付き）

設定詳細は 010 の記事を参照してください。

Cloudflare Family DNS（1.1.1.3 / 1.0.0.3）を使って、アダルトコンテンツやマルウェアを自動ブロックします。

1. **Network** → **Interfaces** → **Add new interface** → Name: `kids`, IP: `192.168.3.1/24`
2. **DHCP Server** タブ → **Advanced Settings** → **DHCP-Options** に追加:
   - `6,1.1.1.3,1.0.0.3`
3. **Network** → **Wireless** → 新規SSID → SSID: `HomeWifi-Kids`, Network: `kids`
4. firewallでkidsゾーンを作成（wanへのforwardのみ許可）

```sh
# DHCP DNSオプションをUCIで設定
uci show dhcp.kids
cp /etc/config/dhcp /etc/config/dhcp.before-kids-dns.$(date +%Y%m%d-%H%M)
uci set dhcp.kids.dhcp_option='6,1.1.1.3,1.0.0.3'
uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart

# 反映後の確認
uci show dhcp.kids
logread | grep -i dnsmasq | tail -n 50
```

最初は「KidsだけDNSを変える」くらいでもかなり意味があります。

---

IoTネットワークは、「信用しきれない機器を分ける」くらいの考え方で十分です。

## ステップ4: IoT専用Wi-Fiの設定（スマート家電隔離）

スマートスピーカー、監視カメラ、ロボット掃除機などのスマート家電を専用ネットワークへ分離します。

1. **Network** → **Interfaces** → **Add new interface** → Name: `iot`, IP: `192.168.4.1/24`
2. **Network** → **Wireless** → 新規SSID → SSID: `HomeWifi-IoT`, Network: `iot`
3. firewallでiotゾーン作成（wanへのforwardのみ許可、lan/kidsへのforwardはREJECT）

IoTネットワークに置く端末の例:
- スマートスピーカー（Amazon Echo, Google Home等）
- ロボット掃除機
- スマートTV（NetflixやYouTube専用として使う場合）
- 監視カメラ
- スマート照明・コンセント

最初は「IoTからLANへ入れない」くらいで十分役立ちます。

---

NASやプリンターは、IPアドレスが変わると家族から「急に使えない」と見えやすい機器です。

## ステップ5: NASやプリンターのIPアドレスを固定する

設定詳細は 012 の記事を参照してください。

DHCPで動的割り当てすると、再起動時にIPアドレスが変わる可能性があります。

NAS、プリンター、スマートTVなどは固定割り当て（DHCP Static Lease）しておくと整理しやすくなります。

1. **Network** → **DHCP and DNS** → **Static Leases** タブ
2. **Add** をクリックして以下を入力:

| 端末 | MACアドレス | 割り当てIP |
|---|---|---|
| NAS | XX:XX:XX:XX:XX:XX | 192.168.1.100 |
| プリンター | XX:XX:XX:XX:XX:XX | 192.168.1.101 |
| スマートTV | XX:XX:XX:XX:XX:XX | 192.168.1.102 |

最初は「NASとプリンターだけ固定する」くらいでもかなり役立ちます。

---

広告ブロックは、「全部を完璧に止める」より、「家庭内の不要広告を減らす」くらいで考えるほうが運用しやすくなります。

## ステップ6: 広告ブロック（adblock）を設定する

設定詳細は 008 の記事を参照してください。自分でAdblock、ブロックリスト、LAN向けDNS配布、Force Local DNS、自動更新を組むのが難しい場合は、動作確認済みの導入スクリプトを使うのが一番早いです。

```sh
curl -sS -o /tmp/adb_setup.sh https://raw.githubusercontent.com/ikm-san/velop/main/adb_setup.sh && sh /tmp/adb_setup.sh -v
```

手動で進めたい場合は、まずパッケージ導入と状態確認だけ行います。

```sh
opkg list-installed | grep -E 'adblock|luci-app-adblock' || true
opkg update
opkg install adblock luci-app-adblock luci-i18n-adblock-ja
/etc/init.d/adblock enable
/etc/init.d/adblock start

/etc/init.d/adblock status
logread | grep -i adblock | tail -n 80
```

LuCI: **Services** → **Adblock** で有効化・ブロックリスト選択・ホワイトリスト追加ができます。導入スクリプトを使った場合も、あとからLuCIで許可リストやブロックリストを調整できます。

最初は「普段よく見るサイトが問題なく開けるか」を確認できれば十分です。

---

## 設定完了後の確認チェックリスト

最初の成功条件としては、次の3つを見れば十分です。最初はここまで確認できれば大きく外していません。

- メインSSIDから普通にインターネットへ出られる
- Guestから家庭内NASや管理画面へ届かない
- KidsやIoTを作った場合は、意図した制限だけ効いている

| 確認事項 | 確認方法 |
|---|---|
| メインSSIDでインターネット接続できる | スマートフォンから確認 |
| Guest SSIDからLAN側NASへ接続できないことを確認 | `ping 192.168.1.100` がタイムアウトになることを確認 |
| KidsネットワークDNSがCloudflare Familyになっているか | `nslookup example.com 1.1.1.3` |
| IoT端末がWANへ出られるか | スマート家電アプリから動作確認 |
| IoT端末からNASへアクセスできないか | NAS IPへpingがタイムアウトになることを確認 |

最初は、「GuestがLANへ届かない」「メインSSIDが安定している」の2つを確認できればかなり十分です。

ルーター側からは、次の確認をまとめて実行すると、どこまで設定できたかを棚卸ししやすくなります。

```sh
# SSIDと紐づくネットワークを確認
uci show wireless | grep -E 'ssid|network|encryption'

# ネットワークとDHCPを確認
uci show network | grep -E 'lan|guest|kids|iot|ipaddr'
uci show dhcp | grep -E 'guest|kids|iot|dhcp_option|host'

# firewall zoneとforwardingを確認
uci show firewall | grep -E 'guest|kids|iot|zone|forwarding'

# 接続中端末とログを確認
cat /tmp/dhcp.leases
logread | grep -Ei 'dnsmasq|firewall|adblock' | tail -n 80
```

---

## まとめ

全部を一度に入れる必要はありません。

まずはメインSSIDとGuestを分け、必要になったらKidsやIoTを順番に足していくほうが無理なく進められます。

自宅では、「あとから増やせる形」で分けておくだけでもかなり扱いやすくなります。

最初は「Main」「Guest」を安定して使えるだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/027/diagram-03.png)

## よくある質問

### 自宅のWi‑FiはSSIDを分けたほうがいい？

来客やIoT機器があるなら、分けたほうが整理しやすくなります。

最初はMainとGuestの2つからでも十分です。

### 自宅Wi‑FiでKids用とIoT用は両方作るべき？

必要性があれば作る、くらいで大丈夫です。

子ども用制御を優先したいならKids、スマート家電が多いならIoTを先に作るほうが整理しやすくなります。

### 自宅向けWi‑Fi設定は最初から全部入れるべき？

一度に全部入れる必要はありません。

Main、Guest、Kids、IoTの順で段階的に足すほうが安全です。

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
