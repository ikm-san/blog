<!-- mirror-source: articles/004-initial-setup-checklist.md -->

# LN6001-JP初期設定チェックリスト: 買って最初に確認すること【OpenWrt集中連載004】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPはOpenWrtベースのルーターですが、買ったあとにOSインストールやファームウェア書き換えから始める製品ではありません。

工場出荷時点でSSID、Wi‑Fiパスワード、管理用ログイン情報などが用意されています。DHCP自動でつながる回線環境なら、まずは回線機器につないで電源を入れるところから始められます。

普通の市販ルーターに近い感覚で使い始められて、必要になればLuCIやSSHで少し深く確認できる。ここがLN6001-JPの扱いやすいところです。

初期設定で大事なのは、最初から全部を触らないことです。まずは管理画面に入れることを確認し、管理パスワードとSSIDを整え、最後にバックアップを取る。ここまでできれば、最初の準備としては十分です。

この記事では、開封してから管理パスワードを変え、SSIDを確認し、最後にバックアップを取るところまでを順番に見ていきます。

![初期設定の流れ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/setup-flow.png)

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-01.png)

## この記事でわかること

- LN6001-JPの初期設定で最初に確認する順番
- 管理画面へ入る前に控えておきたい情報
- 最初に変更してよいもの、慎重に扱うもの
- 初期設定でつまずきやすいポイントと戻し方

## こんな人に向いています

- LN6001-JPを買った直後で、何から触ればいいか迷っている
- HGWやONU配下で最初の接続が不安
- 初期設定で壊さずに進めたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-02.png)

## まずはここまでで十分

初期設定は、最初から全部を触る必要はありません。むしろ、最初は触る範囲を絞ったほうが安全です。

まずは次の3つで十分です。

1. LuCIに入れることを確認する
2. 管理パスワードとSSIDを変える
3. バックアップを取る

最初の目的は、細かな機能追加ではなく「安全に普通に使える入口を作ること」です。

SSHで確認する場合も、最初は読み取り専用にします。

```sh
echo "### system"
date
ubus call system board
uptime

echo "### storage and memory"
free
df -h

echo "### network status"
ifstatus wan
ifstatus wan6
ip route show
ip -6 route show
```

この時点では設定を変えません。まず「ログインできる」「WAN/WAN6の状態が見える」「あとで比較できる基準が残る」ことを優先します。

## 用語ミニ解説

- SSID: スマートフォンやPCに表示されるWi-Fi名です。
- DHCP: ルーターが端末へIPアドレスを自動で配る仕組みです。
- WAN: インターネット回線側です。LN6001-JPのInternet/WANポートは、ONU・モデム・ホームゲートウェイ側へつなぎます。
- LAN: 家や事務所の内側ネットワークです。PC、スマートフォン、NAS、プリンターなどが入ります。
- LuCI: ブラウザで開くOpenWrt系の管理画面です。初期設定はここから始めます。
- IPoE / PPPoE: どちらもインターネット回線へ接続する方式です。IPoEは日本の光回線でよく使われるIPv6系の方式、PPPoEはプロバイダのID/パスワードで接続する方式です。

## まず用意するもの

初期設定では、次のものを手元に用意します。

- LN6001-JP本体
- 付属電源アダプター
- LANケーブル（付属または手持ち）
- ONU、モデム、ホームゲートウェイなどの回線機器
- PCまたはスマートフォン（設定変更はPCを推奨）
- プロバイダの契約情報（接続方式・IDなど）
- ひかり電話契約の有無がわかる情報
- 固定IP契約の有無がわかる情報

IPoEモジュールの導入やSSHでの確認まで進めるなら、PCがあると作業しやすいです。

LuCI管理画面はスマートフォンのブラウザでも開けますが、ネットワーク設定を変えると途中でWi‑Fiが切れることがあります。できれば有線LANでつないだPCから触るほうが安心です。

## 接続前チェック（通電前に確認する）

本体に電源を入れる前に、次を確認します。

**本体底面ラベルから控えるもの:**
- デフォルトのSSID（2.4GHz/5GHz/6GHz それぞれ）
- デフォルトのWi-Fiパスワード
- 管理画面のデフォルトパスワード（初回ログイン用）

**既存機器の確認:**
- ホームゲートウェイ（HGW）やモデムのLAN側IPアドレス
  - NTT系HGW: `192.168.1.1` が多い
  - その他: 機器背面ラベルを確認

LN6001-JPのデフォルトLAN IPも `192.168.1.1` のため、HGWが同じIPを使っている場合は、後述のLAN IP変更を検討します。

ここを事前に把握しておくと、管理画面に入れない時の切り分けがかなり楽になります。

## ケーブル接続の順番

1. ONU/HGWがある場合: ONU/HGWのLANポート → LN6001-JPのInternet（WAN）ポートへLANケーブルを接続
2. PCを有線接続する場合: LN6001-JPのLANポート（1〜4番いずれか）→ PCのLANポートへ接続
3. LN6001-JPに電源アダプターを接続して通電する
4. 前面LEDが白く点灯するまで待つ（赤のままの場合はWAN側の問題）

