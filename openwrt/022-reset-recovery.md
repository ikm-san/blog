<!-- mirror-source: articles/022-reset-recovery.md -->

# リセットと復旧: 初期化前に確認すること【OpenWrt集中連載022】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

設定変更後に「管理画面へ入れない」「Wi‑Fiが消えた」「ネットにつながらない」となっても、すぐ初期化する必要はありません。

LN6001-JPでは、有線LAN・LuCI・SSHを使って設定を戻したり、バックアップから復元できるケースがあります。

ただし、最初から全部を深く調べなくても大丈夫です。

まずは「有線で入れるか」「最後に何を変えたか」「バックアップがあるか」を確認するだけでも、かなり復旧しやすくなります。

この記事では、ソフトリセットとハードリセットの違い、SSHから戻す方法、バックアップ復元の流れ、リセット前に確認したいポイントを、家庭・小規模オフィス向けに整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-01.png)

## この記事でわかること

- リセット前に試したい復旧手順
- ソフトリセットとハードリセットの違い
- SSHやバックアップから戻す考え方
- 「壊れた」と思っても実は戻せるケース

## こんな時に向いています

- 設定変更後にLuCIへ入れなくなって不安
- いきなりリセットしてよいか迷っている
- 初期化せず戻せる可能性を先に確認したい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-02.png)

## まずはここまでで十分

復旧時も、最初から全部試す必要はありません。

まずは次の3つで十分です。

1. 有線LANでLuCIへ入れるか確認する
2. 最後に変更した設定を思い出す
3. バックアップから戻せないか確認する

「すぐ初期化する」より、「まだ戻せる余地があるか」を先に確認するほうが安全です。

特にLAN IP変更やfirewall設定変更直後は、実はルーター自体は壊れていないケースもかなり多くあります。

OpenWrt系では、「初期化しかない」と思われがちですが、設定ファイル単位で戻せるケースもあります。

## リセットの種類

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-01.png)

最初は、「全部初期化」より「直前に触った設定だけ戻せないか」を考えるほうが整理しやすくなります。

---

## リセット前の確認チェックリスト

実際には、「設定は生きているがアクセス先IPが変わっていた」だけのケースもあります。

初期化する前に以下を確認します:

- [ ] LANケーブルでPCとLN6001-JPのLANポートを接続できるか
- [ ] `192.168.1.1`（または変更後のLAN IP）でLuCIにアクセスできるか
- [ ] 直前に変更した設定（LAN IP、firewall、VLAN等）が何かを把握しているか
- [ ] LuCIまたはSSHのバックアップファイルはあるか
- [ ] opkgでインストールしたパッケージのリストを保存してあるか

これらのうち一つでも手がかりがあれば、初期化せず復旧できる可能性があります。

最初は「最後に何を変えたか」を思い出すだけでもかなり役立ちます。

SSHへ入れる場合は、初期化前に今の状態だけでも保存します。

```sh
BACKUP_DIR="/root/recovery-before-reset-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

ubus call system board > "$BACKUP_DIR/system-board.json" 2>/dev/null || true
uptime > "$BACKUP_DIR/uptime.txt"
df -h > "$BACKUP_DIR/df-h.txt"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/ip-route.txt"
ip -6 route show > "$BACKUP_DIR/ip6-route.txt"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt" 2>/dev/null || true
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

sysupgrade -b "$BACKUP_DIR/config-backup.tar.gz"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"

echo "saved: $BACKUP_DIR"
```

このバックアップはルーター本体内に置かれるため、本当に初期化する前にはPCへコピーします。

```sh
# PC側のターミナルで実行
scp -r root@192.168.1.1:/root/recovery-before-reset-* ~/Downloads/
```

---

## ソフトリセット（LuCIから初期化）

LuCIへ入れるなら、まだかなり復旧しやすい状態です。

LuCIにアクセスできる場合:

