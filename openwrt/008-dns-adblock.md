<!-- mirror-source: articles/008-dns-adblock.md -->

# 広告は全部消えない。でもDNSで余計な通信は減らせる｜LN6001-JPのAdblock入門【OpenWrt集中連載008】

広告ブロックって、端末ごとに入れると地味に面倒です。

PCならブラウザ拡張を入れればいい。  
スマホならアプリやブラウザ設定でなんとかなる。  
でもテレビ、ゲーム機、スマート家電、IoT機器、子ども用タブレット、家族のスマホまで全部となると、急にしんどくなります。

そこで出てくるのが、ルーター側のDNS広告ブロックです。

端末ごとにアプリを入れるのではなく、LN6001-JP側でDNS名前解決を見て、広告・トラッキング・不要な通信先の一部を止める考え方です。

これ、うまく使うとかなり便利です。

ただし、最初に正直に言っておきます。

DNS広告ブロックは、広告を全部消す魔法ではありません。

YouTubeやSNSの広告には効きにくいことがあります。  
アプリ内広告が残ることもあります。  
DoHを使う端末には迂回されることもあります。  
そして、強くしすぎるとログイン、決済、画像表示、スマート家電アプリが壊れることもあります。

なので、この記事の方針はこうです。

```txt
広告をゼロにする
```

ではなく、

```txt
余計なDNS問い合わせを減らす
DNSまわりを見えるようにする
壊れたら戻せるようにする
```

です。

LN6001-JPにAdblockを入れて、LuCIで管理しながら、家庭・小さなオフィスで無理なく運用する流れを見ていきます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、広告を完全に消すことではありません。

まずは、ここまでできればOKです。

- DNS広告ブロックでできること・できないことを知る
- Adblockを入れる前にバックアップを取る
- 導入スクリプト、または手動でAdblockを入れる
- LuCIの **Services → Adblock** を開ける
- Force Local DNSとDNS Reportの意味をざっくり理解する
- 誤ブロックが起きた時に一時停止できる
- Allowlistへ必要なドメインを追加できる

広告ブロックは、強くするより先に「戻せる」ことが大事です。

ここ、本当に大事です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-01.png)

## 先にざっくり結論

DNS広告ブロックは、最初から強くしすぎないほうがうまくいきます。

おすすめの流れはこれです。

1. LuCIバックアップを取る
2. 空き容量とパッケージ状態を確認する
3. 導入スクリプト、または手動で `adblock` / `luci-app-adblock` を入れる
4. **Force Local DNS** と **DNS Report** を有効にする
5. ブロックリストは少なめから始める
6. 普段使うWebサイト、アプリ、決済、認証、ゲーム、動画サービスを確認する
7. 問題が出たら、一時停止 → DNS Report確認 → Allowlist追加の順で切り分ける

最初からブロックリストを大量に入れないでください。

強そうに見えますが、必要なサービスまで止める可能性が上がります。

家庭や小さなオフィスでは、

```txt
壊さずに減らす
```

くらいがちょうどいいです。

## こういう人向けです

この記事は、次のような人向けです。

- 家庭内の端末にまとめて広告ブロックを効かせたい
- テレビやIoT機器など、端末側で広告ブロックしにくい機器がある
- 子ども用端末やIoT用ネットワークにDNS制御を入れたい
- 小さなオフィスで不要な広告・トラッキング通信を減らしたい
- Adblockの効果と限界を理解したうえで使いたい
- 誤ブロック時に自分で一時停止やAllowlist追加をできるようにしたい
- DNSまわりを少し見える化したい

逆に、

```txt
とにかく広告を全部消したい
誤ブロックの対応はしたくない
DNSやログは見たくない
```

という人には、少し向かないかもしれません。

DNS広告ブロックは便利ですが、運用もセットです。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-02.png)

## 最初に言葉だけそろえる

Adblockまわりで出てくる言葉を、ざっくり整理しておきます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-01.png)

