<!-- mirror-source: articles/012-dhcp-reservations.md -->

# プリンターが消える前に｜DHCP予約でNAS・カメラ・業務端末のIPを整理する【OpenWrt集中連載012】

プリンターって、普段は空気みたいな存在です。

動いている時は誰も気にしません。

でも、いざ印刷しようとした時に見つからない。  
昨日までつながっていたNASアプリが急につながらない。  
防犯カメラの録画ソフトがカメラを見失う。  
VPNで入りたいミニPCのIPアドレスが変わっている。

こうなると、急にIPアドレスのありがたみを感じます。

家庭や小さなオフィスには、「普段は意識しないけれど、IPアドレスが変わると急に困る機器」があります。

NAS。  
プリンター。  
監視カメラ。  
録画機。  
ミニPC。  
業務端末。  
管理用PC。

こういう機器は、ルーター側で **DHCP予約** しておくとかなり楽です。

端末側へ固定IPを手入力するのではなく、LN6001-JP側で、

```txt
このMACアドレスの機器には、いつもこのIPアドレスを渡す
```

と決めておく方法です。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、プリンター、NAS、監視カメラ、録画機、業務端末のIPアドレスをLuCIとCLIで整理する方法をまとめます。

地味なテーマですが、あとでかなり効きます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、すべての端末に固定IPを振ることではありません。

まずは、ここまでできればOKです。

- 固定IPとDHCP予約の違いが分かる
- IPが変わると困る機器を見分けられる
- DHCP配布範囲と予約IP範囲を分けて考えられる
- LuCIでStatic Leaseを追加できる
- CLIで現在のDHCPリースを確認できる
- NAS、プリンター、カメラのIP管理表を作れる
- MACアドレスランダム化でハマる理由が分かる
- ゲストWi-Fi、Device、Kidsなど複数ネットワークでもIPを整理できる

IP管理は、派手ではありません。

でも、ネットワークをちゃんと育てていくなら、かなり重要です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-01.png)

## 先にざっくり結論

家庭や小さなオフィスでは、端末側に固定IPを手入力するより、まず **ルーター側のDHCP予約** で管理するのがおすすめです。

最初に予約するのは、次のような「IPが変わると困る機器」だけで十分です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-01.png)

逆に、スマートフォン、一時利用PC、ゲストWi-Fi端末などは、通常DHCPのままで十分なことが多いです。

最初のおすすめは、次のように範囲を分けることです。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-02.png)

大事なのは、**通常DHCPで自動配布する範囲と、予約IPとして使う範囲を分けておくこと**です。

完全に必須ではありませんが、分けておくと人間が管理しやすいです。

IP管理は、人間が迷わない設計にしておくのが正義です。

## こういう人向けです

この記事は、次のような人向けです。

- プリンターのIPが変わって印刷できなくなったことがある
- NASやミニPCへ毎回同じIPでアクセスしたい
- 防犯カメラや録画機のIPを整理したい
- 小さなオフィスで業務端末のIPを管理したい
- ゲストWi-Fi、Device、Camera、StaffネットワークごとにIP表を作りたい
- 端末側の固定IPではなく、ルーター側でまとめて管理したい
- VPNやfirewallルールで宛先IPを固定しておきたい

逆に、スマートフォン数台だけの家庭なら、最初から細かく予約しなくても大丈夫です。

まずはNAS、プリンター、カメラ。

この3つだけでも十分効果があります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-02.png)

## 最初に言葉だけそろえる

IP管理まわりで出てくる言葉を、ざっくり整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-03.png)

最初は、次だけ覚えればOKです。

```txt
DHCP = 自動で住所を配る
DHCP予約 = 特定の機器には毎回同じ住所を渡す
MACアドレス = その機器を見分けるための番号
```

## 固定IPとDHCP予約の違い

「固定IP」と言っても、実際には2つのやり方があります。

ここを混ぜると少しややこしくなります。

## 端末側で固定IPを設定する

NAS、PC、プリンターなどの端末側で、IPアドレス、サブネットマスク、ゲートウェイ、DNSを手入力する方法です。

