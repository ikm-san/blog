<!-- mirror-source: articles/031-mlo-setup.md -->

# MLOは速くなる魔法？｜Wi-Fi 7のマルチリンクをLN6001-JPで安全に試す【OpenWrt集中連載031】

Wi-Fi 7の目玉機能といえば、やっぱりMLOです。

MLO。  
Multi-Link Operation。

名前からして強そうです。

複数の無線リンクを同時に使える。  
5GHzと6GHzをうまく使える。  
速くなりそう。  
遅延も減りそう。  
なんか未来っぽい。

いいですよね。  
こういう機能、試したくなります。

でも、ここで一回だけ落ち着きます。

MLOは、

```txt
オンにしたら必ず爆速になる魔法
```

ではありません。

対応ルーター。  
対応端末。  
SSID設定。  
WPA3-SAE。  
同じパスワード。  
6GHz対応。  
端末側のOSやドライバ。  
電波環境。  
距離。  
壁。  
混雑。

このあたりがそろって、ようやく「お、MLOっぽい動きしてるね」と確認しやすくなります。

LN6001-JPはWi-Fi 7対応のトライバンドルーターなので、2.4GHz、5GHz、6GHzを使えます。

ただし、MLOを試す前に、まず普通のWi-Fiが安定していることが大事です。

```txt
5GHzは安定している？
6GHz対応端末で6GHzへつながる？
Wi-Fi 7対応端末はある？
WPA3-SAEで困る古い端末はない？
戻すためのバックアップはある？
```

ここを見ずにMLOをオンにすると、あとで原因が分かりにくくなります。

MLOが悪いのか。  
6GHzが遠いのか。  
WPA3に非対応なのか。  
端末ドライバの問題なのか。  
そもそも通常Wi-Fiから不安定だったのか。

一気に沼ります。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）でMLOを試す前提、LuCIでの有効化手順、確認ポイント、うまくいかない時の戻し方を整理します。

MLOは、焦って常用する機能ではありません。

まずは、小さく試して、合えば使う。

そのくらいの距離感がちょうどいいです。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、MLOを無理に常用することではありません。

まずは、ここまで分かればOKです。

- MLOが通常Wi-Fi接続とどう違うか分かる
- LN6001-JPでMLOを試す前に確認することが分かる
- MLOに必要なSSID、暗号化方式、Keyの条件が分かる
- LuCIでMLOを有効化する流れが分かる
- Associated Stationsで何を見ればよいか分かる
- MLOを無理に使わなくてもよいケースが分かる
- WPA3-SAEで古い端末がつながらない時の考え方が分かる
- MLO用SSIDと互換用SSIDを分ける判断ができる
- MLOを無効化して通常SSID運用へ戻せる

MLOは、Wi-Fi 7らしい楽しい機能です。

でも、家庭や店舗では、

```txt
速いかどうか
```

だけでなく、

```txt
家族やスタッフの端末が困らないか
戻せるか
安定しているか
```

もかなり大事です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-01.png)

## 先にざっくり結論

MLOは便利な機能ですが、最初から必須ではありません。

おすすめの順番はこれです。

1. **通常Wi-Fiで5GHz / 6GHzが安定しているか確認する**
2. **Wi-Fi 7 / MLO対応端末があるか確認する**
3. **MLO用にSSID、暗号化方式、Keyをそろえる**
4. **有線LAN接続のPCからLuCIでMLOを有効化する**
5. **Associated Stationsで複数radioに見えるか確認する**
6. **速度だけでなく、安定性・遅延・切り替わり方を見る**
7. **古い端末が困るなら、MLO用SSIDと互換用SSIDを分ける**
8. **合わなければ、MLOを無効化して通常SSIDへ戻す**

公式手順上、MLOを有効にする前提は次です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-01.png)

ただし、MLOを有効にしても、常に全帯域が同時に最大速度で動くわけではありません。

Associated Stationsで複数radioに見えても、実際にTX/RX rateが出るのはactive radioだけ、という見え方もあります。

MLOは、

```txt
Wi-Fi 7対応端末で複数リンクを使う可能性を開く設定
```

くらいに見ると現実的です。

## この記事の前提

この記事では、LN6001-JPをルーターモードで使い、Main側のWi-FiでMLOを試す前提にします。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-02.png)

Wi-Fiの基本設定、ゲストWi-Fi、IoT用SSID、Kids用SSIDがまだ不安定な場合は、先に通常Wi-Fiを安定させてください。

MLOは、通常Wi-Fi設定の上に乗せるものです。

土台がぐらついている状態でMLOを試すと、切り分けがかなり難しくなります。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-02.png)