最初は、次だけ覚えれば十分です。

```txt
Adblock = DNSで止める
Blocklist = 止めるリスト
Allowlist = 止めないリスト
DNS Report = 何が止まったか見る場所
```

## DNS広告ブロックの仕組み

DNS広告ブロックは、広告やトラッキングに使われるドメイン名を、ルーター側で名前解決させないようにします。

たとえば、端末が次のように通信しようとします。

```txt
端末
  ↓
広告・トラッキング用ドメインをDNS問い合わせ
  ↓
LN6001-JPのdnsmasq / Adblock
  ↓
ブロックリストにあれば止める
  ↓
ブロック対象でなければ通常どおり名前解決
```

端末側に広告ブロック拡張を入れる方式とは違い、ルーター配下の端末へまとめて効かせやすいのが特徴です。

一方で、DNS名で止める方式なので、広告の中身を見て判断しているわけではありません。

ざっくり言うと、

```txt
この通信先、広告っぽい名前だから止める
```

という仕組みです。

## できること

DNS広告ブロックで期待できることは、次のようなものです。

- 一部の広告配信ドメインを止める
- 一部のトラッキングドメインを止める
- IoT機器やテレビなど、端末側で設定しにくい機器にも効かせやすくする
- 家庭内のDNS問い合わせを見える化する
- 子ども用・IoT用・ゲストWi-Fi用ネットワークに段階的にDNS方針を適用する
- 不要な通信先を減らし、ネットワーク運用を整理する

「広告を消す」というより、

```txt
余計な通信先を減らす
```

と考えると、期待値が合いやすいです。

## できないこと

DNS広告ブロックには限界があります。

- すべての広告を消せるわけではない
- YouTubeやSNSなど、同じドメインから本文と広告が配信される場合は効きにくい
- アプリ内広告には効かないことがある
- 端末側がDoHを使うと、ルーターのDNSを迂回されることがある
- VPNアプリやキャリア回線経由の通信には効かない
- 誤ブロックでログイン、決済、画像表示、アプリ連携が壊れることがある
- マルウェア対策やペアレンタルコントロールの完全な代替にはならない

ここで勘違いしやすいのですが、DNS広告ブロックはセキュリティ製品ではありません。

もちろん、怪しい通信先を減らす効果は期待できます。

でも、

```txt
Adblockを入れたから安全
```

ではありません。

あくまでDNSレベルで不要な通信先を減らす道具です。

## 導入前に確認すること

Adblockを入れる前に、まずこのあたりを見ます。

- LN6001-JPがインターネットへ正常につながっている
- LuCIへログインできる
- SSHで入れる場合は `root@192.168.1.1` へログインできる
- 設定バックアップをPCへ保存できる
- パッケージを追加できる空き容量がある
- 業務端末や決済端末に影響しても困らないテスト範囲がある
- すでに独自DNS設定を入れていないか確認する

特に、小さなオフィスや店舗では慎重に進めてください。

決済端末、予約システム、業務アプリ、認証サービスがDNS広告ブロックの影響を受けることがあります。

いきなり全端末へ適用するより、まずは検証用の端末や家庭用端末で試すほうが安全です。

## 導入前にバックアップを取る

Adblockは、DNS、DHCP、firewall、パッケージ状態に関係します。

つまり、少し広い範囲を触ります。

まずLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-adblock-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/adblock-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in dhcp firewall network; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
df -h > "$BACKUP_DIR/df-h.txt"
free > "$BACKUP_DIR/free.txt"
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

バックアップにはSSID、DNS設定、ネットワーク情報などが含まれます。

そのままSNSや公開リポジトリへ出さないようにしてください。

## 空き容量とパッケージを確認する

パッケージ追加前に、空き容量とパッケージ情報を確認します。

```sh
echo "### storage"
df -h

echo "### memory"
free

echo "### update package lists"
opkg update

echo "### adblock related packages"
opkg list | grep -E '^adblock|^luci-app-adblock|^luci-i18n-adblock-ja|^banip|^luci-app-banip'
```

