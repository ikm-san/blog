<!-- mirror-source: articles/009-sqm-latency.md -->

# 子ども用Wi-Fiは“止める”より“分ける”から｜Family DNSと時間ルールを無理なく使う【OpenWrt集中連載009】

子ども用端末のネットワーク設定、けっこう悩みます。

スマートフォン。  
タブレット。  
学校用PC。  
ゲーム機。  
動画用のテレビ。  
学習アプリ。  
オンライン授業。  
ついでにスマートスピーカーやIoT機器。

全部を家族用Wi-Fiに入れておけば、とりあえず動きます。

でも、あとからこう思うことがあります。

「子ども用端末だけDNSフィルタリングしたい」  
「夜だけインターネットを止めたい」  
「学校用PCは止めたくない」  
「スマート家電と子ども用端末は分けたい」  
「でも、家族全員のWi-Fiまで壊したくない」

ここでいきなり強い制限を入れると、だいたい家庭内サポートセンターが開業します。

「授業サイトが開かない」  
「ゲームだけつながらない」  
「動画が止まる」  
「親のスマホまで影響してる」  
「結局どの設定を戻せばいいの？」

つらいです。

なので、家族向けフィルタリングで最初にやることは、強く止めることではありません。

まずは **分けること** です。

子ども用SSIDを分ける。  
子ども用ネットワークを分ける。  
DNS方針を分ける。  
必要なら時間ルールを足す。  
うまくいかなければ戻せるようにする。

この順番がかなり大事です。

LN6001-JPはOpenWrtベースなので、子ども用SSID、DHCP、DNS、firewall zone、時間帯ルールを組み合わせて、家庭のルールに合わせたネットワークを作れます。

ただし、ルーターだけで全部を完全に管理しようとすると無理が出ます。

この記事では、子ども用SSIDを作り、Family DNSと夜間ルールを段階的に入れる考え方をまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、子ども用端末を完全に管理することではありません。

まずは、ここまでできればOKです。

- 子ども用SSIDを分ける理由が分かる
- 子ども用ネットワークを別サブネットにできる
- kids用firewall zoneを作れる
- DHCPでFamily DNSを配れる
- 夜間だけkids → wanを止める考え方が分かる
- DoH、VPN、モバイル回線では迂回されることを理解できる
- 学校用サービスが壊れた時に戻し方を考えられる

家庭ネットワークでは、完璧な制御より、説明できる設計のほうが強いです。

まずは、

```txt
子ども用端末は MyHome_Kids
親や家族共用端末は MyHome
IoT機器は MyHome_IoT
```

くらいに分けるところから始めます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/diagram-01.png)

## 先にざっくり結論

家族向けフィルタリングは、次の順番で小さく始めるのがおすすめです。

1. **子ども用SSIDを分ける**
2. **子ども用ネットワークを別サブネットにする**
3. **kids用firewall zoneを作る**
4. **DHCPでFamily DNSを配布する**
5. **必要ならAdblockを併用する**
6. **夜間だけkids → wanを止める**
7. **学校用サービスや必要なアプリは例外対応する**
8. **DoH、VPN、モバイル回線では迂回されることを理解しておく**

最初から全部を完璧に止めようとしないほうがいいです。

家庭では、強すぎるフィルタリングより、

```txt
どの端末に、どのルールがかかっているか分かる
```

ことのほうが大事です。

## この記事の前提

この記事では、次の構成を前提にします。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-01.png)

AP Bridgeやブリッジモードで使っている場合は、この記事と同じ手順にならないことがあります。

この記事では、LN6001-JPがルーターとしてDHCPとfirewallを担当している前提で進めます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/diagram-02.png)

## こういう人向けです

この記事は、次のような人向けです。

- 子ども用端末だけDNSフィルタリングをかけたい
- 家族全員へ一律の広告ブロックをかけるのは避けたい
- 夜間だけ子ども用端末のインターネットを止めたい
- 学校用PCや学習アプリは使えるようにしておきたい
- IoT機器やゲーム機を家族のメイン端末と分けたい
- ルーター側制御の限界も理解したうえで使いたい
- いきなり厳しい制限ではなく、家庭で説明しやすい構成にしたい

逆に、

```txt
子どもの端末を完全に制御したい
モバイル回線もVPNもアプリも全部止めたい
親子のルール作りは不要で、技術だけで解決したい
```