例:

```txt
IP address: 192.168.1.10
Netmask:    255.255.255.0
Gateway:    192.168.1.1
DNS:        192.168.1.1
```

この方法は、サーバーや業務機器では使われることがあります。

ただし、家庭や小さなオフィスでは、管理が散らばりやすいです。

機器ごとに設定画面を開いて直す必要があります。

ルーターのLAN IPを変えた時や、ネットワークを分けた時にも、端末側を直す必要が出ます。

```txt
設定が端末側に散らばる
```

これが一番の弱点です。

## ルーター側でDHCP予約する

LN6001-JP側で、MACアドレスとIPアドレスを紐づける方法です。

例:

```txt
MAC address: a4:c3:f0:01:23:45
Hostname:    nas-home
IP address:  192.168.1.10
```

端末側はDHCP自動のままです。

ルーターが、

```txt
このMACアドレスの端末には、いつも192.168.1.10を渡す
```

と覚えてくれます。

家庭や小さなオフィスでは、こちらのほうが管理しやすいことが多いです。

端末を交換した時も、ルーター側でMACアドレスを書き換えれば、同じIPアドレスを引き継ぎやすくなります。

## 基本方針は「重要機器だけ予約」

DHCP予約は便利です。

でも、すべての端末を予約する必要はありません。

最初は、次の3種類だけで十分です。

1. NAS、サーバー、ミニPC
2. プリンター、スキャナー、複合機
3. 監視カメラ、録画機、業務端末

スマートフォン、タブレット、一時利用PC、ゲストWi-Fi端末まで固定し始めると、管理表がすぐに膨らみます。

判断基準はシンプルです。

```txt
IPが変わったら困るか？
```

困るなら予約。  
困らないなら通常DHCP。

このくらいで十分です。

## DHCP配布範囲と予約IP範囲を分ける

DHCP予約で大事なのは、通常DHCPで自動配布する範囲と、予約IPとして使う範囲を分けておくことです。

たとえば、LANが `192.168.1.0/24` の場合、次のように分けます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-04.png)

最初からここまで細かくなくても大丈夫です。

まずは次の2つだけでも十分です。

```txt
予約IP:      192.168.1.10-99
通常DHCP:   192.168.1.100-199
```

この分け方にしておくと、

```txt
192.168.1.10台は重要機器
192.168.1.100台は普通の端末
```

と見ただけで分かるようになります。

IPアドレスは、ネットワークの座席表みたいなものです。

席順に意味を持たせると、あとで迷いにくくなります。

## 設定前に現在の状態を保存する

DHCP予約は `/etc/config/dhcp` を変更します。

複数ネットワークを作っている場合は、`network` 側のIP範囲も関係します。

まずLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-dhcp-reservations-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/dhcp-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in dhcp network; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

DHCPリースには端末名、MACアドレス、IPアドレスが含まれます。

そのままSNS、公開リポジトリ、記事のスクリーンショットへ出さないようにしてください。

設定で一番強い人は、全部暗記している人ではありません。

変更前に戻れる人です。

## 現在のDHCP配布範囲を確認する

予約IPを決める前に、現在のDHCP配布範囲を確認します。

## LuCIで確認する

1. **Network** → **Interfaces** を開く
2. `lan` の **Edit** をクリックする
3. **DHCP Server** タブを開く
4. **General Setup** の `Start` と `Limit` を確認する

たとえば、次の設定だとします。

```txt
Start: 100
Limit: 150
```

LAN側IPが `192.168.1.1/24` の場合、DHCP配布範囲はおおよそ次のように考えます。

```txt
192.168.1.100 - 192.168.1.249
```

この場合、DHCP予約は `192.168.1.10-99` のように、通常配布範囲と分けておくと管理しやすいです。

## CLIで確認する

```sh
echo "### LAN DHCP range"
uci get dhcp.lan.start
uci get dhcp.lan.limit
uci get dhcp.lan.leasetime 2>/dev/null || true

echo "### all DHCP ranges"
uci show dhcp | grep -E 'interface|start|limit|leasetime'
```