`opkg update` はインターネット接続が必要です。

ここで失敗する場合は、Adblock以前の問題です。

先にWAN接続、DNS、時刻、回線設定を確認します。

```sh
ifstatus wan
ifstatus wan6
nslookup downloads.openwrt.org
date
```

Adblockを入れる前に、ルーター自身がパッケージを取りに行ける状態であることを確認します。

## 導入方法は2つある

LN6001-JPでAdblockを入れる方法は、大きく2つあります。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-02.png)

どちらが正解というより、目的が違います。

まず実用状態を作って、あとから理解したいなら導入スクリプト。  
中身を理解しながら進めたいなら手動。

このくらいの分け方で大丈夫です。

## 方法1: 導入スクリプトを使う

Adblock、LuCIパッケージ、日本語表示、ブロックリスト、LAN向けDNS配布、Force Local DNS、自動更新cronなどを自分で整えるのが大変な場合は、導入スクリプトを使う方法があります。

- スクリプト集: https://github.com/ikm-san/velop
- 対象スクリプト: `adb_setup.sh`

実行例です。

```sh
curl -sS -o /tmp/adb_setup.sh https://raw.githubusercontent.com/ikm-san/velop/main/adb_setup.sh
sed -n "1,160p" /tmp/adb_setup.sh
sh /tmp/adb_setup.sh -v
```

1行で実行する場合は次の形です。

```sh
curl -sS -o /tmp/adb_setup.sh https://raw.githubusercontent.com/ikm-san/velop/main/adb_setup.sh && sh /tmp/adb_setup.sh -v
```

ただし、いきなり `curl ... && sh ...` で実行する前に、READMEとスクリプト内容を確認してください。

ここ、けっこう大事です。

ルーター設定を変更するスクリプトなので、

```txt
何をするスクリプトか分からないけど、とりあえず実行
```

は避けます。

## スクリプトが行う主な処理

公開されている `adb_setup.sh` では、主に次のような処理を行います。

- `opkg update` をリトライ付きで実行
- `adblock`、`luci-app-adblock`、`luci-i18n-adblock-ja` をインストール
- TOFUフィルターをAdblockのソースへ追加
- LANのDHCPで、ルーター自身のIPv4/IPv6アドレスをDNSとして配布
- AdblockのDNS backendを `dnsmasq` に設定
- Force Local DNSを有効化
- Adblockを有効化・再起動
- `dnsmasq` を再起動
- `t.co` をAllowlistへ追加
- 一部ドメインをBlocklistへ追加
- 毎日3:50にAdblockをreloadするcronを追加
- 最後に再起動するか確認する

独自DNS設定をすでに入れている場合は、特に注意してください。

このスクリプトは `dhcp.lan.dhcp_option` と `dhcp.lan.dns` を整理して、LAN向けDNS配布を設定し直します。

既存のDNS配布設定を使っている環境では、必ずバックアップを取ってから実行します。

## スクリプト利用後に確認すること

スクリプト実行後は、再起動またはサービス再起動後に次を確認します。

```sh
echo "### packages"
opkg list-installed | grep -E 'adblock|luci-app-adblock|luci-i18n-adblock'

echo "### adblock status"
/etc/init.d/adblock status

echo "### dnsmasq status"
/etc/init.d/dnsmasq status

echo "### dhcp DNS options"
uci show dhcp.lan | grep -E 'dhcp_option|dns'

echo "### adblock config"
uci show adblock | grep -E 'adb_enabled|adb_dns|adb_forcedns|adb_trigger|adb_sources'

echo "### recent logs"
logread | grep -Ei 'adblock|dnsmasq' | tail -n 80
```

さらに、普段使うサイトやアプリを実際に開いて確認します。

- 検索
- SNS
- 動画
- ショッピング
- 決済
- 認証
- ゲーム
- スマート家電アプリ