という目的だと、ルーターだけではかなり難しいです。

ルーター側の設定は、家庭内ルールの補助として使うのが現実的です。

## 最初に言葉だけそろえる

子ども用ネットワークで出てくる言葉を、ざっくり整理しておきます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-02.png)

ここで一番大事なのは、SSIDとルールをセットで考えることです。

```txt
MyHome_Kids につないだ端末には kids のルールを適用する
```

この形にしておくと、あとからかなり管理しやすくなります。

## 家族向けフィルタリングで大事な考え方

家庭のフィルタリングは、技術だけでは完結しません。

とくに子ども用端末では、次の3つを分けて考えると運用しやすくなります。

## 1. ネットワークを分ける

まず、子ども用端末をどこへつなぐかを決めます。

メインSSIDに全部混ぜるのではなく、子ども用SSIDを作るとルールを適用しやすくなります。

```txt
MyHome        → 親・家族共用端末
MyHome_Kids   → 子ども用端末
MyHome_IoT    → IoT機器
MyHome_Guest  → ゲストWi-Fi
```

こう分けておくと、DNS、時間制限、firewallルールをSSID単位で考えやすくなります。

最初から完璧に分離しなくても大丈夫です。

まずは「子ども用SSIDはここ」と決めるだけでも前進です。

## 2. DNSで不要な通信先を減らす

DNSフィルタリングは、成人向けコンテンツ、マルウェア、広告・トラッキングなどの一部を名前解決の段階で止める方法です。

端末ごとにアプリを入れなくても、ネットワーク単位で適用できるのが利点です。

ただし、完全ではありません。

DoH、VPN、モバイル回線、アプリ内通信、同一ドメイン配信には効きにくい場合があります。

DNSは「全部止める壁」ではなく、「余計な入口を減らすフィルター」と考えるとちょうどいいです。

## 3. 時間ルールを入れる

夜間だけインターネットを止める。  
食事時間だけ止める。  
平日と休日で変える。

こういうルールはfirewallで作れます。

ただし、時間ルールも万能ではありません。

すでに張られている通信がしばらく残ることがあります。  
アプリによっては、再接続するまで挙動が分かりにくいこともあります。

家庭内では、

```txt
技術で完全に封じる
```

より、

```txt
ルールを説明して、必要な範囲でネットワークが補助する
```

くらいの考え方が長続きします。

## できることと限界

## できること

LN6001-JP側でできることは、次のようなものです。

- 子ども用SSIDを作る
- 子ども用ネットワークを別IPアドレス範囲にする
- 子ども用ネットワークだけFamily DNSを配布する
- 子ども用ネットワークからLAN内機器へ入れないようにする
- 夜間だけkids → wanの通信を止める
- Adblockで広告・トラッキング通信を減らす
- DNS Reportやログで問題を切り分ける
- 必要なドメインをAllowlistへ戻す

## できないこと

一方で、次のような限界があります。

- すべての不適切コンテンツを完全に止めることはできない
- モバイル回線へ切り替えられると、ルーター側の制御は効かない
- VPNアプリを使われると、DNSやfirewallルールを迂回される場合がある
- DoHを使うアプリやブラウザは、ルーターDNSを迂回する場合がある
- 端末内のアプリ利用時間まではルーターだけで細かく管理できない
- 学校管理端末は家庭側で変更できない設定がある
- 誤ブロックで学習サービスや認証が壊れることがある

ここは最初に家族内で共有しておくと、変な期待値になりにくいです。

ルーター側フィルタリングは万能ではありません。

でも、家庭内ネットワークを整理する道具としてはかなり役立ちます。

## 設定前にバックアップを取る

子ども用SSID、DHCP、DNS、firewallを触る前にバックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-family-filter-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/family-filter-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless dhcp firewall; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

バックアップにはSSID、Wi-Fiパスワード、DNS、ネットワーク設定が含まれることがあります。

SNS、公開リポジトリ、記事のスクリーンショットへそのまま出さないようにしてください。

バックアップがあると、人は少し落ち着いて設定できます。

OpenWrt系ルーターでは、これが本当に大事です。

## ステップ1: 子ども用ネットワークを作る

まず、子ども用のネットワークを作ります。

ここでは `kids` というinterfaceを作り、`192.168.30.0/24` を使います。

## LuCIでbridge deviceを作る

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-03.png)

5. **Save** をクリックする

