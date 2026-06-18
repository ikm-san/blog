<!-- mirror-source: articles/012-dhcp-reservations.md -->

> Original note.com article: [固定IPとDHCP予約: プリンター、NAS、監視カメラを整理する【OpenWrt集中連載012】](https://note.com/ikmsan/n/ndd5ae853f033)

# 固定IPとDHCP予約: プリンター、NAS、監視カメラを整理する【OpenWrt集中連載012】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

プリンター、NAS、防犯カメラ、業務端末は、毎回IPアドレスが変わると管理しにくくなります。

ただし、端末側へ固定IPを手入力するより、ルーター側のDHCP予約（Static Lease）で管理したほうが、家庭や小さなオフィスでは運用しやすいことが多いです。

LN6001-JPはOpenWrtベースなので、LuCIからDHCP予約をまとめて管理できます。

最初から全部の端末へ固定IPを付ける必要はありません。まずは「IPが変わると困る機器」だけ整理していけば十分です。

この記事では、DHCP予約の考え方、LuCIでの設定手順、IPアドレス体系の整理方法を、家庭・小規模オフィス向けに順番に整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-01.png)

## この記事でわかること

- DHCP予約が必要な機器と不要な機器の考え方
- LN6001-JPで固定IPを整理する流れ
- DHCP配布範囲と予約IPを安全に分ける方法
- 家庭・小規模オフィス向けのIPアドレス整理例

## こんな人に向いています

- NAS、プリンター、カメラのIPが変わって困ったことがある
- 家庭や小さなオフィスで機器管理を整理したい
- 端末側の固定IPより、ルーター側でまとめて管理したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-02.png)

## まずはここまでで十分

DHCP予約も最初から全部の機器へ入れる必要はありません。むしろ、重要機器だけ整理したほうが管理しやすくなります。

まずは次の3種類だけ押さえれば、かなり整理しやすくなります。

1. NASやサーバー
2. プリンター
3. 防犯カメラや業務端末

「IPが変わると困る機器」から順に予約していくと、無理なく整理できます。

スマートフォンや一時端末まで最初から固定IP化する必要はありません。

## 設定前に現在のDHCP状態を保存する

DHCP予約は `dhcp` 設定を変更します。GuestやCameraなど複数ネットワークを作っている場合は `network` 側の範囲も合わせて確認します。

```sh
BACKUP_DIR="/root/dhcp-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in dhcp network; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
ip addr show > "$BACKUP_DIR/ip-addr.txt"

ls -l "$BACKUP_DIR"
```

DHCPリースには端末名やMACアドレスが含まれるため、そのまま公開場所へ貼らないようにします。

## 用語ミニ解説

- IPアドレス: ネットワーク上の住所です。例: `192.168.1.20`
- DHCP: ルーターが端末へIPアドレスを自動で配る仕組みです。
- DHCP予約（Static Lease）: 特定の端末へ、いつも同じIPアドレスを渡す設定です。
- MACアドレス: 端末ごとに付いている識別番号（例: `aa:bb:cc:dd:ee:ff`）。DHCP予約に使います。

## DHCP予約が必要な機器の種類

| 機器 | DHCP予約が必要な理由 |
|---|---|
| NAS | IPが変わるとバックアップ設定・スマートフォンアプリが壊れる |
| プリンター | IPが変わるとPCからの印刷設定が無効になる |
| 防犯カメラ | 録画ソフトや監視アプリの接続設定が壊れる |
| POS端末 | 業務システムが特定IPで動く場合がある |
| 管理PC | VPN設定やリモートデスクトップのアクセス先が変わる |

逆に、スマートフォンや一時利用PCのように「IPが変わっても困らない端末」は、通常DHCPのままで十分なことが多いです。

家庭や小さなオフィスでは、「全部固定IP」より、「重要機器だけDHCP予約」のほうが管理しやすいです。

端末交換時も、ルーター側の設定変更だけで済みやすくなります。

## DHCP予約の設定手順（LuCI）

### ステップ1: 端末のMACアドレスを確認する

**接続中の端末から確認する方法:**

1. **Network** → **DHCP and DNS** → **Active DHCP Leases** タブを開く
2. 接続中の端末の一覧が表示される
3. 予約したい端末のMACアドレス（例: `a4:c3:f0:01:23:45`）をメモする

まずは、現在つながっている端末を一覧で把握するところから始めます。

LuCIのActive DHCP Leases画面は、「どの端末がどのIPを使っているか」を確認する時にもかなり便利です。

**CLIで確認する場合:**

```sh
# 現在のDHCPリース一覧
cat /tmp/dhcp.leases
# 出力例: 1715000000 a4:c3:f0:01:23:45 192.168.1.105 nas-home *
# 形式:    有効期限  MACアドレス          IPアドレス    ホスト名
```

`cat /tmp/dhcp.leases` は、現在どの端末へIPが配られているかを確認する時に便利です。