広告が減っていても、普段使うサービスが壊れていたら運用としては失敗です。

「広告が減った」だけでなく、「必要なものが壊れていない」ことまで確認します。

## 方法2: LuCIで手動インストールする

手動で進める場合は、LuCIからパッケージを入れるのが分かりやすいです。

1. **System** → **Software** を開く
2. **Update lists** をクリックする
3. 検索ボックスに `adblock` と入力する
4. `luci-app-adblock` を見つけて **Install** をクリックする
5. 必要に応じて `adblock` もInstallする
6. 日本語表示を使いたい場合は `luci-i18n-adblock-ja` も確認する
7. ブラウザを更新する
8. **Services** → **Adblock** が表示されることを確認する

インストール後、メニューに反映されるまで少し時間がかかることがあります。

表示されない場合は、LuCIを更新するか、一度ログアウトして再ログインします。

必要なら、ルーターを再起動します。

```sh
reboot
```

## CLIで手動インストールする

CLIで進める場合は、次のようにします。

```sh
echo "### update package lists"
opkg update

echo "### install adblock packages"
opkg install adblock luci-app-adblock luci-i18n-adblock-ja

echo "### verify installed packages"
opkg list-installed | grep -E 'adblock|luci-app-adblock|luci-i18n-adblock'
```

日本語パッケージが見つからない場合は、`adblock` と `luci-app-adblock` だけでも構いません。

インストール後、LuCIの **Services** に **Adblock** が追加されます。

## Adblockの基本設定

Adblockをインストールしたら、LuCIから基本設定を行います。

1. **Services** → **Adblock** を開く
2. **Settings** を開く
3. 次の項目を確認する

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-03.png)

4. **Save & Apply** をクリックする
5. **Action** またはステータス画面で **Reload** を実行する

Force Local DNSとDNS Reportは、最初から有効にしておくと運用しやすいです。

DNS Reportがあると、どの端末がどのドメインへ問い合わせているか、何がブロックされたかを確認しやすくなります。

## ブロックリストは少なめから始める

Adblockでは、複数のBlocklist Sourcesを選べます。

ここで全部にチェックを入れたくなります。

気持ちは分かります。

でも、最初は少なめがおすすめです。

リストを増やすほど、ブロックできる範囲は広がります。

その一方で、必要なサイトやアプリまで止めてしまう可能性も上がります。

最初は次の方針で十分です。

- まずはデフォルト、または少数の定番リストで始める
- いきなり全部にチェックを入れない
- 普段使うサービスを確認する
- 問題がなければ少しずつ追加する
- 誤ブロックが出たら、どのリストを増やした後か思い出せるようにする

ブロックリスト選びは「強いほど良い」ではありません。

家庭や業務で普通に使うなら、安定性のほうが大事です。

## 動作確認する

設定後、まずAdblockの状態を確認します。

```sh
echo "### adblock status"
/etc/init.d/adblock status

echo "### recent adblock logs"
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100

echo "### block files"
ls -l /tmp/adblock 2>/dev/null || true
find /tmp/adblock -type f -maxdepth 1 2>/dev/null | xargs -r wc -l
```

LuCIでは、**Services** → **Adblock** のDNS Reportを見ます。

DNS Reportで、問い合わせやブロック結果が見えることを確認します。

次に、端末側で普段使うWebサイトやアプリを開きます。

見るポイントは次の通りです。

- Webサイトが普通に開く
- ログインできる
- 決済画面へ進める
- SNSアプリが動く
- 動画アプリが動く
- スマート家電アプリが動く
- 必要な画像やCDNが落ちていない
- 広告やトラッキングが減っている

広告が完全に消えなくても正常です。

DNS広告ブロックで減らせる広告と、減らせない広告があります。

## 端末がルーターDNSを使っているか確認する

Adblockが動いていても、端末が別のDNSを使っていると効果が出ません。

まずDHCPで配布されているDNSを確認します。