Wi-Fiだけで子ども用SSIDを作る場合、Bridge portsは空のままで進めます。

有線ポートも子ども用ネットワークに分けたい場合は、VLAN設計が必要になります。

## LuCIでkids interfaceを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-04.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-05.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-06.png)

この時点では、まだDNS方針は入れていません。

まずは子ども用ネットワークを別IPアドレス範囲として作ります。

## ステップ2: 子ども用SSIDを作る

次に、子ども用端末を接続するSSIDを作ります。

1. **Network** → **Wireless** を開く
2. 5GHz radioの **Add** をクリックする
   - LN6001-JPでは、Linksys公式手順上は `wifi1` が5GHz radioです
3. **Interface Configuration** の **General Setup** で次を設定する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-07.png)

4. **Wireless Security** タブを開く
5. 次のように設定する

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-08.png)

6. **Save** をクリックする

ここで大事なのは、Networkを必ず `kids` にすることです。

SSID名が `MyHome_Kids` でも、Networkが `lan` のままだと、子ども用SSIDの見た目をしたメインLANになります。

ゲストWi-Fiと同じで、SSID名だけでは分離になりません。

## 子ども用SSIDは2.4GHzか5GHzか

子ども用端末がスマートフォン、タブレット、学校用PCなら、まず5GHzでよいことが多いです。

ゲーム機や古い端末、遠い部屋で使う端末がある場合は、2.4GHzも検討します。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-09.png)

最初は5GHzで作り、必要なら2.4GHzにも同じkids networkのSSIDを追加するくらいで大丈夫です。

## ステップ3: kids firewall zoneを作る

子ども用ネットワークからインターネットへ出られるようにしつつ、メインLANへは入れないようにします。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-10.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-11.png)

6. **Save** をクリックする

この設定では、kidsからwanへは出られます。

一方で、kidsからlanへのforwardingは作らないため、メインLAN側のNAS、PC、プリンター、ルーター管理画面へは届きにくくなります。

ただし、Inputを `REJECT` にしているため、DHCPとDNSを明示的に許可する必要があります。

ここはハマりやすいところです。

## ステップ4: kids用DHCPとDNSを許可する

kids端末がIPアドレスを受け取り、DNSを使えるようにします。

## DHCPを許可する

1. **Network** → **Firewall** → **Traffic Rules** を開く
2. **Add** をクリックする
3. 次のように設定する

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-12.png)

4. **Save** をクリックする

## DNSを許可する

1. **Traffic Rules** で **Add** をクリックする
2. 次のように設定する

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-13.png)

3. **Save** をクリックする

この2つがないと、kids zoneのInput `REJECT` によって、端末がIPアドレスを取れなかったり、DNSが使えなかったりします。

Wi-FiにはつながるのにWebが開けない時は、まずここを見ます。

## ステップ5: Family DNSを配布する

次に、子ども用ネットワークだけFamily DNSを配布します。

ここでは例として、Cloudflare for Familiesの「マルウェア + 成人向けコンテンツ」向けDNSを使います。

```txt
1.1.1.3
1.0.0.3
```

LuCIでは次のように設定します。

1. **Network** → **Interfaces** を開く
2. `kids` interfaceの **Edit** をクリックする
3. **DHCP Server** → **Advanced Settings** を開く
4. **DHCP Options** に次を追加する

```txt
6,1.1.1.3,1.0.0.3
```

5. **Save** をクリックする
6. 画面上部の保留中の変更を確認し、**Save & Apply** をクリックする

この設定は、kidsネットワークの端末へ、

```txt
DNSサーバーとして 1.1.1.3 と 1.0.0.3 を使ってね
```

と配るものです。

DNS設定を変更した後は、子ども用端末のWi-Fiを一度切断して再接続します。

端末によっては、再起動したほうが早いこともあります。

## Family DNSの選び方

Family DNSには複数の選択肢があります。

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-14.png)

最初は、仕組みが分かりやすいCloudflare for Familiesの `1.1.1.3` / `1.0.0.3` から試すのが扱いやすいです。

IPv6も子ども用ネットワークへ配る場合は、Cloudflare for FamiliesのIPv6アドレスも確認します。

```txt
2606:4700:4700::1113
2606:4700:4700::1003
```

ただし、IPv6をkidsへ配る場合は、IPv6側のfirewallやDNS方針も合わせて見ます。

