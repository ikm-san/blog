<!-- mirror-source: articles/006-wifi-band-settings.md -->

# Wi-Fiはまず5GHz主力でOK｜LN6001-JPの2.4GHz/5GHz/6GHzを無理なく使い分ける【OpenWrt集中連載006】

Wi-Fi 7ルーターを買うと、ちょっと夢を見たくなります。

6GHzを使えば全部速くなるのでは？  
MLOを有効にすれば最強なのでは？  
2.4GHz、5GHz、6GHzを全部まとめたら、端末がいい感じに選んでくれるのでは？

分かります。

せっかくWi-Fi 7対応ルーターを買ったなら、新しい機能を使いたくなります。

でも、家庭や小さなオフィスで最初に大事なのは、最大速度を追いかけることではありません。

まず大事なのは、

```txt
どの端末を、どのSSIDにつなぐか
```

です。

スマートフォンやPCは5GHz。  
古いプリンターやスマート家電は2.4GHz。  
Wi-Fi 6E/7対応端末だけ6GHz。  
ゲストWi-Fiは、あとでちゃんと本体LANから分ける。

このくらい整理できているだけで、Wi-Fiのトラブルはかなり切り分けやすくなります。

LN6001-JPは、2.4GHz、5GHz、6GHzのトライバンドに対応したOpenWrtベースのWi-Fi 7ルーターです。

だからこそ、最初から全部を盛るより、役割ごとに分けて始めるほうが扱いやすいです。

この記事では、LN6001-JPで2.4GHz / 5GHz / 6GHzをどう使い分けるか、LuCIでSSIDをどう設定するか、MLOを最初から有効にすべきかを、家庭・小さなオフィス目線で整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、Wi-Fi 7の全機能を使い切ることではありません。

まずは、ここまで分かればOKです。

- 2.4GHz、5GHz、6GHzの役割をざっくり分けられる
- 家庭や小さなオフィスでSSIDをどう分けるか考えられる
- LuCIでSSID、暗号化、Wi-Fiパスワードを変更できる
- 6GHzが見えない時に、まず端末対応を疑える
- MLOは最初から必須ではないと分かる
- Wi-Fi設定変更前にバックアップを取る理由が分かる

Wi-Fi 7は楽しいです。

でも、最初からMLOまで全部やらなくて大丈夫です。

まずは、5GHzを主力にして、2.4GHzと6GHzを役割分担させるところから始めましょう。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-01.png)

## 先にざっくり結論

最初はこの3つで十分です。

1. **5GHzを主力SSIDにする**
2. **2.4GHzはIoTや古い端末向けに分ける**
3. **6GHzやMLOは対応端末がそろってから試す**

最初からすべての帯域を同じSSIDにまとめたり、MLOを有効にしたりする必要はありません。

むしろ、初期段階では次のようなシンプル構成のほうが管理しやすいです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-01.png)

Wi-Fi設定で一番大事なのは、

```txt
速そうな帯域へ全部つなぐこと
```

ではなく、

```txt
端末ごとの置き場所を決めること
```

です。

ここが整理できていると、あとでゲストWi-Fi、IoT分離、MLO、VLANへ進む時もかなり楽になります。

## こういう人向けです

この記事は、次のような人向けです。

- 2.4GHz、5GHz、6GHzのどれを使えばいいか迷っている
- Wi-Fi 7ルーターを買ったけど、SSID設計で止まっている
- IoT機器だけ別SSIDにまとめたい
- 6GHzを使いたいが、どの端末が対応しているか分からない
- MLOを有効にすべきか迷っている
- 家庭や小さなオフィスで、分かりやすいSSID構成にしたい
- ゲストWi-Fiを作る前に、まず帯域設計を決めたい
- LuCIでWi-Fi設定を変える流れを確認したい

逆に、すでにWi-Fi 7 / MLO / WPA3-SAEまわりを理解している人には、かなり基本寄りです。

でも、家庭や店舗ではこの基本がかなり効きます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-02.png)

## 最初に言葉だけそろえる

Wi-Fi設定で出てくる言葉を、ざっくりそろえておきます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-02.png)

最初は、次だけ覚えておけば大丈夫です。