```sh
echo "### dhcp lan options"
uci show dhcp.lan | grep -E 'dhcp_option|dns'

echo "### leases"
cat /tmp/dhcp.leases
```

端末側でもDNSサーバーを確認します。

### Mac

```sh
scutil --dns | grep nameserver
```

### Windows PowerShell

```powershell
Get-DnsClientServerAddress
```

DNS設定を変えた後は、端末のWi-Fiを一度切断して再接続すると、新しいDHCP情報を受け取りやすくなります。

端末によっては、再起動したほうが早いこともあります。

## Force Local DNSの考え方

Force Local DNSは、端末が外部DNSへ直接問い合わせようとする通常のDNS通信を、ルーター側へ寄せるための設定です。

たとえば、端末が `8.8.8.8` や `1.1.1.1` へ直接DNS問い合わせを投げようとしても、ルーター側のDNSフィルタを通すようにできます。

ただし、これは主に通常のDNS通信、つまりポート53のDNSを対象にした考え方です。

DoHのようにHTTPSの中にDNS問い合わせを入れる方式は、別の扱いになります。

つまり、

```txt
Force Local DNS = 通常DNSの寄せ直し
DoH対策 = それとは別に考える
```

です。

Force Local DNSを有効にしても、DoHまで完全に防げるとは考えないほうが安全です。

## DoHへの対応

ブラウザやアプリがDoHを使うと、ルーターのDNSを迂回する場合があります。

DoH自体は悪いものではありません。

外部ネットワークでDNSの盗聴や改ざんを防ぐ目的では便利な仕組みです。

ただし、家庭や小さなオフィスでルーター側のAdblockを通したい場合は、DoHがAdblockを迂回する原因になります。

## まず端末側で確認する

ブラウザの設定を確認します。

- Chrome: 設定 → プライバシーとセキュリティ → セキュリティ → セキュアDNSを使用する
- Firefox: 設定 → 一般 → ネットワーク設定 → DNS over HTTPS
- Edge: 設定 → プライバシー、検索、サービス → セキュアDNS

家庭内でAdblockを効かせたい端末では、まず端末側のDoH設定を確認します。

## banIPを使う方法

Linksys公式のDoH設定手順では、DoH迂回への対策例として `luci-app-banip` を追加し、banIP側でDoHをIPv4/IPv6ともに有効化する流れが案内されています。

手順の大枠は次の通りです。

1. **System** → **Software** で `luci-app-banip` をインストールする
2. ブラウザを更新する
3. **Services** → **banIP** を開く
4. banIPを有効にする
5. DoHをIPv4/IPv6の両方で有効にする
6. **Save & Apply** をクリックする
7. 必要に応じて再起動する

CLIでインストールする場合は次のようにします。

```sh
opkg update
opkg install banip luci-app-banip
```

ただし、banIPもブロック機能です。

有効化する範囲が広がるほど、誤ブロックの可能性もあります。

まずは端末側DoHを確認し、それでも必要ならbanIPを使う。

この順番が現実的です。

## 誤ブロックへの対応フロー

広告ブロック運用で一番大事なのは、問題が起きた時に戻せることです。

次の順番で切り分けると落ち着いて対応できます。

1. Adblockを一時停止する
2. 問題が解消するか確認する
3. DNS Reportやログで怪しいドメインを見る
4. Allowlistへ必要なドメインを追加する
5. AdblockをReloadする
6. ブロックリストを減らすか見直す

いきなりBlocklistを全部消す必要はありません。

まず止める。  
直るか見る。  
怪しいドメインを見る。  
必要なものだけ許可する。

この順番です。

## Adblockを一時停止する

LuCIでは、**Services** → **Adblock** でEnabledを外し、**Save & Apply** します。

CLIでは次を使います。

```sh
/etc/init.d/adblock stop
```

一時停止で問題が解消するなら、Adblockによるブロックが原因の可能性が高くなります。