1. **System** → **Backup / Flash Firmware** を開く
2. **Perform reset** ボタンをクリック
3. 確認ダイアログで **OK** をクリック
4. ルーターが再起動し、工場出荷設定に戻る（約2分待つ）

最初は「バックアップを取ってから初期化する」だけでもかなり安全性が変わります。

初期化後は次が工場出荷状態へ戻ります:
- 管理者パスワード: 本体底面ラベルの初期パスワードに戻る
- LAN IP: `192.168.1.1`
- Wi-Fi SSID/パスワード: 工場出荷時のもの（本体ラベルに記載）

---

## ハードリセット（リセットボタン）

LuCIへ入れない場合でも、有線LANやSSHがまだ使えるケースがあります。

物理リセットは最後の手段として考えるほうが安全です。

LuCIにアクセスできない場合（管理パスワード忘れ等）:

1. LN6001-JPの電源が入った状態で待機
2. 本体裏面または底面の **RESET** ボタンを確認
3. クリップや細いピンを使い、リセットボタンを約10秒間押し続ける
4. LEDが赤点灯から青点滅へ変わり、再起動していることを確認する
5. 再起動後、インターネット接続を検知すると白点灯になるまで待つ
6. 工場出荷設定で起動完了

最初は「本当にLuCIもSSHも入れないか」を確認してから実施するほうが整理しやすくなります。

> **注意**: ハードリセット後は全設定が消えます。事前にLuCIバックアップを取っておくことを強くおすすめします。

Linksys公式手順でも、MBE70/LN6001系のリセットは本体底面のリセットボタンを約10秒押し、LEDが赤点灯から青点滅へ変わる流れで案内されています。

初期化後にインターネットを検知すると白点灯になります。赤点灯のままなら、初期化ではなくWAN配線や回線設定を確認します。

---

## SSHからのリカバリー（firstbootコマンド）

SSHへ入れるなら、まだかなり多くの復旧手段が残っています。

```sh
# 工場出荷設定に戻すコマンド（実行すると全設定が消える）
sysupgrade -b /tmp/before-firstboot-$(date +%Y%m%d-%H%M).tar.gz
opkg list-installed > /tmp/opkg-before-firstboot-$(date +%Y%m%d-%H%M).txt

firstboot -y
reboot now
```

`firstboot` は、OpenWrt系でまず覚えておきたい復旧コマンドです。

ただし、実行すると設定は消えるため、最初はバックアップ確認を優先します。

管理パスワードを覚えていてSSHに入れる場合は、特定の設定ファイルだけ戻すこともできます:

```sh
# 直前に変更したファイルのバックアップを確認
ls /etc/config/*.backup.*

# 戻す前に現在のファイルも保存
cp /etc/config/network /etc/config/network.before-rollback.$(date +%Y%m%d-%H%M)
cp /etc/config/firewall /etc/config/firewall.before-rollback.$(date +%Y%m%d-%H%M)

# バックアップから特定の設定を復元
cp /etc/config/network.backup.20260618 /etc/config/network
/etc/init.d/network restart

# firewallだけ戻す
cp /etc/config/firewall.backup.20260618 /etc/config/firewall
/etc/init.d/firewall restart

# 戻した後の確認
ifstatus lan 2>/dev/null || true
ifstatus wan 2>/dev/null || true
logread | tail -n 80
```

最初は「networkだけ」「firewallだけ」のように、直前に変更した設定だけ戻すほうが切り分けしやすくなります。

---

## LuCIバックアップからの復元

バックアップがある場合は、「全部再設定」より復元のほうがかなり速いことがあります。

バックアップファイル（`.tar.gz`）がある場合:

1. **System** → **Backup / Flash Firmware** を開く
2. **Restore backup** セクションで **Browse** をクリック
3. バックアップファイル（`backup-openwrt-*.tar.gz`）を選択して **Upload archive** をクリック
4. 確認後 **Proceed** をクリック
5. ルーターが再起動し、バックアップ時点の設定が復元される

最初は「LuCIへ戻れるか」を確認できるだけでもかなり重要です。