```txt
2.4GHz = IoT・古い端末
5GHz = 普段使いの主力
6GHz = 新しい高速端末
MLO = Wi-Fi 7端末が増えてからでOK
```

## 帯域ごとの特徴

2.4GHz、5GHz、6GHzは、数字が大きいほど必ず良いわけではありません。

それぞれ得意なことが違います。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-03.png)

Wi-Fi設定で迷った時は、

```txt
どの帯域が速いか
```

より、

```txt
どの端末をどこへ置くと分かりやすいか
```

で考えると整理しやすくなります。

スマートフォンやPCを5GHzへ。  
IoTや古い端末を2.4GHzへ。  
Wi-Fi 6E/7対応の新しい端末だけ6GHzへ。

まずはこれで十分です。

## 2.4GHz: IoTと古い端末の受け皿

2.4GHzは、古い端末やIoT機器で使いやすい帯域です。

壁や床の影響を受けにくく、遠い部屋まで届きやすい傾向があります。

一方で、混みやすいです。

電子レンジ、Bluetooth、近隣Wi-Fiなどの影響も受けます。

つまり、2.4GHzは「高速道路」ではなく「生活道路」みたいなものです。

速さより、対応機器の多さと届きやすさが強みです。

## 2.4GHzに向いている端末

- スマートスピーカー
- スマート照明
- センサー類
- 古いプリンター
- 古いゲーム機
- Wi-Fi 5GHz非対応のIoT機器
- ルーターから遠い場所にある低速端末

## 2.4GHzのSSID例

IoT用として分けるなら、こんな名前が分かりやすいです。

```txt
MyHome_IoT
```

店舗なら、たとえばこうです。

```txt
Shop_Device
```

SSID名は、できれば英数字中心にします。

古いIoT機器では、日本語SSIDや記号の多いSSIDで相性が出ることがあります。

## 2.4GHzの暗号化

2.4GHzをIoT用にする場合は、互換性重視で考えます。

新しい端末だけならWPA3も選択肢になります。

ただ、IoT機器はWPA2前提のものも多いです。

最初はWPA2-PSKなど、接続できる端末が多い設定を選ぶほうが現実的です。

ここで全部WPA3にすると、

```txt
スマート家電だけつながらない
古いプリンターだけ見えない
ゲーム機だけ怒っている
```

みたいなことが起きがちです。

IoTは、速さより「ちゃんとつながること」を優先します。

## 2.4GHzで注意すること

2.4GHzへ何でも詰め込むと混雑します。

スマートフォン、PC、テレビ、ゲーム機など、速度がほしい端末は5GHzまたは6GHzへ寄せます。

2.4GHzは、

```txt
古い端末とIoTを受け止める場所
```

と考えると分かりやすいです。

## 5GHz: 日常利用の主力

5GHzは、家庭や小さなオフィスで主力にしやすい帯域です。

2.4GHzより速度が出やすく、6GHzより対応端末が多い。

つまり、いちばん現実的な主役です。

## 5GHzに向いている端末

- ノートPC
- デスクトップPC
- スマートフォン
- タブレット
- テレビ
- ゲーム機
- Web会議用端末

## 5GHzのSSID例

メインSSIDは、5GHzを中心に考えると扱いやすいです。

```txt
MyHome
```

小さなオフィスなら:

```txt
Office_Staff
```

店舗なら:

```txt
Shop_Staff
```

家族やスタッフが普段使う端末は、まずこのSSIDにつなぎます。

## 5GHzの暗号化

新しい端末中心ならWPA3-SAEも選択肢になります。

ただし、古い端末が混ざる場合は、互換性を優先した設定のほうがトラブルが少ないことがあります。

ここは環境次第です。

家庭なら、

```txt
新しいスマホ・PC中心 → WPA3も検討
古い端末やゲーム機が混ざる → 互換性重視
```

くらいで見ればOKです。

## 5GHzで注意すること

5GHzは、2.4GHzより壁や床の影響を受けやすいです。

また、チャネルやDFSの影響で一時的に待機や切り替えが発生することがあります。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

国内向け製品としての前提を維持したうえで、SSID設計と接続先の整理を行います。

## 6GHz: 新しい高速端末の専用レーン

6GHzは、Wi-Fi 6E / Wi-Fi 7対応端末で使える帯域です。

