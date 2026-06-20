<!-- mirror-source: articles/032-ap-bridge-wds.md -->

# APモード・ブリッジ・WDS: LN6001-JPを既存ネットワークに組み込む【OpenWrt集中連載032】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPは、既存ルーター配下へアクセスポイント（AP）やWDS子機として追加できます。

「既存ルーターはそのまま使いたい」「Wi‑Fiだけ強化したい」「有線が引けない場所へ子機を置きたい」といったケースでかなり役立ちます。

ただし、最初からLuCIやCLIで手動設定しなくても大丈夫です。

Linksys公式サポートでは、Setup Assistantを使ってAP BridgeやWDS子機を設定する流れが案内されています。

この記事では、AP BridgeとWDSの違い、Setup Assistantを使う流れ、設定後に管理画面へ戻れなくならないためのポイントを整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-01.png)

## この記事でわかること

- AP BridgeとWDS子機の違い
- LN6001-JPを既存ネットワークへ追加する考え方
- Setup Assistantを使って安全に進める流れ
- 設定後に管理IPを見失わないポイント

## こんな人に向いています

- 既存ルーターはそのまま使い、LN6001-JPをAPとして追加したい
- 有線バックホールか無線子機かで迷っている
- 手動設定より、公式Setup Assistantから始めたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-02.png)

## まずはここまでで十分

この構成は、最初から手動で細かく触るより、まず次の3つを守るほうが安全です。

1. AP BridgeとWDSのどちらにするか決める
2. Setup Assistantを使って設定する
3. 変更後の管理IPアドレスを必ず控える

特に管理IPを見失わないことが、設定後に戻れなくならないための基本です。

最初は、「既存ルーターを残すか」「LN6001-JPを主ルーターにするか」を分けて考えると整理しやすくなります。

## 構成パターンの比較

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-01.png)

最初は、「有線ならAP Bridge」「有線が難しいならWDS」くらいで整理すると分かりやすくなります。

---

最初は、LuCIやCLIを直接触るより、公式Setup Assistantから始めるほうが切り分けしやすくなります。

Setup Assistantを入れる前に、管理IPと現在設定を控えます。

```sh
BACKUP_DIR="/root/ap-wds-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

ubus call system board > "$BACKUP_DIR/system-board.json" 2>/dev/null || true
uci show network > "$BACKUP_DIR/network.uci.txt"
uci show wireless > "$BACKUP_DIR/wireless.uci.txt"
uci show firewall > "$BACKUP_DIR/firewall.uci.txt"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/wireless "$BACKUP_DIR/wireless"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
ifstatus lan > "$BACKUP_DIR/ifstatus-lan.json" 2>/dev/null || true
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
wifi status > "$BACKUP_DIR/wifi-status.json" 2>/dev/null || true

echo "backup: $BACKUP_DIR"
```

## Setup Assistantの導入手順

1. Linksys公式サポートページ（https://support.linksys.com/kb/article/7195-jp/）から `setup-assistant.ipk` をダウンロード
2. LuCIで **System** → **Software** を開く
3. **Update lists** をクリックしてパッケージリストを更新
4. 完了後、**Dismiss** で閉じる
5. **Upload Package** をクリック
6. ダウンロードした `setup-assistant.ipk` を **Browse** で選び、**Upload**
7. 確認画面で **Install** をクリック
8. `installed in root is up to date.` と表示されたら **Dismiss** をクリック
9. ブラウザをリロードするか開き直し、**Services** → **Setup Assistant** が表示されることを確認

最初は、「Services → Setup Assistant」が見えるかを確認できれば十分です。

SSHから確認する場合は、次を見ます。

```sh
opkg list-installed | grep -Ei 'setup|assistant'
logread | grep -Ei 'setup|assistant|opkg' | tail -n 80
```

---

AP Bridgeは、「既存ルーター配下へWi‑Fiを追加する」構成として考えると整理しやすくなります。

## AP Bridgeの設定手順

AP Bridgeでは、親ルーター配下でLN6001-JPをアクセスポイントとして使います。

公式Setup Assistantでは、必要項目を入力してまとめて反映します。

1. **Services** → **Setup Assistant** を開く
2. **AP Bridge** タブを選ぶ
3. 以下の項目を入力する

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-02.png)

4. **Save & Apply** をクリック
5. 念のためLN6001-JPを再起動
6. 変更後は **Access Point IP** に指定したIPアドレスでLuCIへアクセス