確認後、再開する場合は次を実行します。

```sh
/etc/init.d/adblock start
/etc/init.d/adblock reload
```

## ブロックされたドメインを確認する

特定のドメインが怪しい場合は、runtime listやログを確認します。

```sh
DOMAIN="domain-to-check.example"

echo "### search runtime blocklists"
grep -R "$DOMAIN" /tmp/adblock 2>/dev/null || true

echo "### recent logs"
logread | grep -Ei "adblock|dnsmasq|$DOMAIN" | tail -n 100
```

LuCIのDNS Reportも確認します。

どの端末が、どのドメインへ問い合わせ、ブロックされたかを見ます。

## Allowlistに追加する

必要なドメインが誤ブロックされている場合は、Allowlistへ追加します。

### LuCIで追加する

1. **Services** → **Adblock** を開く
2. **Allowlist** を開く
3. 許可したいドメインを1行1ドメインで追加する

例:

```txt
example.com
api.business-service.jp
payment-gateway.example
```

4. **Save & Apply** をクリックする
5. AdblockをReloadする

### CLIで追加する

```sh
echo "example.com" >> /etc/adblock/adblock.whitelist
/etc/init.d/adblock reload
```

Allowlistへ追加する時は、広すぎるドメインを入れないようにします。

たとえば、原因が `api.example.com` だけなのに、いきなり `example.com` 全体を許可すると、不要な通信まで戻してしまうことがあります。

最初は必要最小限で追加します。

## よくある誤ブロック

DNS広告ブロックで起きやすい誤ブロックは次の通りです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-04.png)

とくに業務利用では、決済、認証、予約、会計、POS、勤怠、チャット、クラウドストレージの動作確認を優先します。

業務端末があるネットワークでは、強いブロックリストをいきなり有効にしないほうが安全です。

## ネットワークごとに適用範囲を考える

家庭や小さなオフィスでは、最初から全端末へ一括適用するより、ネットワークごとに方針を分けると運用しやすくなります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-05.png)

Adblockだけですべてを分けるより、SSID、VLAN、DHCP、DNS、firewall zoneを組み合わせて考えると整理しやすいです。

子ども用・IoT用・ゲストWi-Fi用のネットワークを分けている場合は、それぞれのDHCPやDNS配布方針を決めます。

## DHCPで別DNSを配布する

特定のネットワークに別のDNSを配布したい場合は、DHCP Optionsを使います。

たとえば、子ども用ネットワーク `kids` にCloudflare Family DNSを配る例です。

LuCIでは次のように設定します。

1. **Network** → **Interfaces** を開く
2. 子ども用インターフェース、たとえば `kids` の **Edit** をクリックする
3. **DHCP Server** → **Advanced Settings** を開く
4. **DHCP Options** に次を追加する

```txt
6,1.1.1.3,1.0.0.3
```

5. **Save & Apply** をクリックする

CLIでは次のようにします。

```sh
echo "### backup DHCP config"
cp /etc/config/dhcp /etc/config/dhcp.backup.$(date +%Y%m%d-%H%M)

echo "### add DNS option for kids"
uci add_list dhcp.kids.dhcp_option='6,1.1.1.3,1.0.0.3'
uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart
```

DNS配布を変更した後は、端末のWi-Fiを一度切断して再接続します。

端末によっては、再起動したほうが早いこともあります。

## 既存DNS設定との衝突に注意する

Adblock、Force Local DNS、DHCP Options、DoH、banIP、VPN、ゲストWi-Fi、VLANは、DNSまわりで関係します。

たとえば、次のような構成では注意が必要です。

- すでにDHCPで外部DNSを配っている
- 端末側に固定DNSが設定されている
- VPNクライアントが独自DNSを配っている
- TailscaleやWireGuardでMagicDNSや内部DNSを使っている
- ゲストWi-Fiだけ別DNSにしている
- 子ども用ネットワークでFamily DNSを使っている