CLIは、最初は設定変更より「今どう割り当てられているか」を確認する用途で使うくらいで十分です。

**端末側で確認する方法:**
- Windows: `ipconfig /all` → 「物理アドレス」
- Mac: 設定 → ネットワーク → 接続中のWi-Fi/有線 → 詳細 → ハードウェア
- iPhone/Android: Wi-Fi設定 → 接続中のSSIDの詳細画面

### ステップ2: DHCP予約を追加する

1. **Network** → **DHCP and DNS** を開く
2. **Static Leases** タブを選択
3. **Add** をクリック
4. 以下を入力:

| フィールド | 入力値 | 例 |
|---|---|---|
| Hostname | 端末の名前 | `nas-home` |
| MAC address | 端末のMACアドレス | `a4:c3:f0:01:23:45` |
| IP address | 割り当てるIPアドレス | `192.168.1.10` |

5. **Save** をクリック
6. 他の機器も同様に追加する
7. 全部追加したら **Apply** をクリック

最初は「nas-home」「printer-living」のように、役割が分かる名前を付けておくと、あとから管理しやすくなります。

### ステップ3: 端末を再接続して確認する

予約設定後は、端末のWi‑Fiをオフにしてからオンへ戻し、再接続します。

IPアドレスが予約した値になっていることを確認します。

端末のIPアドレス確認方法:
- Windows: `ipconfig` コマンドで確認
- Mac: 設定 → ネットワーク → 使用中の接続 → IPアドレス
- スマートフォン: Wi-Fi設定の接続済みSSIDの詳細

ここで「再接続後も同じIPを取る」ことが確認できれば、DHCP予約は正常に動いています。

## IPアドレス体系の設計

場当たり的にIPを割り当てると、後からかなり混乱しやすくなります。

用途ごとに範囲を決めておくと、あとから見返した時にも整理しやすくなります。

### 家庭向けの設計例

| 範囲 | 用途 |
|---|---|
| 192.168.1.1 | ルーター（LN6001-JP） |
| 192.168.1.10–19 | NAS・サーバー |
| 192.168.1.20–29 | プリンター |
| 192.168.1.30–49 | IoTカメラ・家電 |
| 192.168.1.100–199 | 通常のDHCP端末（スマートフォン・PC） |
| 192.168.1.200–220 | 来客・一時端末 |

最初はここまで細かく分けなくても大丈夫です。

まずは「予約IP帯」と「通常DHCP帯」を分けるだけでも、かなり管理しやすくなります。

### 小さなオフィス向けの設計例

| ネットワーク | 範囲 | 用途 |
|---|---|---|
| Staff（LAN） | 192.168.1.0/24 | 管理PC・業務端末 |
| Guest | 192.168.2.0/24 | 来客Wi-Fi |
| Camera | 192.168.3.0/24 | 防犯カメラ |
| 予約帯 | 192.168.x.10–30 | 固定IP端末 |
| DHCP帯 | 192.168.x.100–200 | 自動割り当て端末 |

GuestやCameraを別ネットワークへ分けている場合は、それぞれのネットワーク内でも「予約帯」と「DHCP帯」を分けておくと整理しやすくなります。

## 機器管理表を作る

設定画面を触る前に、機器一覧表を作っておくと管理がかなり楽になります。

特にNAS、プリンター、カメラが増えてくると、「どのIPがどの機器だったか」を見失いやすくなります。

| 機器名 | 用途 | 接続方法 | MACアドレス | 予約IP | 設置場所 |
|---|---|---|---|---|---|
| nas-home | 写真・バックアップ | 有線 | a4:c3:f0:01:23:45 | 192.168.1.10 | 書斎 |
| printer-living | 印刷 | Wi-Fi 2.4G | b8:27:eb:11:22:33 | 192.168.1.20 | リビング |
| camera-front | 監視 | 有線 | dc:a6:32:44:55:66 | 192.168.1.31 | 玄関 |

この表は、Markdown、Excel、Googleスプレッドシートなど、自分が管理しやすい形式で十分です。

LuCIのDHCP一覧からMACアドレスをコピーすると、比較的楽に整理できます。

## CLIでDHCP予約を管理する

DHCP予約はCLIでも設定できますが、最初はLuCIから設定したほうが状態を理解しやすいです。

CLIを使う場合も、まず現在設定を確認してから変更するほうが安全です。

### 現在の設定を確認する

```sh
echo "### DHCP config"
uci show dhcp

echo "### active leases"
cat /tmp/dhcp.leases

echo "### static lease entries"
uci show dhcp | grep -A 4 'host'
```

`uci show dhcp` と `cat /tmp/dhcp.leases` は、現在のDHCP状態を確認する時にかなり便利です。

### CLIで予約を追加する

