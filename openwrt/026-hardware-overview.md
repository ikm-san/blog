<!-- mirror-source: articles/026-hardware-overview.md -->

> Original note.com article: [ハードウェア概要: LN6001-JPのポート・LED・ボタンの見方【OpenWrt集中連載026】](https://note.com/ikmsan/n/nf5e2df270ea3)

# ハードウェア概要: LN6001-JPのポート・LED・ボタンの見方【OpenWrt集中連載026】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JP（Linksys Velop WRT Pro 7）は、Wi‑Fi 7（802.11be）対応のOpenWrtベース無線ルーターです。

2.4GHz / 5GHz / 6GHz のトライバンドとMLOに対応し、LuCIやSSHを使った高度な設定もできます。

ただし、最初から細かな仕様を全部覚えなくても大丈夫です。

まずは「WANポート」「LANポート」「LED状態」「リセットボタン」の4つだけ分かれば、初期設定やトラブル切り分けはかなり進めやすくなります。

この記事では、LN6001-JP本体まわりの見方、各ポートやLEDの意味、最初に確認したいポイントを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-01.png)

## この記事でわかること

- LN6001-JPのWAN、LAN、LED、リセットボタンの役割
- 初期設定前に確認したい本体まわり
- LED表示からどこまで切り分けできるか
- 2.5GbEやWi‑Fi 7の基本的な見方

最初は、「全部のスペックを理解する」より、「どこへ何をつなぐか」を把握するほうがかなり重要です。

## 主要スペック

| 項目 | 仕様 |
|---|---|
| Wi-Fi規格 | Wi-Fi 7（IEEE 802.11be） |
| 対応バンド | 2.4GHz / 5GHz / 6GHz（トライバンド） |
| WANポート | 2.5GbE × 1 |
| LANポート | 1GbE × 4 |
| OS/ファームウェア | OpenWrtベース（Linksys独自ファームウェア） |
| 管理画面 | LuCI（https://192.168.1.1） |
| 日本向けモデル | LN6001-JP（技適取得済み） |

最初は、「WANは回線側」「LANは端末側」という切り分けだけでもかなり役立ちます。

> 詳細スペックはLinksys公式製品ページおよびサポートページ（https://support.linksys.com/kb/article/6274-jp/）をご確認ください。
>
> 実際の通信速度は、接続端末・距離・障害物・回線側構成でも変わります。

SSHで実機側の基本情報を見るなら、まずは読み取り専用の確認だけで十分です。

```sh
echo "### system board"
ubus call system board

echo "### cpu"
cat /proc/cpuinfo | grep -E 'system type|machine|model|processor|cpu MHz|BogoMIPS' | head -n 30

echo "### memory and storage"
free
df -h

echo "### uptime"
uptime
```

この出力で、ファームウェアのベース情報、CPU、メモリ、ストレージ、稼働時間を確認できます。スペック表と完全に同じ言葉で表示されるとは限りませんが、トラブル相談時の基準情報として役に立ちます。

![本体まわりの確認ポイント](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/hardware-map.png)

---

ルーター初期設定時は、「WANとLANを逆に挿していた」だけのケースもかなり多くあります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-02.png)

## ポートの配置と役割

### 背面ポート（後ろから見て）

| ポート | 種別 | 役割 |
|---|---|---|
| WAN | 2.5GbE（黄/青色ポート） | ONU・HGW・モデムと接続する入口 |
| LAN 1〜4 | 1GbE × 4 | PCや有線デバイスを接続するLANポート |
| DC IN | 電源 | 付属ACアダプターを接続 |

### ポート接続のポイント

- **WANポートとLANポートを間違えない**: WANポートはONU/HGWへ接続。LANポートはPCやスイッチへ接続
- **2.5GbEのWANポート**: ONU/HGWが2.5GbE対応の場合は2.5Gbpsで接続可能。未対応の場合は自動で1Gbpsにネゴシエート
- **LANポート間の通信**: LAN 1〜4のポート間はブリッジ接続でルーターを経由せずに通信

最初は「ONU/HGW → WAN」「PC → LAN」の向きだけ確認できれば十分です。

SSHでポートやインターフェース名を確認する場合は、次のように読み取ります。

```sh
echo "### interfaces"
ip link show

echo "### IPv4/IPv6 addresses"
ip addr show

echo "### bridge and VLAN related config"
uci show network | grep -E 'device|bridge|ports|vlan|ifname|proto'
```

LN6001-JPの内部インターフェース名は、画面上の「WAN」「LAN 1」などのラベルと完全に同じ名前とは限りません。まずはLuCIの **Network** → **Interfaces** と合わせて読み、いきなり設定を書き換えないようにします。

リンク速度を見たい場合は、環境によって取れる項目が変わります。取れるものだけ表示するには次のようにします。