## こういう人向けです

この記事は、次のような人向けです。

- Wi-Fi 7対応スマートフォンやPCを持っている
- 6GHz対応端末を持っている
- 5GHzと6GHzをまとめて使う挙動を試したい
- 低遅延や安定性の改善に興味がある
- 通常SSID運用とMLO運用を比べてみたい
- WPA3-SAEへ移行しても問題ない端末構成になっている
- MLO専用SSIDを作って安全に試したい
- 失敗した時に通常SSIDへ戻せるようにしておきたい

逆に、次のような場合は、急いでMLOを使わなくても大丈夫です。

- Wi-Fi 7対応端末がない
- 6GHz対応端末がない
- 古いIoT機器やプリンターが多い
- WPA3-SAEに非対応の端末が多い
- 家族やスタッフにSSID変更を説明しにくい
- 店舗や業務利用で、安定性を最優先したい
- 5GHzの通常SSIDで十分安定している

MLOは、使わないと損な機能ではありません。

使える環境がそろっているなら試すと面白い、くらいで大丈夫です。

## 先に読んでおくと楽な記事

MLOは少し上級寄りです。

先に次の記事を読んでおくと、かなり分かりやすくなります。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-03.png)

MLOは「通常Wi-Fi設定の上に乗る機能」です。

通常SSID、5GHz、6GHzが不安定な状態でMLOを試すと、原因が分かりにくくなります。

まず普通のWi-Fiを安定させる。

MLOはその次です。

## 用語ミニ解説

MLOまわりの言葉を、ざっくり整理しておきます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-04.png)

最初は、次だけで十分です。

```txt
MLO = 複数リンクを使うWi-Fi 7の仕組み
SSID / WPA3 / Keyをそろえる必要がある
対応端末がないと効果は見えにくい
```

## MLOとは

従来のWi-Fiでは、端末は基本的に1つの帯域へ接続します。

たとえば、2.4GHz、5GHz、6GHzのどれかです。

バンドステアリングでは、端末や電波状況に応じて接続先帯域を切り替えることがあります。

一方、MLOでは、対応端末が複数のリンクを使えるようになります。

ざっくり比較すると、次のようになります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-05.png)

ただし、MLOを有効にしただけで、必ず常に高速化するわけではありません。

端末。  
ドライバ。  
距離。  
障害物。  
混雑。  
帯域幅。  
暗号化方式。  
radio状態。

いろいろな条件で見え方が変わります。

MLOは「速度テストの数字を必ず上げる機能」というより、

```txt
Wi-Fi 7端末が複数リンクを使えるようにするための土台
```

と見たほうが安全です。

## MLOを試す前に確認すること

## 1. 端末がWi-Fi 7 / MLO対応か

Wi-Fi 7対応ルーターがあっても、端末側が対応していないとMLOにはなりません。

確認するものです。

- スマートフォンの仕様
- PCのWi-Fiチップ仕様
- OSの対応状況
- ドライバ更新状況
- 6GHz対応の有無
- MLO対応の明記があるか
- 端末メーカーのサポート情報

「Wi-Fi 7対応」と書かれていても、端末側の実装やOSの状態によって挙動差が出ることがあります。

まず端末側の仕様を確認します。

ここを飛ばすと、ルーター側をいくら触ってもMLOにならない、という悲しい時間が発生します。

## 2. 6GHzを使える端末か

MLOの検証では、6GHz対応端末があると分かりやすいです。

ただし、6GHzは対応端末が必要です。

古いスマートフォンやPCでは、6GHz SSIDが見えません。

6GHzが見えない場合に確認することです。

- 端末がWi-Fi 6EまたはWi-Fi 7対応か
- OSやドライバが6GHz対応か
- ルーター側の6GHz radioが有効か
- 端末が日本国内の6GHz利用条件に対応しているか
- ルーターと端末の距離が遠すぎないか
- 端末側で6GHz利用が無効化されていないか

6GHzは高速ですが、距離や障害物の影響を受けやすいです。

最初の確認では、ルーターの近くで試します。

壁2枚越しの部屋でいきなりMLO検証するのは、ちょっとハードモードです。

## 3. WPA3-SAEで困る端末がないか

MLOの前提として、暗号化方式はWPA3-SAEへそろえる必要があります。

ただし、古い端末、IoT機器、プリンター、ゲーム機の中にはWPA3-SAEに対応していないものがあります。

そのため、家庭や小さなオフィスでは次のどちらかを選びます。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-06.png)

古い端末が多い場合は、Main SSIDをいきなりWPA3-SAEへ変更しないほうが安全です。

