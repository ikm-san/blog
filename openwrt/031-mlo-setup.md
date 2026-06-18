<!-- mirror-source: articles/031-mlo-setup.md -->

> Original note.com article: [MLOセットアップ: Wi‑Fi 7のマルチリンク動作を活かす【OpenWrt集中連載031】](https://note.com/ikmsan/n/n00750e23f49c)

# MLOセットアップ: Wi‑Fi 7のマルチリンク動作を活かす【OpenWrt集中連載031】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

MLO（Multi-Link Operation）は、Wi‑Fi 7の目玉機能の1つで、2.4GHz・5GHz・6GHzの複数バンドを同時利用する技術です。

通信速度向上だけでなく、遅延低減や接続安定性改善も期待できます。

ただし、最初から「絶対にMLOを使うべき」と考えなくても大丈夫です。

まずは「Wi‑Fi 7端末があるか」「5GHzが安定しているか」「通常SSID運用でも困っていないか」を確認するだけでも、かなり整理しやすくなります。

この記事では、LN6001-JPでMLOを有効化する条件、通常SSIDとの違い、MLOを無理に使わなくてもよいケースを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-01.png)

## この記事でわかること

- MLOが通常Wi‑Fi接続とどう違うか
- LN6001-JPでMLOを有効化する条件
- MLOを試すべきケースと無理に使わなくてもよいケース
- MLO確認時に見たいポイント

## こんな人に向いています

- Wi‑Fi 7対応端末を持っていてMLOを試したい
- 6GHzを含めた構成をどうそろえるか知りたい
- MLOを使うべきか通常SSID運用で十分か迷っている

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-02.png)

## まずはここまでで十分

MLOは必須機能ではありません。

まずは次の3つで十分です。

1. ルーターと端末が両方ともWi‑Fi 7対応か確認する
2. 全帯域でSSID、パスワード、WPA3-SAEをそろえる
3. 効果が薄ければ通常SSID運用へ戻せる前提で試す

MLOは、「使えたら便利」くらいで考えるほうが無理なく判断しやすくなります。

最初は、「理論最大速度」より、「複数バンドをまとめて安定利用できる技術」と考えるほうが整理しやすくなります。

## MLOとは

| 項目 | 従来のバンドステアリング | MLO（Wi-Fi 7） |
|---|---|---|
| バンド切り替え | どれか1つのバンドを使用 | 複数バンドを同時使用 |
| 遅延 | 切り替え時に遅延が発生 | 複数リンクで冗長性・低遅延 |
| 速度 | 1バンドの最大速度 | 複数バンドの合算に近い速度 |
| 要件 | なし | 端末・ルーター両方がWi‑Fi 7対応必要 |

MLOの恩恵を受けるには、次が必要です:

1. LN6001-JP（ルーター側）: ファームウェア1.2.0.15以降
2. 接続端末: Wi‑Fi 7（IEEE 802.11be）対応スマートフォン・PCが必要

Wi-Fi 7非対応端末は従来どおり1バンドで接続します（MLOにはならないが通常通り使える）。

最初は、「Wi‑Fi 7対応 = 必ずMLO対応」ではない点だけ理解できればかなり役立ちます。

MLOでは、「全部同じ設定へそろえる」ことがかなり重要です。

## MLOのための設定方針

MLOを使うには、次の条件をそろえる必要があります:

- 全バンド（2.4GHz/5GHz/6GHz）に **同じSSID** を設定
- 全バンドに **同じパスワード** を設定
- 暗号化方式を **WPA3-SAE** に統一（WPA2では動作しない場合あり）

最初は、「SSID」「暗号化」「パスワード」の3つをそろえることを優先するだけでも十分です。

> **重要**: 電波出力・チャンネル・国コードなどの無線設定は、最初は変更しないほうが安全です。
>
> まずは標準状態でMLO動作確認するほうが切り分けしやすくなります。

MLO設定は、最初はLuCIからそろえるほうが整理しやすくなります。

変更前に現在の無線設定を保存しておきます。

```sh
BACKUP_DIR="/root/mlo-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/wireless "$BACKUP_DIR/wireless"
cp /etc/config/network "$BACKUP_DIR/network"
uci show wireless > "$BACKUP_DIR/wireless.uci.txt"
wifi status > "$BACKUP_DIR/wifi-status-before.json" 2>/dev/null || true
iw dev > "$BACKUP_DIR/iw-dev-before.txt" 2>/dev/null || true
```

## LuCIでのMLO設定手順

### 2.4GHzバンドの設定

1. LuCI: **Network** → **Wireless** を開く
2. `radio0`（2.4GHz）の行にある既存SSIDの **Edit** をクリック
3. **Interface Configuration** タブで設定:

| 設定項目 | 値 |
|---|---|
| ESSID（SSID） | HomeWifi（全バンド共通） |
| Encryption | WPA3-SAE |
| Key | 共通パスワード（12文字以上推奨） |
| Network | lan |

4. **Save** → 次のバンドへ

最初は、「2.4GHzだけ別SSIDになっていないか」を確認するだけでもかなり役立ちます。

### 5GHzバンドの設定

1. `radio1`（5GHz）の行にある既存SSIDの **Edit** をクリック
2. 同様に SSID: `HomeWifi`、Encryption: WPA3-SAE、同じKey を入力
3. Network: `lan`
4. **Save**

5GHzは、実際にはMLOの主力帯域になるケースがかなり多くあります。

### 6GHzバンドの設定

1. `radio2`（6GHz）の行にある既存SSIDの **Edit** をクリック
2. 同様に SSID: `HomeWifi`、Encryption: WPA3-SAE、同じKey を入力
3. Network: `lan`
4. **Save** → **Apply**

6GHzは高速ですが、距離や障害物には比較的弱めです。

