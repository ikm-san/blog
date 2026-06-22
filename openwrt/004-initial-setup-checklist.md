<!-- mirror-source: articles/004-initial-setup-checklist.md -->

# 買ったらまずここまで｜LN6001-JP初期設定チェックリスト【OpenWrt集中連載004】

新しいルーターを買った直後って、ちょっとテンション上がります。

Wi-Fi 7だし。  
OpenWrtベースだし。  
LuCIもSSHも使えるし。  
VLANもVPNもIPoEも触れそう。

「よし、全部設定するぞ」と言いたくなります。

分かります。

でも、LN6001-JPを買って最初にやることは、全部の機能を触ることではありません。

まずやるべきことは、かなりシンプルです。

```txt
普通につながる
管理画面に入れる
パスワードを変える
Wi-Fi名を確認する
バックアップを1本取る
```

ここまでです。

ゲストWi-Fi、VLAN、VPN、DNS広告ブロック、MLO、Tailscale、IPoEモジュール。

このあたりは後からで大丈夫です。

OpenWrtベースのルーターは、できることが多いぶん、初日に全部を変えると、何か起きた時に原因が追いにくくなります。

初日は、まず「つながる」「ログインできる」「戻せる」を作る。

これくらいの距離感がちょうどいいです。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、LN6001-JPを完璧に設定することではありません。

まずは、ここまでできればOKです。

- 本体底面ラベルの情報を確認する
- ONU / モデム / ホームゲートウェイとWANポートを正しくつなぐ
- `https://192.168.1.1` からLuCIへログインする
- 管理者パスワードを変更する
- SSIDとWi-Fiパスワードを確認する
- WAN接続を確認する
- 初期バックアップを1本保存する

ここまでできれば、初期設定としてはかなりいい感じです。

OpenWrtの本番はここからですが、まずは足場を作りましょう。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-01.png)

## 先にざっくり結論

初期設定の流れはこれです。

```txt
本体ラベルを確認
  ↓
WANとLANをつなぐ
  ↓
LuCIへログイン
  ↓
管理者パスワード変更
  ↓
SSID / Wi-Fiパスワード確認
  ↓
WAN接続確認
  ↓
タイムゾーンとホスト名
  ↓
バックアップ保存
```

最初の成功条件は、次の3つです。

```txt
LuCIへ再ログインできる
普段使う端末がWi-Fiにつながる
初期バックアップを1本保存できている
```

この3つができたら、初日はもう勝ちです。

ゲストWi-Fi、VLAN、VPN、Adblock、IPoEモジュールなどは、後から1つずつ足していけば大丈夫です。

## こういう人向けです

この記事は、次のような人向けです。

- LN6001-JPを買った直後で、何から触ればいいか迷っている
- OpenWrtベースのルーターを壊さずに使い始めたい
- ONU、モデム、ホームゲートウェイ配下での接続が不安
- IPoE、PPPoE、固定IPなどの回線方式がよく分からない
- LuCIやSSHは気になるけど、最初は安全に進めたい
- ゲストWi-FiやVPNは後からでいいので、まず初期設定を終わらせたい

逆に、すでにOpenWrt運用に慣れていて、VLANやVPNまで一気に作る人には少し基本寄りです。

でも、慣れている人でも初期バックアップだけは本当におすすめです。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-02.png)

## 最初に言葉だけそろえる

初期設定で出てくる言葉を、軽くそろえておきます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-01.png)

最初は全部を深く理解しなくて大丈夫です。

まずは、

```txt
WAN = 回線側
LAN = 家や事務所の内側
LuCI = 管理画面
```

くらいで十分です。

## まず用意するもの

初期設定では、次を手元に用意します。

- LN6001-JP本体
- 付属電源アダプター
- LANケーブル
- ONU、モデム、ホームゲートウェイなどの回線機器
- PCまたはスマートフォン
- プロバイダの契約情報
- ひかり電話契約の有無が分かる情報
- 固定IP契約の有無が分かる情報
- パスワードマネージャー、または安全にメモを残せる場所

