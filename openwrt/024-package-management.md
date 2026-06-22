<!-- mirror-source: articles/024-package-management.md -->

# opkgは楽しい。でも入れすぎ注意｜LN6001-JPで追加パッケージを安全に使う【OpenWrt集中連載024】

`opkg` は楽しいです。

これ、本当に楽しいです。

OpenWrtベースのルーターに、あとから機能を足せます。

`tcpdump` で通信を見る。  
`curl` で疎通を確認する。  
Adblockを入れてDNS広告ブロックを試す。  
DDNSのLuCI画面を追加する。  
ちょっとした診断ツールを入れる。

普通の家庭用Wi-Fiルーターだと、ここまで自由には触れません。

LN6001-JPはLuCI、SSH、opkgに対応しているので、必要な機能をあとから足せます。

ここがOpenWrtベースルーターの楽しいところです。

ただし、ここで一度ブレーキです。

ルーターはLinux PCではありません。

ノートPCやサーバーみたいに、

```txt
便利そうだから全部入れておこう
全部アップデートして最新にしておこう
```

というノリで触ると、あとで困ることがあります。

ストレージは限られます。  
メモリも限られます。  
`kmod-*` はカーネルバージョンに依存します。  
ファームウェア更新後に追加パッケージ本体が消えることもあります。  
DNSやfirewall系のパッケージは、入れた瞬間に通信へ影響します。

つまり `opkg` は、

```txt
便利そうなものを全部入れる道具
```

ではありません。

```txt
必要なものを、少しずつ、戻せる状態で入れる道具
```

です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で `opkg` を使う時に、何を確認してから入れるか、何を入れすぎないほうがよいか、どう削除するか、ファームウェア更新後にどう戻すかを整理します。

地味ですが、かなり大事な運用記事です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、`opkg` で何でも入れられるようになることではありません。

まずは、ここまでできればOKです。

- `opkg update` と `opkg upgrade` の違いが分かる
- パッケージ追加前に `/overlay` の空き容量を確認できる
- 追加前に `opkg list-installed` を保存できる
- LuCIとCLIでパッケージを探す・入れる・削除する流れが分かる
- `kmod-*` がカーネルバージョンに依存すると分かる
- VPN / IPoE系はLinksys公式モジュールを優先する理由が分かる
- ファームウェア更新後に追加パッケージを戻す考え方が分かる
- 入れたパッケージを棚卸しする習慣が分かる

`opkg` は便利です。

でも、入れるより大事なのは、

```txt
なぜ入れたか
どう戻すか
更新後にどう再導入するか
```

です。

ここまで考えられると、LN6001-JPをかなり安心して育てられます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-01.png)

## 先にざっくり結論

`opkg` を使う時は、まずこの5つを守ればかなり安全です。

1. **インストール前に `opkg update` を実行する**
2. **`df -h` で `/overlay` の空き容量を見る**
3. **導入前に `opkg list-installed` を保存する**
4. **用途が明確なパッケージだけ入れる**
5. **`opkg upgrade` で全体更新しない**

特に大事なのは、`opkg update` と `opkg upgrade` を混同しないことです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-01.png)

`opkg update` は、パッケージ一覧を取り直すだけです。

一方で、`opkg upgrade` はインストール済みパッケージを更新します。

名前は似ています。

でも、重みがぜんぜん違います。

OpenWrt系では、ルーター上で大量のパッケージを一括更新するより、公式ファームウェア更新と、必要パッケージの再導入で管理するほうが安全です。

LN6001-JPでは、WireGuard / Tailscale、オートIPoEなど、Linksys公式モジュールが案内されている機能もあります。

そういう機能は、まず公式モジュールを優先します。

## こういう人向けです

この記事は、次のような人向けです。

- `opkg` で何ができるのか知りたい
- `tcpdump` や `curl` などのツールを追加したい
- AdblockやLuCI拡張を入れる前に注意点を知りたい
- ファームウェア更新後に追加パッケージが消えて困ったことがある
- `opkg update` と `opkg upgrade` の違いを整理したい
- WireGuardやTailscaleを普通のopkgで入れてよいか迷っている
- IPoE系パッケージを手動で入れる前に考えるべきことを知りたい
- 小さなオフィスや店舗で、追加パッケージを増やしすぎない運用にしたい

