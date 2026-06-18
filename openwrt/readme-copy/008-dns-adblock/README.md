<!-- mirror-source: articles/008-dns-adblock.md -->

> Original note.com article: [DNS広告ブロック: LN6001-JPにAdblockを入れて設定する【OpenWrt集中連載008】](https://note.com/ikmsan/n/n4759cf81d0e1)

# DNS広告ブロック: LN6001-JPにAdblockを入れて設定する【OpenWrt集中連載008】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

DNS広告ブロックは、端末ごとに広告ブロックアプリを入れるのではなく、ルーター側のDNS名前解決で不要な通信先を減らす考え方です。

LN6001-JPはOpenWrtベースなので、`luci-app-adblock` を使ってLuCIから比較的簡単に設定できます。

ただし、最初から大量のブロックリストを入れすぎると、誤ブロックでWebサイトやアプリが正常に動かなくなることがあります。まずは少ないリストから始め、必要に応じて広げていくほうが扱いやすいです。

また、DNS広告ブロックは「すべての広告を完全に消す機能」ではありません。アプリ内広告やDoH（DNS over HTTPS）を使う端末には効きにくいこともあります。

この記事では、Adblockパッケージの導入、基本設定、誤ブロック対応、DoH対策までを、家庭・小規模オフィス向けに順番に整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-01.png)

## この記事でわかること

- LN6001-JPでDNS広告ブロックを導入する流れ
- Adblockでできることと限界
- 誤ブロックが起きた時の戻し方
- 子ども用・IoT用ネットワークへ段階的に適用する考え方

## こんな人に向いています

- 家庭内の端末へまとめて広告ブロックを効かせたい
- 子ども用やIoT用ネットワークへ段階的に入れたい
- Adblockの効果と限界を理解したうえで使いたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-02.png)

## まずはここまでで十分

Adblockも最初から全機能を有効にする必要はありません。むしろ、最初はシンプルな構成のほうが誤ブロックを切り分けやすくなります。

まずは次の3つで十分です。

1. バックアップを取る
2. `adblock` と `luci-app-adblock` を入れる
3. 少ないブロックリストで試す

最初は「広告を全部消す」より、「誤ブロックなく安定して使えるか」を見ながら、適用範囲をゆっくり広げるほうが安全です。

## DNS広告ブロックの仕組みと限界

DNS広告ブロックは、広告・トラッキングに使われるドメイン名をルーター側でブロックします。

端末ごとにアプリを入れる方法とは異なり、家庭内の全端末へまとめて適用できるところが特徴です。

**できること:**
- 一部の広告・トラッキングドメインを名前解決で抑える
- IoT機器や子ども用端末の通信先を整理しやすくする
- 端末ごとの設定負担を減らす

**できないこと（限界）:**
- すべての広告を消せるわけではない（同一ドメインからの広告は効かない）
- アプリ内広告には効きにくい
- 端末側がDoH（DNS over HTTPS）を使うとルーターDNSを迂回される
- 誤ブロックが起きることがある

つまり、DNS広告ブロックは「万能な広告除去機能」というより、「不要な通信先を減らしてネットワークを整理する仕組み」と考えると分かりやすいです。

## 導入前に確認すること

DNS設定の変更は、業務用ソフトウェアや決済システムに影響する可能性があります。

業務端末が含まれるネットワークへの適用は、まずテスト用SSIDやGuestネットワークで試してから広げるほうが安全です。

いきなり全端末へ適用するより、まず家族用またはIoT用ネットワークで試してから広げるほうが、問題を切り分けしやすくなります。

## 導入前の状態確認とバックアップ

AdblockはDNS、DHCP、firewall、パッケージ状態に関係します。導入前に、現在の状態を保存しておきます。

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