最初はIPv4で構成を安定させ、その後でIPv6を検討するくらいでも大丈夫です。

## Google SafeSearchについて

Google SafeSearchの強制は、単にDNSサーバーを変えるだけではありません。

Googleの検索ドメインを `forcesafesearch.google.com` へ向けるDNS設定が必要になります。

家庭で最初に始めるなら、

```txt
kidsへFamily DNSを配る
端末側の検索・アプリ・OSのペアレンタル設定も併用する
```

くらいが現実的です。

SafeSearch強制は、必要になってから個別に検討します。

## AdblockとFamily DNSの使い分け

008の記事で扱ったAdblockと、この記事のFamily DNSは役割が少し違います。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-15.png)

DNSだけで全部をやろうとしないのがコツです。

子ども用端末では、Family DNS、OS側ペアレンタル設定、必要に応じてfirewall時間ルールを組み合わせると現実的です。

## Force Local DNSとの関係

008の記事でAdblockを導入し、Force Local DNSを有効にしている場合は注意が必要です。

Force Local DNSは、端末が外部DNSへ直接行こうとした時に、ルーター側DNSへ寄せるための設定です。

一方で、この記事のFamily DNS配布は、kids端末へ `1.1.1.3` / `1.0.0.3` のような外部DNSを使わせる方法です。

つまり、設定の組み合わせによっては意図がぶつかることがあります。

最初は次のどちらかに分けて考えると安全です。

## パターンA: kidsはFamily DNSを直接使う

- kids DHCPで `6,1.1.1.3,1.0.0.3` を配布する
- Force Local DNSの影響範囲を確認する
- 端末側でDNSがCloudflare Family DNSになっているか確認する

## パターンB: ルーター側Adblockを使う

- 端末にはLN6001-JPをDNSとして使わせる
- AdblockとAllowlistで調整する
- DoH対策は端末側設定やbanIPで考える

最初は、構成を混ぜすぎないほうがトラブルを切り分けやすくなります。

「kidsだけFamily DNS」「メインLANはAdblock」など、ネットワークごとに方針を決めてから設定します。

## ステップ6: 夜間だけkidsのインターネットを止める

DNSフィルタリングだけでなく、時間帯で通信を止めたい場合はfirewallのTraffic Ruleを使います。

ここでは例として、子ども用ネットワークからインターネットへの通信を、夜22:00から朝6:00まで止めます。

日付をまたぐ時間帯は、2つのルールに分けると分かりやすいです。

- 22:00〜23:59
- 00:00〜06:00

## LuCIで夜間ルールを作る

1. **Network** → **Firewall** → **Traffic Rules** を開く
2. **Add** をクリックする
3. 1つ目のルールを作る

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-16.png)

4. **Save** をクリックする
5. 2つ目のルールを作る

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-17.png)

6. **Save** をクリックする
7. ルールの順番を確認する
8. **Save & Apply** をクリックする

firewallの時間ルールは、ルールの順番や既存接続の扱いによって見え方が変わります。

すでに張られている通信がすぐ切れないこともあるため、動作確認では端末のWi-Fiを切断・再接続したり、アプリを再起動したりして確認します。

## CLIで夜間ルールを作る例

UCIで作る場合は、バックアップを取ってから行います。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.$(date +%Y%m%d-%H%M)

uci add firewall rule
uci set firewall.@rule[-1].name='Block-Kids-WAN-Night-1'
uci set firewall.@rule[-1].src='kids'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].target='REJECT'
uci set firewall.@rule[-1].start_time='22:00:00'
uci set firewall.@rule[-1].stop_time='23:59:59'

uci add firewall rule
uci set firewall.@rule[-1].name='Block-Kids-WAN-Night-2'
uci set firewall.@rule[-1].src='kids'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].target='REJECT'
uci set firewall.@rule[-1].start_time='00:00:00'
uci set firewall.@rule[-1].stop_time='06:00:00'

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

曜日を指定する場合は、LuCIで作ってから `uci show firewall` で実際の項目名を確認するのがおすすめです。

ファームウェアやLuCIのバージョンで表示や項目が変わることがあります。

## 設定後に確認すること

設定後は、実際の子ども用端末で確認します。

LuCIの画面だけ見て安心したくなりますが、最後は端末側で見ます。

## 子ども用SSIDに接続できるか

端末を `MyHome_Kids` へ接続します。

期待する状態は次です。