Guest、Kids、Deviceなど複数ネットワークを作っている場合は、それぞれのDHCP範囲も確認します。

## 機器管理表を作る

DHCP予約を入れる前に、機器一覧表を作っておくと管理がかなり楽になります。

特にNAS、プリンター、カメラが増えてくると、

```txt
このIP、何の機器だっけ？
```

が起きます。

表にしておくと、未来の自分が助かります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-05.png)

Markdown、Excel、Googleスプレッドシート、Notion、何でもOKです。

大事なのは、機器名、MACアドレス、IPアドレス、用途が分かることです。

IP管理は、ネットワークの家計簿みたいなものです。

細かすぎなくていいですが、何に使っているかは残しておきます。

## DHCP予約が向いている機器

## NAS / ミニPC / サーバー

NASやミニPCは、IPアドレスが変わると困りやすい機器です。

- 共有フォルダの接続先が変わる
- スマートフォンアプリが接続できなくなる
- バックアップ先が見つからなくなる
- SSHやWeb管理画面へ入りにくくなる
- VPN経由でアクセスする時に困る

NASやミニPCは、まず予約対象にしてよい機器です。

例:

```txt
nas-home     → 192.168.1.10
mini-server  → 192.168.1.11
```

## プリンター / 複合機

プリンターは、IPアドレスが変わるとPC側の印刷設定が不安定になることがあります。

特に小さなオフィスでは、プリンターを固定しておくとトラブルが減ります。

```txt
printer-office  → 192.168.1.20
printer-receipt → 192.168.1.21
```

複数台ある場合は、役割や設置場所が分かる名前にしておくと便利です。

## 監視カメラ / 録画機

防犯カメラや録画機は、録画ソフトやアプリが特定IPを見に行くことがあります。

IPが変わると、カメラがオフラインに見えることがあります。

カメラ用ネットワークを分けている場合は、そのネットワーク内で予約IPを整理します。

例:

```txt
camera-front → 192.168.3.10
camera-back  → 192.168.3.11
nvr-shop     → 192.168.3.20
```

カメラや録画機は、IP管理表との相性がかなり良いです。

どこにあるカメラか、どのIPか、どの録画機につながっているかを残しておくと、故障時の切り分けが楽になります。

## 業務端末 / POS周辺機器

POSや決済端末は、ベンダー要件を優先します。

固定IPや特定ネットワークが必要な場合もあれば、勝手に変更するとサポート対象外になる場合もあります。

この記事では、一般的なDHCP予約の考え方を扱います。

ただし、POSや決済端末は必ずベンダーの手順を確認してください。

```txt
POSや決済端末は、カメラやIoTと同じノリで触らない
```

ここは店舗運用ではかなり大事です。

## 管理用PC

ルーター、NAS、カメラ、録画機の管理に使うPCは、固定しておくと便利です。

たとえば、StaffネットワークからDeviceネットワークのカメラ管理を許可する場合、管理PCのIPを固定しておくとfirewallルールを作りやすくなります。

```txt
admin-pc → 192.168.1.30
```

管理用PCを決めておくと、トラブル対応の起点にもなります。

## DHCP予約が不要なことが多い機器

次のような端末は、最初から予約しなくても問題ないことが多いです。

- スマートフォン
- タブレット
- 一時利用PC
- ゲストWi-Fi端末
- 来訪者の端末
- 短期間だけ使う検証端末

もちろん、家庭の運用によってはスマートフォンを固定したいこともあります。

ただ、最初から全部を予約すると、管理対象が増えすぎます。

まずは、IPが変わると困る機器だけに絞るのがおすすめです。

## LuCIでDHCP予約を追加する

ここから、LuCIでStatic Leaseを追加します。

## ステップ1: 端末のMACアドレスを確認する

接続中の端末なら、LuCIで確認できます。

1. **Network** → **DHCP and DNS** を開く
2. **Active DHCP Leases** を確認する
3. 予約したい端末のMACアドレス、IPアドレス、Hostnameを控える

CLIで確認する場合は、次を使います。

```sh
cat /tmp/dhcp.leases
```