設定変更は、できればPCから行うのがおすすめです。

スマートフォンのブラウザでもLuCIは開けます。

ただ、Wi-Fi名やIPアドレスを変えると、途中で接続が切れることがあります。

初期設定では、有線LANでつないだPCのほうが落ち着いて作業できます。

特に次を触る時は、有線LANのPCがあると安心です。

- LAN IPアドレス
- WAN接続方式
- VLAN
- firewall
- Wi-Fi名や暗号化方式

スマホだけで全部やろうとすると、途中でWi-Fiが切れて「今どこ？」となりがちです。

## 通電前に確認すること

電源を入れる前に、まず本体底面ラベルと既存の回線機器を確認します。

ここ、地味ですが大事です。

### 本体底面ラベルで見るもの

本体底面ラベルから、次を確認します。

- デフォルトSSID
- デフォルトWi-Fiパスワード
- 管理画面ログインに使う初期パスワード
- 型番
- シリアル番号
- 認証表示

LN6001-JPでは、初期SSIDやパスワードなどが本体底面ラベルで確認できます。

あとから家族やスタッフへWi-Fi情報を案内する場合も、最初に控えておくと楽です。

ただし、ラベルの写真をSNSや記事に出さないでください。

SSID、Wi-Fiパスワード、管理用情報が含まれることがあります。

### 既存機器で見るもの

回線側では、次を確認します。

- ONU、モデム、ホームゲートウェイの有無
- 既存ルーターのLAN側IPアドレス
- ホームゲートウェイのLAN側IPアドレス
- プロバイダの接続方式
- ひかり電話契約の有無
- 固定IP契約の有無

NTT系ホームゲートウェイでは、LAN側IPアドレスが `192.168.1.1` になっていることがあります。

LN6001-JPの初期LAN IPアドレスも `192.168.1.1` です。

つまり、HGWとLN6001-JPが同じIPアドレスになってしまうことがあります。

ここで詰まると、

```txt
管理画面に入れない
つながったり切れたりする
なぜか既存ルーターの画面が出る
```

みたいなことが起きます。

最初に既存機器のIPを見ておくと、あとでかなり楽です。

## ケーブル接続の順番

まずはシンプルにつなぎます。

```txt
ONU / モデム / HGW
  ↓
LN6001-JPのInternet / WANポート
  ↓
LN6001-JPのLANポート
  ↓
PC
```

手順は次の通りです。

1. ONU、モデム、またはホームゲートウェイのLANポートから、LN6001-JPのInternet / WANポートへLANケーブルを接続する
2. PCを有線接続する場合は、LN6001-JPのLANポートからPCへLANケーブルを接続する
3. LN6001-JPに電源アダプターを接続する
4. 電源を入れる
5. LEDが白く点灯するか確認する

公式サポートでは、インターネット接続を検知するとライトが白く点灯すると案内されています。

赤のままでも、すぐ故障と決めつけなくて大丈夫です。

まずはここを見ます。

- WANケーブルがInternet / WANポートに入っているか
- 回線機器側のLANポートが有効か
- ONU / HGW側が起動しているか
- 回線機器を再起動する必要がないか
- 回線方式がDHCP自動でよいか

初期設定のミスで多いのは、難しい設定ではなく、WANとLANの挿し間違いです。

まず物理から見ます。

## LuCI管理画面へログインする

LN6001-JPの初期設定は、LuCIの管理画面から始めます。

### 接続する

PCまたはスマートフォンを、LN6001-JPに接続します。

方法はどちらでも構いません。

- 有線LAN: LN6001-JPのLANポートとPCをLANケーブルで接続
- Wi-Fi: 本体底面ラベルに記載されたデフォルトSSIDへ接続

おすすめは有線LANです。

Wi-Fi名やWi-Fiパスワードを変えると、無線接続は一度切れます。

有線なら、そのまま作業を続けやすいです。

### ブラウザで開く