混雑が少なく、高速な通信を期待できます。

ただし、対応端末が必要です。

ここが大事です。

6GHzは、ルーター側が対応していても、端末側が対応していなければ見えません。

古いスマートフォンやPCでは、6GHz用SSIDが一覧に出ないことがあります。

## 6GHzに向いている端末

- Wi-Fi 6E対応スマートフォン
- Wi-Fi 7対応スマートフォン
- Wi-Fi 6E/7対応ノートPC
- ルーターに近い場所で使う高速端末

## 6GHzのSSID例

対応端末向けに分けるなら、こんな名前が分かりやすいです。

```txt
MyHome_6G
```

小さなオフィスなら:

```txt
Office_6G
```

6GHzは、対応端末だけの専用レーンとして使うイメージです。

## 6GHzの暗号化

6GHzでは、WPA3-SAEを前提に考えます。

古い端末を6GHzへつなぐことは基本的にないので、ここは新しい端末向けに割り切りやすいです。

## 6GHzで注意すること

6GHzは高速ですが、距離や壁の影響を受けやすいです。

「6GHzなら家中どこでも爆速」というより、

```txt
ルーターに近い場所で、新しい端末を混雑の少ない帯域へ逃がす
```

と考えるほうが現実的です。

6GHzが見えない時は、まず端末側の対応を疑います。

ルーターが悪いとは限りません。

## まずおすすめするSSID構成

初期設定直後は、次のような構成が扱いやすいです。

### 家庭向けシンプル構成

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-04.png)

家庭では、まずこれだけでかなり整理できます。

ゲストWi-Fiをすぐに作りたい場合でも、最初はメインSSIDとIoT用SSIDを安定させ、そのあとゲストWi-Fiの記事でnetworkとfirewallまで含めて作るほうが安全です。

### 家庭向け + ゲストWi-Fi構成

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-05.png)

ここで大事なのは、`MyHome_Guest` というSSIDを作っただけでは、まだ安全なゲストWi-Fiとは限らないことです。

本体LANから分けるには、後続記事で扱うguest network、DHCP、firewall zoneが必要です。

### 小さなオフィス向け

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-06.png)

小さなオフィスでは、SSID名だけでなく、どのSSIDがどのnetworkに紐づくかが重要になります。

最初はSSIDを整理し、次のステップでVLANやfirewall zoneを使ってStaff、Guest、Deviceを分ける流れが扱いやすいです。

## 同じSSIDにするか、分けるか

Wi-Fi設定でよく迷うのが、2.4GHz / 5GHz / 6GHzを同じSSIDにするか、別々のSSIDにするかです。

## 同じSSIDにする場合

同じSSIDにすると、端末側が自動で帯域を選びます。

ユーザーに案内するWi-Fi名が少なくなるので、シンプルに見えます。

ただし、端末がどの帯域へつながっているか分かりにくくなることがあります。

```txt
あれ、このスマホは5GHz？
それとも2.4GHz？
なんでテレビだけ遅い？
```

こういう時に、切り分けしにくくなります。

## 分ける場合

SSIDを分けると、端末を意図した帯域へつなぎやすくなります。

```txt
IoTは2.4GHz
PCやスマホは5GHz
新しい端末は6GHz
```

このように整理できます。

その代わり、Wi-Fi名は増えます。

家族やスタッフへ説明する手間も少し増えます。

## 最初は分けたほうが分かりやすい

この連載では、最初は分ける構成をおすすめします。

理由は、トラブル時に切り分けしやすいからです。

「この端末は2.4GHzにいる」  
「このPCは5GHzにいる」  
「このスマートフォンは6GHzにいる」

これが分かるだけで、原因の見通しがかなり良くなります。

慣れてきたら、使い方に合わせてSSIDを統合しても構いません。

最初は分ける。  
慣れたらまとめる。

この順番が安全です。

## LuCIでWi-Fi設定を変更する

Wi-Fi設定は、LuCIの **Network → Wireless** から変更します。

公式手順でも、Wi-Fi設定変更時は有線LAN接続したPCから作業することが案内されています。

これはかなり大事です。

Wi-Fi設定を変更すると、作業中の無線接続が一時的に切れることがあります。