出力例:

```txt
1715000000 a4:c3:f0:01:23:45 192.168.1.105 nas-home *
```

おおよそ次のように読みます。

```txt
有効期限  MACアドレス  IPアドレス  ホスト名  クライアントID
```

端末側で確認する方法もあります。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-06.png)

## MACアドレスランダム化に注意する

スマートフォンやPCでは、SSIDごとにランダムなMACアドレスを使う機能があります。

これはプライバシー保護には有効です。

ただし、DHCP予約では注意が必要です。

端末側でMACアドレスが変わると、ルーター側の予約と一致しなくなります。

予約したい端末では、対象SSIDに対して次を確認します。

- iPhone / iPad: Wi-Fi設定 → 対象SSID → プライベートWi-Fiアドレス
- Android: Wi-Fi設定 → 対象SSID → プライバシー / MACアドレスの種類
- Windows / macOS: ランダムハードウェアアドレスやプライベートアドレス設定

NAS、プリンター、カメラのような固定機器では問題になりにくいです。

一方で、スマートフォンやPCを予約する場合は、ここでハマることがあります。

「予約したのに効かない」と思ったら、まずランダムMACを疑います。

## ステップ2: Static Leaseを追加する

1. **Network** → **DHCP and DNS** を開く
2. **Static Leases** タブを開く
3. **Add** をクリックする
4. 次の項目を入力する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-07.png)

5. **Save** をクリックする
6. 他の機器も同様に追加する
7. 画面上部の保留中の変更を確認し、**Save & Apply** をクリックする

ホスト名は、役割と場所が分かる名前にしておくと便利です。

例:

```txt
nas-home
printer-office
camera-front
nvr-shop
mini-server
```

日本語名やスペースを含む名前は避け、英数字とハイフン中心にしておくと扱いやすいです。

## ステップ3: 端末を再接続して確認する

予約設定後、端末が古いDHCPリースを持ったままだと、すぐにはIPが変わらないことがあります。

端末側で次のどれかを行います。

- Wi-Fiを切断して再接続する
- 有線LANを抜き差しする
- 端末を再起動する
- Windowsなら `ipconfig /release` → `ipconfig /renew` を使う

Windowsの例です。

```powershell
ipconfig /release
ipconfig /renew
ipconfig
```

Macでは、Wi-Fiをオフ/オンするか、ネットワーク設定からDHCPリースを更新します。

その後、LuCIの **Active DHCP Leases** やCLIで確認します。

```sh
cat /tmp/dhcp.leases
```

期待する状態は、対象端末が予約したIPアドレスを取得していることです。

## ローカル名でアクセスする

Static Leaseでホスト名を設定すると、dnsmasqのローカルDNSで名前解決しやすくなります。

たとえば、NASに次のような名前を付けたとします。

```txt
nas-home
```

環境によっては、次のような名前でアクセスできます。

```txt
nas-home
nas-home.lan
```

確認するには、PCから次を実行します。

```sh
nslookup nas-home
nslookup nas-home.lan
```

ルーター側から確認する場合は次を使います。

```sh
nslookup nas-home 127.0.0.1
nslookup nas-home.lan 127.0.0.1
```

IPアドレスを覚えるより、名前で管理できるほうが楽です。

ただし、端末やOSによって名前解決の挙動が違うことがあります。

うまく引けない場合は、LuCIの **Network** → **DHCP and DNS** → **Hostnames** で手動登録する方法もあります。

## 複数ネットワークでのIP設計

ゲストWi-Fi、カメラ用ネットワーク、Staffネットワークを分けている場合は、それぞれのネットワークでIP範囲を整理します。

例:

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-08.png)

ゲストWi-Fiでは、基本的に予約IPは不要です。

Deviceネットワークでは、カメラ、録画機、設備機器を予約することが多くなります。

Kidsネットワークでは、ゲーム機や学校用PCなど、必要な端末だけ予約します。

## 小さなオフィス向けの例

小さなオフィスでは、機器名とIP範囲をある程度ルール化しておくと便利です。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-09.png)