逆に、OpenWrtのビルドシステムやImage Builderでカスタムファームウェアを作っている人には基本寄りです。

この記事では、LN6001-JPを普通に運用しながら、必要な機能だけ安全に足すことを優先します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-02.png)

## 最初に言葉だけそろえる

パッケージ管理まわりの言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-02.png)

この記事は、LN6001-JPのOpenWrtベース firmware version 1.2.0.15 を前提に、`opkg` で説明します。

OpenWrt本家では、将来バージョンや別系列で `apk` が使われることがあります。

ただし、この記事ではLN6001-JP実機運用として、`opkg` 前提で進めます。

## opkgでできること

`opkg` を使うと、LN6001-JPへ追加機能を入れられます。

代表的には次のような用途です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-03.png)

`opkg` の魅力は、必要なものをあとから足せるところです。

たとえば、トラブル時に `tcpdump` を入れて通信を見る。  
Adblockを入れてDNS広告ブロックを試す。  
DDNSをLuCIから管理する。

こういう柔軟さがあります。

ただし、追加できるからといって、全部入れる必要はありません。

家庭や小さなオフィスでは、まず「トラブル切り分けに役立つもの」「実際に運用で使うもの」だけで十分です。

## opkgで苦手なこと

`opkg` は万能ではありません。

次のようなことは苦手、または注意が必要です。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-04.png)

`opkg` は「必要な部品を追加する道具」です。

Linux PCのように「全部アップデートして常に最新化する道具」と考えないほうが安全です。

## 最初に覚えるコマンド

まずはこのあたりだけで十分です。

```sh
# 本体情報を見る
ubus call system board

# ストレージ空き容量を見る
df -h

# パッケージ一覧を更新する
opkg update

# パッケージを検索する
opkg list | grep tcpdump

# パッケージ情報を見る
opkg info tcpdump

# パッケージを入れる
opkg install tcpdump

# 入っているか確認する
opkg list-installed | grep tcpdump

# パッケージが配置したファイルを見る
opkg files tcpdump

# パッケージを削除する
opkg remove tcpdump

# インストール済み一覧を保存する
opkg list-installed > /tmp/installed-packages-$(date +%Y%m%d-%H%M).txt
```

最初は、

```txt
検索する
情報を見る
入れる
削除する
一覧を保存する
```

だけ覚えれば十分です。

いきなり依存関係やfeedの深いところへ行かなくて大丈夫です。

## LuCIでパッケージを管理する

CLIに慣れていない場合は、LuCIから確認する方法もあります。

1. LuCIへログインする
2. **System** → **Software** を開く
3. **Update lists** をクリックする
4. 検索ボックスでパッケージ名を探す
5. 必要なパッケージの **Install** をクリックする
6. インストール完了後、必要ならブラウザを更新する

LuCIアプリを追加した場合、メニューに反映されるまで少し時間がかかることがあります。

反映されない時は、次を試します。

```sh
/etc/init.d/uhttpd restart
```

それでも出ない場合は、LuCIへ再ログイン、またはルーター再起動を試します。

ただし、LuCIで入れたパッケージも、裏側では `opkg` で管理されています。

LuCIとCLIは別世界ではありません。

同じパッケージ管理を、画面で見るか、コマンドで見るかの違いです。

## パッケージ追加前のチェックリスト

インストール前に、次を確認します。

```txt
□ 何のために入れるパッケージか説明できる
□ 公式モジュールで扱うべき機能ではない
□ `df -h` で空き容量を確認した
□ `opkg update` を実行した
□ 現在のインストール済み一覧を保存した
□ カーネルモジュール系ではないか確認した
□ DNS / firewall / VPN / IPoEへ影響するパッケージではないか確認した
□ ファームウェア更新後に再導入が必要な可能性を理解した
```

特に、次の2つは毎回見ます。

```sh
df -h
opkg update
```

`df -h` では、特に `/overlay` の空きを見ます。

```sh
df -h | grep -E 'Filesystem|overlay|root'
```

`/overlay` がかなり埋まっている場合は、パッケージ追加をいったん止めます。

不要なパッケージや一時ファイルがないか確認します。

`/overlay` がパンパンのルーターは、だいたい機嫌が悪くなります。