スマホだけで作業していると、

```txt
SSID変えた瞬間に画面が固まった
今、何が反映されたのか分からない
```

となりがちです。

できれば、有線LANでつないだPCから作業してください。

## radioの対応

公式手順では、LN6001-JPのradioは次のように案内されています。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-07.png)

LuCIの表示を見ながら、対象radioを間違えないようにします。

## 既存SSIDを編集する

1. ブラウザで `https://192.168.1.1` を開き、LuCIへログインする
2. **Network** → **Wireless** を開く
3. 変更したいradioの **Edit** をクリックする
4. **Interface Configuration** の **ESSID** にSSID名を入力する
5. **Wireless Security** タブを開く
6. **Encryption** を選ぶ
7. **Key** にWi-Fiパスワードを入力する
8. **Save** をクリックする
9. 必要なradioで同じ作業を繰り返す
10. 画面上部の保留中の変更を確認し、**Save & Apply** で反映する
11. Wi-Fi接続が切れた場合は、新しいSSIDとパスワードで再接続する

反映中は、Wi-Fiが一時的に切れます。

これは正常です。

ここで焦ってリセットボタンを押さなくて大丈夫です。

## 新しいSSIDを追加する

ゲストWi-FiやIoT用SSIDを追加する場合は、対象radioで新しいWi-Fiインターフェースを追加します。

1. **Network** → **Wireless** を開く
2. 追加したいradioの **Add** をクリックする
3. ESSIDを入力する
4. **Wireless Security** で暗号化方式とKeyを設定する
5. **Network** で接続先networkを選ぶ
6. **Save** → **Save & Apply** で反映する

ここで注意です。

SSIDを追加しただけでは、ネットワーク分離まで完成しません。

たとえば、`MyHome_Guest` というSSIDを作っても、それが `lan` networkに紐づいていれば、見た目だけゲストWi-Fiで、中身はMain LANと同じです。

ゲストWi-Fiを本当に分けたい場合は、後続記事で扱うguest network、DHCP、firewall zoneまで設定します。

## 暗号化方式の考え方

Wi-Fiの暗号化は、端末の対応状況に合わせて考えます。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/table-08.png)

WPA3は新しい方式です。

ただし、古いIoT機器や古いPCが対応していない場合があります。

「セキュリティを上げたいから全部WPA3」にすると、古い端末がつながらなくなることがあります。

なので、SSIDごとに役割を分けます。

```txt
古い端末・IoT → 2.4GHz / WPA2-PSK
新しいPC・スマホ → 5GHz / WPA3も検討
6GHz端末 → 6GHz / WPA3-SAE
```

これが現実的です。

## MLOは最初から有効にすべきか

MLOはWi-Fi 7らしい機能です。

名前も強いです。

Multi-Link Operation。  
複数のリンクを使う。  
いかにも速そう。

試したくなります。

でも、最初から必須ではありません。

MLOを使うには、端末側もWi-Fi 7 / MLOに対応している必要があります。

さらに公式手順では、MLOを有効にする前に、3つのSSID名、暗号化方式、セキュリティキーをそろえる必要があります。

## MLOを使う前提

MLOを試すなら、まず次をそろえます。

- 端末側がWi-Fi 7 / MLOに対応している
- 2.4GHz / 5GHz / 6GHzのSSID名を完全に同じにする
- 3帯域すべての暗号化を `WPA3-SAE` にする
- 3帯域すべてで同じWi-Fiパスワードを使う
- できれば有線LAN接続のPCから設定する

## LuCIでのMLO有効化

1. 2.4GHz / 5GHz / 6GHzのSSID名を同じにする
2. 3帯域すべてで暗号化を `WPA3-SAE` にする
3. 3帯域すべてで同じWi-Fiパスワードを設定する
4. **Save & Apply** で反映する
5. **Network** → **MLO** を開く
6. **AP MLO Enable** にチェックを入れる
7. **Save & Apply** をクリックする
8. **Network** → **Wireless** → **Associated Stations** で接続状態を見る

MLO有効化後は、対応端末が2つまたは3つのradioに接続しているかをAssociated Stationsで確認します。

ただし、複数radioに見えても、常に全部のradioで最大速度が出るわけではありません。

