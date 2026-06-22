<!-- mirror-source: articles/017-port-forwarding-caution.md -->

# そのポート、本当に開ける？｜NAS・カメラを外に出す前にVPNを考える【OpenWrt集中連載017】

NASを外から見たい。  
監視カメラをスマホで確認したい。  
ゲームサーバーを友人に開放したい。  
自宅のミニPCへSSHしたい。  
小さなオフィスの管理画面へ外出先から入りたい。

こういう時、まず出てくる言葉が「ポート開放」です。

ポート開放は便利です。

でも、言い方を変えると、

```txt
インターネット側から入れるドアを1つ増やす
```

設定です。

しかも、そのドアは家の中だけではなく、インターネット側から見えます。

NASの管理画面。  
カメラのWeb UI。  
録画機。  
SSH。  
RDP。  
ルーターの管理画面。

こういうものを直接外に出すと、便利になる一方で、見つけられる可能性も上がります。

もちろん、ポート開放そのものが悪いわけではありません。

ゲームサーバーや公開Webサーバーのように、外部から直接アクセスさせたい用途では必要になることがあります。

でも、自分だけが使う。  
家族だけが見る。  
スタッフだけが管理する。

それなら、まずVPNでよくない？  
というのがこの記事の出発点です。

LN6001-JPでは、LuCIからPort Forwardsを設定できます。  
さらに、Linksys公式のVPN Assistantを使ってWireGuardやTailscaleも扱えます。

この記事では、ポート開放のやり方だけではなく、開ける前に考えること、VPNで代替できるケース、IPoE環境で詰まりやすいところ、開けたあとに棚卸しするポイントをまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、ポート開放をたくさん作れるようになることではありません。

まずは、ここまで分かればOKです。

- ポート開放が「外から入れる入口」だと理解できる
- NAS、カメラ、LuCI、SSH、RDPを直接公開しないほうがよい理由が分かる
- 自分や家族だけが使うなら、VPNを先に考えられる
- IPoE / IPv4 over IPv6環境でポート開放が難しい場合があると分かる
- IPv4ポートフォワードとIPv6公開は別物だと分かる
- 公開先機器のIPをDHCP予約で固定できる
- LuCIとCLIでPort Forwardsを確認できる
- 不要になったポート開放を削除・棚卸しできる

ポート開放は、設定できたら終わりではありません。

```txt
開けたあと管理できるか
```

が大事です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-01.png)

## 先にざっくり結論

ポート開放は、設定方法を調べる前に、次の順番で考えるのがおすすめです。

1. **本当にインターネットへ直接公開する必要があるか確認する**
2. **自分・家族・スタッフだけが使うならVPNを優先する**
3. **公開先機器のIPアドレスをDHCP予約で固定する**
4. **開けるポートは必要最小限にする**
5. **可能なら送信元IPアドレスを制限する**
6. **公開先機器の更新・認証・ログを確認する**
7. **不要になったらすぐ閉じる**
8. **定期的にPort Forwardsを棚卸しする**

特に、次は直接公開しないほうがいいです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-01.png)

判断はかなりシンプルです。

```txt
自分だけが使う → VPN
外部の人にも直接使わせる → ポート開放を検討
```

この順番で考えるだけで、かなり安全側に寄せられます。

## こういう人向けです

この記事は、次のような人向けです。

- NASを外出先から見たい
- 監視カメラや録画機を外から確認したい
- ゲームサーバーを公開したい
- ポート開放とVPNのどちらを選ぶべきか迷っている
- IPoE / IPv4 over IPv6環境でポート開放できるか分からない
- 既存のPort Forwardsを棚卸ししたい
- LuCIやSSHを外から開けてよいか不安
- 小さなオフィスや店舗で、外部公開ルールを整理したい

逆に、すでに公開サーバー運用、リバースプロキシ、WAF、証明書管理、ログ監視まで慣れている人には基本寄りです。

この記事では、家庭・小さなオフィス・店舗で「まず事故らない判断」を優先します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-02.png)

## 最初に言葉だけそろえる