## 導入前スナップショットを取る

大きめのパッケージや、DNS、VPN、firewallに関係するパッケージを入れる前は、状態を保存します。

```sh
BACKUP_DIR="/root/opkg-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

echo "### system"
ubus call system board > "$BACKUP_DIR/system-board.json"
df -h > "$BACKUP_DIR/df-h.txt"
free > "$BACKUP_DIR/free.txt"

echo "### packages"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
opkg list-upgradable > "$BACKUP_DIR/opkg-list-upgradable.txt" 2>/dev/null || true

echo "### configs"
for cfg in network wireless firewall dhcp system; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

echo "### logs"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

PCへ保存する場合です。

```sh
scp -r root@192.168.1.1:/root/opkg-before-YYYYMMDD-HHMM ~/Downloads/
```

LAN IPを変更している場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/root/opkg-before-YYYYMMDD-HHMM ~/Downloads/
```

このスナップショットには、SSID、内部IP、MACアドレス、設定情報が含まれることがあります。

公開しないように注意してください。

## 最初に入れるなら何がよいか

最初は、用途が明確で、トラブル切り分けに役立つものから試すのがおすすめです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-05.png)

最初は、`tcpdump` や `curl` のように、何をするものか分かりやすいものから始めると安全です。

一方で、VPN、IPoE、カーネルモジュール、firewallに深く関係するパッケージは、慎重に扱います。

「入れると便利」ではなく、

```txt
今の自分の運用で本当に必要か
```

で判断します。

## インストール例: tcpdump

`tcpdump` は、通信の中身を確認するための定番ツールです。

まずはインストールだけ行います。

```sh
echo "### update package lists"
opkg update

echo "### install tcpdump"
opkg install tcpdump

echo "### verify"
opkg list-installed | grep '^tcpdump'
opkg files tcpdump | head
tcpdump --help | head
```

実際に使う例です。

```sh
# 通信を少しだけ見る例
tcpdump -i any -n -c 20
```

`tcpdump` は出力が多くなります。

長時間回しっぱなしにしないで、必要な時だけ短く使うのがおすすめです。

特に小さなオフィスや店舗では、営業時間中に重いキャプチャを長時間回すのは避けます。

## インストール例: curl

`curl` は、HTTP/HTTPS疎通確認やAPI確認に便利です。

```sh
opkg update
opkg install curl

curl --version
curl -I https://www.google.co.jp
```

ただし、インターネット上のスクリプトを `curl | sh` でいきなり実行するのは避けます。

まずファイルとして保存し、中身を確認します。

```sh
curl -sS -o /tmp/script.sh https://example.com/script.sh
sed -n "1,160p" /tmp/script.sh
sh /tmp/script.sh
```

ルーター設定を変更するスクリプトは、実行前に内容を読みます。

ルーターはネットワークの入口です。

よく分からないスクリプトを流す場所ではありません。

## LuCIアプリを入れる例

LuCIアプリは、管理画面にメニューを追加するパッケージです。

例としてDDNSです。

```sh
opkg update
opkg install luci-app-ddns
/etc/init.d/uhttpd restart
```

インストール後、LuCIをリロードし、メニューに反映されるか確認します。

LuCIアプリを入れた時は、次も確認します。

```sh
opkg list-installed | grep luci-app-ddns
opkg files luci-app-ddns | head
logread | tail -n 80
```

メニューが出ない時は、ブラウザ更新、LuCI再ログイン、uhttpd再起動を試します。

それでも出ない場合は、パッケージ名、依存関係、ログを確認します。

## Adblockを入れる場合

Adblockは便利です。

でもDNSに影響します。

つまり、失敗すると「一部サイトが開かない」「アプリがログインできない」「スマート家電アプリが見えない」といった症状が出ることがあります。

導入前にバックアップを取り、DNSが壊れた時に戻せるようにします。

```sh
opkg update
opkg install adblock luci-app-adblock luci-i18n-adblock-ja
```

日本語パッケージが見つからない場合は、`adblock` と `luci-app-adblock` だけでも構いません。

インストール後に確認します。

```sh
/etc/init.d/adblock status 2>/dev/null || true
opkg list-installed | grep -Ei 'adblock|luci-app-adblock'
logread | grep -Ei 'adblock|dnsmasq' | tail -n 100
```

