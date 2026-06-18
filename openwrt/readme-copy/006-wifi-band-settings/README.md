<!-- mirror-source: articles/006-wifi-band-settings.md -->

> Original note.com article: [LN6001-JPのWi‑Fi設定: 2.4GHz/5GHz/6GHzをどう使い分けるか【OpenWrt集中連載006】](https://note.com/ikmsan/n/ndd6f12876bc8)

# LN6001-JPのWi‑Fi設定: 2.4GHz/5GHz/6GHzをどう使い分けるか【OpenWrt集中連載006】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPは、2.4GHz、5GHz、6GHzを使えるWi‑Fi 7ルーターです。

ただ、Wi‑Fi 7という言葉だけを見ると、「とにかく全部を高速化すればいい」と考えがちですが、実際には“どの端末をどの帯域へつなぐか” を整理するほうが、家庭や小さなオフィスではかなり重要です。

目安としては、互換性重視の2.4GHz、日常利用の主力になる5GHz、Wi‑Fi 6E/7対応端末向けの6GHzという分け方が分かりやすいです。

また、MLO（マルチリンクオペレーション）のようなWi‑Fi 7機能もありますが、最初から無理に有効化しなくても大丈夫です。まずは安定してつながる構成を作り、そのあと必要に応じて調整していくほうが扱いやすくなります。

この記事では、LuCIでの具体的な設定手順と、帯域ごとの使い分けを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-01.png)

## この記事でわかること

- 2.4GHz、5GHz、6GHzをどう使い分けると分かりやすいか
- LN6001-JPでSSIDをどう分けると運用しやすいか
- MLOを使うべきかどうかの考え方
- 家庭・小規模オフィス向けの現実的なSSID構成例

## こんな人に向いています

- 2.4GHz、5GHz、6GHzのどれを使えばいいか迷っている
- 家庭や小さなオフィスでSSIDの分け方を整理したい
- MLOを使うべきか、通常のSSID構成で十分か判断したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-02.png)

## まずはここまでで十分

最初から全帯域を複雑に使い分けなくても大丈夫です。むしろ、最初はシンプルな構成のほうが管理しやすくなります。

まずは次の3つで十分です。

1. 5GHzを主力SSIDにする
2. 2.4GHzはIoTや古い端末向けに分ける
3. 6GHzやMLOは対応端末がそろってから検討する

最初の段階では、「最大速度」より「どの端末をどこへつなぐか」が整理できているほうが運用しやすいです。

## 用語ミニ解説

- SSID: スマートフォンやPCに表示されるWi-Fi名です。
- 2.4GHz: 古い端末やIoT機器で使いやすく、届きやすい一方で混雑しやすい帯域です。
- 5GHz: PC、スマートフォン、テレビなど日常利用の主力になりやすい帯域です。
- 6GHz: Wi-Fi 6E/7対応端末で使える新しい帯域です。対応端末が必要です。
- MLO: Wi-Fi 7の機能のひとつ。複数帯域を同時使用します。端末側の対応も必要です。

## 帯域ごとの特徴と向く端末

![表 01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-01.png)

Wi‑Fi設定で迷った時は、「どの端末をどの帯域へ逃がしたいか」で考えると整理しやすくなります。

例えば、IoT機器を2.4GHzへまとめておくと、スマートフォンやPCを5GHz側へ寄せやすくなります。

Linksys公式のWi‑Fi設定ページでは、LuCIの `Network > Wireless` から各無線を編集する流れが案内されています。`wifi0` が2.4GHz、`wifi1` が5GHz、`wifi2` が6GHzとして説明されています。

最初は全部を細かく調整する必要はありません。まずはSSID名とパスワードを整理し、どの帯域をどの用途に使うかだけ決めれば十分です。

設定を変える前に、現在の無線状態を読み取りだけで控えておくと安心です。

```sh
echo "### wireless config summary"
uci show wireless | grep -E 'wifi-device|wifi-iface|ssid|encryption|network|disabled'

echo "### radio status"
wifi status

echo "### wireless interfaces"
iw dev
```

この段階では、SSID名、暗号化方式、どのnetworkに紐付いているか、有効/無効状態を見るだけにします。無線の国設定、送信出力、DFS関連の値はこの連載では変更しません。

## LuCIでのSSID設定手順

SSID設定は、日常運用で最も触ることが多い設定のひとつです。

最初は「家族用」「IoT用」「来客用」くらいに整理できれば十分実用になります。

### 既存SSIDを編集する

1. **Network** → **Wireless** を開く
2. 変更したい無線（2.4G / 5G / 6G）の **Edit** をクリック
3. **Interface Configuration** タブ:

![表 02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-02.png)

4. **Wireless Security** タブ:

![表 03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-03.png)

5. **Save** をクリック
6. 各帯域分（2.4 / 5 / 6）繰り返す
7. **Apply** ボタンで設定を反映

### 新しいSSIDを追加する（Guest用など）