ブラウザのアドレスバーに次を入力します。

```txt
https://192.168.1.1
```

`http://` ではなく `https://` です。

初回アクセス時に、証明書警告が出ることがあります。

これはローカル管理画面で自己署名証明書を使う場面ではよくある表示です。

次を確認してから進みます。

- LN6001-JPへ直接つないでいる
- アクセス先が `https://192.168.1.1`
- 変な外部サイトではない

問題なければ、ブラウザの「詳細設定」などから続行します。

### ログインする

ログイン情報は次です。

```txt
ユーザー名: root
パスワード: 本体底面ラベルに記載された初期パスワード
```

公式FAQでは、デフォルトの管理者パスワードは本体底面のデフォルトWi-Fiパスワードと同じと案内されています。

ログインできたら、LuCIのStatus画面が表示されます。

ここまで来たら、最初の大きな山は越えています。

## 管理者パスワードを変更する

LuCIへ入れたら、まず管理者パスワードを変更します。

Wi-Fi名より先に、ここです。

管理者パスワードは、LuCIやSSHでルーターへ入るための鍵です。

Wi-Fiパスワードとは役割が違います。

### LuCIでの手順

1. 上部メニューの **System** を開く
2. **Administration** を選択する
3. **Router Password** セクションを確認する
4. **Password** に新しいパスワードを入力する
5. **Confirmation** に同じパスワードを再入力する
6. **Save** をクリックする

パスワードは、12文字以上を目安にします。

英大文字、英小文字、数字、記号を混ぜると安心です。

覚えやすさだけで決めるより、パスワードマネージャーに保存する前提で強い文字列にしておくほうがおすすめです。

### SSHで変更する場合

SSHでログインできる環境なら、次でも変更できます。

```sh
passwd
```

実行すると、新しいパスワードを2回入力するよう求められます。

ただし、初期設定ではLuCIから変更すれば十分です。

SSHは、最初は状態確認やバックアップ取得に使うくらいで大丈夫です。

## SSIDとWi-Fiパスワードを確認・変更する

工場出荷時のSSIDとWi-Fiパスワードは、端末ごとに用意されています。

そのまま使い始めることもできます。

ただ、家族やスタッフに案内しやすい名前にしたい場合や、既存ルーターから置き換える場合は、このタイミングで変更しておくと後が楽です。

### 変更する前に決めること

SSIDを変更する前に、次を軽く決めておきます。

- 2.4GHz / 5GHz / 6GHzを同じSSIDにするか、分けるか
- 家族用・業務用のメインSSIDをどうするか
- 既存ルーターと同じSSIDにして端末の再設定を減らすか
- 新しいSSIDにして、古い接続情報を整理するか
- ゲストWi-Fiは今作るか、後で別記事を見ながら作るか

初期設定の日に、ゲストWi-Fiまで一気に作らなくても構いません。

まずメインSSIDを安定させる。  
バックアップを取る。  
その後でゲストWi-Fiを作る。

この順番のほうが安全です。

### LuCIでの手順

1. 上部メニューの **Network** → **Wireless** を開く
2. 変更したい無線インターフェースの **Edit** をクリックする
3. **Interface Configuration** の **ESSID** に新しいWi-Fi名を入力する
4. **Wireless Security** タブを開く
5. **Encryption** を選ぶ
6. **Key** に新しいWi-Fiパスワードを入力する
7. **Save** をクリックする
8. 必要に応じて2.4GHz / 5GHz / 6GHzで同じ作業をする
9. 画面上部の **Save & Apply** で反映する

反映すると、Wi-Fi接続は一度切れます。

これは正常です。

新しいSSIDとパスワードで再接続してください。

### CLIではまず確認だけにする

初期設定では、Wi-Fi設定の変更はLuCIから行うのが安全です。

CLIでは、まず現在の設定確認とバックアップに留めます。