TX/RX rateが出るactive radioが1つだけに見えることもあります。

## 最初は通常SSID構成でよい

MLOは面白いです。

でも、家庭や小さなオフィスでは、最初は通常のSSID構成で十分なことが多いです。

特に、古い端末やIoT機器が多い環境では、MLOのために3帯域すべてをWPA3-SAEへ統一すると、接続できない端末が出る可能性があります。

まずは、

```txt
5GHz主力
2.4GHz IoT
6GHz対応端末
```

で安定させます。

MLOは、Wi-Fi 7対応端末が増えてから試すくらいで大丈夫です。

## LuCIでAssociated Stationsを見る

接続端末がどのSSIDやradioにつながっているかは、LuCIでも確認できます。

1. **Network** → **Wireless** を開く
2. 画面下部の **Associated Stations** を見る
3. 接続端末、signal、TX/RX rate、接続先radioを確認する

ここを見ると、PCやスマートフォンが本当に5GHzや6GHzへつながっているか確認できます。

MLOを有効にした場合も、この画面で複数radioへの接続状況を確認します。

ただし、端末名が分かりにくい場合もあります。

MACアドレスと端末名を照合したい場合は、DHCPリースや端末側の設定も合わせて確認します。

## CLIでWi-Fi状態を確認する

CLIでは、最初は読み取りだけに留めます。

Wi-Fi設定の変更は、LuCIで対象radioやSSIDを確認しながら行うほうが安全です。

## 現在のWi-Fi設定を見る

```sh
echo "### wireless config summary"
uci show wireless | grep -E 'wifi-device|wifi-iface|ssid|encryption|network|disabled'

echo "### radio status"
wifi status

echo "### wireless interfaces"
iw dev
```

SSID名、暗号化方式、どのnetworkに紐づいているか、有効/無効状態を確認します。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

## 接続端末を見る

```sh
echo "### connected stations"
for ifc in $(iw dev | awk '$1 == "Interface" {print $2}'); do
  echo "### $ifc"
  iw dev "$ifc" station dump | grep -E 'Station|signal:|tx bitrate:|rx bitrate:|connected time:' || true
done
```

どの無線インターフェースに端末がぶら下がっているかを確認できます。

MLO有効時は、同じ端末が複数radioに見える場合があります。

その場合も、LuCIのAssociated Stationsと合わせて見たほうが分かりやすいです。

## 無線ログを見る

```sh
echo "### recent wireless logs"
logread | grep -Ei 'wireless|wifi|wlan|hostapd|dfs|radar' | tail -n 80
```

Wi-Fiが切れる、SSIDが見えない、6GHzにつながらない、といった時に直近ログを見ます。

ただし、ログにDFSやregulatory関連の文字が出てきても、意味を理解せず設定変更しないでください。

この連載では、無線の国設定、送信出力、DFS関連の値は変更しません。

## 設定変更前のバックアップ

Wi-Fi設定は、SSID名や暗号化方式を少し変えただけでも端末が再接続できなくなることがあります。

変更前には、LuCIのバックアップに加えて、CLIで現在の無線設定を控えておくと安心です。

```sh
BACKUP_DIR="/root/wireless-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

cp /etc/config/wireless "$BACKUP_DIR/wireless"
uci show wireless > "$BACKUP_DIR/wireless.uci.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
iw dev > "$BACKUP_DIR/iw-dev.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、SSIDやWi-Fiパスワードが含まれることがあります。

そのままチャット、SNS、公開リポジトリ、記事のスクリーンショットへ出さないようにしてください。

## 戻す場合

変更後にWi-Fiへつながらなくなった場合は、有線LANでPCをつなぎ、バックアップから戻します。

```sh
cp /root/wireless-before-change-YYYYMMDD-HHMM/wireless /etc/config/wireless
wifi reload
```

`YYYYMMDD-HHMM` は実際のバックアップフォルダ名に置き換えます。

Wi-Fiが切れている状態では、無線接続で復旧できないことがあります。

Wi-Fi設定を触る時は、有線LANで入れるPCを1台用意しておくと安心です。

## 帯域の有効・無効を切り替える

使わない帯域やSSIDは、一時的に無効化できます。

ただし、初期段階ではCLIで無理に切り替えるより、LuCIから行うのが安全です。

## LuCIで切り替える

1. **Network** → **Wireless** を開く
2. 対象SSIDの **Disable** または **Enable** をクリックする
3. **Save & Apply** で反映する

たとえば、6GHz対応端末がまったくない場合は、最初は6GHzを無理に使わなくても構いません。

ただし、将来Wi-Fi 6E/7対応端末を追加する予定があるなら、有効化したままでも問題ありません。

## CLIでは確認だけにする

```sh
echo "### SSID and disabled flags"
uci show wireless | grep -E 'ssid|disabled'

