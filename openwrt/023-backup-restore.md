<!-- mirror-source: articles/023-backup-restore.md -->

> Original note.com article: [バックアップと復元: 設定を守る習慣【OpenWrt集中連載023】](https://note.com/ikmsan/n/n3c25190a8e94)

# バックアップと復元: 設定を守る習慣【OpenWrt集中連載023】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPはOpenWrtベースで設定自由度が高いぶん、設定変更やアップデート前のバックアップがかなり重要です。

特にIPoE、VLAN、VPN、DHCP予約などを触り始めると、「どこを戻せば復旧できるか」が分かるだけでも安心感がかなり変わります。

ただし、最初から完璧なバックアップ運用を作らなくても大丈夫です。

まずは「LuCIでバックアップを取る」「PCへ保存する」「重要変更前だけ個別コピーする」くらいでも、かなり復旧しやすくなります。

この記事では、LN6001-JPでバックアップに含まれるもの・含まれないもの、LuCIとSSHでの保存方法、復元後に確認したいポイントを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-01.png)

## この記事でわかること

- LN6001-JPバックアップに含まれるもの・含まれないもの
- LuCIとSSHでバックアップを取る流れ
- 個別ファイルだけ戻す考え方
- 復元後にどこを確認すると切り分けしやすいか

## こんな人に向いています

- 設定変更で壊した時に戻せるようにしておきたい
- ファームウェア更新前に何を保存すべきか知りたい
- LuCIとSSHのどちらでバックアップすべきか迷っている

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-02.png)

## まずはここまでで十分

バックアップも、最初から完璧に管理しなくて大丈夫です。

まずは次の3つを押さえれば十分です。

1. LuCI でフルバックアップを取ってPCへ保存する
2. 重要な変更前だけ個別ファイルも控える
3. `opkg list-installed` も別で保存する

バックアップ本体とパッケージ一覧が分かれているだけでも、復旧しやすさはかなり変わります。

最初は「全部を自動化する」より、「戻せる状態を残す」ことを優先するほうが安全です。

OpenWrt系では、「設定」と「追加パッケージ」が別管理になっている点がかなり重要です。

## バックアップに含まれるもの・含まれないもの

| 含まれるもの | 含まれないもの |
|---|---|
| `/etc/config/` 以下の全設定ファイル | opkgでインストールしたパッケージのバイナリ |
| ホスト名・パスワード・タイムゾーン | `/tmp/` ディレクトリ以下（揮発性） |
| ネットワーク・Wi-Fi・firewall設定 | ファームウェア本体 |
| DHCP予約・DNS設定 | adblockのブロックリストキャッシュ |
| VPN設定（WireGuard鍵含む） | ログファイル |

最初は、「設定は戻るが、追加パッケージ本体は別」という整理だけでもかなり役立ちます。

> バックアップ復元後に、opkgでインストールしたパッケージ（adblock等）は再インストールが必要になることがあります。インストール済みパッケージ一覧も合わせて保存しておくと整理しやすくなります。

---

## LuCIからバックアップを取る

最初はLuCIバックアップだけでも、かなり復旧しやすさが変わります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Backup** セクションの **Generate archive** ボタンをクリック
3. `backup-openwrt-YYYY-MM-DD.tar.gz` という名前でPCにダウンロードされる

最初は「設定変更前だけバックアップする」くらいでも十分役立ちます。

Linksys公式手順でも、バックアップはLuCIの **System** → **Backup / Flash Firmware** から **Generate archive**、復元は **Upload archive** を使う流れです。

CLIは、この公式UI手順を補完して「何が入っているか」「追加パッケージは別に控えたか」を確認する用途で使います。

> **推奨タイミング**: 設定変更前・設定変更後・ファームウェアアップデート前
>
> 特にfirewall、VLAN、IPoE、VPN設定変更前はバックアップを残しておくと安心です。

---

## SSHからバックアップを取る