```sh
echo "### backup wireless config"
cp /etc/config/wireless /etc/config/wireless.backup.$(date +%Y%m%d-%H%M)

echo "### SSID and encryption"
uci show wireless | grep -E 'ssid|encryption|disabled'

echo "### radio status"
wifi status
```

`default_radio0` のようなセクション名は、環境やファームウェアによって変わることがあります。

確認せずに `uci set` をコピペ実行しないでください。

最初はLuCIで変更し、CLIは「今どうなっているかを見る」ために使うくらいで十分です。

## 回線タイプを確認する

初期設定でいちばん詰まりやすいのはWAN側です。

ここは、自分の契約と回線機器の構成を確認しながら進めます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-02.png)

OCNバーチャルコネクト、transix、v6プラス、クロスパスなどのIPv4 over IPv6サービスを使う場合は、別記事「NTT IPoEとIPv4 over IPv6」で詳しく扱います。

初期設定の記事では、まず

```txt
自分の回線がどのパターンに近いか
```

を把握できれば十分です。

## WAN接続状態を確認する

### LuCIで確認する

1. **Status** → **Overview** を開く
2. **Network** セクションを確認する
3. IPv4 WAN StatusやIPv6 WAN Statusを見る
4. WAN側IPアドレスが表示されているか確認する

WAN側IPアドレスが表示されていれば、回線側との接続はかなり前進しています。

ただし、IPv4とIPv6のどちらがつながっているかは、回線方式によって見え方が変わります。

IPoE環境では、IPv6は生きているけれどIPv4 over IPv6側だけまだ、ということもあります。

### CLIで確認する

SSHで入れる場合は、読み取り中心で確認します。

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

echo "### routes"
ip route show
ip -6 route show

echo "### reachability"
ping -c 4 8.8.8.8
ping -c 4 google.com
```

`ifstatus wan` と `ifstatus wan6` は、設定変更ではなく、まず今の状態を見るためのコマンドです。

判断の目安です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-03.png)

最初は、ここまで見られれば十分です。

## タイムゾーンとホスト名を設定する

地味ですが、ログを見る時に大事です。

1. **System** → **System** を開く
2. **General Settings** を確認する
3. **Hostname** にルーター名を入れる

例:

```txt
LN6001-home
LN6001-office
LN6001-shop
```

4. **Timezone** で `Asia/Tokyo` を選ぶ
5. **Save & Apply** をクリックする

ログの時刻がずれていると、トラブル時にかなり見づらくなります。

最初のうちにタイムゾーンを合わせておくと、あとで監視やログ確認の記事へ進んだ時にも楽です。

## 初期バックアップを取る

WAN接続、LuCIログイン、管理者パスワード変更、SSID確認が終わったら、必ずバックアップを取ります。

ここが初期設定のゴールです。

バックアップが1本あるだけで、後からゲストWi-Fi、VLAN、VPN、IPoEモジュール、DNS広告ブロックを試す時の安心感がかなり変わります。

バックアップがあると、人は少し大胆になれます。

OpenWrt系ルーターでは、これが本当に大事です。

### LuCIでバックアップする

1. **System** → **Backup / Flash Firmware** を開く
2. **Backup** セクションの **Generate archive** をクリックする
3. `.tar.gz` ファイルをダウンロードする
4. ファイル名に日付と状態を入れて保存する

ファイル名の例です。

```txt
backup-LN6001-initial-20260621.tar.gz
```

バックアップファイルには、ネットワーク設定やWi-Fi設定など、重要な情報が含まれます。

そのままSNSや公開リポジトリへアップロードしないでください。

### CLIで状態メモを残す

LuCIのバックアップに加えて、状態確認ログを残しておくと、あとで比較しやすくなります。

```sh
BACKUP_DIR="/root/initial-setup-check-$(date +%Y%m%d-%H%M)"
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
logread | tail -200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

PCへ転送する場合は、PC側のターミナルで次のように実行します。

```sh
scp -r root@192.168.1.1:/root/initial-setup-check-YYYYMMDD-HHMM ./
```

