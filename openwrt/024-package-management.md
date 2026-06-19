<!-- mirror-source: articles/024-package-management.md -->

> Original note.com article: [パッケージ管理: opkgでできることとできないこと【OpenWrt集中連載024】](https://note.com/ikmsan/n/n2bc5e8447e77)

# パッケージ管理: opkgでできることとできないこと【OpenWrt集中連載024】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPはOpenWrtベースのため、`opkg` パッケージマネージャーを使って機能を追加できます。

tcpdumpや一部のLuCI拡張などを追加できる一方で、ストレージ容量や依存関係、ファームウェア更新時の再インストールも意識する必要があります。

ただし、最初から大量のパッケージを管理しなくても大丈夫です。

まずは「空き容量を見る」「更新前に一覧を保存する」「よく分からないパッケージを入れすぎない」だけでも、かなり安全に運用しやすくなります。

WireGuardとTailscaleは、Linksys公式VPNアシスタントモジュールが案内されています。

この記事では、`opkg` の基本的な使い方、追加前に確認したいポイント、ファームウェア更新後に戻しやすくする考え方を、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-01.png)

## この記事でわかること

- LN6001-JPで `opkg` で入れてよいものと注意が必要なもの
- パッケージ追加前に確認したい空き容量
- ファームウェア更新後に戻しやすくする準備
- Linksys公式モジュールを優先したほうがよいケース

## こんな人に向いています

- 追加パッケージを入れてよいか不安
- `opkg` で何でも入れてよいのか判断に迷っている
- 更新後にパッケージが消える前提で整理しておきたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-02.png)

## まずはここまでで十分

パッケージ管理も、最初から複雑に考えなくて大丈夫です。

まずは次の3つだけ守ればかなり安全です。

1. `opkg update` を先に実行する
2. `df -h` で空き容量を見る
3. 更新前に `opkg list-installed` を保存する

追加パッケージは便利ですが、「入れる前の確認」を省かないことがかなり重要です。

最初は「便利そうだから全部入れる」より、「本当に必要なものだけ追加する」ほうが安全です。

`opkg` は、OpenWrt系で追加機能を管理するための基本ツールです。

最初は「インストール」「削除」「一覧確認」だけ覚えておけば十分です。

## 基本コマンド一覧

```sh
# まず本体情報と空き容量を確認
ubus call system board
df -h

# パッケージリストを更新（インストール前に必ず実行）
opkg update

# パッケージを検索
PACKAGE_KEYWORD="tcpdump"
opkg list | grep "$PACKAGE_KEYWORD"
opkg list | grep tcpdump

# パッケージをインストール
PACKAGE_NAME="tcpdump"
opkg install "$PACKAGE_NAME"

# パッケージをアンインストール
PACKAGE_NAME="tcpdump"
opkg remove "$PACKAGE_NAME"

# インストール済みパッケージの一覧
opkg list-installed

# パッケージの詳細情報
PACKAGE_NAME="tcpdump"
opkg info "$PACKAGE_NAME"

# パッケージが配置したファイルを確認
opkg files "$PACKAGE_NAME"

# ストレージ容量の確認
df -h
```

`opkg update` と `opkg list-installed` は、まず最初に覚えておきたいコマンドです。

OpenWrt系では、ストレージ容量不足が原因で不安定になるケースもあります。

## インストール前の確認

```sh
# ストレージ空き容量の確認
df -h

# 現在のパッケージ一覧を保存
BACKUP_DIR="/root/opkg-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
opkg list-upgradable > "$BACKUP_DIR/opkg-list-upgradable.txt" 2>/dev/null || true
ubus call system board > "$BACKUP_DIR/system-board.json"

# /overlayの使用量に特に注目
df -h | grep overlay
```

`/overlay` の空き容量が少ない（10MB未満など）場合は、不要パッケージをアンインストールしてから進めます。

最初は「容量に余裕があるか」を確認するだけでもかなり役立ちます。

最初は、「トラブル切り分けに役立つもの」や「普段の運用を楽にするもの」から追加するほうが整理しやすくなります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-01.png)

最初は、tcpdump のような「用途が明確で、インストール後に何を確認したいかが分かっているもの」から追加するほうが安全です。

WireGuard/Tailscaleは、Linksys公式サポートのVPNアシスタントモジュールを使います。

VPN系は、まず公式手順を優先したほうが切り分けしやすくなります。

最初は、LuCI対応パッケージを一緒に入れると状態確認しやすくなります。

## インストール例: tcpdump

```sh
# パッケージリスト更新
opkg update

# tcpdumpをインストール
opkg install tcpdump

# インストール確認
opkg list-installed | grep tcpdump
opkg files tcpdump | head

# 使い方確認
tcpdump --help | head
```

最初は「インストールできたか」「コマンドが起動するか」を確認できれば十分です。

OpenWrt系では、「設定は残るが追加パッケージ本体は消える」ケースがあります。