DNS広告ブロックは便利ですが、DNSはネットワーク全体の土台です。

何かがおかしくなった時は、いきなりブロックリストを増やすのではなく、まず「端末がどのDNSを使っているか」を確認します。

## Adblockを更新・再読み込みする

ブロックリストは更新されます。

手動で再読み込みする場合は次を使います。

```sh
/etc/init.d/adblock reload
```

ログを確認します。

```sh
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100
```

スクリプトを使った場合は、毎日3:50にAdblockをreloadするcronが追加されます。

cronを確認するには次を実行します。

```sh
crontab -l
```

必要に応じて、時刻を変えることもできます。

ただし、深夜にルーターやDNSが一瞬不安定になると困る環境では、業務時間外や利用が少ない時間へ調整します。

## 無効化・アンインストールする場合

問題が大きい場合は、一時停止や無効化で戻します。

## 一時停止

```sh
/etc/init.d/adblock stop
```

## 自動起動を止める

```sh
/etc/init.d/adblock disable
```

## LuCIで無効化する

1. **Services** → **Adblock** を開く
2. **Enabled** のチェックを外す
3. **Save & Apply** をクリックする
4. 必要に応じて `dnsmasq` を再起動する

```sh
/etc/init.d/dnsmasq restart
```

## パッケージを削除する

完全に削除する場合は、設定バックアップを取ってから行います。

```sh
opkg remove luci-app-adblock adblock
/etc/init.d/dnsmasq restart
```

`luci-i18n-adblock-ja` を入れている場合は、必要に応じて削除します。

```sh
opkg remove luci-i18n-adblock-ja
```

削除後、DHCPで配っているDNS設定やForce Local DNS関連の設定が残っていないか確認します。

```sh
uci show dhcp.lan | grep -E 'dhcp_option|dns' || true
uci show adblock 2>/dev/null || true
```

## トラブル時の見方

## Webサイトが開かない

まずAdblockを一時停止します。

```sh
/etc/init.d/adblock stop
```

その状態でWebサイトが開くなら、Adblock側の可能性が高いです。

次にDNS Reportとログを見ます。

```sh
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100
```

必要なドメインをAllowlistへ追加します。

## 広告が減らない

次を確認します。

- Adblockが有効か
- Blocklistが取得できているか
- 端末がLN6001-JPのDNSを使っているか
- Force Local DNSが有効か
- 端末側でDoHを使っていないか
- その広告がDNSブロックで止められる種類か

CLIでは次を見ます。

```sh
/etc/init.d/adblock status
uci show adblock | grep -E 'adb_enabled|adb_forcedns|adb_dns'
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100
```

広告が残っていても、それだけで設定失敗とは限りません。

DNS広告ブロックでは止めにくい広告もあります。

## DNSが不安定になった

まずAdblockを止めて、dnsmasqを再起動します。

```sh
/etc/init.d/adblock stop
/etc/init.d/dnsmasq restart
```

その後、端末のWi-Fiを切断・再接続します。

それでも不安定なら、バックアップから戻すことを検討します。