```txt
IP address: 192.168.30.x
Gateway: 192.168.30.1
DNS: 1.1.1.3 / 1.0.0.3
```

端末によってはDNS表示が見えにくいことがあります。

PCならOS側のDNS表示コマンドで確認します。

Mac:

```sh
scutil --dns | grep nameserver
```

Windows PowerShell:

```powershell
Get-DnsClientServerAddress
```

## インターネットへ出られるか

通常時間帯にWebサイトを開きます。

```sh
ping -c 4 8.8.8.8
ping -c 4 example.com
```

`8.8.8.8` に届くのに `example.com` が引けない場合はDNSを確認します。

どちらも届かない場合は、kids → wan forwardingやWAN接続を確認します。

## LAN内機器へ届かないか

子ども用SSIDにつないだ端末から、メインLAN側の機器へアクセスしてみます。

```txt
https://192.168.1.1
http://192.168.1.x
ssh root@192.168.1.1
```

期待する状態は、届かないことです。

ただし、プリンターやNASなどを子ども用端末から使わせたい場合は、必要な機器だけ例外ルールを作る必要があります。

最初から広く許可せず、必要なものだけ許可するほうが安全です。

## 夜間ルールが効くか

設定した時間帯に、子ども用SSIDからWebサイトを開いて確認します。

すぐに止まらない場合は、端末のWi-Fiを切断・再接続し、アプリやブラウザも再起動します。

時間ルールは「新しい通信を止める」動きに見えることがあるため、すでに張られた通信がしばらく残る場合があります。

## CLIで状態を確認する

メインLAN側からSSHでLN6001-JPへ入り、状態を確認します。

```sh
echo "### kids interface"
uci show network.kids
ifstatus kids

echo "### kids dhcp"
uci show dhcp.kids

echo "### firewall kids"
uci show firewall | grep -E 'kids|Allow-Kids|Block-Kids'

echo "### wireless kids"
uci show wireless | grep -E 'kids|MyHome_Kids|ssid|network'

echo "### dhcp leases"
cat /tmp/dhcp.leases

echo "### dns logs"
logread | grep -Ei 'dnsmasq|kids|dhcp' | tail -n 80
```

firewallの実際の展開を見たい場合は、次も使えます。

```sh
iptables-save 2>/dev/null | grep -i kids -A5 -B5
ip6tables-save 2>/dev/null | grep -i kids -A5 -B5
```

最初は `uci show firewall` だけでも十分です。

## よくある失敗と対処

## 子ども用端末がIPアドレスを取れない

期待するIPアドレスは `192.168.30.x` です。

取得できない場合は、次を確認します。

- 子ども用SSIDのNetworkが `kids` になっているか
- `kids` interfaceでDHCP Serverが有効になっているか
- `Allow-Kids-DHCP` があるか
- 子ども用端末のWi-Fiを一度切断・再接続したか

CLIでは次を確認します。

```sh
uci show dhcp.kids
logread | grep -i dnsmasq | tail -n 50
cat /tmp/dhcp.leases
```

## DNSフィルタリングが効かない

次を確認します。

- kids DHCP Optionsに `6,1.1.1.3,1.0.0.3` が入っているか
- 端末が新しいDHCP情報を受け取っているか
- 端末側に固定DNSが設定されていないか
- 端末側でDoHが有効になっていないか
- VPNアプリを使っていないか
- Force Local DNSやAdblock設定と衝突していないか

端末側でDNSを確認します。

Macなら:

```sh
scutil --dns | grep nameserver
```

Windowsなら:

```powershell
Get-DnsClientServerAddress
```

## 学校用サービスや学習アプリが動かない

Family DNSやAdblockで誤ブロックされている可能性があります。

まず切り分けます。

- 一時的に通常DNSへ戻す
- Adblockを止めて確認する
- DNS Reportやログを見る
- 必要なドメインをAllowlistへ追加する
- 子ども用SSIDではなくメインSSIDで試して差を見る

学校用PCは、学校側の管理ポリシーやVPN、独自DNSを使っている場合があります。

家庭側だけで原因を決めつけず、端末側の管理状況も確認します。

## 夜間ルールが効かない

時間ルールが効かない時は、次を確認します。

- LN6001-JPのタイムゾーンが `Asia/Tokyo` になっているか
- ルールのStart Time / Stop Timeが正しいか
- 日付をまたぐ時間帯を2つのルールに分けているか
- ルールの順番が適切か
- kids → wanの通信を許可するルールより前にブロックルールがあるか
- 既存接続が残っていないか