```sh
echo "### backup DHCP config"
cp /etc/config/dhcp /etc/config/dhcp.backup.$(date +%Y%m%d-%H%M)

echo "### add static lease"
uci add dhcp host
uci set dhcp.@host[-1].name='nas-home'
uci set dhcp.@host[-1].mac='a4:c3:f0:01:23:45'
uci set dhcp.@host[-1].ip='192.168.1.10'
uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart

echo "### verify static lease"
uci show dhcp | grep -A 4 '@host'
logread | grep -i dnsmasq | tail -n 50
```

CLIで追加した場合も、最後にLuCIのStatic Leases画面で確認しておくと安心です。

### ホスト名でアクセスできるようにする（ローカルDNS）

DHCP予約でホスト名を設定すると、`nas-home.lan` や `nas-home` のような名前でアクセスできるようになります（dnsmasqの機能）:

IPアドレスを覚えなくてもアクセスしやすくなるため、家庭でもかなり便利です。

LuCIの **Network** → **DHCP and DNS** → **Hostnames** タブで手動ホスト名も追加できます。

NASやプリンターが増えてくると、「IPアドレス管理」より「名前管理」のほうが分かりやすくなることがあります。

## DHCP配布範囲の設定確認

DHCP予約と配布範囲が重複しないようにします。

ここがDHCP管理で一番つまずきやすいポイントです。

### LuCIで確認

1. **Network** → **Interfaces** → `lan` の **Edit**
2. **DHCP Server** タブで Start と Limit を確認

例: Start=`100`、Limit=`150` の場合、`192.168.1.100`〜`192.168.1.249` がDHCP配布範囲になります。予約IPはこの範囲**外**（例: `192.168.1.10`〜`192.168.1.50`）に設定します。

最初は「予約は10〜50」「通常DHCPは100以降」のように、大きく離しておくと管理しやすくなります。

### CLIで確認

```sh
echo "### LAN DHCP range"
uci get dhcp.lan.start   # 例: 100
uci get dhcp.lan.limit   # 例: 150

echo "### all DHCP ranges"
uci show dhcp | grep -E 'interface|start|limit|leasetime'
```

## 端末交換時の対応

端末を交換した場合（NAS・カメラ・プリンターの買い替え等）も、DHCP予約なら比較的楽に引き継げます:

1. 古い予約のMACアドレスを新しい端末のMACアドレスに変更する
2. LuCI: **Network** → **DHCP and DNS** → **Static Leases** → 対象を **Edit**
3. MACアドレスを新しい端末の値に変更 → **Save & Apply**

この操作だけで、同じIPアドレスを新しい端末に引き継げます。端末側・接続先システム側の設定変更が不要になります。

端末側へ固定IPを直接設定している場合より、入れ替え時の手戻りを減らしやすいところがDHCP予約のメリットです。

## よくある失敗と対処

DHCP予約で困る時は、「端末が古いIPを持ったまま」か、「予約帯とDHCP帯が重なっている」ケースがかなり多いです。

### 予約後も端末のIPが変わる

**原因:** 端末が古いリースを保持している
**対処:** 端末のWi‑FiをOff/Onするか、`ipconfig /release && ipconfig /renew`（Windows）でDHCPを更新する

### 予約したIPに別の端末がいる

**原因:** DHCP配布範囲と予約IP範囲が重複していて、別端末に同じIPが配布されていた
**対処:** DHCP配布範囲（Start/Limit）と予約IP範囲が重複しないよう整理し直す

### 端末のホスト名で名前解決できない

**確認:** LuCIのDHCP Static Leasesでホスト名が設定されているか確認
**対処:** ホスト名を設定して `/etc/init.d/dnsmasq restart` を実行

## まとめ

DHCP予約は、次の順番で進めると整理しやすいです:

1. **Network** → **DHCP and DNS** → **Static Leases** → **Add**
2. ホスト名・MACアドレス・IPアドレスを入力して **Save & Apply**
3. 端末を再接続して予約IPが割り当てられたことを確認

設定前に機器一覧表を作り、IPアドレス体系を整理してから進めると、後からの管理がかなり楽になります。

特に大事なのは、「DHCP配布範囲」と「予約IP範囲」を重ねないことです。

最初から全部の端末を固定IP化するより、「IPが変わると困る機器」だけ整理していくほうが、家庭や小さなオフィスでは運用しやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-03.png)

## よくある質問

### 固定IPは端末側で設定するよりDHCP予約がいい？

家庭や小さなオフィスなら、まずはルーター側のDHCP予約で管理するほうが分かりやすいです。

端末交換時の手戻りも少なくなり、IP管理をルーター側へまとめやすくなります。

### DHCP予約が必要な機器はどれ？

プリンター、NAS、防犯カメラ、業務PC、VPN先として使う機器など、IPが変わると困る機器に向いています。

### DHCP予約で一番気をつけることは？

DHCP配布範囲と予約IP範囲を重ねないことです。

先にアドレス設計を決めてから登録すると、あとから機器が増えても整理しやすくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt DHCP configuration: https://openwrt.org/docs/guide-user/base-system/dhcp

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