1. **Network** → **Wireless** を開く
2. 追加したい帯域（例: 5GHz `wifi1`）の **Add** をクリック
3. 上記と同様に ESSID・セキュリティを設定
4. **Network** タブで、接続するネットワーク（Guestネットワークが必要な場合は先にNetworkで作成）を選択
5. **Save** → **Apply**

Guest用SSIDを作る場合は、後続の記事で紹介する firewall zone や VLAN と組み合わせると、家庭用ネットワークと分離しやすくなります。

## 無線の有効・無効の切り替え

### LuCIで切り替える

1. **Network** → **Wireless** を開く
2. 対象のSSIDの **Disable**（無効化）または **Enable**（有効化）をクリック

帯域ごとに独立して有効/無効を切り替えられます。

使わない帯域を無効化すると、電波干渉の軽減や電力削減につながることがあります。

### CLIで状態を確認

```sh
echo "### wireless config"
uci show wireless

echo "### SSID / disabled flags"
uci show wireless | grep -E 'ssid|disabled'

echo "### radio status"
wifi status
```

帯域の有効/無効切り替えはLuCIで行うのが安全です。

`wireless.default_radio2` のようなセクション名は環境で変わることがあります。CLIは、最初は変更より状態確認に使うくらいで十分です。

## 帯域別の推奨設定と使い方

### 2.4GHz: IoTと古い端末向け

**推奨設定:**
- SSID: `MyHome_IoT`（用途がわかる名前）
- Encryption: `WPA2-PSK`（古い端末との互換性を重視）
- チャネル: 自動（Auto）または1/6/11から選択

**向く端末:**
- スマートスピーカー（Amazon Echo, Google Home等）
- スマートテレビ（Wi-Fi 5以下のモデル）
- 古いゲーム機（Nintendo DS, 初代Fire TV等）
- IoTセンサー、スマート照明
- 遠い部屋の端末（壁や床を通りやすい）

**注意点:**
- 電子レンジ・Bluetooth・近隣Wi-Fiの影響を受けやすい
- 複数のIoT機器をつなぐと混雑しやすい
- 速度を重視する用途は5GHz/6GHzへ移す

IoT機器を2.4GHz側へまとめておくと、「なぜかIoTだけ切れる」「5GHzへつながらない」といった切り分けがしやすくなります。

### 5GHz: 日常利用の主力

**推奨設定:**
- SSID: `MyHome` または `MyHome_5G`
- Encryption: `WPA3-SAE`（新しい端末）または `WPA2-PSK`（混在環境）
- チャネル幅: `80MHz`（通常）または `160MHz`（高速な場合）

**向く端末:**
- ノートPC・デスクトップPC
- スマートフォン（iPhone/Android）
- タブレット
- ゲーム機（PS5, Nintendo Switch等）
- スマートテレビ（最新モデル）

**5GHzが2.4GHzより良い点:**
- 混雑が少なく安定しやすい
- 速度が出やすい
- 動画視聴・オンライン会議に向く

家庭や小さなオフィスでは、まず5GHzを中心に構成しておくとバランスを取りやすいです。

### 6GHz: Wi-Fi 7対応端末向け

**推奨設定:**
- SSID: `MyHome_6G`（MLO設計にする場合は他帯域と同じSSID名）
- Encryption: `WPA3-SAE`（必須）
- チャネル幅: `80MHz` または `160MHz`

**向く端末:**
- Wi-Fi 7対応スマートフォン（2023年以降のフラグシップモデル）
- Wi-Fi 7対応ノートPC（2024年以降の一部モデル）

**6GHzの注意点:**
- 対応端末がない場合は使えない
- 壁や床を通りにくく、近距離で効果的
- すべての端末を6GHzへまとめることはできない

「6GHzを有効化すれば全部速くなる」というより、対応端末だけを混雑の少ない帯域へ逃がせるイメージに近いです。

## MLO（マルチリンクオペレーション）の設定

MLOはWi‑Fi 7の機能で、複数帯域を同時使用することで速度・安定性を向上させます。

ただし、現時点では「まず最初に有効化すべき必須機能」というより、対応端末がそろってきた時に検討する機能、と考えるほうが現実的です。

**MLOを使う場合の前提:**
- 端末側もWi-Fi 7対応が必要
- 2.4GHz / 5GHz / 6GHzの3つのSSID名を完全に同じにする
- 暗号化はすべて `WPA3-SAE`
- セキュリティキーも同一にする

**LuCIでのMLO設定:**
1. 2.4GHz / 5GHz / 6GHz のSSIDをすべて同じ名前（例: `MyHome`）に設定
2. すべての帯域で Encryption を `WPA3-SAE` に統一
3. すべての帯域で同じパスワードを設定
4. Apply で反映
5. **Network** → **MLO** を開き、**AP MLO Enable** にチェックを入れて **Save & Apply**

Linksys公式のMLO設定手順では、有線接続したPCから作業し、3つのSSID名・WPA3-SAE・セキュリティキーをそろえたうえで **Network** → **MLO** から有効化する流れが案内されています。端末側がWi-Fi 7対応であれば、自動的に最適な帯域を組み合わせて使用します。