ls -l "$BACKUP_DIR"
```

パッケージを追加する前に空き容量も確認します。

```sh
df -h
opkg update
opkg list | grep -E '^adblock|^luci-app-adblock|^banip|^luci-app-banip'
```

## セットアップが難しい場合は導入スクリプトを使う

Adblockを手動で入れて、ブロックリスト、Force Local DNS、DNS Report、DoH対策まで自分で整えるのが大変なら、以下のGitHubで公開している広告ブロック導入スクリプトを使う方法があります。

最初から全部を手で組むより、まず動く状態を作ってから中身を理解していくほうが入りやすい人も多いです。

- スクリプト: https://github.com/ikm-san/velop
- 内容: AdblockとLuCI用パッケージの導入、TOFUフィルターの追加、LAN向けDNS設定、Force Local DNS、自動更新cronなどをまとめて設定する
- 効果の目安: ブラウザ上の広告表示を大きく減らし、ルーター配下のスマホやPC全体に効かせやすくする
- 向く場面: 家庭や小規模オフィスで広告を減らしたい場合、子ども用端末のノイズを少し抑えたい場合

実行例:

```sh
curl -sS -o /tmp/adb_setup.sh https://raw.githubusercontent.com/ikm-san/velop/main/adb_setup.sh && sh /tmp/adb_setup.sh -v
```

curl実行前に、GitHub上のREADMEやスクリプト内容を確認してから進めることをおすすめします。

スクリプトはルーター設定を変更し、最後に再起動確認も行います。

広告ブロックの効果はサイトやアプリの作りによって変わるため、実行前にLuCIのバックアップを取得し、READMEとスクリプト内容を確認してください。

業務端末や決済端末が同じネットワークにある場合は、いきなり全体へ適用せず、影響範囲を確認してから使います。

## Adblockパッケージのインストール

LN6001-JPでは、LuCIからパッケージを追加できます。

最初はCLIより、LuCIのSoftware画面から進めたほうが確認しやすいです。

### LuCIでインストール

1. **System** → **Software** を開く
2. **Update lists** をクリックしてパッケージリストを更新する
3. 検索ボックスに `adblock` と入力
4. `luci-app-adblock` を見つけて **Install** をクリック
5. `adblock` も見つけて **Install** をクリック（依存関係として自動インストールされる場合もある）
6. LuCIページを更新する（F5）

インストール直後は少し時間がかかることがあります。画面が更新されるまで数十秒待つと反映される場合があります。

### CLIでインストール

```sh
echo "### update package lists"
opkg update

echo "### install adblock packages"
opkg install adblock luci-app-adblock

echo "### verify installed packages"
opkg list-installed | grep adblock
```

CLIの `opkg install` は便利ですが、最初はLuCIでパッケージ名を確認しながら進めるほうが分かりやすいです。

インストール後、LuCIメニューの **Services** に **Adblock** が追加されます。

## Adblockの基本設定

最初は「少ないブロックリストで安定動作するか」を優先すると、あとから調整しやすくなります。

### LuCIからAdblockを設定する

1. **Services** → **Adblock** を開く
2. **Settings** タブの主要設定:

![表 01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/table-01.png)

3. **Blocklist Sources** タブ:
   - 利用するブロックリストにチェックを入れる
   - 推奨の初期設定:
     - `adguard`: 広告・トラッキング
     - `disconnect`: トラッキング中心
     - `hagezi` または `yoyo`: 広告中心（日本語サイトにも比較的有効）
   - 最初は2〜3リストから始め、誤ブロックが出たら調整する

最初から大量のリストを有効化すると、誤ブロック時の原因切り分けが難しくなります。まずは2〜3個から始めるほうが現実的です。

4. **Save & Apply** をクリック
5. **Action** タブで **Reload** をクリックしてブロックリストを適用

### CLIで確認・操作

```sh
echo "### Adblock status"
/etc/init.d/adblock status

echo "### Reload Adblock"
/etc/init.d/adblock reload

echo "### Blocklist files"
ls -l /tmp/adblock 2>/dev/null || true
wc -l /tmp/adblock/adb_list* 2>/dev/null || true

echo "### DNS service"
/etc/init.d/dnsmasq status

echo "### Recent Adblock logs"
logread | grep -Ei 'adblock|dnsmasq' | tail -n 80
```

`/etc/init.d/adblock status` と `logread | grep -i adblock` は、まず最初に確認したいコマンドです。

CLIは、最初は設定変更より「今どう動いているか」を確認する用途で使うくらいで十分です。

## ホワイトリスト（許可リスト）の設定

誤ブロックが発生した場合は、特定のドメインをホワイトリストへ追加します。

広告ブロック運用では、「必要なものだけ戻せる」ことがかなり重要です。

### LuCIでホワイトリストを追加

1. **Services** → **Adblock** → **Allowlist** タブを開く
2. テキストボックスに許可するドメインを1行1ドメインで入力:

```
example.com
api.business-service.jp
payment-gateway.com
```

3. **Save & Apply** をクリック
4. **Action** → **Reload** でリストを再適用

### UCIでホワイトリストを編集

```sh
# ホワイトリストファイルを直接編集
vi /etc/adblock/adblock.whitelist

# 追記する場合
echo "example.com" >> /etc/adblock/adblock.whitelist

# Adblockをリロード
/etc/init.d/adblock reload
```

最初はLuCIから追加したほうが分かりやすいですが、複数台を管理する場合はCLI管理のほうが整理しやすくなることがあります。

## DNSを端末グループ別に設定する

Guest Wi‑FiやVLANを使っている場合は、ネットワークごとにDNS方針を変えることもできます。

例えば、家族用は通常DNS、子ども用はFamily DNS、IoT用は広告ブロックDNSという分け方も可能です。

### 子ども用ネットワークにフィルタリングDNSを設定する

フィルタリングDNS（例: Cloudflare 1.1.1.3、Google SafeSearch）を子ども用SSIDのDHCPで配布します:

1. **Network** → **Interfaces** で子ども用インターフェース（例: `kids`）の **Edit** をクリック
2. **DHCP Server** → **Advanced Settings** タブを開く
3. **DHCP Options** に以下を追加:
   - `6,1.1.1.3,1.0.0.3`（Cloudflare Family DNS）
   - または `6,8.8.8.8,8.8.4.4`（Google DNS）
4. **Save & Apply**

### CLIでDHCPのDNS配布を設定

```sh
echo "### backup DHCP config"
cp /etc/config/dhcp /etc/config/dhcp.backup.$(date +%Y%m%d-%H%M)