Adblockを入れたあとに一部サイトやアプリが壊れたら、一時停止して切り分けます。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
nslookup www.google.co.jp 127.0.0.1
```

戻す場合です。

```sh
/etc/init.d/adblock start 2>/dev/null || true
/etc/init.d/adblock reload 2>/dev/null || true
```

Adblockは008の記事で詳しく扱っています。

最初から強いブロックリストを盛りすぎないのがコツです。

## VPN系は公式VPN Assistantを優先する

WireGuardやTailscaleは、一般的なOpenWrtパッケージとしても存在します。

ただし、LN6001-JPではLinksys公式のVPN Assistantモジュールが案内されています。

この連載では、WireGuard / Tailscaleはまず公式VPN Assistantを使う方針にします。

理由は次の通りです。

- LN6001-JP向けの導線が用意されている
- LuCI上で設定しやすい
- 関連パッケージや設定の前提を合わせやすい
- サポート記事と照合しやすい
- 切り分け時に「公式手順どおりか」を確認しやすい

VPN系を普通の `opkg install` だけで組もうとすると、カーネルモジュール、LuCIアプリ、firewall、ルーティング、鍵管理、NAT越えが絡みます。

まずVPN Assistantを使い、必要になってから上級者向けに手動構成を検討するほうが安全です。

## IPoE系も公式Internet Assistantを優先する

NTTフレッツ系のIPoE / IPv4 over IPv6では、MAP-E、DS-Lite、IPIPなどが関係します。

LN6001-JPでは、Linksys公式のオートIPoE / Internet Assistantモジュールが案内されています。

そのため、最初から汎用OpenWrt記事を見て手動でMAP-EやDS-Liteを組むより、まず公式モジュールを確認します。

特に次の環境では重要です。

- OCNバーチャルコネクト
- transix
- v6プラス
- クロスパス
- v6コネクト
- 固定IP / IPIP系サービス
- ONU直下のNTT IPoE環境

IPoE系は回線契約やプロバイダ方式に依存します。

`opkg` で関連パッケージを探して入れる前に、014の記事とLinksys公式サポートを確認してください。

IPv4 over IPv6は、ノリで組むとかなり沼ります。

## kmod系パッケージに注意する

`kmod-*` は、カーネルモジュールのパッケージです。

例:

```txt
kmod-wireguard
kmod-usb-storage
kmod-fs-ext4
```

これらは、実行中のカーネルバージョンと合っている必要があります。

カーネルバージョンが合わないと、依存関係エラーやインストール不可になることがあります。

確認します。

```sh
ubus call system board
uname -a
opkg info kmod-wireguard 2>/dev/null || true
```

`kmod-*` を無理に入れたり、別バージョンの `.ipk` を強制インストールしたりするのは避けます。

特にVPN、USB、ファイルシステム、ネットワークドライバ系のkmodは、ファームウェアとの相性が重要です。

迷う場合は、公式モジュールまたは公式ファームウェア更新を優先します。

## `opkg upgrade` を基本的に使わない

OpenWrt系では、一般的なLinux PCのように「全パッケージを一括アップグレード」する運用はおすすめしません。

つまり、次は基本的に使いません。

```sh
opkg upgrade
```

理由は次の通りです。

- ベースシステムと追加パッケージの整合性が崩れることがある
- カーネルモジュールとカーネルのバージョンが合わなくなることがある
- `/overlay` の容量を圧迫する
- 依存関係や設定ファイルの衝突が起きることがある
- ファームウェア更新で対応すべき内容までopkgで更新しようとしてしまう
- soft-brickのような厄介な状態になる可能性がある

使ってよい基本コマンドは、次です。

```sh
opkg update
opkg install <package>
opkg remove <package>
opkg list-installed
opkg info <package>
opkg files <package>
```

`opkg update` は、パッケージ一覧を更新するだけです。

`opkg upgrade` は、インストール済みパッケージを更新する操作です。

名前が似ていますが、意味が大きく違います。

ここは本当に間違えやすいです。

```txt
updateはOK
upgradeは基本NG
```

最初はこれで覚えてください。

## `.ipk` ファイルを手動インストールする場合

Linksys公式モジュールや、特定の `.ipk` をアップロードして入れる場合があります。

LuCIでは、**System → Software → Upload Package** から `.ipk` をアップロードします。

CLIでは次のようにします。

```sh
opkg install /tmp/package-name.ipk
```

ただし、`.ipk` 手動インストールは注意が必要です。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-06.png)

出所不明の `.ipk` は入れないでください。

ルーターはネットワークの入口です。

よく分からないパッケージを入れるリスクは、PCより大きいと考えておくほうが安全です。

## パッケージ削除の前に確認する

不要になったパッケージを削除する時も、いきなり消さずに確認します。

```sh
PACKAGE_NAME="tcpdump"