ポート開放まわりで出てくる言葉を、ざっくり整理します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-02.png)

最初は、これだけ覚えれば大丈夫です。

```txt
ポート開放 = 外から入れるドア
VPN = 認証された専用通路
DHCP予約 = 公開先機器の住所を固定する
```

## ポート開放とは何か

ポート開放は、インターネット側から来た通信を、LAN内の特定機器へ転送する設定です。

たとえば、外部から次のようにアクセスしたとします。

```txt
https://<自宅やオフィスのWAN側IP>:8443
```

LN6001-JP側で、次のようなPort Forwardを作っているとします。

```txt
WAN TCP 8443 → 192.168.1.10 TCP 443
```

この場合、外部からWAN側の8443番ポートへ来た通信を、LAN内のNAS `192.168.1.10` の443番ポートへ転送します。

図にすると、こうです。

```txt
インターネット
  ↓ TCP 8443
LN6001-JP WAN
  ↓ DNAT / Port Forward
NAS 192.168.1.10:443
```

OpenWrtのfirewall設定では、こうしたポートフォワードは `redirect` セクションとして扱われます。

つまり、LuCIのPort Forwards画面で作った設定は、裏側ではfirewallのredirect設定として保存されます。

## ポート開放が必要になるケース

ポート開放が必要になるのは、外部の人や端末から、LAN内サービスへ直接アクセスさせたい場合です。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-03.png)

「外から使いたい」だけなら、ポート開放が唯一の方法ではありません。

自分や家族、スタッフだけが使うなら、WireGuardやTailscaleのようなVPNを使うほうが安全に設計しやすいです。

## 直接公開しないほうがよいもの

これはかなり大事です。

次のようなものは、原則としてインターネットへ直接公開しないほうがよいです。

- ルーターのLuCI管理画面
- ルーターのSSH
- NASの管理画面
- RDP
- VNC
- 監視カメラの管理画面
- 録画機の管理画面
- プリンターの管理画面
- Home Assistantなど家庭内管理系サービス
- 管理用ミニPCのSSHやWeb UI

これらは、だいたい「自分だけが使う」用途です。

それなら、外へ直接出すよりVPNです。

```txt
外出先端末
  ↓ VPN
LN6001-JP
  ↓ LAN内
NAS / カメラ / 管理画面
```

この形なら、NASやカメラの管理画面をインターネットへ直接見せずに済みます。

## VPNで代替できるかを先に考える

ポート開放前に、まずこの表で考えます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-04.png)

判断の目安はこれです。

```txt
自分・家族・スタッフだけが使う → VPNを優先
不特定多数や外部プレイヤーが使う → ポート開放を検討
```

VPNは、外から入るための“専用通路”です。

ポート開放は、外から見える“ドア”です。

この違いを意識するだけで、かなり判断しやすくなります。

## IPoE環境ではポート開放できるとは限らない

日本の光回線では、IPoE / IPv4 over IPv6がよく使われます。

ここで注意したいのは、IPv4の外部着信がいつでも自由にできるとは限らないことです。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-05.png)

`wan6` が正常でも、IPv4ポート開放ができるとは限りません。

IPv4 over IPv6の方式によって、外部から利用できるポート範囲が制限されたり、そもそもIPv4外部着信できなかったりします。

まず、現在の回線状態を確認します。

```sh
echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### possible IPoE logs"
logread | grep -Ei 'ipoe|map|mape|dslite|ipip|wan|wan6|odhcp6c|netifd' | tail -n 100
```

IPoE環境でポート開放がうまくいかない場合は、設定ミスだけでなく、回線方式の制約も疑います。

固定IPや外部公開が必要な業務用途では、契約中のプロバイダ仕様を確認してください。

## IPv4ポートフォワードとIPv6公開は別物

もうひとつ大事なのがIPv6です。

IPv4の家庭内ネットワークでは、多くの場合NATがあり、Port Forwardで外部からLAN内機器へ転送します。

一方、IPv6ではLAN内端末にグローバルIPv6アドレスが付くことがあります。