最初は、「新しいIPアドレスでLuCIへ入り直せるか」を確認できればかなり十分です。

設定後は、旧IPではなく新しく指定した **Access Point IP** に接続します。

```sh
# PC側から確認
AP_IP="192.168.1.2"
ping "$AP_IP"
ssh root@"$AP_IP"

# ルーター側へ入れたら状態確認
uci show network.lan
ip route show
cat /tmp/dhcp.leases
logread | tail -n 80
```

---

WDSは、「有線が引けない場所へ子機を置く」用途とかなり相性があります。

## WDS子機モードの設定手順

WDSは、有線バックホールではなく、親SSIDへ無線接続して子機側を構成する方式です。

Linksys公式ページでは、WDS子機として使う場合もSetup Assistantで設定します。

1. **Services** → **Setup Assistant** を開く
2. **WDS Configuration** タブを選ぶ
3. 以下の項目を入力する

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-03.png)

4. **Save & Apply** をクリック
5. 念のためLN6001-JPを再起動
6. 子機側のIPアドレスでLuCIへアクセスできるか確認

最初は、「子機側IPアドレスへ入れるか」を確認するだけでもかなり役立ちます。

WDS後の確認では、親SSIDへ接続できているか、子機側SSIDへ端末が接続できるかを分けます。

```sh
# 無線インターフェースと接続状態を確認
wifi status
iw dev
WLAN_IF="ath00"  # iw devで表示されたInterface名に置き換える
iw dev "$WLAN_IF" link 2>/dev/null || true

# WDS/無線関連ログ
logread | grep -Ei 'wds|wifi|hostapd|wpa' | tail -n 100
```

---

設定直後は、「壊れた」のではなく、「IPアドレスやWi‑Fi状態が変わっただけ」のケースもかなり多くあります。

公式ページでは、うまくいかない場合に **Network** → **Wireless** を開き、Wi‑Fi関連インターフェース表示を確認して初期調整する手順が案内されています。

初回表示で確認画面が出た場合は **Continue** をクリックし、Wi‑Fiサービス再起動後に画面が正しく表示されることを確認します。

最初は、「新しいIPアドレスで入れているか」を確認するだけでもかなり切り分けしやすくなります。

どうしても戻れない場合でも、SSHへ入れるなら直前のバックアップから戻せます。

```sh
# 変更前バックアップから戻す例
cp /root/ap-wds-before-YYYYMMDD-HHMM/network /etc/config/network
cp /root/ap-wds-before-YYYYMMDD-HHMM/wireless /etc/config/wireless
cp /root/ap-wds-before-YYYYMMDD-HHMM/firewall /etc/config/firewall

/etc/init.d/network restart
wifi reload
/etc/init.d/firewall restart
```

---

AP BridgeやWDSでは、「設定自体は成功しているが、アクセス先IPが変わっていた」ケースがかなり多くあります。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/table-04.png)

最初は、「新しい管理IPへ入れるか」「親SSIDへ接続できているか」の2つを優先確認すると整理しやすくなります。

---

## まとめ

LN6001-JPは、既存ルーター配下へAP BridgeやWDS子機として追加できます。

まずは、Setup Assistantを使って安全に構成し、変更後IPアドレスを必ず控えることがかなり重要です。

有線バックホールならAP Bridge、無線で中継したいならWDS、と分けて考えると整理しやすくなります。

最初は、「新しいIPでLuCIへ入り直せる」「親ネットワークへ普通に接続できる」の2つを確認できれば十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/032/diagram-03.png)

## よくある質問

### LN6001-JPのAP BridgeとWDSは何が違う？

AP Bridgeは有線で親ルーター配下へ入る構成、WDSは親SSIDへ無線接続する子機構成です。

最初は、有線のAP Bridgeのほうが安定しやすくなります。

### 既存ルーター配下でLN6001-JPをアクセスポイントとして使える？

使えます。

Setup AssistantのAP Bridgeを使うと、同一サブネットのアクセスポイントとして組み込みやすくなります。

### LN6001-JPのWDSはメッシュWi‑Fiと同じ？

同じではありません。

WDSは親SSIDへぶら下がる子機構成で、EasyMeshやVelop系メッシュとは別の仕組みです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys AP/bridge/WDS setup assistant: https://support.linksys.com/kb/article/7195-jp/

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