Linksys公式では、白点灯でインターネット接続を検知、赤点灯のままの場合は回線側の接続またはインターネット設定を確認するよう案内されています。

赤のままでも、すぐに故障と決めつけなくて大丈夫です。まずはWANケーブル、回線機器、WAN設定の順に確認します。

## LuCI管理画面へのアクセス手順

### ブラウザでアクセス

1. PCまたはスマートフォンをLN6001-JPに接続（有線または本体底面のSSIDへWi-Fi接続）
2. ブラウザのアドレスバーに以下を入力してEnter:

```
https://192.168.1.1
```

証明書警告が出る場合は「詳細設定」→「このサイトにアクセスする（または続行）」を選択します（自己署名証明書のため正常な動作です）。

初回アクセスでは少し不安に見えますが、ローカル管理画面の自己署名証明書なので、この場面ではよくある表示です。

### ログイン

- ユーザー名: `root`
- パスワード: 本体底面ラベルの管理パスワード（初回）

ログインすると、LuCIのStatus（概要）画面が表示されます。

## 管理者パスワードの変更（最初にやること）

初回ログイン後は、まず管理者パスワードを変更します。ここは最初に済ませておきたい作業です。

### LuCIでの手順

1. 上部メニューの **System** をクリック
2. **Administration** を選択
3. **Router Password** セクションを確認
4. **Password** フィールドに新しいパスワードを入力
5. **Confirmation** フィールドに同じパスワードを再入力
6. **Save** をクリック

パスワードは12文字以上を目安に、英大文字・英小文字・数字・記号を混ぜておくと安心です。

覚えきれない文字列にして、パスワードマネージャーへ保存しておくのがおすすめです。

### SSHでの変更

```sh
passwd
# New password: （新しいパスワードを入力）
# Retype new password: （同じパスワードを再入力）
```

## SSIDとWi-Fiパスワードの変更

工場出荷時のSSIDとWi‑Fiパスワードは、必要に応じて変更します。端末ごとに違う文字列が割り振られているので、無理に変更しなくても構いません。

ただ、家族やスタッフに案内しやすい名前にしたい場合は、このタイミングで変えておくと後が楽です。

### LuCIでの手順

1. 上部メニューの **Network** → **Wireless** を選択
2. 設定したいSSIDの右側にある **Edit** ボタンをクリック
3. **Interface Configuration** タブの **ESSID** フィールドに新しいWi-Fi名を入力
4. **Wireless Security** タブを開く
5. **Encryption** で `WPA2-PSK` または `WPA3-SAE` を選択
6. **Key** フィールドに新しいWi-Fiパスワードを入力（8文字以上）
7. **Save** をクリック
8. 同じ操作を2.4GHz / 5GHz / 6GHz それぞれで設定する（または同じSSIDにまとめる）
9. ページ上部に出る「Unsaved changes」の **Apply** ボタンをクリックして反映

変更適用後はSSIDへの接続が一度切れます。これは正常です。新しいSSIDとパスワードで再接続してください。

### CLIで現在の設定を確認する

```sh
echo "### backup wireless config"
cp /etc/config/wireless /etc/config/wireless.backup.$(date +%Y%m%d-%H%M)

echo "### SSID and encryption"
uci show wireless | grep -E 'ssid|encryption|disabled'

echo "### radio status"
wifi status
```

Wi‑Fi名とパスワードの変更は、LuCIのWireless画面で行うのが安全です。

`default_radio0` のようなセクション名は環境で変わることがあるため、確認せずにコピペで `uci set` を実行しないでください。CLIは、最初は変更より確認に使うくらいで十分です。

## 回線タイプを確認する

初期設定で詰まりやすいのはWAN側の回線です。ここは焦らず、自分の契約と回線機器の構成を確認しながら進めます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-01.png)

OCNバーチャルコネクト、transixなどのIPv4 over IPv6サービスを使う場合は、別記事「NTT IPoEとIPv4 over IPv6」も参照してください。

## WAN接続状態の確認方法

### LuCIで確認

1. **Status** → **Overview** を開く
2. **Network** セクションで以下を確認:
   - IPv4 WAN Status: `Connected` と表示されているか
   - IPv4 Address: WANのIPアドレスが表示されているか
   - IPv6 WAN Status: `Connected` と表示されているか（IPoE環境）

### CLIで確認

```sh
echo "### WAN status"
ifstatus wan
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### reachability"
ping -c 4 8.8.8.8
ping -c 4 google.com
```

`ifstatus wan` と `ifstatus wan6` は、いきなり設定を書き換えるためではなく、まず今の状態を見るためのコマンドです。最初は読み取り用として使うだけでも十分役に立ちます。

## タイムゾーンとホスト名の設定

1. **System** → **System** を選択
2. **General Settings** タブで以下を設定:
   - Hostname: ルーターの名前（例: `LN6001-home`）
   - Timezone: `Asia/Tokyo` を選択