## インストール済みパッケージの保存（ファームウェアアップデート前に必須）

ファームウェアアップデート後は、opkgでインストールしたパッケージが消えることがあります（設定ファイルは残る場合あり）。

アップデート前に一覧を保存します:

```sh
# インストール済みパッケージ一覧を保存
opkg list-installed > /tmp/installed-packages-$(date +%Y%m%d-%H%M).txt

# PCにコピー（PC側のターミナルで実行）
scp root@192.168.1.1:/tmp/installed-packages-20260618-1200.txt ~/Downloads/
```

最初は「主要パッケージだけ再インストールできる状態」を残しておくだけでもかなり安心感が変わります。

ファームウェアアップデート後の再インストール:

```sh
opkg update
# 保存したリストを参照しながら主要パッケージを再インストール
opkg install tcpdump

# 戻したあとに確認
opkg list-installed | grep tcpdump
tcpdump --help | head
```

最初は「実際に追加したパッケージ」「VPN」「IPoE関連」が戻るかを優先確認すると整理しやすくなります。

`opkg` は便利ですが、「普通のLinux PC」と同じ感覚で大量導入すると整理しにくくなることがあります。

## opkgの注意事項

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/table-02.png)

最初は、「本当に必要なものだけ追加する」くらいで十分です。

```sh
# LuCIパネルの再読み込み
/etc/init.d/uhttpd restart
```

最初は「LuCIメニューへ反映されたか」を見るだけでもかなり役立ちます。

インストール失敗時は、「依存関係」「容量」「カーネル不一致」のどれかが原因のことがかなり多くあります。

## アンインストール前後の確認

不要になったパッケージを消す時も、いきなり削除するより「何を消すか」を先に見ます。

```sh
# 対象パッケージの情報と配置ファイルを確認
PACKAGE_NAME="tcpdump"
opkg info "$PACKAGE_NAME"
opkg files "$PACKAGE_NAME"

# 設定ファイルがある場合は先に保存
BACKUP_DIR="/root/remove-package-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp -a /etc/config "$BACKUP_DIR/etc-config"
opkg list-installed > "$BACKUP_DIR/opkg-before.txt"

# パッケージ削除
opkg remove "$PACKAGE_NAME"

# 削除後の確認
opkg list-installed | grep "$PACKAGE_NAME" || true
df -h
logread | tail -n 80
```

OpenWrtでは、設定ファイルだけ残る場合があります。

再インストールする可能性があるものは、設定を残しておくほうが安全です。

## パッケージがインストールできない場合

インストール前後の成功条件としては、次の3つが見えれば十分です。最初はここまで確認できれば大きく外していません。

- 空き容量に余裕がある
- 依存関係エラーなしでインストールできる
- LuCI拡張なら画面更新後にメニューへ出てくる

```sh
# エラー: Cannot satisfy dependencies
# → opkg updateを先に実行
PACKAGE_NAME="tcpdump"
opkg update
opkg install "$PACKAGE_NAME"

# エラー: kmod-xxx conflicts with installed kmod
# → カーネルバージョン不一致。ファームウェアバージョンを確認
ubus call system board | grep kernel

# エラー: No space left on device
# → 不要なパッケージをアンインストール
UNNEEDED_PACKAGE="example-package"
opkg remove "$UNNEEDED_PACKAGE"
df -h  # 空き容量再確認

# どのパッケージを最近入れたか確認
opkg list-installed | tail -n 50
```

最初は「容量不足か」「update不足か」を分けるだけでもかなり切り分けしやすくなります。

## まとめ

1. パッケージ追加前は `opkg update` を実行する
2. `df -h` でストレージ空き容量を確認する
3. tcpdump など用途が明確なものから追加する
4. ファームウェア更新前は `opkg list-installed` をPCへ保存する
5. WireGuard/TailscaleはLinksys公式VPNアシスタントモジュールを優先する
6. `luci-app-*` インストール後はLuCIを再読み込みする

`opkg` は、LN6001-JPへ機能を追加できる強力な仕組みです。

ただし、「何でも大量に入れる」より、「必要なものだけ追加する」ほうが整理しやすくなります。

最初は「空き容量確認」「一覧保存」「主要パッケージだけ導入」を意識するだけでも、かなり安全に運用しやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/024/diagram-03.png)

## よくある質問

### LN6001-JPでは好きなOpenWrtパッケージを何でも入れていい？

何でも無条件に入れるのはおすすめしません。

空き容量や依存関係、公式モジュールとの重複を確認してから進めるほうが安全です。

### LN6001-JPでパッケージを入れる前に何を確認すればいい？

`df -h` で空き容量を見て、`opkg update` を実行します。

必要なら現在のインストール済み一覧も保存しておくと戻しやすくなります。

### LN6001-JPでWireGuardやTailscaleも `opkg` で入れるべき？

この連載ではLinksys公式VPNアシスタントモジュールを優先しています。

一般パッケージより、まず公式手順を入口にしたほうが安全です。

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