この場合、考え方は「ポートフォワード」ではなく、**IPv6 firewallで外部からの通信を許可するかどうか**になります。

つまり、IPv4とIPv6は別に考えます。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-06.png)

IPv6で不用意に機器を公開しないように注意してください。

IPv4でポートフォワードしていないから安全、とは限りません。

IPv6側で外部から届くルールを作っていないか、必要に応じて確認します。

```sh
echo "### IPv6 addresses"
ip -6 addr show

echo "### IPv6 routes"
ip -6 route show

echo "### firewall IPv6 hints"
uci show firewall | grep -Ei 'ip6|ipv6|wan|src|dest|port|rule'
```

この連載では、IPv6公開は必要最小限にし、基本はVPN経由をおすすめします。

## 設定前にバックアップを取る

ポート開放はfirewall設定を変更します。

設定前にLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-port-forward-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/port-forward-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in firewall network dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
iptables-save > "$BACKUP_DIR/iptables-save.txt" 2>/dev/null || true
ip6tables-save > "$BACKUP_DIR/ip6tables-save.txt" 2>/dev/null || true

logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、IPアドレス、ポート開放ルール、回線情報、端末情報が含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

ポート開放で一番強い人は、設定コマンドを暗記している人ではありません。

不要になったら閉じられる人です。

## 公開先機器のIPを固定する

ポート開放では、転送先のIPアドレスが重要です。

たとえば、NASを `192.168.1.10` として設定しているのに、再起動後に `192.168.1.123` へ変わると、Port Forwardが壊れます。

そのため、公開先機器はDHCP予約で固定します。

例:

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-07.png)

現在のDHCPリースはCLIで確認できます。

```sh
cat /tmp/dhcp.leases
```

LuCIでは、**Network** → **DHCP and DNS** → **Static Leases** でDHCP予約を作ります。

詳しくは012の固定IPとDHCP予約の記事で扱っています。

## 既存の公開ルールを棚卸しする

新しいポートを開ける前に、今すでに何が公開されているか確認します。

```sh
echo "### existing redirects"
uci show firewall | grep -E 'redirect|src_dport|dest_ip|dest_port|target|name'

echo "### detailed redirect sections"
uci show firewall | sed -n '/=redirect/,+12p'

echo "### iptables / ip6tables hints"
iptables-save 2>/dev/null | grep -Ei 'dnat|redirect|8443|51820' -A3 -B3 || true
ip6tables-save 2>/dev/null | grep -Ei 'dnat|redirect|8443|51820' -A3 -B3 || true
```

何のために作ったか分からないルールがある場合は、すぐ削除する前にメモや運用履歴を確認します。

分からない公開ルールがある状態で新しいルールを足すと、あとから管理できなくなります。

## LuCIでポートフォワードを作る

LN6001-JPでは、LuCIのFirewall画面からPort Forwardsを設定できます。

## ステップ1: Port Forwardsを開く

1. LuCIへログインする
2. **Network** → **Firewall** を開く
3. **Port Forwards** タブを開く
4. **Add** をクリックする

## ステップ2: 転送ルールを設定する

例として、NASのHTTPS管理画面を外部ポート8443から内部ポート443へ転送する場合です。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-08.png)

設定後、**Save** → **Save & Apply** をクリックします。

ただし、NAS管理画面の直接公開はおすすめしません。

この例は、Port Forwardsの仕組みを説明するためのものです。

自分だけがNASへ入る用途なら、VPNを優先してください。

## 外部ポートを変えれば安全か

内部ポート443を外部ポート8443へ変えると、一般的な自動スキャンに少し見つかりにくくなることがあります。

しかし、これは本質的な防御ではありません。

```txt
WAN 8443 → LAN 192.168.1.10:443
```

このようにしても、インターネット側へ入口を作っていることに変わりはありません。

外部ポート番号を変えることは、認証強化、更新、送信元IP制限、VPN化の代わりにはなりません。

## CLIでポートフォワードを作る