echo "### add DNS option for kids"
uci add_list dhcp.kids.dhcp_option='6,1.1.1.3,1.0.0.3'
uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart
```

DNS設定を変更したあと、端末側でWi‑Fiを一度切断・再接続すると、新しいDNS設定を受け取りやすくなります。

## DoH（DNS over HTTPS）への対応

端末側がDoHを使うと、ルーターのDNS設定を迂回します。

主にChromeブラウザやFirefoxがDoHを自動的に使う場合があります。

**確認方法:**
ブラウザのDNS設定でDoHが有効になっているか確認します。
- Chrome: 設定 → プライバシーとセキュリティ → セキュリティ → セキュアDNSを使用する
- Firefox: 設定 → 一般 → ネットワーク設定 → DNS over HTTPS

**対応策（一例）:**
Linksys公式手順では、DoH迂回への対策例として `luci-app-banip` を追加し、banIP側でDoHブロックを有効化する流れが案内されています。家庭内で簡単に運用するなら、まず端末側のDoHを無効化し、必要に応じてbanIPを使う順番が現実的です。

まずは「端末側DoHを無効化する」くらいから始め、必要に応じてbanIPを追加するほうが運用しやすいです。

## 誤ブロックへの対応フロー

広告ブロック運用で一番大事なのは、「問題が起きた時にすぐ戻せること」です。

そのため、まず一時停止 → 原因確認 → ホワイトリスト追加、の順で切り分けると分かりやすくなります。

1. **Adblockを一時無効化して確認する**:
   - LuCI: Services → Adblock → Settings → Enabled のチェックを外す → Save & Apply
   - CLI: `/etc/init.d/adblock stop`
   - 一時無効化で問題が解決すれば、Adblock側が原因と切り分けやすくなる

2. **どのドメインがブロックされているか確認する**:

```sh
echo "### Check blocked domain in runtime lists"
grep "domain-to-check.com" /tmp/adblock/adb_list*

echo "### Recent DNS/Adblock logs"
logread | grep -Ei 'adblock|dnsmasq|domain-to-check' | tail -n 80
```

3. **ホワイトリストに追加する**（上記の手順参照）

4. **ブロックリストを見直す**:
   - Services → Adblock → Blocklist Sources でリストを減らす

## よくある誤ブロックの例

- ログイン画面が開かない（認証サービスのドメインがブロックされている）
- アプリ内の画像が表示されない（CDNドメインがブロックされている）
- 決済画面が止まる（決済サービスのドメインがブロックされている）
- スマート家電のクラウド連携が不安定になる（IoTクラウドドメインがブロックされている）

特に決済・認証・CDN系ドメインは誤ブロック時の影響が大きいため、店舗や業務用途では慎重に確認したほうが安心です。

## 設定変更前のバックアップ

```sh
BACKUP_DIR="/root/dns-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in dhcp firewall network; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

opkg list-installed | grep -Ei 'adblock|banip|dnsmasq' > "$BACKUP_DIR/dns-packages.txt" || true

ls -l "$BACKUP_DIR"
```

DNSやfirewall周辺は、あとからGuest Wi‑FiやVLAN設定とも関係してきます。変更前バックアップを残しておくと、切り分けがかなり楽になります。

## まとめ

Adblockは「広告が全部消える機能」ではなく、DNS名前解決の段階で不要な通信先を減らすツールです。

期待値を合わせた上で、まずは少ないブロックリストから始め、段階的に適用範囲を広げるほうが運用しやすくなります。

設定後は、次の3つが確認できれば十分です。最初から完璧な広告ブロックを目指す必要はありません。

- インターネット自体は普段どおり使える
- 体感できる範囲で広告や不要通信が減っている
- 誤ブロック時に一時停止やホワイトリスト追加で戻せる

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/008/diagram-03.png)

## よくある質問

### LN6001-JPで広告ブロックはできる？

できます。`adblock` と `luci-app-adblock` を入れると、LuCIからDNS広告ブロックを管理できます。

### DNS広告ブロックで広告は全部消える？

消えません。同一ドメイン配信の広告やアプリ内広告には効きにくく、DoHを使われると迂回されることもあります。

「全部消す」より、「不要な通信を減らして快適にする」くらいの期待値で使うほうが現実的です。

### 誤ブロックが起きたらどうする？

まずAdblockを一時停止して原因を切り分け、その後でホワイトリストへ必要なドメインを追加する流れが安全です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT Adblock setup: https://support.linksys.com/kb/article/6711-en/?section_id=175
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