SSHから復元する場合:

```sh
# PCからバックアップファイルをルーターにコピー
scp backup-openwrt-20260507.tar.gz root@192.168.1.1:/tmp/

# SSHでルーターにログインして復元
ssh root@192.168.1.1
sysupgrade -r /tmp/backup-openwrt-20260507.tar.gz
```

`sysupgrade -r` は、バックアップ復元時にまず覚えておきたいコマンドです。

---

## よくある「壊れていないのに入れない」ケース

実際には、「壊れた」のではなく、「アクセス先や権限が変わっただけ」のケースもかなり多くあります。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/table-02.png)

```sh
# LuCIのWebサーバー（uhttpd）を再起動
/etc/init.d/uhttpd restart

# uhttpdとSSHの状態確認
/etc/init.d/uhttpd status
/etc/init.d/dropbear status

# ルーター自身のLAN IPを確認
uci show network.lan
```

最初は「LuCI自体が止まっていないか」を見るだけでもかなり役立ちます。

---

## 初期化後の再設定チェックリスト

初期化前の成功条件としては、次の3つを見れば十分です。最初はここまで確認できれば大きく外していません。

- LuCIかSSHのどちらかへまだ入れる
- 直前に変更した設定の見当がついている
- バックアップか再設定材料が残っている

ハードリセット後は、「普段使う機能から順番に戻す」ほうが整理しやすくなります。

ハードリセット後に必要な再設定項目:

1. [ ] 管理者パスワードを設定
2. [ ] ホスト名・タイムゾーンを設定（Asia/Tokyo）
3. [ ] SSIDとWi-Fiパスワードを設定
4. [ ] WANの回線種別を設定（IPoE/PPPoE等）
5. [ ] LAN IPアドレスを設定（HGW配下の場合は変更必須）
6. [ ] ゲストWi-Fi・Kids/IoTネットワークを再作成
7. [ ] DHCP固定割り当てを再登録
8. [ ] VPN（WireGuard/Tailscale）を再設定
9. [ ] opkgでインストールしたパッケージを再インストール（adblock等）
10. [ ] IPoEモジュールを再インストール（必要な場合）

最初は「WAN」「Wi‑Fi」「普段使う端末」が戻ることを優先確認すると整理しやすくなります。

設定の詳細は各記事（Article 004: 初期セットアップ、Article 007: ゲストWi-Fi等）を参照してください。

---

## まとめ

1. まず有線LAN接続でLuCIやSSHへ入れるか確認する
2. 最後に変更した設定（LAN IP、firewall、VLAN等）を思い出す
3. バックアップファイルがあるか確認する
4. LuCIやSSHから戻せるなら、部分復元を先に試す
5. それでも戻せない場合だけハードリセットを検討する
6. リセット後はWAN・Wi‑Fi・普段使う機能から順番に戻す

「壊れたからすぐ初期化」ではなく、「まだ戻せる余地があるか」を確認するだけでも、かなり安全に復旧しやすくなります。

特にLAN IP変更やfirewall設定変更は、「実はルーター自体は正常」なケースもかなり多くあります。

バックアップを定期的に残しておくと、復旧時間を大幅に短縮できます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/022/diagram-03.png)

## よくある質問

### LN6001-JPがおかしい時、すぐリセットしたほうがいい？

すぐに押すより、有線LAN、LuCI、SSHで戻せないかを先に確認するほうが安全です。

バックアップがあるなら、復元のほうが早いこともあります。

### ソフトリセットとハードリセットは何が違う？

どちらも最終的には初期化ですが、LuCIやSSHから操作できるなら状況確認やバックアップの余地が残ります。

物理ボタンは最後の手段として考えるほうが安全です。

### LN6001-JPをリセットした後は何からやり直す？

まず004の初期設定手順に戻り、管理パスワード、SSID、WAN確認の順で整えると戻しやすくなります。

最初は「インターネットへ出られる状態」を優先確認すると整理しやすいです。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
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