MLOは機能としてはかなり面白いのですが、現時点では「まず使うべき必須機能」というほどではありません。

対応端末がそろっていないと効果を実感しにくく、日常利用では5GHzや6GHzを素直に使い分けるだけでも十分なことが多いです。

LN6001-JPを2台使って無線バックホールまで含めて組むような場面なら検討の余地がありますが、興味がなければ無理に有効化しなくても大丈夫です。

MLOを有効化する前後も、CLIではまず読み取りに留めます。

```sh
echo "### MLO prerequisite check from wireless config"
uci show wireless | grep -E 'ssid|encryption|disabled'

echo "### associated stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done
```

同じ端末が複数radioに見える場合がありますが、Linksys公式の説明どおり、実際にTX/RXレートが出るactive radioは端末や電波状況で変わります。MLO確認はLuCIの **Network** → **Wireless** → **Associated Stations** と合わせて見るのが分かりやすいです。

## 推奨のSSID構成例

### 家庭向け（シンプル構成）

![表 04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-04.png)

### Wi-Fi 7 MLO構成（対応端末あり）

![表 05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-05.png)

### 小さなオフィス向け

![表 06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-06.png)

## CLIで状態を確認する

```sh
echo "### radio status"
wifi status

echo "### wireless config"
uci show wireless

echo "### wireless interfaces"
iw dev

echo "### connected stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done

echo "### recent wireless logs"
logread | grep -Ei 'wireless|wifi|wlan|hostapd' | tail -n 80
```

`wifi status` は、どの帯域が有効か、SSIDがどう設定されているかを見る時に便利です。

まずは設定変更より、「今どの帯域が動いているか」を確認する用途で使うと分かりやすいです。

## 設定変更前のバックアップ

```sh
BACKUP_DIR="/root/wireless-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/wireless "$BACKUP_DIR/wireless"
uci show wireless > "$BACKUP_DIR/wireless.uci.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
iw dev > "$BACKUP_DIR/iw-dev.txt"

ls -l "$BACKUP_DIR"
```

このバックアップにはWi-Fiパスワードが含まれるため、そのままチャットや公開記事に貼り付けないようにします。

変更後に問題が出た場合の復元:

```sh
cp /root/wireless-before-change-YYYYMMDD-HHMM/wireless /etc/config/wireless
wifi restart
```

Wi‑Fi設定は、SSID名や暗号化方式を少し変えただけでも端末が再接続できなくなることがあります。変更前バックアップを残しておくと安心です。

## 帯域の選び方で迷ったら

2.4GHz・5GHz・6GHzは、数字が大きいほど常によいわけではありません。

それぞれ「届きやすさ」「安定性」「速度」「対応端末」が違います。

- IoT機器や古い端末 → **2.4GHz**（互換性と届きやすさを優先）
- PC・スマートフォン・テレビ → **5GHz**（速度とバランス）
- Wi-Fi 7対応の最新端末 → **6GHz or MLO**（最高速度）

最初は家族用・来客用・IoT用の3つのSSIDに分けるだけで整理しやすくなります。複雑にしすぎず、まずシンプルな構成から始めてください。

設定後は、次の3つが確認できれば十分です。最初から完璧なWi‑Fi設計を目指さなくても大丈夫です。

- 主力の端末が想定どおり 5GHz か 6GHz へつながっている
- IoT 機器や古い端末が 2.4GHz で安定している
- 6GHz や MLO を使わなくても日常利用で困っていない

## まとめ

LuCIでのSSID設定は **Network** → **Wireless** → **Edit** から行います。

帯域ごとに適切な端末を割り振ることで、家庭・店舗全体のWi‑Fi体験が改善します。

Wi‑Fi 7のMLOを使いたい場合は、3帯域すべてのSSID名・WPA3-SAE・パスワードを統一し、**Network** → **MLO** で **AP MLO Enable** を有効にします。

通常のSSID設計とMLO設計は方向性が違うため、まずはシンプルなSSID構成から始め、必要に応じてMLOを検討するくらいが扱いやすいです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-03.png)

## よくある質問

### LN6001-JPでは2.4GHz、5GHz、6GHzを全部使い分けるべき？

最初から細かく分ける必要はありません。まずは5GHzを主力にし、IoT用に2.4GHz、対応端末があるなら6GHzを足すくらいで十分です。

シンプルに始めたほうが、あとからトラブル切り分けもしやすくなります。

### LN6001-JPの6GHzはどんな端末なら使える？

Wi-Fi 6E や Wi-Fi 7 に対応した端末が必要です。古いスマートフォンやPCでは見えないことがあります。

### LN6001-JPのMLOは最初から有効にしたほうがいい？

必須ではありません。対応端末が少ないうちは、通常のSSID設計で運用したほうが分かりやすいです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT WiFi settings: https://support.linksys.com/kb/article/221-en/?section_id=175
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