このフォルダにもパスワードや設定に関わる情報が含まれることがあります。

保管場所には注意してください。

## 初期設定後に控える情報

初期設定が終わったら、次の情報をメモしておきます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/table-04.png)

このメモは、あとでトラブル対応、ファームウェア更新、VLAN設定、VPN設定へ進む時に役立ちます。

未来の自分にかなり感謝されます。

## 最初に変更してよいもの、慎重に扱うもの

初期設定では、触るものと触らないものを分けます。

### 最初に変更してよいもの

- 管理者パスワード
- SSID
- Wi-Fiパスワード
- タイムゾーン
- ホスト名

このあたりは、最初に整えておくと運用しやすくなります。

### 慎重に扱うもの

- LAN側IPアドレス
- WAN接続方式
- DHCP設定
- VLAN
- firewall zone
- VPN
- IPoEモジュール
- パッケージ追加

これらは、設定ミスで管理画面へ入れなくなったり、インターネットへ出られなくなったりすることがあります。

触る前にバックアップを取り、1つ変更したら確認する、という進め方がおすすめです。

### この連載では変更しないもの

- 無線の国設定
- 地域コード
- 送信出力
- DFS関連
- 国内向け製品としての前提に関わる無線設定

LN6001-JPは日本向けモデルです。

この連載では、技適や国内利用の前提に関わる無線設定は変更しません。

## 初期設定でつまずきやすいところ

初期設定でつまずいても、原因はだいたい限られています。

まずは「管理画面」「WAN接続」「IPアドレス重複」の3つに分けて見ます。

## 管理画面にアクセスできない

確認する順番は次です。

1. `https://192.168.1.1` で開いているか
2. PCまたはスマートフォンがLN6001-JPのLAN側に接続されているか
3. Wi-Fi接続先が既存ルーターではなくLN6001-JPになっているか
4. 有線LANでLN6001-JPのLANポートへ直接つないで試したか
5. HGWや既存ルーターとIPアドレスが重複していないか
6. 証明書警告で止まっていないか

ブラウザが自動で検索してしまう場合は、アドレスバーへ `https://192.168.1.1` を直接入力します。

ここで焦ってリセットボタンを押したくなります。

でも、まだ早いです。

まず有線LANで直接つなぎ、PCがどのIPアドレスを取っているかを見ます。

## Wi-Fiにはつながるがインターネットへ出られない

次の順番で見ます。

1. LEDが白点灯か、赤点灯か
2. WANケーブルがInternet / WANポートに入っているか
3. ONU、モデム、HGW側のLANポートが有効か
4. LuCIのStatus画面でWAN側IPアドレスが表示されているか
5. SSHで `ifstatus wan` / `ifstatus wan6` を確認する
6. 回線方式がDHCP自動でよいか、PPPoEやIPoEモジュールが必要かを確認する

HGWあり環境では、LN6001-JP側はDHCP自動で動く場合があります。

HGWなしのNTTフレッツIPoE環境では、オートIPoEモジュールが必要になる場合があります。

## HGWとIPアドレスが重複している

HGWとLN6001-JPがどちらも `192.168.1.1` を使っている場合、管理画面アクセスや通信が不安定になることがあります。

この場合は、LN6001-JP側のLAN IPアドレスを変更します。

ただし、これは慎重に扱う設定です。

作業前にバックアップを取り、有線LANで接続してから行います。

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

変更後は、管理画面のURLも変わります。

```txt
https://192.168.10.1
```

IPアドレス変更後に、PCが古いIPアドレスを持ったままになることがあります。

その場合は、PCのWi-Fiや有線LANを一度切断して再接続するか、PCを再起動します。

## 初期設定の日にやらなくてよいこと

LN6001-JPはできることが多いので、初日に全部触りたくなります。

でも、次の作業は初期設定が安定してからで大丈夫です。

- ゲストWi-Fi作成
- VLAN設定
- WireGuard / Tailscale設定
- DNS広告ブロック
- 詳細なfirewall設定
- MLO設定
- パッケージ追加
- AP / ブリッジ / WDS子機設定