タイムゾーンは、LuCIの **System** → **System** で確認できます。

CLIでは次を見ます。

```sh
date
uci show system | grep timezone
uci show firewall | grep -E 'Block-Kids|start_time|stop_time'
```

## ゲームや動画だけ止まらない

ゲームや動画アプリは、すでに張った接続を維持したり、別の通信経路を使ったりすることがあります。

時間ルールを入れても、すぐに見た目が変わらない場合があります。

確認時は、次も試します。

- 端末のWi-Fiを切断・再接続する
- アプリを完全終了して起動し直す
- 端末を再起動する
- VPNやモバイル回線を使っていないか確認する

ルーター側でできる制限には限界があります。

ゲーム機やスマートフォンでは、OS側やアカウント側のペアレンタル設定も併用するのがおすすめです。

## 端末ごとに管理したい場合

子ども用SSID全体ではなく、端末ごとに管理したい場合は、固定DHCP割り当てを使います。

1. **Network** → **DHCP and DNS** を開く
2. **Static Leases** を確認する
3. 子ども用端末のMACアドレスに固定IPを割り当てる

例:

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-18.png)

そのうえで、firewallルールをsource IPごとに分けることもできます。

ただし、MACアドレスのランダム化が有効な端末では、端末側のプライベートアドレス設定によってMACアドレスが変わることがあります。

家庭内で固定DHCPを使う場合は、端末側のWi-Fi設定も確認してください。

## 子ども用ネットワークでプリンターを使わせたい場合

kids → lanを完全に拒否すると、メインLAN側のプリンターやNASへ届きません。

プリンターだけ使わせたい場合は、kids → lanを広く許可するのではなく、プリンターのIPアドレスだけ例外にします。

例:

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/table-19.png)

プリンターは機種によって使う通信が違います。

印刷だけ許可したつもりでも、探索、AirPrint、IPP、mDNSなどが絡むことがあります。

最初は無理に例外を増やさず、本当に必要な機器だけ検証しながら許可します。

## DoH、VPN、モバイル回線への考え方

子ども用フィルタリングでよくあるのが、DoH、VPN、モバイル回線による迂回です。

- DoH: ブラウザやアプリがHTTPSでDNS問い合わせを行う
- VPN: 端末から外部VPNへ入り、家庭内ルーターのDNSやfirewallを迂回する
- モバイル回線: Wi-Fiではなく携帯回線で通信する

ルーター側で全部を完全に防ぐのは難しいです。

家庭では、次の順番で考えるのが現実的です。

1. まず子ども用SSIDを作る
2. Family DNSやAdblockでDNS方針を決める
3. 端末側のDoH設定を確認する
4. 必要ならbanIPなどでDoH対策を検討する
5. OSやアカウント側のペアレンタル設定を併用する
6. モバイル回線やVPNについては家庭内ルールとして話す

技術で穴を全部埋めようとすると、親子どちらもしんどくなります。

ルーター側設定は、家庭内ルールの補助として使うのが長続きします。

## 家庭内で説明しやすいルールにする

フィルタリングは、設定が複雑になるほど説明しづらくなります。

家庭では、次のくらいの説明にしておくと運用しやすいです。

```txt
MyHome:
  親と家族共用端末用。通常のDNSとAdblock。

MyHome_Kids:
  子ども用端末。Family DNSを使う。
  夜22時から朝6時まではインターネットを止める。

MyHome_IoT:
  IoT機器用。必要最小限の通信だけ。
```

「なぜ分けているのか」を説明できることが大事です。

SSID名、パスワード、時間ルール、例外対応をメモしておくと、あとから家族内で共有しやすくなります。

## 運用メモのテンプレート

設定したら、次のようなメモを残しておきます。

```txt
子ども用SSID:
  MyHome_Kids

子ども用ネットワーク:
  192.168.30.0/24

DNS:
  Cloudflare for Families 1.1.1.3 / 1.0.0.3

時間ルール:
  22:00-06:00 は kids → wan を拒否

例外:
  学校用PCは必要に応じて別ルール
  プリンター利用は未設定 / 必要時に検討

確認日:
  2026-06-21

バックアップ:
  backup-LN6001-before-family-filter-20260621.tar.gz
```