MLO用に別SSIDを作って試すほうが切り分けやすくなります。

家庭では、こちらがかなりおすすめです。

## 4. 通常Wi-Fiが安定しているか

MLOを有効化する前に、通常Wi-Fiとして安定しているか確認します。

```sh
echo "### wireless config"
uci show wireless | grep -E 'ssid|encryption|network|disabled'

echo "### wifi status"
wifi status

echo "### wireless interfaces"
iw dev

echo "### wireless logs"
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 120
```

見るポイントです。

- Main SSIDへ接続できる
- 5GHzが安定している
- 6GHz対応端末で6GHzへ接続できる
- DNSやDHCPに問題がない
- ログにhostapdやradio再起動が大量に出ていない
- 端末が想定どおりのIPアドレスを取れている

通常Wi-Fiが不安定な状態でMLOを試すと、MLOが原因なのか、元からの問題なのか分からなくなります。

MLOの前に、普通のWi-Fi。

ここはかなり大事です。

## MLOを使うべきケース

MLOを試す価値があるのは、次のようなケースです。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-07.png)

MLOは、古い端末をたくさん混ぜた環境より、新しい端末中心の環境で試すほうがスムーズです。

特に、Wi-Fi 7対応PCやスマートフォンを持っているなら、MLO用SSIDを作って試す価値はあります。

## MLOを無理に使わなくてもよいケース

次のような場合は、通常SSID運用で十分なことがあります。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-08.png)

MLOは使わないと損、という機能ではありません。

通常SSIDで安定しているなら、そのままでも問題ありません。

Wi-Fi設定は、派手な機能より安定が正義の場面も多いです。

## 設定前にバックアップを取る

MLO設定では、Wi-Fi設定を変更します。

SSID、暗号化方式、Keyをそろえるため、古い端末がつながらなくなる可能性があります。

作業前にバックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-mlo-20260622.tar.gz
```

SSHでも状態メモを残します。

```sh
BACKUP_DIR="/root/mlo-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in wireless network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

wifi status > "$BACKUP_DIR/wifi-status-before.json" 2>/dev/null || true
iw dev > "$BACKUP_DIR/iw-dev-before.txt" 2>/dev/null || true
logread | grep -Ei 'wireless|wifi|hostapd|mlo' | tail -n 200 > "$BACKUP_DIR/wireless-log-before.txt"

ls -l "$BACKUP_DIR"
```

PCへコピーする場合です。

```sh
scp -r root@192.168.1.1:/root/mlo-before-* ~/Downloads/
```

LAN IPを変えている場合は、IPアドレスを置き換えます。

```sh
scp -r root@192.168.10.1:/root/mlo-before-* ~/Downloads/
```

このバックアップにはSSID、Wi-Fiパスワード、ネットワーク構成などが含まれることがあります。

公開リポジトリ、SNS、記事スクリーンショットへそのまま出さないでください。

MLOは楽しいですが、戻せる状態を作ってから触るのが安全です。

## MLO用SSID設計

MLOを試す時は、既存Main SSIDをそのままMLO用へ変えるか、MLO専用SSIDを作るかを決めます。

ここはかなり大事です。

## パターンA: Main SSIDをMLO化する

```txt
HomeWifi
  2.4GHz / 5GHz / 6GHz
  WPA3-SAE
  同じKey
```

向いているケースです。

- 家庭内端末が新しい
- WPA3-SAE非対応端末がほぼない
- SSIDを増やしたくない
- MainをWi-Fi 7中心へ寄せたい
- 自分でトラブル対応できる

注意点です。

- 古い端末やIoT機器がつながらなくなる可能性
- トラブル時に影響範囲が広い
- 家族の端末再接続が必要になる場合がある
- 店舗や業務利用では避けたほうがよい場合がある

家庭のMainをいきなりMLO化するのは、ちょっと攻めた設定です。

端末が新しめで、戻し方も分かっている人向けです。

## パターンB: MLO専用SSIDを作る

```txt
HomeWifi
  通常利用

HomeWifi_MLO
  MLO検証用
  WPA3-SAE
  Wi-Fi 7端末だけ接続
```

向いているケースです。

- 古い端末やIoT機器が多い
- MLOをまず試したい
- 家族全体へ影響させたくない
- 速度や遅延を比較したい
- 店舗や小さなオフィスで本番SSIDへ影響させたくない

家庭では、最初はMLO専用SSIDで試すほうが安全です。

問題なければ、あとからMainへ統合するか考えれば十分です。

まず小さく試す。

これが一番平和です。

## パターンC: 2.4GHz互換SSIDを分ける

WPA3-SAEへそろえると古い端末が困る場合は、2.4GHzに互換用SSIDを残す方法もあります。

```txt
HomeWifi_MLO
  5GHz / 6GHz中心
  WPA3-SAE
  Wi-Fi 7端末用