このくらい決めておくと、あとから「このIPは何だっけ？」が減ります。

## 店舗向けの例

店舗では、POS、カメラ、ゲストWi-Fi、Staff端末を混ぜないことが大事です。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/table-10.png)

POSや決済端末は、一般的なIoT機器と同じ扱いにしないほうが安全です。

ベンダー指定のIP範囲、DNS、通信先、サポート条件がある場合は、それを優先してください。

## CLIでDHCP予約を確認する

CLIでは、まず読み取り確認から始めます。

```sh
echo "### active leases"
cat /tmp/dhcp.leases

echo "### DHCP config"
uci show dhcp

echo "### static lease entries"
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 80
```

`cat /tmp/dhcp.leases` は、現在どの端末にどのIPが配られているかを見る時に便利です。

`uci show dhcp` は、Static LeaseやDHCP範囲を確認する時に使います。

最初は読むだけで十分です。

## CLIでDHCP予約を追加する

CLIで追加する場合は、必ずバックアップしてから行います。

```sh
echo "### backup DHCP config"
cp /etc/config/dhcp /etc/config/dhcp.backup.$(date +%Y%m%d-%H%M)

echo "### add static lease"
uci add dhcp host
uci set dhcp.@host[-1].name='nas-home'
uci set dhcp.@host[-1].mac='a4:c3:f0:01:23:45'
uci set dhcp.@host[-1].ip='192.168.1.10'

echo "### check changes"
uci changes dhcp

echo "### apply changes"
uci commit dhcp
/etc/init.d/dnsmasq restart

echo "### verify"
uci show dhcp | grep -E '@host|nas-home|a4:c3:f0:01:23:45|192.168.1.10'
logread | grep -i dnsmasq | tail -n 50
```

CLIで追加した場合も、あとでLuCIの **Static Leases** 画面で確認しておくと安心です。

`uci changes` を見てから `uci commit`。

ここは習慣にしておくと安全です。

## 複数台をまとめて追加する例

カメラを複数台追加する場合は、UCIコマンドを並べるより、先に表を作ってから慎重に入れるほうが安全です。

例:

```sh
cp /etc/config/dhcp /etc/config/dhcp.backup.before-cameras.$(date +%Y%m%d-%H%M)

uci add dhcp host
uci set dhcp.@host[-1].name='camera-front'
uci set dhcp.@host[-1].mac='dc:a6:32:44:55:66'
uci set dhcp.@host[-1].ip='192.168.3.10'

uci add dhcp host
uci set dhcp.@host[-1].name='camera-back'
uci set dhcp.@host[-1].mac='dc:a6:32:44:55:67'
uci set dhcp.@host[-1].ip='192.168.3.11'

uci add dhcp host
uci set dhcp.@host[-1].name='nvr-shop'
uci set dhcp.@host[-1].mac='00:11:22:33:44:55'
uci set dhcp.@host[-1].ip='192.168.3.20'

uci changes dhcp
uci commit dhcp
/etc/init.d/dnsmasq restart
```

複数台を一気に入れる場合は、MACアドレスとIPアドレスの取り違えが起きやすいです。

作業後、必ず `cat /tmp/dhcp.leases` とLuCIで確認します。

## 重複チェック

DHCP予約で怖いのは、IPアドレスやMACアドレスの重複です。

重複すると、端末が想定外のIPを取ったり、dnsmasqがエラーを出したりすることがあります。

簡単な確認です。

```sh
echo "### duplicate reserved IPs"
uci show dhcp | sed -n "s/.*\\.ip='\\([^']*\\)'.*/\\1/p" | sort | uniq -d

echo "### duplicate reserved MACs"
uci show dhcp | sed -n "s/.*\\.mac='\\([^']*\\)'.*/\\1/p" | tr 'A-F' 'a-f' | sort | uniq -d

echo "### current leases"
cat /tmp/dhcp.leases
```

何か表示された場合は、同じIPやMACが複数の予約に入っていないか確認します。

IP重複は、見つけるまでが地味に大変です。

最初から表で管理しておくとかなり防げます。

## 端末交換時の対応