CLIで設定する場合は、名前付きセクションで作ると後から削除しやすくなります。

例: NASのHTTPSを `8443 → 443` で転送する。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-nas-https.$(date +%Y%m%d-%H%M)

uci -q delete firewall.nas_https
uci set firewall.nas_https='redirect'
uci set firewall.nas_https.name='NAS-HTTPS'
uci set firewall.nas_https.src='wan'
uci set firewall.nas_https.proto='tcp'
uci set firewall.nas_https.src_dport='8443'
uci set firewall.nas_https.dest='lan'
uci set firewall.nas_https.dest_ip='192.168.1.10'
uci set firewall.nas_https.dest_port='443'
uci set firewall.nas_https.target='DNAT'

echo "### pending changes"
uci changes firewall

uci commit firewall
/etc/init.d/firewall restart

echo "### verify"
uci show firewall.nas_https
logread | grep -Ei 'firewall|redirect|dnat|nat' | tail -n 80
```

`firewall.nas_https` のように名前付きにしておくと、削除時にインデックス番号を探さなくて済みます。

## 送信元IPを制限する

可能なら、誰でもアクセスできる状態ではなく、特定の送信元IPアドレスだけ許可します。

たとえば、別拠点の固定IP `203.0.113.10` からだけNASへアクセスさせる例です。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-nas-source-limit.$(date +%Y%m%d-%H%M)

uci set firewall.nas_https.src_ip='203.0.113.10'

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart

uci show firewall.nas_https
```

LuCIでは、Port Forwardの詳細設定でSource IP addressを指定します。

固定IPの拠点からだけアクセスする用途なら、送信元IP制限はかなり有効です。

ただし、スマートフォンのモバイル回線など、送信元IPが変わる環境では運用しにくい場合があります。

その場合も、ポート開放ではなくVPNを検討します。

## ポート範囲を広く開けない

避けたい設定の例です。

```txt
WAN TCP 1-65535 → 192.168.1.10
WAN UDP 1-65535 → 192.168.1.10
```

また、よく分からないまま複数ポートをまとめて開けるのも避けます。

ポート開放では、原則として次のように考えます。

- 必要なポートだけ
- 必要なプロトコルだけ
- 必要な期間だけ
- 必要なら送信元IPだけ
- 不要になったら削除

ゲームサーバーなどで複数ポートが必要な場合も、公式ドキュメントで必要なポートを確認し、必要な分だけ設定します。

## UPnPは便利。でも小さなオフィスでは慎重に

UPnPを有効にすると、LAN内のアプリや機器が自動的にポート開放を要求できる場合があります。

ゲーム機や一部アプリでは便利です。

一方で、ユーザーが意識しないうちにポートが開くこともあります。

家庭のゲーム用途では助かる場面があります。

でも、小さなオフィスや店舗では、UPnPを安易に有効化しないほうが管理しやすいです。

確認する場合は、パッケージや設定を見ます。

```sh
opkg list-installed | grep -Ei 'upnp|miniupnp' || true
uci show upnpd 2>/dev/null || true
logread | grep -Ei 'upnp|miniupnp' | tail -n 80
```

UPnPを使う場合でも、どの機器が何を開けているか棚卸しできる状態にしておきます。

## 外部から確認する

Port Forwardを設定したら、LAN内からではなく外部ネットワークから確認します。

たとえば、スマートフォンをWi-Fiから切り、モバイル回線で確認します。

```txt
https://<WAN側IPまたはDDNS名>:8443
```

同じLAN内からWAN側IPへアクセスして確認すると、NAT reflection / hairpin NATの有無に左右されます。

そのため、外部公開確認は必ず別ネットワークから行います。

確認することです。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-09.png)

## WAN側IPを確認する

LuCIでは、**Status** → **Overview** のIPv4 WAN Statusを見ます。

CLIでは次です。

```sh
ifstatus wan
```

WAN側IPがプライベートIPの場合、上位ルーターやCGNAT配下の可能性があります。

例:

```txt
10.x.x.x
172.16.x.x - 172.31.x.x
192.168.x.x
100.64.x.x - 100.127.x.x
```

この場合、LN6001-JPでPort Forwardを設定しても、さらに上位側で転送が必要だったり、そもそも外部から届かなかったりします。

HGWや上位ルーター配下では、二重ルーター構成にも注意します。

## 二重ルーターの場合

構成が次のようになっている場合です。

```txt
Internet
  ↓
HGW / 既存ルーター
  ↓
LN6001-JP
  ↓
NAS / サーバー
```

この場合、LN6001-JPでポート開放しても、上位ルーター側が外部通信をLN6001-JPへ転送していないと届きません。

選択肢は主に次です。

- 上位ルーター側でもポート開放する
- LN6001-JPをAP / ブリッジとして使う
- 上位ルーターのDMZを使う
- ポート開放をやめてTailscaleなどVPNを使う

DMZは広く転送する設定になりがちなので、安易にはおすすめしません。

二重ルーターで外部公開が面倒な場合は、Tailscaleのほうが楽なことがあります。

## 公開後の確認コマンド

LN6001-JP側で設定を確認します。

```sh
echo "### redirects"
uci show firewall | sed -n '/=redirect/,+12p'

echo "### iptables DNAT hints"
iptables-save 2>/dev/null | grep -Ei 'dnat|8443|NAS-HTTPS' -A5 -B5 || true

echo "### ip6tables DNAT hints"
ip6tables-save 2>/dev/null | grep -Ei 'dnat|8443|NAS-HTTPS' -A5 -B5 || true

echo "### firewall logs"
logread | grep -Ei 'firewall|dnat|redirect|reject|drop' | tail -n 120
```

公開先機器側でもログを確認します。

NASやWebサーバーなら、アクセスログや認証失敗ログを見ます。

ルーター側だけでなく、公開先機器側のログを見ることが重要です。

## 公開先機器で確認すること

Port Forwardを作る前後で、公開先機器も確認します。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-10.png)

ルーター側でポート開放しても、公開先機器の認証が弱ければ危険です。

公開するなら、公開先機器の更新と認証をセットで確認します。

## 削除・無効化の手順

ポート開放は、不要になったら削除します。

LuCIでは次の手順です。

1. **Network** → **Firewall** を開く
2. **Port Forwards** タブを開く
3. 対象ルールの **Delete** をクリックする
4. **Save & Apply** をクリックする

CLIで名前付きセクションを削除する例です。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-delete-nas-https.$(date +%Y%m%d-%H%M)

uci delete firewall.nas_https

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart

echo "### verify"
uci show firewall | grep -E 'NAS-HTTPS|nas_https|8443' || true
```

インデックス番号で削除する場合は、必ず対象を確認してから行います。

```sh
uci show firewall | grep -n '=redirect'
```

ただし、インデックスは設定追加・削除で変わることがあります。

できれば名前付きセクションで管理するほうが安全です。

## 定期棚卸しする

月1回、または設定変更時に、公開ルールを棚卸しします。

確認用コマンドです。

```sh
echo "### port forwards inventory"
uci show firewall | sed -n '/=redirect/,+12p'
```

棚卸し表の例です。

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-11.png)

「用途が説明できない公開ルール」は、削除候補です。

## 一時公開の運用

検証や一時作業でポートを開ける場合は、最初から期限を決めます。

```txt
目的:
  取引先に一時的に検証環境を見せる

公開ルール:
  Temp-Test TCP 8443 -> 192.168.1.60:443

公開期間:
  2026-06-21 15:00 - 2026-06-21 18:00

削除予定:
  作業終了後すぐ

確認:
  削除後、外部から接続できないことを確認
```

一時公開のつもりで残り続けるのが、かなり危ないです。

作業メモに「削除確認」まで入れておきます。

## ポート開放しない代替案

ポート開放以外にも選択肢があります。

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/table-12.png)

どれもメリット・デメリットがあります。

ただ、自分や限られた人だけが使うなら、まずTailscaleやWireGuardのようなVPNが分かりやすいです。

## 運用メモのテンプレート

ポート開放を設定したら、必ずメモを残します。

```txt
ルール名:
  Game-Server-UDP