HomeWifi_IoT
  2.4GHz
  WPA2-PSK
  IoT / 古い端末用
```

この場合、公式MLO手順の「3つのSSIDをそろえる」構成とは違う運用になります。

ただし、家庭や店舗では、実用上こちらのほうが扱いやすいことがあります。

考え方はこうです。

```txt
MLO検証を優先する
  → 公式手順どおり3帯域をそろえる

古い端末互換を優先する
  → MLO用と互換用を分ける
```

どちらが正しいというより、目的が違います。

## 公式手順の前提

MLOを有効化する前に、次をそろえます。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-09.png)

LuCIでradioを確認します。

Linksys公式手順上、radio名は次の通りです。

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-10.png)

SSHで確認する場合は次です。

```sh
echo "### wireless devices"
uci show wireless | grep '=wifi-device'

echo "### SSID / encryption / network"
uci show wireless | grep -E 'ssid|encryption|key|network|disabled'
```

`key` にはWi-Fiパスワードが表示されることがあります。

出力を公開しないでください。

## LuCIでSSIDと暗号化をそろえる

作業は、有線LANで接続したPCから行うのがおすすめです。

Wi-Fi設定変更中に無線が一時的に切れることがあるためです。

## 2.4GHzを設定する

1. **Network** → **Wireless** を開く
2. 2.4GHz radioのSSIDを確認する
   - Linksys公式手順上、`wifi0` が2.4GHzです
3. 対象SSIDの **Edit** をクリックする
4. **Interface Configuration** → **General Setup** を開く
5. ESSIDをMLO用SSIDへそろえる

例:

```txt
HomeWifi_MLO
```

6. **Wireless Security** を開く
7. Encryptionを `WPA3-SAE` にする
8. Keyを入力する
9. Networkが `lan` であることを確認する
10. **Save** をクリックする

## 5GHzを設定する

1. 5GHz radioのSSIDを確認する
   - Linksys公式手順上、`wifi1` が5GHzです
2. 対象SSIDの **Edit** をクリックする
3. ESSIDを同じMLO用SSIDにする
4. Encryptionを `WPA3-SAE` にする
5. Keyを2.4GHzと同じにする
6. Networkを `lan` にする
7. **Save** をクリックする

5GHzは、多くの端末で主力になる帯域です。

MLO検証でも5GHzが安定しているかは重要です。

## 6GHzを設定する

1. 6GHz radioのSSIDを確認する
   - Linksys公式手順上、`wifi2` が6GHzです
2. 対象SSIDの **Edit** をクリックする
3. ESSIDを同じMLO用SSIDにする
4. Encryptionを `WPA3-SAE` にする
5. Keyを同じにする
6. Networkを `lan` にする
7. **Save** をクリックする

6GHzは高速ですが、距離や壁の影響を受けやすいです。

最初の確認では、ルーターの近くで接続します。

## Save & Apply前に確認する

3つのradioで設定をそろえたら、反映前にもう一度確認します。

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-11.png)

ここで1つでも不安があるなら、いったん止めて大丈夫です。

MLOは逃げません。

先にバックアップと戻し方を確認しましょう。

## MLOを有効化する

SSID、暗号化、Keyをそろえたら、MLOを有効化します。

1. 有線LANでPCをLN6001-JPへ接続する
2. LuCIへログインする
3. **Network** → **MLO** を開く
4. **AP MLO Enable** にチェックを入れる
5. **Save & Apply** をクリックする
6. Wi-Fiが再反映されるまで待つ
7. Wi-Fi 7対応端末でMLO用SSIDへ接続する

この時、Wi-Fi端末が一時的に切断されることがあります。

作業は有線LAN接続のPCから行うのが安全です。

スマホだけで作業していると、設定反映中に自分がWi-Fiから落ちて、ちょっと焦ります。

有線PC、かなり大事です。

## 設定後にLuCIで確認する

まず、LuCIで確認します。

1. **Network** → **Wireless** を開く
2. **Associated Stations** へ移動する
3. MLO対応端末が複数radioに見えるか確認する
4. TX/RX rateがどのradioで出ているか見る

複数radioに表示されても、すべてのradioで常に通信しているとは限りません。

実際にactiveなradioだけTX/RX rateが表示され、ほかのradioはinactiveに見えることがあります。

ここで、

```txt
複数radioに出ているのに、1つしかrateが出ていない
```

となっても、すぐ失敗と判断しないでください。

公式手順上も、active radioだけTX/RX rateが表示されることがあります。

MLOは、見え方がちょっとクセあります。

## SSHで確認する

SSHでは、まず読み取り中心で確認します。

```sh
echo "### wireless config"
uci show wireless | grep -Ei 'mlo|ssid|encryption|network|disabled'