SSHバックアップは、「今の状態を丸ごと保存したい」時に便利です。

CLIは、最初は設定変更より「バックアップを残す」用途で使うくらいでも十分です。

```sh
# LN6001-JPにSSH接続後、バックアップを作成
BACKUP_NAME="/tmp/backup-ln6001-$(date +%Y%m%d-%H%M).tar.gz"
sysupgrade -b "$BACKUP_NAME"

# バックアップ対象ファイル一覧を確認
sysupgrade -l > /tmp/sysupgrade-file-list-$(date +%Y%m%d-%H%M).txt

# 破損確認用のハッシュを作成
sha256sum "$BACKUP_NAME" > "$BACKUP_NAME.sha256"

# PCにバックアップファイルをコピー（PC側のターミナルで実行）
scp root@192.168.1.1:/tmp/backup-ln6001-20260618-1200.tar.gz ~/Downloads/
scp root@192.168.1.1:/tmp/backup-ln6001-20260618-1200.tar.gz.sha256 ~/Downloads/
```

`sysupgrade -b` は、OpenWrt系でまず覚えておきたいバックアップコマンドです。

または、個別設定ファイルだけバックアップする:

```sh
# 設定変更前にまとめてコピー（日付付きバックアップ）
BACKUP_DIR="/root/config-backup-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp system; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

# バックアップ一覧確認
ls -l "$BACKUP_DIR"
```

最初は「network」「wireless」「firewall」「dhcp」だけでも個別保存しておくと、かなり戻しやすくなります。

---

## インストール済みパッケージのリストを保存する

追加パッケージは、バックアップ復元後に消えている場合があります。

ファームウェアアップデート後はパッケージが消えるため、事前にリストを保存します:

```sh
# インストール済みパッケージ一覧をファイルに出力
opkg list-installed > /tmp/installed-packages-$(date +%Y%m%d-%H%M).txt

# Linksys公式/追加モジュールだけざっくり抽出
opkg list-installed | grep -Ei 'adblock|wireguard|tailscale|vpn|ipoe|map|dslite|ipip' > /tmp/important-packages.txt

# PCにコピー
scp root@192.168.1.1:/tmp/installed-packages-20260618-1200.txt ~/Downloads/
scp root@192.168.1.1:/tmp/important-packages.txt ~/Downloads/
```

最初は「Adblock」「VPN」「IPoE関連」が残っているか確認できれば十分です。

---

## LuCIからバックアップを復元する

バックアップがある場合は、「全部再設定」より復元のほうがかなり速いことがあります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Restore backup** セクションの **Browse** をクリック
3. 保存済みの `.tar.gz` ファイルを選択して **Upload archive** をクリック
4. 確認ダイアログで **Proceed** をクリック
5. ルーターが再起動し、バックアップ時点の設定が復元される

最初は「LuCIへ戻れるか」を確認できるだけでもかなり重要です。

---

## SSHからバックアップを復元する

SSHへ入れるなら、まだかなり柔軟に復旧できます。

```sh
# PCからバックアップファイルをルーターに転送（PC側のターミナルで実行）
scp backup-ln6001-20260618-1200.tar.gz root@192.168.1.1:/tmp/

# SSHでルーターにログインして復元
ssh root@192.168.1.1
sysupgrade -r /tmp/backup-ln6001-20260618-1200.tar.gz
```

`sysupgrade -r` は、バックアップ復元時にまず覚えておきたいコマンドです。

個別設定ファイルのロールバック:

```sh
# networkだけ戻す
cp /root/config-backup-20260618-1200/network /etc/config/network
/etc/init.d/network restart

# wirelessだけ戻す
cp /root/config-backup-20260618-1200/wireless /etc/config/wireless
wifi reload

# firewallだけ戻す
cp /root/config-backup-20260618-1200/firewall /etc/config/firewall
/etc/init.d/firewall restart

# 復元後の確認
ifstatus wan 2>/dev/null || true
wifi status 2>/dev/null || true
uci show firewall | head
logread | tail -n 80
```