目的:
  家族・友人向けゲームサーバー

公開先:
  192.168.1.50

外部ポート:
  UDP 2456-2458

内部ポート:
  UDP 2456-2458

送信元IP制限:
  なし / あり

公開開始:
  2026-06-21

削除予定:
  常設 / 2026-06-22削除

代替検討:
  VPNでは代替不可

確認:
  外部モバイル回線から接続OK
  不要ポートなし
  公開先機器の更新済み

バックアップ:
  backup-LN6001-before-port-forward-20260621.tar.gz
```

「何のために開けたか」を残しておくと、棚卸しでかなり助かります。

## よくある失敗

### LAN内からは見えるが外から見えない

よくある原因です。

- 同じWi-Fi内から確認している
- WAN側IPがプライベートIP
- 二重ルーターになっている
- IPoE / IPv4 over IPv6方式の制約
- 上位HGWで転送していない
- firewall ruleが正しくない
- 公開先機器のfirewallが拒否している

確認します。

```sh
ifstatus wan
ip route show default
uci show firewall | sed -n '/=redirect/,+12p'
logread | grep -Ei 'firewall|dnat|redirect|drop|reject' | tail -n 100
```

外部テストはモバイル回線など別ネットワークから行います。

### ポートチェックサイトでclosedになる

ポートチェックサイトでclosedになる原因はいくつかあります。

- 公開先サービスが起動していない
- 公開先機器のIPが変わっている
- TCPではなくUDPのサービスをTCPで確認している
- IPoE方式の制約で外から届かない
- 上位ルーターで止まっている
- firewall ruleが違う
- 公開先機器側firewallが拒否している

特にUDPサービスは、ポートチェックサイトでは分かりにくいことがあります。

ゲームサーバーなどでは、実際のクライアントから接続確認します。

### NASを公開したらログイン失敗が増えた

インターネットへ公開すると、自動スキャンやログイン試行が来ることがあります。

対処:

- 直接公開をやめてVPNへ移行する
- 送信元IP制限を入れる
- NASのファームウェアを更新する
- 強いパスワードと2FAを使う
- 不要な管理機能を無効化する
- ログを確認する
- 使わないポートを閉じる

NAS管理画面の直接公開は、できれば避けたい構成です。

### ルーター管理画面を公開したい

おすすめしません。

LuCIやSSHは、インターネットへ直接公開しないでください。

外出先から管理したい場合は、TailscaleやWireGuardでVPN接続してから開きます。

### RDPを公開したい

おすすめしません。

RDPは直接公開せず、VPN経由で使います。

小さなオフィスや店舗でリモート作業が必要な場合も、まずVPN、端末側の強い認証、アカウント管理をセットで考えます。

### IPoEで使いたいポートが開けない

MAP-Eなどでは、利用可能ポートが制限される場合があります。

DS-LiteではIPv4外部着信が難しいことがあります。

対処:

- プロバイダのIPv4 over IPv6方式を確認する
- 利用可能ポート範囲を確認する
- 固定IPサービスを検討する
- PPPoE固定IPを検討する
- Tailscaleなどポート開放不要のVPNを検討する

### IPv6で意図せず公開されていないか不安

IPv6では、端末にグローバルIPv6アドレスが付くことがあります。

IPv4のPort Forwardとは別に、IPv6 firewallを確認します。

```sh
ip -6 addr show
uci show firewall | grep -Ei 'ip6|ipv6|wan|src|dest|port|rule'
ip6tables-save 2>/dev/null | grep -Ei 'ip6|tcp dport|udp dport' -A3 -B3 || true
```

必要のないIPv6公開ルールは作らないようにします。

IPv6の細かい設計は、IPv6の落とし穴の記事と合わせて確認してください。

## まとめ

ポート開放は、外からLAN内機器へ入口を作る設定です。

便利ですが、管理責任も増えます。

安全に考えるなら、次の順番がおすすめです。

1. 本当に直接公開が必要か確認する
2. 自分や家族だけが使うならVPNを優先する
3. 公開先機器のIPをDHCP予約で固定する
4. 必要なポートだけ開ける
5. 可能なら送信元IPを制限する
6. IPoE / IPv4 over IPv6の制約を確認する
7. IPv6側の公開も別に確認する
8. 公開先機器の更新・認証・ログを確認する
9. 不要になったらすぐ削除する
10. 定期的に公開ルールを棚卸しする

「ポート開放できた」はゴールではありません。

公開後に安全に管理できる状態まで作って、初めて運用に乗せられます。

特に、NAS、カメラ、ルーター管理画面、RDP、SSHは、まずVPNで代替できないかを考えるのがおすすめです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/017/diagram-03.png)

## 次に読むなら

ポート開放を検討しているなら、次の記事も合わせて読むと判断しやすくなります。

- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [Tailscaleの使いどころ](https://note.com/ikmsan/n/n8d21425036a9)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

自分だけが使うアクセスなら、まずVPN記事へ。

公開先機器のIPが変わると困る場合は、固定IPとDHCP予約の記事へ。

IPv6側の公開が気になる場合は、IPv6の落とし穴の記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

OpenWrt系のfirewall表示は、バージョンやターゲットによって変わります。LN6001-JPでは `iptables-save`、`ip6tables-save`、iptables互換表示を確認します。

この記事では、まず `uci show firewall` で設定を確認し、必要に応じて `iptables-save` や `ip6tables-save` を読む方針にしています。

LuCIの画面名、firewall設定項目、Port Forwardsの表示、コマンド出力は、ファームウェア更新や追加モジュールで変わることがあります。

IPoE / IPv4 over IPv6、PPPoE、固定IP、IPIPの外部着信条件は、契約している回線やプロバイダによって異なります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポート、OpenWrtの最新ドキュメント、契約中のプロバイダ情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### NASやカメラを見るならポート開放が必要？

必ずしも必要ではありません。

自分や家族だけが見るなら、TailscaleやWireGuardのようなVPNを優先するほうが安全に設計しやすいです。

### ルーター管理画面を外部公開してもいい？

おすすめしません。

LuCIやSSHはインターネットへ直接公開せず、VPN経由でアクセスしてください。

### 外部ポートを変えれば安全？

完全な対策ではありません。

たとえば443を8443へ変えると一部の自動スキャンに引っかかりにくくなることはありますが、公開していることに変わりはありません。

認証、更新、送信元IP制限、VPN化を優先してください。

### IPoE環境でポート開放できないのはなぜ？

MAP-Eでは利用可能ポートが制限される場合があり、DS-LiteではIPv4の外部着信が難しいことがあります。

契約中のIPv4 over IPv6方式と、プロバイダの仕様を確認してください。

### IPv6でもポート開放が必要？

IPv6ではIPv4のようなDNAT型のPort Forwardではなく、対象IPv6アドレス・ポートへのfirewall許可として考えることが多いです。

IPv4とIPv6は別に確認してください。

### UPnPは有効にしていい？

家庭のゲーム用途では便利な場合がありますが、小さなオフィスや店舗では安易に有効化しないほうが管理しやすいです。

有効にする場合は、どの機器が何を開けているか確認できる状態にしてください。

### 使わなくなったポート開放はどうすればいい？

すぐ削除してください。

LuCIの **Network** → **Firewall** → **Port Forwards** から削除し、外部から接続できないことを確認します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt ポート開放・ポートフォワーディング機能の設定方法 WRT Pro 7: https://support.linksys.com/kb/article/7034-jp/
- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- OpenWrt firewall configuration: https://openwrt.org/docs/guide-user/firewall/firewall_configuration
- OpenWrt port forwarding: https://openwrt.org/docs/guide-user/firewall/fw3_configurations/port_forwarding
- OpenWrt NAT examples: https://openwrt.org/docs/guide-user/firewall/fw3_configurations/fw3_nat

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