echo "### wifi status"
wifi status

echo "### iw dev"
iw dev

echo "### associated stations by interface"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done

echo "### MLO / Wi-Fi logs"
logread | grep -Ei 'mlo|wifi|wireless|wlan|hostapd' | tail -n 120
```

見るポイントです。

- MLO用SSIDが各radioで同じか
- 暗号化がWPA3-SAEへそろっているか
- 端末が複数radioに見えるか
- TX/RX rateがどこで出ているか
- hostapdやMLO関連ログにエラーがないか
- 端末が想定どおりのIPを取れているか

DHCPも見ておきます。

```sh
echo "### DHCP leases"
cat /tmp/dhcp.leases
```

MLO以前にIPを取れていなければ通信できません。

Wi-Fi接続とIP取得は分けて見ます。

## 端末側で確認する

端末側でも確認します。

ただし、端末ごとにMLO表示の分かりやすさはかなり違います。

ルーター側のAssociated Stationsと、端末側表示を両方見るのがおすすめです。

## Windows

確認するものです。

- Wi-FiアダプターがWi-Fi 7対応か
- ドライバが最新か
- 接続規格がWi-Fi 7 / 802.11beとして表示されるか
- リンク速度
- 使用帯域
- チャネル
- セキュリティ方式

PowerShellでざっくり確認する例です。

```powershell
netsh wlan show interfaces
```

表示項目の中で、Radio type、Receive rate、Transmit rate、Channelなどを見ます。

Windowsでは、Wi-Fiドライバ更新で挙動が変わることがあります。

Wi-Fi 7対応PCなのに思ったように動かない場合は、OSとドライバも確認します。

## macOS

Macでは、メニューバーのWi-FiアイコンをOptionキーを押しながらクリックすると、接続情報を見られる場合があります。

確認するものです。

- PHY Mode
- Channel
- Tx Rate
- RSSI
- Noise
- Security

Wi-Fi 7対応機種かどうかは、Mac本体の仕様も確認します。

機種によっては、6GHz対応やWi-Fi 7対応状況が異なります。

## iPhone / iPad

iOS / iPadOSでは、設定画面で見える情報は限られます。

確認するものです。

- Wi-Fi 7対応機種か
- 6GHz対応か
- WPA3接続できているか
- 速度測定や体感で安定しているか
- ルーター側Associated Stationsで複数radioに見えるか

細かなMLO状態は、端末側よりルーター側で確認するほうが分かりやすいです。

## Android

Android端末は、メーカーやOSバージョンにより表示が違います。

確認するものです。

- Wi-Fi 7対応か
- 6GHz対応か
- 接続速度
- 周波数
- セキュリティ方式
- ルーター側Associated Stationsで複数radioに見えるか

Androidはメーカーごとの実装差が出やすいです。

同じWi-Fi 7対応でも、MLO表示や挙動が違うことがあります。

## 速度測定の考え方

MLOを有効にしたら、速度も測りたくなります。

分かります。

でも、インターネット速度テストだけでは、MLOの効果を正しく見にくいことがあります。

理由は次です。

- 回線速度が上限になる
- 測定サーバーの混雑が影響する
- 端末側の性能が上限になる
- WANが2.5GbEでもLANポートは1GbE
- Wi-Fi以外の要素がボトルネックになる
- MLOは速度だけでなく遅延や安定性にも関係する

確認するなら、次を分けて見ます。

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-12.png)

最初は、理論最大速度より、実用で不安定にならないかを見るほうが大事です。

数字だけ追いかけると、だいたい楽しくなって寝不足になります。

## LAN内で試す時の注意

LAN内速度を測る場合、相手側もボトルネックになります。

たとえば、有線NASがLN6001-JPのLANポートへ接続されている場合、LANポートは1GbEです。

つまり、Wi-Fi側が速くても、NAS側が1GbEで頭打ちになることがあります。

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-13.png)

MLOを評価する時は、

```txt
何がボトルネックか
```

を分けて考えます。

## MLOで古い端末がつながらない時

MLO設定ではWPA3-SAEへそろえる必要があります。

そのため、古い端末やIoT機器がつながらなくなることがあります。

## 対処案

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-14.png)

家庭では、MLO用SSIDと互換用SSIDを分けるのが現実的なことが多いです。

例:

```txt
HomeWifi
  通常利用