まずは、普通に使える状態とバックアップを作る。

そのあとで、必要な機能を一つずつ足していくほうが安全です。

OpenWrtは、初日に全部盛りしなくてもちゃんと楽しいです。

## まとめ

LN6001-JPの初期設定で大事なのは、最初から全部を触ることではありません。

まずは、次の3つです。

```txt
つながる
ログインできる
戻せる
```

最初のステップをもう一度まとめると、こうです。

1. 本体底面ラベルを確認する
2. ONU / モデム / HGWとWANポートを接続する
3. `https://192.168.1.1` でLuCIへログインする
4. 管理者パスワードを変更する
5. SSIDとWi-Fiパスワードを確認・必要なら変更する
6. WAN接続を確認する
7. タイムゾーンとホスト名を設定する
8. バックアップを取る

ここまで終われば、初期設定としては十分です。

次は、IPoE設定、ゲストWi-Fi、VLAN、VPNなど、目的に合わせて一つずつ進めていけば大丈夫です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/004/diagram-03.png)

## 次に読むなら

初期設定が終わったら、次は目的に合わせて進みます。

- [NTT IPoEとIPv4 over IPv6: OCNバーチャルコネクト/transixをどう設定するか](https://note.com/ikmsan/n/n97ddc6c12ca8)
- [LN6001-JPのWi-Fi設定](https://note.com/ikmsan/n/ndd6f12876bc8)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

日本の回線設定が不安ならIPoE記事へ。

Wi-Fi名や帯域、6GHzまわりを整理したいならWi-Fi設定記事へ。

家庭用・IoT用・ゲストWi-Fiを分けたいなら、ゲストWi-Fi記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### LN6001-JPは買ってすぐ使える？

DHCP自動でつながる回線環境なら、回線機器につないで電源を入れるだけで使い始められる場合があります。

ただし、管理者パスワードの変更、SSIDとWi-Fiパスワードの確認、初期バックアップは最初に済ませておくほうが安心です。

### 初期設定で最初に変えるべきものは？

まずは管理者パスワードです。

その次にSSIDとWi-Fiパスワードを確認し、必要なら変更します。

最後にWAN接続を確認し、バックアップを取る流れにすると迷いにくくなります。

### 管理画面に入れない時は何を確認すればいい？

`https://192.168.1.1` で開いているか、PCやスマートフォンがLN6001-JPに接続されているか、証明書警告で止まっていないかを確認します。

それでも入れない場合は、HGWや既存ルーターとIPアドレスが重複していないかを確認します。

有線LANでLN6001-JPのLANポートへ直接つないで試すと切り分けしやすいです。

### 初期設定でゲストWi-Fiまで作ったほうがいい？

初日は作らなくて大丈夫です。

まずはメインSSIDで安定して使える状態を作り、バックアップを取ります。

そのあと、ゲストWi-Fiの記事を見ながら、家庭用、IoT用、ゲスト用の分離を考えるほうが安全です。

### IPoEモジュールは最初に必ず必要？

必ずではありません。

回線構成によります。

HGWあり環境では、HGW側がIPoEを担当し、LN6001-JPはDHCP自動で動く場合があります。

HGWなしのNTTフレッツIPoE環境では、オートIPoEモジュールが必要になる場合があります。

まずは自分の回線方式と回線機器の構成を確認してください。

### 初期設定後、すぐSSHを使うべき？

必須ではありません。

LuCIだけでも初期設定はできます。

ただ、SSHで `ifstatus wan`、`cat /tmp/dhcp.leases`、`logread | tail -n 50` などを見られると、後のトラブル対応がかなり楽になります。

最初は読み取りだけで十分です。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へログインする方法: https://support.linksys.com/kb/article/7038-jp/
- Velop WRT Pro 7 OpenWrt ルーターの設定をファイル保存・復元する方法: https://support.linksys.com/kb/article/7041-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/

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