SSID・暗号化・キーをそろえたあとでMLOを有効化するほうが整理しやすくなります。

### MLOを有効化する

1. 有線LANでPCをLN6001-JPに接続する
2. LuCIにログインする
3. **Network** → **MLO** を開く
4. **AP MLO Enable** にチェックを入れる
5. **Save & Apply** をクリック

最初は、有線LAN接続したPCから設定するほうが安全です。

CLIでは、最初は「設定変更」より「現在状態確認」を優先するほうが安全です。

```sh
# 現在のSSID設定を確認
uci show wireless | grep ssid

# 暗号化方式を確認
uci show wireless | grep encryption

# 各SSIDがどのnetworkに紐づいているか確認
uci show wireless | grep network

# 無線デバイスとSSIDの状態を確認
wifi status
```

`wifi status` は、まず最初に確認したいコマンドです。

最初は「SSIDがそろっているか」を見るだけでもかなり役立ちます。

`wireless.default_radio0` のようなセクション名は環境ごとに変わることがあります。

コピペで `uci set` を実行せず、LuCIのWireless画面で各バンドを確認してから変更するほうが安全です。

MLOは、「設定しただけ」で終わりではなく、実際に端末側が使えているか確認することも重要です。

## MLO動作の確認方法

### 端末側での確認（iPhone/Android）

Wi-Fi 7対応端末でSSIDに接続後、Wi-Fi接続の詳細情報でリンク速度や使用バンドを確認します。

- **iPhone（Wi-Fi 7対応機種）**: 設定 → Wi-Fi → 接続済みSSID名の「i」ボタン → リンク速度・使用バンドを確認
- **Mac**: Option + Wi‑Fiアイコン → 接続情報でPHYモードが「802.11be」であることを確認

### ルーター側での確認

LuCIでは **Network** → **Wireless** の **Associated Stations** で、端末が2つまたは3つのradioに接続しているかを確認します。複数radioに表示されても、実際にTX/RXレートが出るactive radioは端末や電波状況により変わります。

```sh
# 各バンドの接続状態を確認
wifi status

# MLO設定らしき項目が反映されているか確認
uci show wireless | grep -Ei 'mlo|ssid|encryption|key'

# 接続中の端末と使用バンドを確認
# 実際のwlan名は iw dev で確認
iw dev
iw dev <wlan名> station dump

# ログ確認
logread | grep -Ei 'wifi|hostapd|mlo' | tail -n 100
```

最初は、「複数radioへ接続が見えるか」を確認できれば十分役立ちます。

MLOは比較的新しい機能のため、「Wi‑Fi 7対応」と書かれていても挙動差が出ることがあります。

うまくいかない時は、まずMLOを疑うより、通常Wi-Fiとして安定しているかを確認します。

```sh
# SSID、暗号化、keyが全バンドで一致しているか確認
uci show wireless | grep -E 'ssid|encryption|key'

# 一時的にMLOを無効化する場合はLuCIの Network > MLO から戻す
# その後、Wi-Fiを再読み込み
wifi reload
logread | grep -Ei 'wifi|hostapd' | tail -n 80
```

## よくある問題

| 症状 | 確認事項 |
|---|---|
| Wi-Fi 7端末だが速度が改善しない | 端末がMLO対応しているか確認（Wi-Fi 7対応でもMLO非対応モデルあり） |
| 6GHz帯に接続できない | 端末が6GHz対応しているか確認（Wi-Fi 6Eまたは7が必要） |
| WPA3-SAEに設定したら古い端末が繋がらない | WPA2-PSK / WPA3-SAE mixedモードを使うか、古い端末用に別SSIDを作る |
| SSIDが3つ見える | 全バンドで同じSSIDを設定できていない → 手順を再確認 |

最初は、「MLOが動かない」より、「通常Wi‑Fiとして安定動作するか」を優先確認するほうが整理しやすくなります。

MLOでは、「全部の帯域が常に最大速度」になるわけではありません。

## 帯域幅と対応チャンネルの目安

| バンド | 最大チャンネル幅 | MLO時の役割 |
|---|---|---|
| 2.4GHz | 40MHz | 障害物を超える安定リンク（補助） |
| 5GHz | 160MHz | バランスリンク（主力） |
| 6GHz | 320MHz | 超高速・低遅延リンク（高速） |

電波の強さや端末の位置によって、MLO時の実際のリンク組み合わせは自動的に最適化されます。

最初は、「5GHz主体 + 必要に応じて6GHz追加」くらいで考えるほうが現実的です。

## まとめ

MLOは、Wi‑Fi 7環境で複数バンドをまとめて使える便利な機能です。

ただし、必須ではなく、「通常SSID運用で十分安定している」なら無理に使わなくても問題ありません。

まずはSSID・WPA3-SAE・パスワードを全帯域でそろえ、LuCIから安全に試すほうが整理しやすくなります。

最初は、「Wi‑Fi 7端末が普通に安定接続できる」だけでも十分価値があります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-03.png)

## よくある質問

### LN6001-JPのMLOは有効にすると必ず速くなる？

必ずしもそうではありません。

ルーターと端末の両方がWi‑Fi 7とMLOへ対応していて、環境条件も合って初めて効果を感じやすくなります。

### LN6001-JPでMLOを使うにはSSIDをどう設定すればいい？

2.4GHz、5GHz、6GHzの3帯域すべてで、同じSSID、同じパスワード、同じWPA3-SAE設定へそろえる必要があります。

### LN6001-JPでMLOを有効にすると古い端末はつながらなくなる？

古い端末はMLOにはなりませんが、通常Wi‑Fi接続として使えることが多いです。

ただし、WPA3-SAEとの相性は確認が必要です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT MLO setup: https://support.linksys.com/kb/article/8652-en/?section_id=175

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