HomeWifi_MLO
  Wi-Fi 7端末用

HomeWifi_IoT
  古い端末・IoT用
```

MLOのために家中のIoTが悲鳴を上げるのは、あまり幸せではありません。

## MLOで速度が上がらない時

速度が上がらない時も、すぐ失敗とは限りません。

確認することです。

- 端末が本当にWi-Fi 7 / MLO対応か
- 端末が6GHz対応か
- ルーターから近い位置で確認しているか
- Associated Stationsで複数radioに見えるか
- active radioにTX/RX rateが出ているか
- 速度測定のボトルネックがWANやサーバーではないか
- LAN側が1GbEで頭打ちになっていないか
- 5GHz単体でも十分速い環境ではないか
- 端末側ドライバやOSが最新か

MLOは、常に「速度テストの数字が大きくなる」機能ではありません。

混雑時や移動時、複数リンクの使い分けで安定性を感じる場面もあります。

速度が上がらないから即オフ、ではなく、まず条件を見ます。

ただし、実用上差がないなら通常SSID運用でOKです。

## MLOでWi-Fiが不安定になった時

MLO有効化後に不安定になったら、まず通常Wi-Fiへ戻して切り分けます。

確認します。

```sh
echo "### wireless logs"
logread | grep -Ei 'mlo|wifi|wireless|hostapd|disconnect|deauth|auth' | tail -n 150

echo "### associated stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done
```

次に、LuCIの **Network** → **MLO** で **AP MLO Enable** を外し、**Save & Apply** します。

その後、Wi-Fiを再読み込みします。

```sh
wifi reload
```

MLOを無効化して安定するなら、MLO設定、端末相性、WPA3-SAE、6GHz、ドライバ周辺を疑います。

不安定なまま根性で使い続ける必要はありません。

## MLOを無効化して通常運用へ戻す

MLOを使わないと決めた場合は、通常SSID運用へ戻します。

## LuCIで戻す

1. 有線LANでPCを接続する
2. LuCIへログインする
3. **Network** → **MLO** を開く
4. **AP MLO Enable** のチェックを外す
5. **Save & Apply** をクリックする
6. **Network** → **Wireless** を開く
7. 必要に応じてSSIDや暗号化方式を戻す
8. Wi-Fi端末を再接続する

## wireless設定だけ戻す

作業前に保存した `wireless` を戻す場合です。

```sh
cp /root/mlo-before-YYYYMMDD-HHMM/wireless /etc/config/wireless
wifi reload
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えてください。

## LuCIバックアップから戻す

LuCIバックアップから戻す場合は、**System** → **Backup / Flash Firmware** から復元します。

復元後、LAN IPやSSIDがバックアップ時点へ戻ることがあります。

復元後にLuCIへ入れない場合は、PC側のDefault Gatewayを確認してください。

管理画面の住所が戻っただけ、ということもあります。

## MLO運用メモ

MLOを使うなら、設定メモを残しておくと後で楽です。

```txt
MLO運用メモ:

製品:
  LN6001-JP

ファームウェア:
  1.2.0.15

MLO:
  enabled: yes

MLO SSID:
  HomeWifi_MLO

対象radio:
  2.4GHz / 5GHz / 6GHz

暗号化:
  WPA3-SAE

互換用SSID:
  HomeWifi_IoT
  2.4GHz / WPA2-PSK

確認端末:
  Wi-Fi 7対応スマートフォン
  Wi-Fi 7対応PC

確認:
  Associated Stationsで複数radio表示あり
  TX/RX rateはactive radioのみ表示
  6GHz近距離確認OK

戻し方:
  Network -> MLO -> AP MLO Enableを外す
  または /root/mlo-before-.../wireless を戻す

バックアップ:
  backup-LN6001-before-mlo-20260622.tar.gz
```

MLOは端末相性や環境差が出やすいです。

どの端末で確認したかを残しておくと、あとからかなり助かります。

## よくある失敗

### SSIDは同じだが暗号化方式が違う

MLOでは、SSIDだけでなく暗号化方式もそろえる必要があります。

確認します。

```sh
uci show wireless | grep -E 'ssid|encryption|key'
```

`key` にはWi-Fiパスワードが出ることがあります。

共有時は伏せてください。

### Keyが1つだけ違う

SSID名とWPA3-SAEがそろっていても、Keyが1つだけ違うと条件がそろいません。

LuCIで3つのradioをそれぞれ確認します。