```sh
cp /root/adblock-before-YYYYMMDD-HHMM/dhcp /etc/config/dhcp
cp /root/adblock-before-YYYYMMDD-HHMM/firewall /etc/config/firewall
cp /root/adblock-before-YYYYMMDD-HHMM/network /etc/config/network

uci changes
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
/etc/init.d/network restart
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えてください。

LuCIのバックアップファイルを使う場合は、**System** → **Backup / Flash Firmware** から復元します。

## 運用のおすすめ

家庭や小さなオフィスでは、次の運用が扱いやすいです。

- 最初は軽めのブロックリストから始める
- 業務端末や決済端末にはいきなり適用しない
- 誤ブロックが出たらAllowlistで戻す
- 変更前にLuCIバックアップを取る
- DNS Reportを見て、何がブロックされているか確認する
- DoHは端末側設定とbanIPの両方で段階的に考える
- 子ども用やIoT用は、SSID/VLAN/DHCPとセットで設計する
- 「広告ゼロ」より「壊さず減らす」を優先する

広告ブロックは、強くしようと思えばいくらでも強くできます。

でも、日常利用では「強さ」より「戻せること」のほうが大事です。

## まとめ

LN6001-JPでは、OpenWrtベースの柔軟性を活かして、AdblockによるDNS広告ブロックを導入できます。

ただし、DNS広告ブロックは万能ではありません。

- 端末ごとにアプリを入れず、ルーター側で不要な通信先を減らせる
- LuCIからAdblockを管理できる
- Force Local DNSとDNS Reportを有効にすると運用しやすい
- DoHを使う端末には効きにくい場合がある
- 誤ブロックに備えてAllowlistと一時停止の手順を持っておく
- 業務端末や決済端末には慎重に適用する

最初は、導入スクリプトまたは手動導入で動く状態を作り、普段使うサイトやアプリに問題がないか確認します。

そのあとで、ブロックリスト、Allowlist、DoH対策、ネットワーク別DNS方針を少しずつ整えていくのがおすすめです。

広告を全部消すより、壊さず減らす。

このくらいの距離感が、家庭や小さなオフィスではちょうどいいと思います。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-03.png)

## 次に読むなら

DNS広告ブロックを使い始めたら、次は目的に合わせて進みます。

- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

子ども用端末にDNS方針を分けたい人は、家族向けフィルタリングへ。

ゲストWi-FiやIoT用SSIDごとにDNS方針を変えたい人は、ゲストWi-FiやVLANの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### LN6001-JPで広告ブロックはできますか？

できます。

`adblock` と `luci-app-adblock` を入れると、LuCIからDNS広告ブロックを管理できます。

導入スクリプトを使う方法と、LuCI / CLIから手動で入れる方法があります。

### DNS広告ブロックで広告は全部消えますか？

消えません。

同じドメインから配信される広告、アプリ内広告、DoHを使う端末、VPN経由の通信などには効きにくいことがあります。

「全部消す」より、「不要な通信先を減らす」くらいの期待値で使うほうが現実的です。

### Force Local DNSを有効にすれば十分ですか？

通常のDNS通信をルーター側へ寄せるには役立ちます。

ただし、DoHのようにHTTPSでDNSを送る方式は別です。

DoH対策まで考えるなら、端末側のDoH設定確認やbanIPのDoHブロックも検討します。

### 誤ブロックが起きたらどうしますか？

まずAdblockを一時停止して原因を切り分けます。

Adblock停止で問題が解消するなら、DNS Reportやログを確認し、必要なドメインをAllowlistへ追加します。

それでも難しい場合は、ブロックリストを減らします。

### 子ども用ネットワークだけ強めにできますか？

できます。

ただし、Adblock単体で考えるより、子ども用SSID、VLAN、DHCP、DNS配布、firewall zoneをセットで設計するほうが分かりやすいです。

必要に応じてFamily DNSの配布も検討できます。

### 店舗や業務ネットワークに入れてもいいですか？

慎重に進めるなら使えます。

ただし、決済端末、POS、予約システム、会計、認証サービス、業務チャット、クラウドストレージなどは誤ブロックの影響が大きいです。

まず検証用ネットワークで試し、問題がないことを確認してから広げてください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターにAdblockを追加してセットアップする方法: https://support.linksys.com/kb/article/7050-jp/
- Velop WRT Pro 7 OpenWrt DNS-over-HTTPS（DoH）の設定方法: https://support.linksys.com/kb/article/7047-jp/
- OpenWrt Wiki - Ad blocking: https://openwrt.org/docs/guide-user/services/ad-blocking
- OpenWrt Wiki - DNS hijacking: https://openwrt.org/docs/guide-user/firewall/fw3_configurations/intercept_dns
- ikm-san velop 広告ブロック導入スクリプト: https://github.com/ikm-san/velop

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