3. **Save & Apply** をクリック

## バックアップを取る（必須）

設定が安定したら、必ず初期バックアップを取ります。ここはかなり大事です。

バックアップが1本あるだけで、あとからGuest Wi‑Fi、VLAN、VPN、IPoE設定を試す時の安心感がかなり変わります。これが後の回復の基点になります。

### LuCIでのバックアップ手順

1. **System** → **Backup / Flash Firmware** を選択
2. **Backup** セクションの **Generate archive** ボタンをクリック
3. `.tar.gz` ファイルがダウンロードされる
4. ファイル名に日付を付けてPCに保存（例: `backup-LN6001-20260507.tar.gz`）

### CLIでのバックアップ

```sh
BACKUP_DIR="/root/initial-setup-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json"
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"

ls -l "$BACKUP_DIR"
```

このバックアップにはWi-Fiパスワードや管理に関わる情報が含まれるため、そのまま他人へ共有しないようにします。

PCへ転送する場合は、PC側のターミナルで実行します。

```sh
scp -r root@192.168.1.1:/root/initial-setup-backup-YYYYMMDD-HHMM ./
```

## 初期設定後に控える情報

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-02.png)

## 最初に変更してよいもの、慎重に扱うもの

変更しやすいもの:

- SSID・Wi-Fiパスワード
- 管理パスワード
- タイムゾーン・ホスト名

慎重に扱うもの:

- LAN側IPアドレス
- WAN接続方式
- VLAN・firewall zone
- 無線の国設定・送信出力・DFS関連（この連載では変更しない）

## 初期設定でつまずきやすい場面と対処

初期設定でつまずいても、原因はだいたい限られています。まずは管理画面、WAN接続、IPアドレス重複の3つに分けて見ると切り分けしやすいです。

### 管理画面にアクセスできない

- ブラウザが `http://` でアクセスしている → `https://192.168.1.1` を使う
- HGWとIPが重複している → PCをLN6001-JPのLANポートに有線接続して試みる
- 証明書エラーで止まっている → 「詳細」→「続行」で進む（初回のみ正常）

### Wi-Fiにはつながるがインターネットへ出られない

1. `ifstatus wan` でWAN接続状態を確認
2. LEDが赤のままの場合はWANケーブルの接続を確認
3. HGWあり環境 → LN6001-JPのWAN設定が `DHCP` になっているか確認
4. HGWなし環境 → IPoEモジュールの導入が必要かを確認

### 既存HGWとIPアドレスが重複している

この変更を行うと、管理画面のURLも変わります。作業前に現在のバックアップを取ってから進めると安心です。

HGWのIPが `192.168.1.1` でLN6001-JPも同じ場合:

```sh
echo "### current LAN config"
uci show network.lan

echo "### backup network config"
cp /etc/config/network /etc/config/network.backup.$(date +%Y%m%d-%H%M)

echo "### change LN6001-JP LAN IP"
uci set network.lan.ipaddr='192.168.10.1'
uci set network.lan.netmask='255.255.255.0'
uci changes network
uci commit network
/etc/init.d/network restart
```

変更後は `https://192.168.10.1` で管理画面にアクセスします。

## まとめ

最初の成功条件としては、次の3つが確認できれば十分です。最初から細かい最適化まで終わらせる必要はありません。

- LuCI に再ログインできる
- ふだん使う端末が Wi-Fi につながる
- バックアップを1本保存できている

最初の6ステップ:

1. ONU/HGWにWANポートを接続して通電
2. `https://192.168.1.1` でLuCIにアクセス
3. 管理パスワードを変更
4. SSIDとWi-Fiパスワードを設定
5. WAN接続を確認
6. バックアップを取得

この6ステップを終えたら、IPoE設定、Guest Wi‑Fi、VLAN、VPNなど、必要な機能を一つずつ足していけば十分です。初期設定は“全部を完成させる作業”ではなく、“安全に次へ進むための土台作り”と考えると進めやすいです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-03.png)

## よくある質問

### LN6001-JPは買ってすぐ使える？

DHCP自動でつながる回線環境なら、回線機器につないで電源を入れれば使い始めやすいです。ただし、管理パスワード変更と初期バックアップは最初に済ませておくほうが安心です。

### 初期設定で最初に変えるべきものは？

まずは管理パスワードです。その次にSSIDとWi-Fiパスワード、最後にWAN接続確認とバックアップ、という順だと迷いにくくなります。

### 管理画面に入れない時は何を確認すればいい？

`https://192.168.1.1` で開いているか、HGWとLAN IPが重複していないか、証明書警告で止まっていないかを先に確認します。有線接続したPCから試すと切り分けしやすいです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT router setup: https://support.linksys.com/kb/article/219-en/?section_id=175
- Linksys OpenWRT web interface login: https://support.linksys.com/kb/article/223-en/?section_id=175
- Linksys OpenWRT admin password change: https://support.linksys.com/kb/article/222-en/?section_id=175
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/

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