```txt
2.4GHz: HomeWifi_MLO / WPA3-SAE / same key
5GHz:   HomeWifi_MLO / WPA3-SAE / same key
6GHz:   HomeWifi_MLO / WPA3-SAE / same key
```

この「完全一致」が大事です。

### WPA3-SAEにしたらIoT機器がつながらない

古いIoT機器はWPA3-SAEに対応していないことがあります。

対処:

- IoT用2.4GHz SSIDを別に作る
- IoT用SSIDはWPA2-PSKなど互換性重視にする
- MLO用SSIDはWi-Fi 7端末専用にする

### 6GHz SSIDが見えない

確認することです。

- 端末がWi-Fi 6E / Wi-Fi 7対応か
- 6GHz radioが有効か
- 端末OSやドライバが最新か
- ルーターの近くで確認しているか
- MLO用SSIDと通常SSIDを混同していないか

6GHzが見えない時は、まず端末側の対応を確認します。

ルーターだけではどうにもならないことがあります。

### Associated Stationsに複数radioが出ない

確認することです。

- 端末がMLO対応か
- SSID、WPA3-SAE、Keyが3帯域でそろっているか
- AP MLO Enableが有効か
- 端末側で一度SSIDを削除して再接続したか
- 端末がルーターから遠すぎないか
- 6GHz対応端末か
- 端末側ドライバやOSが最新か

端末によっては、Wi-Fi 7対応でもMLO表示が分かりにくいことがあります。

ルーター側Associated Stationsと端末側情報の両方を見ます。

### MLO有効後に家族の端末がつながらない

Main SSIDをMLO用に変更した場合、WPA3-SAE非対応端末がつながらない可能性があります。

家庭では、MLO専用SSIDを作るほうが安全です。

```txt
HomeWifi
  家族用通常SSID

HomeWifi_MLO
  Wi-Fi 7端末用

HomeWifi_IoT
  古い端末・IoT用
```

MLOの検証で家族全員のWi-Fiを巻き込むと、家庭内の空気がちょっと悪くなります。

小さく試しましょう。

### MLOとゲストWi-Fi・IoTを混ぜてしまう

MLOはMainや専用検証SSIDで試すのがおすすめです。

Guest、Kids、IoTは、目的が違います。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/table-15.png)

MLOをゲストWi-FiやIoTへ無理に入れる必要はありません。

目的ごとにSSIDを分けます。

## 店舗や小さなオフィスで使う場合

店舗や小さなオフィスでは、MLOはかなり慎重に扱ったほうがよいです。

業務で大事なのは、速度より安定性です。

Staff用端末が新しく、Wi-Fi 7対応端末があるなら、MLO専用SSIDを作って検証するのはありです。

ただし、次の端末を巻き込まないようにします。

- POS
- 決済端末
- レシートプリンター
- 監視カメラ
- 録画機
- スマートロック
- 古い業務タブレット
- 来客用ゲストWi-Fi

店舗では、次のように分けるのが無難です。

```txt
Shop_Staff
  業務用通常SSID

Shop_Staff_MLO
  Wi-Fi 7対応スタッフ端末の検証用

Shop_Guest
  ゲストWi-Fi

Shop_Device
  カメラ・設備機器用
```

MLOを業務本番SSIDへ入れるのは、検証してからで十分です。

## MLO変更時のチェックリスト

作業前です。

```txt
□ 有線LANのPCで作業している
□ LuCIバックアップをPCへ保存した
□ /etc/config/wireless の控えを取った
□ Wi-Fi 7 / MLO対応端末がある
□ 6GHz対応端末がある
□ WPA3-SAE非対応端末の影響を確認した
□ MLO用SSIDをMainと分けるか決めた
□ 戻し方を確認した
```

作業後です。

```txt
□ MLO用SSIDへ接続できた
□ Associated Stationsで複数radio表示を確認した
□ active radioのTX/RX rateを確認した
□ DHCPでIPを取れている
□ Webサイトが開ける
□ 速度だけでなく安定性も確認した
□ 古い端末やIoT機器に影響がない
□ 不安定ならMLOを無効化して戻した
□ 動作確認後のバックアップを取った
```

## まとめ

MLOは、Wi-Fi 7らしい機能です。

ただし、家庭や小さなオフィスでは、最初から必須ではありません。

大事なのは次です。

1. まず通常Wi-Fiが安定していることを確認する
2. Wi-Fi 7 / MLO対応端末があるか確認する
3. SSID、WPA3-SAE、Keyをそろえる
4. 有線LAN接続のPCからLuCIで設定する
5. **Network → MLO → AP MLO Enable** で有効化する
6. Associated Stationsで複数radio表示とactive TX/RX rateを見る
7. 古い端末が困るならMLO専用SSIDを分ける
8. 効果が薄ければ通常SSID運用へ戻す
9. 作業前にバックアップを取る
10. 無線の国設定、送信出力、DFS関連の値は変更しない