echo "### package info"
opkg info "$PACKAGE_NAME"

echo "### files"
opkg files "$PACKAGE_NAME"

echo "### save package list before remove"
BACKUP_DIR="/root/remove-package-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
opkg list-installed > "$BACKUP_DIR/opkg-before.txt"

echo "### remove"
opkg remove "$PACKAGE_NAME"

echo "### verify"
opkg list-installed | grep "$PACKAGE_NAME" || true
df -h
logread | tail -n 80
```

OpenWrtでは、パッケージを削除しても設定ファイルが残ることがあります。

将来再インストールする可能性があるものは、設定を残しておくほうが便利な場合があります。

逆に、完全に不要な設定が残っていると混乱する場合もあります。

削除後は、LuCIメニュー、サービス、ログを確認します。

## サービスの起動状態を見る

パッケージを入れたら、サービスが追加されることがあります。

確認します。

```sh
echo "### init scripts"
ls /etc/init.d

echo "### related services"
ls /etc/init.d | grep -Ei 'adblock|banip|tailscale|wireguard|ddns|dnsmasq|uhttpd'
```

サービス状態を見る例です。

```sh
/etc/init.d/adblock status 2>/dev/null || true
/etc/init.d/dnsmasq status 2>/dev/null || true
/etc/init.d/uhttpd status 2>/dev/null || true
```

自動起動の有効化・無効化は、パッケージによって扱いが異なります。

一般的な形式は次です。

```sh
/etc/init.d/service-name enable
/etc/init.d/service-name disable
/etc/init.d/service-name start
/etc/init.d/service-name stop
/etc/init.d/service-name restart
```

ただし、VPN AssistantやInternet Assistantのような公式モジュールは、LuCI側の設定と合わせて扱います。

CLIだけで勝手に止めたり消したりする前に、LuCIの状態も確認します。

## ファームウェア更新前に保存する

ファームウェア更新前には、追加パッケージ一覧を保存します。

```sh
PKG_DIR="/root/packages-before-firmware-$(date +%Y%m%d-%H%M)"
mkdir -p "$PKG_DIR"

opkg list-installed > "$PKG_DIR/opkg-list-installed.txt"
opkg list-installed | awk '{print $1}' > "$PKG_DIR/opkg-package-names.txt"

opkg list-installed | grep -Ei 'adblock|banip|wireguard|tailscale|vpn|ipoe|internet|assistant|map|dslite|ipip|luci-app' > "$PKG_DIR/important-packages.txt" || true

ubus call system board > "$PKG_DIR/system-board.json"
df -h > "$PKG_DIR/df-h.txt"

ls -l "$PKG_DIR"
```

PCへコピーします。

```sh
scp -r root@192.168.1.1:/root/packages-before-firmware-YYYYMMDD-HHMM ~/Downloads/
```

ファームウェア更新後に確認します。

```sh
opkg list-installed > /tmp/opkg-list-installed-after.txt
```

差分を見る例です。

```sh
diff /root/packages-before-firmware-YYYYMMDD-HHMM/opkg-list-installed.txt /tmp/opkg-list-installed-after.txt 2>/dev/null || true
```

ただし、古い一覧をそのまま全部入れ直さないでください。

ファームウェア更新後は、まず更新後の環境で `opkg update` を行い、そのファームウェアに合ったパッケージを必要なものだけ再導入します。

## ファームウェア更新後の再導入方針

更新後は、次の順番で確認します。

1. LuCIへ入れる
2. WAN / WAN6が戻っている
3. DNSが使える
4. Wi-Fiが使える
5. `opkg update` が通る
6. 必要な追加パッケージが残っているか見る
7. 消えているものを必要な順に戻す

確認コマンドです。

```sh
echo "### system"
ubus call system board