家庭内ネットワークでも、こうしたメモを残しておくと、あとから自分が助かります。

未来の自分は、だいたい設定内容を忘れています。

## まとめ

家族向けフィルタリングは、いきなり強い制限を入れるより、子ども用SSIDとDNS方針を分けるところから始めるのがおすすめです。

LN6001-JPでは、OpenWrtベースの柔軟性を使って、次のような構成を作れます。

- 子ども用SSID `MyHome_Kids`
- 子ども用ネットワーク `192.168.30.0/24`
- kids → wan は許可
- kids → lan は拒否
- DHCPでFamily DNSを配布
- 必要なら夜間だけkids → wanを拒否
- AdblockやAllowlistで誤ブロックに対応
- DoH、VPN、モバイル回線の限界も理解する

家庭では、「完全に防ぐ」より「ルールを分かりやすくし、必要に応じて戻せる」ことが大事です。

まずは小さく始める。

子ども用端末、IoT、ゲストWi-Fi、家族共用端末を少しずつ整理していく。

このくらいの距離感が、無理なく続けやすいと思います。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/009/diagram-03.png)

## 次に読むなら

家族向けフィルタリングを入れたら、次は目的に合わせて進みます。

- [DNS広告ブロックの始め方](https://note.com/ikmsan/n/n4759cf81d0e1)
- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)

DNS広告ブロックをまだ入れていない人は、008の記事へ。

子ども用、IoT用、ゲストWi-Fi用のネットワークをもっと分けたい人は、VLANとfirewall zoneの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### 子ども用SSIDは作ったほうがいい？

作ったほうが管理しやすいです。

メインSSIDに全部混ぜると、DNS方針や時間ルールを端末ごとに管理する必要があります。

子ども用SSIDを分ければ、SSID単位でDNS、firewall、時間ルールを考えやすくなります。

### Family DNSだけで安全になる？

完全にはなりません。

Family DNSは成人向けコンテンツやマルウェアの一部をDNSで減らす仕組みです。

DoH、VPN、モバイル回線、アプリ内通信、同一ドメイン配信などには効きにくいことがあります。

OSやアカウント側のペアレンタル設定も併用するのがおすすめです。

### AdblockとFamily DNSはどちらを使えばいい？

目的が違います。

広告やトラッキングを減らしたいならAdblock。

子ども用端末で成人向けコンテンツやマルウェアを減らしたいならFamily DNS。

両方使う場合は、Force Local DNSやDHCP Optionsとの衝突に注意し、ネットワークごとに方針を決めます。

### 夜間制限は確実に効く？

多くの通信は止められますが、完全ではありません。

すでに張られている通信がしばらく残ることがあります。

確認時は端末のWi-Fiを切断・再接続したり、アプリを再起動したりして見ます。

### 学校用PCにも適用していい？

慎重に進めてください。

学校管理端末は、VPN、証明書、独自DNS、管理ポリシーを使っている場合があります。

いきなり強いフィルタリングをかけると、授業、認証、教材、提出システムが動かなくなることがあります。

まずはテストし、問題が出たらAllowlistや別SSIDで調整します。

### 子どもがモバイル回線やVPNを使ったらどうなる？

LN6001-JP側の制御は効きにくくなります。

家庭内Wi-Fiを通らない通信や、外部VPNへ入る通信は、ルーター側DNSやfirewallを迂回する場合があります。

そこは技術だけではなく、端末側のペアレンタル設定や家庭内ルールとして考える必要があります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターにAdblockを追加してセットアップする方法: https://support.linksys.com/kb/article/7050-jp/
- Velop WRT Pro 7 OpenWrt DNS-over-HTTPS（DoH）の設定方法: https://support.linksys.com/kb/article/7047-jp/
- OpenWrt Wiki - Parental controls: https://openwrt.org/docs/guide-user/firewall/fw3_configurations/fw3_parent_controls
- OpenWrt Wiki - DHCP and DNS configuration: https://openwrt.org/docs/guide-user/base-system/dhcp
- OpenWrt Wiki - DHCP and DNS examples: https://openwrt.org/docs/guide-user/base-system/dhcp_configuration
- Cloudflare 1.1.1.1 for Families: https://developers.cloudflare.com/1.1.1.1/setup/
- Google Search Help - Lock SafeSearch for devices you manage: https://support.google.com/websearch/answer/186669

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