MLOは「必ず速くなる魔法」ではありません。

でも、Wi-Fi 7対応端末があるなら試す価値はあります。

最初は、Main全体を変えるより、MLO専用SSIDで小さく試す。

うまくいけば使い続ける。

不安定なら通常SSIDへ戻す。

このくらいの距離感が、LN6001-JPでMLOを試す時にはちょうどよいです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/031/diagram-03.png)

## 次に読むなら

MLOを試す前後には、次の記事も合わせて読むと整理しやすいです。

- [Wi-Fi設定](https://note.com/ikmsan/n/n4dbf8f72744c)
- [ハードウェア概要](https://note.com/ikmsan/n/nf5e2df270ea3)
- [SSH基本コマンド集](https://note.com/ikmsan/n/n99897dd3cae6)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

まず通常の2.4GHz / 5GHz / 6GHz設定を整理したい人はWi-Fi設定へ。

MLO後につながらない端末が出た場合は、つながらない時の切り分けとバックアップ記事へ進むと戻しやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

MLO画面、Wi-Fi radio名、`wifi status`、`iw dev`、Associated Stationsの見え方、端末側のMLO表示は、ファームウェア、端末、OS、ドライバ、電波環境によって変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずバックアップを取り、Linksys公式サポートや端末メーカーの最新情報も確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### MLOを有効にすると必ず速くなりますか？

必ず速くなるわけではありません。

端末がWi-Fi 7 / MLO対応で、SSIDや暗号化条件がそろい、電波環境も合っている時に効果を確認しやすくなります。

速度だけでなく、遅延や安定性も含めて見ます。

### MLOには何が必要ですか？

公式手順では、3つのSSID名を完全に同じにし、3つともWPA3-SAEにし、3つとも同じKeyを使う必要があります。

そのうえで、LuCIの **Network → MLO** から **AP MLO Enable** を有効にします。

### Wi-Fi 7端末なら必ずMLOになりますか？

必ずとは言えません。

端末側のMLO対応、OS、ドライバ、接続状態、電波環境によって変わります。

Wi-Fi 7対応とMLO動作は分けて確認します。

### 古い端末はMLO有効後も使えますか？

MLO自体は使えません。

また、MLO用にWPA3-SAEへそろえると、WPA3非対応の古い端末やIoT機器がつながらないことがあります。

その場合は、互換用SSIDを別に作るのがおすすめです。

### 2.4GHzもMLOに含めるべきですか？

公式MLO手順では、3つのSSID、WPA3-SAE、Keyをそろえることが前提です。

ただし、家庭では古い端末やIoT機器向けに2.4GHzを別SSIDへ分ける運用も現実的です。

MLO検証を優先するなら公式手順どおりそろえます。

互換性を優先するならMLO用SSIDとIoT用SSIDを分けます。

### MLOはゲストWi-FiやIoTにも使うべきですか？

基本的にはMainまたはMLO専用SSIDで試すのがおすすめです。

ゲストWi-FiはLAN分離、IoTは互換性と安定性が主目的です。

MLOをゲストWi-FiやIoTへ無理に入れる必要はありません。

### Associated Stationsで複数radioに見えるのに、TX/RX rateが1つだけです

それだけで失敗とは限りません。

複数radioに見えても、実際にactiveなradioだけTX/RX rateが表示されることがあります。

端末側の挙動や通信状態も合わせて確認します。

### MLOをやめたい時はどう戻しますか？

LuCIの **Network → MLO** で **AP MLO Enable** を外し、**Save & Apply** します。

SSIDや暗号化方式も戻したい場合は、作業前に取ったバックアップの `/etc/config/wireless` を戻すか、LuCIから通常SSIDへ戻します。

### MLO検証で一番安全なやり方は？

MLO専用SSIDを作り、有線LAN接続のPCから設定する方法です。

Main SSIDをいきなりMLO化すると、家族や古い端末へ影響が出やすくなります。

### 店舗でMLOを使うべきですか？

試すなら、まずStaff向けのMLO専用SSIDで検証するのがおすすめです。

POS、決済端末、カメラ、スマートロック、ゲストWi-Fiを巻き込まないようにしてください。

店舗では速度より安定運用を優先します。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- How to enable MLO on your Linksys OpenWRT router: https://support.linksys.com/kb/article/8652-en/?section_id=175
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/

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