echo "### network"
ifstatus wan
ifstatus wan6
nslookup www.google.co.jp

echo "### opkg"
opkg update
opkg list-installed | grep -Ei 'adblock|banip|wireguard|tailscale|vpn|ipoe|internet|assistant|luci-app' || true
```

戻す優先順位は、環境によって変わります。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-07.png)

回線接続やVPNに関係するものを先に戻し、便利ツールは後からでも構いません。

まずネットが戻る。  
次にVPNが戻る。  
そのあと便利ツール。

この順番が安全です。

## トラブル時の見方

## `opkg update` が失敗する

よくある原因です。

- WANがつながっていない
- DNSが壊れている
- 時刻が大きくずれている
- IPv6だけ不安定
- パッケージリポジトリへ到達できない
- 証明書やHTTPS関連の問題

確認します。

```sh
ifstatus wan
ifstatus wan6
ip route show default
ip -6 route show default
ping -c 4 8.8.8.8
nslookup downloads.openwrt.org
date
logread | grep -Ei 'opkg|wget|uclient|ssl|certificate|dnsmasq|wan' | tail -n 120
```

`opkg update` が失敗する時は、パッケージ管理より前に、WAN、DNS、時刻を見ます。

パッケージの問題ではなく、そもそもルーターが外へ出られていないことがあります。

## `Cannot satisfy dependencies` が出る

依存関係を満たせない時に出ます。

原因の例です。

- `opkg update` していない
- パッケージ名が違う
- リポジトリに対象パッケージがない
- ファームウェアとパッケージのバージョンが合わない
- `kmod-*` のカーネルバージョンが合わない

確認します。

```sh
opkg update
opkg info <package-name>
ubus call system board
uname -a
logread | grep -Ei 'opkg|dependency|kernel|kmod' | tail -n 120
```

`kmod-*` で依存エラーが出る場合は、無理に入れないでください。

ファームウェアや公式モジュールの対応を確認します。

## `No space left on device` が出る

ストレージ不足です。

確認します。

```sh
df -h
du -h -d 1 /overlay 2>/dev/null | sort -h
opkg list-installed | tail -n 80
```

対応の考え方です。

- 不要パッケージを削除する
- 大きなログや一時ファイルがないか見る
- 何でも入れる方針をやめる
- 本当に必要なら、ファームウェア構成や外部ストレージ設計を考える

不要パッケージ削除例です。

```sh
opkg remove <package-name>
df -h
```

ただし、依存関係で必要なものを削除しないよう注意します。

`opkg info` と `opkg files` を見てから削除します。

## LuCIメニューに出てこない

LuCIアプリを入れたのにメニューが出ない場合です。

確認します。

```sh
opkg list-installed | grep luci-app
/etc/init.d/uhttpd restart
logread | grep -Ei 'uhttpd|luci|rpcd' | tail -n 80
```

次を試します。

- ブラウザをリロードする
- LuCIからログアウトして再ログインする
- uhttpdを再起動する
- 必要ならルーターを再起動する
- パッケージ名が正しいか確認する

## インストール後に通信が不安定になった

DNS、firewall、VPN系のパッケージを入れた時に起きやすいです。

まず直前に入れたものを確認します。

```sh
opkg list-installed | tail -n 80
logread | tail -n 150
```

DNS系なら確認します。

```sh
/etc/init.d/dnsmasq status 2>/dev/null || true
/etc/init.d/adblock status 2>/dev/null || true
logread | grep -Ei 'dnsmasq|adblock|banip' | tail -n 120
```

VPN系なら確認します。

```sh
tailscale status 2>/dev/null || true
wg show 2>/dev/null || true
logread | grep -Ei 'tailscale|wireguard|vpn|wg' | tail -n 120
```

一時停止できるサービスなら、一時停止して切り分けます。

```sh
/etc/init.d/adblock stop 2>/dev/null || true
/etc/init.d/dnsmasq restart
```

直前に入れたものを疑う。

これはかなり効きます。

## どのパッケージが大きいか見たい

`opkg` だけでは、実際の使用容量を完全に見やすく出せるわけではありません。

ざっくり見るなら、`/overlay` を確認します。

```sh
df -h
du -h -d 1 /overlay 2>/dev/null | sort -h
```

パッケージが置いたファイルを見るなら次です。

```sh
PACKAGE_NAME="tcpdump"
opkg files "$PACKAGE_NAME"
```

不要になったものは削除します。

```sh
opkg remove "$PACKAGE_NAME"
df -h
```

`/overlay` の空き容量は、たまに見るだけでもかなり違います。

入れっぱなし、増やしっぱなしは避けます。

## パッケージ運用メモ

追加パッケージを使い始めたら、簡単なメモを残します。

```txt
LN6001-JP package memo:

Firmware:
  1.2.0.15

追加パッケージ:
  tcpdump
    purpose: packet capture
    installed: 2026-06-21

  curl
    purpose: HTTP/HTTPS疎通確認
    installed: 2026-06-21

  adblock
    purpose: DNS広告ブロック
    installed: 2026-06-21
    related config: /etc/config/adblock

  VPN Assistant
    purpose: WireGuard/Tailscale
    installed from: Linksys公式サポート
    installed: 2026-06-21

  Auto IPoE / Internet Assistant
    purpose: NTT IPv4 over IPv6
    installed from: Linksys公式サポート

更新前保存:
  opkg-list-installed-20260621-1200.txt

注意:
  opkg upgradeは使わない
  ファームウェア更新後に必要パッケージを再確認
```

この程度でも、あとからかなり助かります。

「これ何のために入れたんだっけ？」を減らせます。

## 定期点検すること

月1回、またはファームウェア更新前に、次を確認します。

```sh
echo "### storage"
df -h

echo "### installed packages count"
opkg list-installed | wc -l

echo "### important packages"
opkg list-installed | grep -Ei 'adblock|banip|wireguard|tailscale|vpn|ipoe|internet|assistant|luci-app' || true

echo "### services"
ls /etc/init.d | grep -Ei 'adblock|banip|tailscale|wireguard|vpn|ddns' || true