最初は「networkだけ」「firewallだけ」のように、直前に変更した部分だけ戻すほうが切り分けしやすくなります。

---

## バックアップのタイミング・管理方法

バックアップは、「定期的」より「変更前後」で残すほうが実運用ではかなり役立ちます。

| タイミング | 推奨アクション |
|---|---|
| 初期セットアップ完了時 | 必ずバックアップを取る |
| 設定変更前 | 変更するファイルを個別バックアップ |
| 設定変更後（動作確認済み） | LuCIバックアップを取ってPCに保存 |
| ファームウェアアップデート前 | 必ずフルバックアップを取る |
| VPN鍵を生成した直後 | バックアップを取る（鍵の再生成が面倒なため） |

最初は、「今安定している状態」を1つ残すだけでもかなり安心感が変わります。

PCの `~/Downloads/router-backup/` などへ日付付きで整理しておくと、過去設定へ戻しやすくなります。

ルーター本体だけへ保存していると、本体故障時に取り出せなくなることがあります。

---

## よくある失敗

復元時は、「バックアップは戻ったが、周辺条件が変わっていた」というケースもあります。

復元後の最初の確認は、次の3つで十分です。最初はここまで確認できれば大きく外していません。

- LuCIへ再ログインできる
- WANとWi‑Fiが普段どおり使える
- 追加パッケージ不足を把握できている

| 症状 | 原因 | 対処 |
|---|---|---|
| 復元後にインターネットに接続できない | バックアップが回線設定変更前のもの | WANの設定を手動で再設定 |
| 復元後にadblockが動かない | パッケージは復元されない | `opkg install` で再インストール |
| バックアップファイルが見つからない | 保存先を忘れた | PCの「ダウンロード」フォルダを探す |
| 復元後にWi-Fiに接続できない | SSIDやパスワードが以前の設定に戻った | 以前のSSID/パスワードで再接続 |

最初は「インターネットへ戻れるか」を優先確認すると整理しやすくなります。

復元直後は、次の確認をまとめて行うと抜け漏れを見つけやすくなります。

```sh
ubus call system board
ifstatus wan
ifstatus wan6
wifi status
opkg list-installed | grep -Ei 'adblock|wireguard|tailscale|vpn|ipoe|map|dslite|ipip' || true
logread | tail -n 100
```

---

## まとめ

1. LuCIまたは `sysupgrade -b` でバックアップを作成する
2. バックアップは必ずPC側にも保存する
3. `opkg list-installed` で追加パッケージ一覧も保存する
4. 重要変更前は `network` `wireless` `firewall` などを個別コピーする
5. 復元後はWAN・Wi‑Fi・普段使う機能から確認する
6. 必要なら `sysupgrade -r` や個別ファイルコピーで戻す

最初から完璧なバックアップ運用を作る必要はありません。

「変更前に1本バックアップを取る」だけでも、設定変更への安心感はかなり変わります。

特にIPoE、VLAN、VPN、firewall設定を触る前は、バックアップを残しておくと復旧がかなり楽になります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/023/diagram-03.png)

## よくある質問

### LN6001-JPのバックアップには何が入る？

主に `/etc/config/` 以下の設定が入ります。

ファームウェア本体や追加パッケージ実体は別で考える必要があります。

### LN6001-JPのバックアップはいつ取るべき？

設定変更の前後と、ファームウェア更新前です。

特にfirewall、VLAN、IPoE、VPN変更前はかなり重要です。

### LN6001-JPで復元したらすぐ元通りになる？

基本設定は戻しやすいですが、追加パッケージは再インストールが必要なことがあります。

`opkg list-installed` を別保存しておくと安心です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys OpenWRT backup and restore: https://support.linksys.com/kb/article/218-en/?section_id=175

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