```sh
for dev in /sys/class/net/*; do
  name="$(basename "$dev")"
  [ -r "$dev/speed" ] || continue
  printf "%s speed: " "$name"
  cat "$dev/speed" 2>/dev/null || true
done
```

ここで `2500` や `1000` のような値が見える場合があります。ただし、すべての仮想インターフェースや無線インターフェースで表示されるわけではありません。

---

LEDを見るだけでも、「起動中なのか」「回線問題なのか」をかなり切り分けしやすくなります。

## LEDの見方

LN6001-JPは、本体上部に1つのステータスLEDがあります。

| LED状態 | 意味 |
|---|---|
| 白点灯（常時） | 正常動作中・インターネット接続中 |
| 赤点灯（常時） | インターネット接続なし（WAN未接続または設定エラー） |
| 青点滅 | 起動中 |
| 消灯 | 電源オフ / LED無効化設定 |

最初は、「白=正常」「赤=回線問題系」「青点滅=起動中」くらいで覚えるだけでも十分役立ちます。

LEDの状態と対処:

- **赤点灯が続く**: WANケーブル接続、ONU/HGW電源、回線種別設定を確認する（014の記事参照）
- **起動後も青点滅が続く**: 起動に時間がかかることがある（最大3分程度待つ）。それ以上続く場合はリセットや再起動を検討する

---

リセットボタンは便利ですが、「最後の手段」として考えるほうが安全です。

## リセットボタン

本体底面（または裏面）に**リセットボタン**があります。

| 操作 | 方法 | 結果 |
|---|---|---|
| ファクトリーリセット | 電源を入れた状態でリセットボタンを約10秒間押し続ける | 全設定を初期化 |

リセット開始後はLEDが赤点灯から青点滅へ変わり、再起動後にインターネット接続を検知すると白点灯になります。

最初は、LuCIやSSHで戻せないかを先に確認するほうが整理しやすくなります。

---

Wi‑Fi 7や6GHzは高速ですが、「距離が伸びれば常に最速」というわけではありません。

## Wi-Fiアンテナ・電波の仕様

| バンド | 周波数帯 | 最大規格速度の目安 | 特徴 |
|---|---|---|---|
| 2.4GHz（radio0） | 2400〜2484MHz | 〜591Mbps | 障害物に強い・遠くまで届く |
| 5GHz（radio1） | 5150〜5850MHz | 〜2883Mbps | 速度と距離のバランスが良い |
| 6GHz（radio2） | 5925〜7125MHz | 〜5765Mbps | 最高速・干渉少ない・短距離向き |

最初は、「2.4GHzは遠距離向き」「5GHzはバランス型」「6GHzは近距離高速」くらいで整理すると分かりやすくなります。

---

OpenWrt系では、「今どのバージョンを使っているか」を確認できるだけでもかなり重要です。

## ファームウェアバージョンの確認

LuCIのホーム画面に表示されています。

SSHでは次のように確認できます。

```sh
echo "### firmware / release"
cat /etc/openwrt_release

echo "### system board"
ubus call system board

echo "### installed base packages"
opkg list-installed | grep -E 'base-files|busybox|dnsmasq|firewall|dropbear|uhttpd'
```

記事やサポートへ相談する時は、機種名だけでなく、ファームウェアのバージョン、稼働時間、WAN構成も一緒に控えておくと話が早くなります。

```sh
echo "### quick support bundle"
date
ubus call system board
uptime
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
```

---

## まとめ

最初に覚えておきたいのは、「WANポートは回線機器側」「LANポートはPCやスイッチ側」という切り分けと、LEDの白・赤・青点滅で状態を見ることです。

細かなスペックを全部暗記しなくても、この2点が分かっていれば初期設定や切り分けはかなり進めやすくなります。

最初は「どこへ何をつなぐか」「LEDが何色か」を見られるだけでも十分です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/026/diagram-03.png)

## よくある質問

### LN6001-JPのWANポートとLANポートはどこを見分ければいい？

WANは回線機器につなぐ入口、LANはPCやスイッチにつなぐ側です。

最初にここを間違えないだけでも切り分けがかなり楽になります。

### LN6001-JPのLEDが赤い時は故障？

すぐに故障とは限りません。

まずはWANケーブル、ONU/HGW電源、回線設定を順に確認すると切り分けしやすくなります。

### LN6001-JPのリセットボタンはいつ使う？

LuCIやSSHで戻せない時の最後の手段として考えるのが安全です。

押す前にバックアップや復旧手順を確認したほうが安心です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 light behavior: https://support.linksys.com/kb/article/216-en/?section_id=175
- Linksys MBE70 reset: https://support.linksys.com/kb/article/220-en/?section_id=175

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