echo "### radio status"
wifi status
```

`wireless.default_radio2` のようなセクション名は、環境やファームウェアによって変わることがあります。

確認せずに `uci set` をコピペ実行しないでください。

## 6GHzのSSIDが見えない時

6GHzのSSIDが見えない時は、まず端末側を確認します。

見るポイントは次です。

- 端末がWi-Fi 6EまたはWi-Fi 7に対応しているか
- 端末のOSやWi-Fiドライバが古くないか
- 6GHz radioがLuCIで有効になっているか
- 暗号化方式が6GHz対応端末で使える設定になっているか
- 端末がルーターから近い場所にあるか

6GHzは、対応端末が必要です。

古いスマートフォンやPCでは、SSID一覧に表示されないことがあります。

ルーターが悪いとは限りません。

まずは、対応端末をルーターの近くに置き、LuCIの **Network → Wireless** と **Associated Stations** を確認します。

## 2.4GHz IoTがつながらない時

IoT機器がつながらない時は、だいたい次のどれかです。

- 5GHz / 6GHz専用SSIDへつなごうとしている
- WPA3に対応していない
- SSID名やパスワードに相性がある
- ルーターから遠すぎる
- IoT機器側の初期化や再設定が必要
- アプリ側で同じLANにいる必要がある

IoT用SSIDは、2.4GHz専用にして、暗号化は互換性重視にするほうが安定しやすいです。

また、SSID名に日本語や記号を使うと、古い端末やIoT機器で相性が出ることがあります。

最初は英数字中心のSSIDにしておくのがおすすめです。

## ゲストWi-FiはSSIDだけでは完成しない

ここ、かなり大事です。

ゲストWi-Fiは、SSIDを追加するだけなら簡単です。

でも、それだけでは本体LANから分離できていないことがあります。

たとえば、`MyHome_Guest` というSSIDを作っても、そのSSIDが `lan` networkに紐づいていたら、中身はMain LANと同じです。

つまり、ゲスト端末からNASやプリンターが見える可能性があります。

ゲストWi-Fiをちゃんと分けたい場合は、次をセットで考えます。

- ゲスト用SSID
- ゲスト用network
- ゲスト用DHCP
- firewall zone
- LANへの通信制限
- 必要ならVLAN

SSID名だけで安心しない。

これがゲストWi-Fiでいちばん大事です。

この006では、まず「メインSSID」「IoT用SSID」「6GHz用SSID」「ゲストWi-Fi用SSID候補」をどう整理するかまでを押さえます。

ゲストWi-Fiの本格設定は、次の記事で扱います。

## 設定後に確認すること

Wi-Fi設定を変えたら、実端末で確認します。

LuCIの画面だけ見て安心したくなりますが、最後は端末側で確認します。

チェックするのは次です。

- PCやスマートフォンがメインSSIDへつながる
- IoT機器が2.4GHz用SSIDへつながる
- Wi-Fi 6E/7対応端末が6GHz用SSIDへつながる
- 端末がインターネットへ出られる
- LuCIのAssociated Stationsで接続先radioを確認できる
- 変更前バックアップが残っている
- 家族やスタッフへ案内するSSID名とパスワードが整理されている

最初から完璧なWi-Fi設計を目指さなくて大丈夫です。

主力端末が5GHzまたは6GHzへつながる。  
IoT機器が2.4GHzで安定する。  
ゲストWi-Fiを後から分離できる見通しがある。

ここまでできれば、初期のWi-Fi設計としては十分です。

## まとめ

LN6001-JPのWi-Fi設定は、LuCIの **Network → Wireless** から変更します。

2.4GHz、5GHz、6GHzは、それぞれ役割を分けて考えると分かりやすいです。

- 2.4GHz: IoT、古い端末、遠い部屋
- 5GHz: PC、スマートフォン、テレビなど日常利用の主力
- 6GHz: Wi-Fi 6E/7対応端末
- MLO: Wi-Fi 7対応端末が増えてから検討

MLOを使う場合は、3帯域すべてのSSID名、WPA3-SAE、Wi-Fiパスワードをそろえ、**Network → MLO** で **AP MLO Enable** を有効にします。

ただし、最初からMLOを使う必要はありません。

まずは、5GHz主力、2.4GHz IoT、6GHz対応端末というシンプル構成で安定させるのがおすすめです。

Wi-Fi設定で一番大事なのは、最大速度よりも置き場所です。

どの端末を、どのSSIDへつなぐか。

ここが決まるだけで、家庭や小さなオフィスのWi-Fiはかなり扱いやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/006/diagram-03.png)

## 次に読むなら

Wi-Fi帯域の使い分けが決まったら、次は目的に合わせて進みます。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [家族向けフィルタリング](https://note.com/ikmsan/n/n284bb49cd1e3)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [MLOを有効化する前に確認すること](https://note.com/ikmsan/n/n00750e23f49c)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

ゲストWi-Fiを本体LANから分けたい人は、ゲストWi-Fi記事へ。

家庭内の子ども用端末やIoTを整理したい人は、家族向けフィルタリング記事へ。

小さなオフィスや店舗でStaff、Guest、Deviceを分けたい人は、VLAN記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### LN6001-JPでは2.4GHz、5GHz、6GHzを全部使い分けるべき？

最初から細かく分ける必要はありません。

まずは5GHzを主力にし、IoT用に2.4GHz、対応端末があるなら6GHzを足すくらいで十分です。

シンプルに始めたほうが、あとからトラブル切り分けもしやすくなります。

### 2.4GHzと5GHzは同じSSIDにしてもいい？

できます。

ただし、最初は分けたほうが端末の接続先を把握しやすいです。

IoT機器が多い家庭では、2.4GHzを `MyHome_IoT` のように分け、PCやスマートフォンを5GHzへ寄せる構成が分かりやすいです。

### 6GHzのSSIDが見えないのはなぜ？

端末がWi-Fi 6EまたはWi-Fi 7に対応していない可能性があります。

6GHzは対応端末が必要です。

端末が対応している場合でも、OSやWi-Fiドライバが古い、ルーターから遠い、6GHz radioが無効になっている、といった可能性があります。

### MLOは最初から有効にしたほうがいい？

必須ではありません。

MLOを使うには、端末側のWi-Fi 7対応に加えて、3帯域のSSID名、暗号化方式、Wi-Fiパスワードをそろえる必要があります。

対応端末が少ないうちは、通常のSSID設計で運用したほうが分かりやすいです。

### ゲストWi-FiはSSIDを追加するだけで作れる？

SSIDを追加するだけなら作れます。

ただし、ゲストWi-Fiを本体LANから分けたい場合は、ゲスト用network、DHCP、firewall zoneの設定も必要です。

安全に分離したい場合は、ゲストWi-Fiの記事でnetworkとfirewallまで含めて設定してください。

### CLIでWi-Fi設定を変更してもいい？

慣れていればできますが、最初はLuCIから変更するのがおすすめです。

CLIでは、まず `uci show wireless`、`wifi status`、`iw dev` などで読み取り確認に使うくらいが安全です。

### IoT用SSIDは2.4GHzだけでいい？

多くの場合、最初は2.4GHzだけで十分です。

スマート家電や古い機器は2.4GHzしか対応していないことが多いためです。

ただし、機器によっては5GHz対応のものもあります。

まずは機器の仕様を確認してください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 よくある質問 FAQ: https://support.linksys.com/kb/article/6899-jp/
- Configuring the WiFi Settings of the Linksys OpenWRT router: https://support.linksys.com/kb/article/221-en/
- How to enable MLO on your Linksys OpenWRT router: https://support.linksys.com/kb/article/8652-en/?section_id=175
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- 株式会社アスク製品ページ: https://www.ask-corp.jp/products/linksys/router/velop-wrt-pro-7.html

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