NAS、カメラ、プリンターを買い替えた場合、DHCP予約なら引き継ぎが簡単です。

1. 新しい機器をネットワークへ接続する
2. 新しいMACアドレスを確認する
3. LuCIの **Static Leases** で既存予約を編集する
4. MACアドレスを新しい機器のものへ変更する
5. **Save & Apply** する
6. 新しい機器を再接続する

これで、同じIPアドレスを新しい機器へ引き継ぎやすくなります。

端末側へ固定IPを手入力している場合は、端末側の設定画面を開いて直す必要があります。

ルーター側で集中管理できるのが、DHCP予約の大きなメリットです。

## 端末側固定IPが必要なケース

基本はDHCP予約がおすすめですが、端末側固定IPが必要な場面もあります。

たとえば、次のようなケースです。

- DHCPがないネットワークで使う機器
- 業務機器やPOSでベンダーが端末側固定IPを指定している
- ネットワーク初期設定時に一時的に固定IPが必要
- ルーターより先に起動する必要がある特殊機器
- 別拠点へ持ち出す機器で、固定設定が前提になっている

この場合も、IPアドレス管理表に必ず記録します。

端末側固定IPは、ルーター側から見えにくいため、管理表がないと重複しやすくなります。

## よくある失敗と対処

### DHCP予約したのにIPが変わらない

端末が古いDHCPリースを持ったままの可能性があります。

対処:

- Wi-Fiを切断して再接続する
- 有線LANを抜き差しする
- 端末を再起動する
- Windowsなら `ipconfig /release` → `ipconfig /renew`
- ルーター側で `dnsmasq` を再起動する

```sh
/etc/init.d/dnsmasq restart
```

### 予約したIPに別の端末がいる

通常DHCPの配布範囲と予約IP範囲が重なっていたり、別端末が同じIPを持っていたりする可能性があります。

対処:

- DHCP配布範囲を確認する
- 予約IPを分かりやすい範囲へ移す
- すでにそのIPを使っている端末を再接続する
- 機器管理表を見直す

```sh
cat /tmp/dhcp.leases
uci show dhcp | grep -E 'start|limit|@host|\\.ip='
```

### 同じMACアドレスで複数予約している

端末交換や設定変更時に、古い予約が残っていることがあります。

対処:

- LuCIのStatic Leasesで重複を削除する
- CLIで重複を確認する
- `dnsmasq` を再起動する

```sh
uci show dhcp | grep -E '@host|\\.name=|\\.mac=|\\.ip='
```

### ホスト名でアクセスできない

まず、Static LeaseにHostnameが入っているか確認します。

```sh
uci show dhcp | grep -E 'nas-home|printer|camera'
```

次に、名前解決を確認します。

```sh
nslookup nas-home 192.168.1.1
nslookup nas-home.lan 192.168.1.1
```

うまくいかない場合は、端末側DNSがLN6001-JPを向いているか確認します。

Adblock、Family DNS、外部DNS、DoHを使っている場合は、ローカル名解決が期待どおり動かないことがあります。

### スマートフォンの予約が効かない

MACアドレスランダム化が原因のことがあります。

対象SSIDの設定で、プライベートアドレスやランダムMACを確認してください。

ただし、スマートフォンはIPが変わっても困らないことが多いため、無理に予約しなくてもよい場合があります。

### GuestやDeviceで予約したのに別ネットワークのIPになる

SSIDの紐づくNetworkが間違っている可能性があります。

たとえば、Device用SSIDを作ったつもりでも、Networkが `lan` のままだと `192.168.1.x` を取得します。

確認:

```sh
uci show wireless | grep -E 'ssid|network|device|guest|kids|iot'
cat /tmp/dhcp.leases
```

LuCIの **Network** → **Wireless** で、対象SSIDのNetworkを確認してください。

## 運用メモのテンプレート

DHCP予約を設定したら、次のようなメモを残しておくと便利です。