echo "### logs"
logread | grep -Ei 'opkg|adblock|banip|tailscale|wireguard|error|fail' | tail -n 120
```

見るポイントです。

- `/overlay` が埋まっていないか
- 使っていないパッケージが増えていないか
- 何のために入れたか分からないパッケージがないか
- ファームウェア更新前の一覧を保存しているか
- DNSやVPN系サービスが正常か

追加パッケージは、増えるほど管理対象も増えます。

使っていないものは、棚卸しして削除を検討します。

## やってはいけないこと

`opkg` 運用で避けたいことです。

- `opkg upgrade` を習慣的に実行する
- `/overlay` の空き容量を見ずに入れ続ける
- 出所不明の `.ipk` を入れる
- `kmod-*` の依存エラーを無理に回避する
- VPN / IPoE系を公式手順を見ずに手動で組む
- ファームウェア更新前にパッケージ一覧を保存しない
- 何のために入れたか分からないパッケージを放置する
- DNSやfirewall系パッケージを一度に複数入れる
- インストール後にログや状態を確認しない
- `curl | sh` を中身確認なしで実行する

`opkg` は強力です。

だからこそ、便利そうなものを全部入れるより、必要なものを少しずつ追加するほうが安全です。

## まとめ

LN6001-JPでは、`opkg` を使って機能を追加できます。

ただし、普通のPCと同じ感覚で大量に入れたり、全体更新したりするのは避けたほうが安全です。

ポイントは次の通りです。

1. パッケージ追加前に `df -h` で空き容量を見る
2. インストール前に `opkg update` を実行する
3. 必要なパッケージだけ入れる
4. `opkg upgrade` は基本的に使わない
5. `kmod-*` はカーネルバージョン依存に注意する
6. LuCIアプリ追加後はLuCIをリロードする
7. ファームウェア更新前に `opkg list-installed` を保存する
8. 更新後は必要なパッケージだけ再導入する
9. VPNやIPoEはLinksys公式モジュールを優先する
10. パッケージを入れた理由をメモしておく

最初は、`tcpdump` や `curl` のような用途が分かりやすいものから始めるのがおすすめです。

Adblock、VPN、IPoE、banIPのようにDNSやfirewall、回線に関係するものは、バックアップを取ってから一つずつ進めます。

`opkg` は、LN6001-JPを育てるための道具です。

でも、育てすぎてジャングルにしない。

これくらいの距離感がちょうどいいです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-03.png)

## 次に読むなら

パッケージ管理を理解したら、次の記事も合わせて読むと運用しやすくなります。

- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [ファームウェア更新運用](https://note.com/ikmsan/n/nff6b598da354)
- [DNS広告ブロックの始め方](https://note.com/ikmsan/n/n4759cf81d0e1)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)

ファームウェア更新後にパッケージが消える不安がある人は、ファームウェア更新運用へ。

AdblockやDNS系パッケージを入れたい人は、DNS広告ブロックの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LN6001-JPの現行ファームウェアでは `opkg` を前提にしていますが、OpenWrt本家の新しい系列では `apk` へ移行しているものがあります。

LuCIの画面名、`opkg` の出力、パッケージ名、依存関係、VPN Assistant、オートIPoE、Adblockなどの扱いは、ファームウェアやモジュール更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントを確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPでは好きなOpenWrtパッケージを何でも入れていい？

何でも無条件に入れるのはおすすめしません。

空き容量、依存関係、ファームウェアとの相性、公式モジュールの有無を確認してから進めます。

特に `kmod-*`、VPN、IPoE、firewall、DNS系は慎重に扱います。

### `opkg update` は何をしている？

パッケージ一覧を更新します。

インストール前に必要な操作です。

ただし、インストール済みパッケージを更新する `opkg upgrade` とは違います。

### `opkg upgrade` は使っていい？

基本的に使わない方針をおすすめします。

OpenWrt系では、ファームウェア全体の更新は公式ファームウェア更新で行い、追加パッケージは必要なものだけ再導入するほうが安全です。

### パッケージを入れる前に何を確認すればいい？

まず `df -h` で `/overlay` の空き容量を確認し、`opkg update` を実行します。

大きめの変更なら、`opkg list-installed` と `/etc/config/` の主要設定も保存します。

### WireGuardやTailscaleはopkgで入れるべき？

LN6001-JPでは、まずLinksys公式VPN Assistantモジュールを優先するのがおすすめです。

一般的なOpenWrtの手順で手動導入する前に、公式サポートの手順を確認してください。

### オートIPoEもopkgで入れる？

LN6001-JP向けには、Linksys公式のオートIPoE / Internet Assistantモジュールが案内されています。

NTT IPoE / IPv4 over IPv6系は回線方式が絡むため、まず公式手順を確認してください。

### ファームウェア更新後にパッケージは残る？

残らない場合があります。

更新前に `opkg list-installed` を保存し、更新後に必要なものだけ再導入します。

設定ファイルが残っていても、パッケージ本体がなければ機能しないことがあります。

### LuCIアプリを入れたのにメニューが出ない

ブラウザを更新し、LuCIへ再ログインします。

必要なら次を実行します。

```sh
/etc/init.d/uhttpd restart
```

それでも出ない場合は、パッケージ名やインストール結果、ログを確認します。

### `No space left on device` が出たら？

`/overlay` の空き容量が不足しています。

`df -h` と `du -h -d 1 /overlay` を確認し、不要パッケージを削除します。

容量不足のまま追加し続けないでください。

### OpenWrt本家でapkへ移行している話は関係ある？

LN6001-JPのこの記事では、firmware version 1.2.0.15 の `opkg` 前提で扱います。

ただし、OpenWrt本家の新しい系列では `apk` が使われる場合があります。

他のOpenWrt機器や将来のファームウェアを見る時は、その環境のパッケージ管理方式を確認してください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- OpenWrt Wiki - Opkg package manager: https://openwrt.org/docs/guide-user/additional-software/opkg
- OpenWrt Wiki - Packages: https://openwrt.org/packages/start
- OpenWrt Wiki - Upgrading packages warning: https://openwrt.org/meta/infobox/upgrade_packages_warning
- OpenWrt Wiki - Preserving OpenWrt packages: https://openwrt.org/docs/guide-user/installation/sysupgrade.packages
- OpenWrt Wiki - How do I install packages?: https://openwrt.org/faq/how_to_install_packages
- OpenWrt Wiki - apk package manager: https://openwrt.org/docs/guide-user/additional-software/apk

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