```txt
LAN:
  router: 192.168.1.1
  DHCP range: 192.168.1.100-199
  reserved range: 192.168.1.10-99

Reserved:
  nas-home
    MAC: a4:c3:f0:01:23:45
    IP: 192.168.1.10
    location: 書斎

  printer-office
    MAC: b8:27:eb:11:22:33
    IP: 192.168.1.20
    location: 事務所

Device:
  camera-front
    MAC: dc:a6:32:44:55:66
    IP: 192.168.3.10
    location: 入口

Backup:
  backup-LN6001-before-dhcp-reservations-20260621.tar.gz
```

こうしたメモは、地味ですがかなり効きます。

あとからVPN、監視、VLAN、ゲストWi-Fi、店舗構成例へ進む時にも使い回せます。

## まとめ

DHCP予約は、家庭や小さなオフィスのIP管理をかなり楽にしてくれます。

ポイントは次の通りです。

1. 端末側固定IPより、まずルーター側DHCP予約で管理する
2. 予約するのは、IPが変わると困る機器だけでよい
3. 通常DHCPの配布範囲と予約IP範囲を分ける
4. NAS、プリンター、カメラ、録画機、管理PCから始める
5. MACアドレスランダム化に注意する
6. ホスト名も設定して、名前でアクセスしやすくする
7. 設定前にバックアップを取る
8. 機器管理表を残す

最初から全部の端末を固定IP化する必要はありません。

まずは、NAS、プリンター、カメラの3種類だけ整理する。

それだけでも、家庭や小さなオフィスのネットワーク管理はかなり見通しが良くなります。

IPアドレス管理は地味です。

でも、トラブルが起きた時に「あの時やっておいてよかった」と思うタイプの設定です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/012/diagram-03.png)

## 次に読むなら

DHCP予約を整理したら、次は目的に合わせて進みます。

- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)
- [店舗向け構成例](https://note.com/ikmsan/n/n506a774c0680)

カメラやPOS、ゲストWi-Fiを分けたい人は、VLANとfirewall zoneの記事へ。

外出先からNASやミニPCへ安全に入りたい人は、VPNの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

また、OpenWrt系ではdnsmasqやodhcpdの設定・挙動がバージョンによって変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートやOpenWrtの最新ドキュメントも確認してください。

## よくある質問

### 固定IPは端末側で設定するよりDHCP予約がいい？

家庭や小さなオフィスでは、まずDHCP予約がおすすめです。

端末側はDHCP自動のままにして、LN6001-JP側でMACアドレスとIPアドレスを紐づけます。

端末交換時もルーター側の設定変更で済みやすくなります。

### DHCP予約が必要な機器はどれ？

NAS、プリンター、監視カメラ、録画機、ミニPC、管理用PC、業務端末などです。

IPが変わると設定や接続先が壊れる機器を優先します。

### スマートフォンもDHCP予約したほうがいい？

基本的には不要です。

スマートフォンはIPが変わっても困らないことが多く、MACアドレスランダム化の影響もあります。

必要な理由がある端末だけ予約すれば十分です。

### DHCP配布範囲と予約IP範囲は重なってもいい？

技術的に動く場合もありますが、運用上は分けておくほうが分かりやすいです。

たとえば、通常DHCPを `192.168.1.100-199` にし、予約IPを `192.168.1.10-99` に分けると管理しやすくなります。

### ホスト名でアクセスできる？

Static Leaseでホスト名を設定すると、`nas-home` や `nas-home.lan` のような名前でアクセスできる場合があります。

ただし、端末側DNSがLN6001-JPを向いている必要があります。

Adblock、外部DNS、DoH、Family DNSを使っている場合は、ローカル名解決の挙動を確認してください。

### GuestやDeviceネットワークでもDHCP予約できる？

できます。

Guestでは通常不要ですが、Deviceネットワークではカメラや録画機を予約することが多いです。

それぞれのネットワーク内で、予約IP範囲とDHCP配布範囲を分けて管理します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys Velop WRT Pro 7 LAN IPアドレス変更手順: https://support.linksys.com/kb/article/226-en/?section_id=175
- OpenWrt DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt DHCP and DNS examples: https://openwrt.org/docs/guide-user/base-system/dhcp_configuration

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
