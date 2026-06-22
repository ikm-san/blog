<!-- mirror-source: articles/010-family-filtering.md -->

# 家庭のWi-Fi、全部同じで大丈夫？｜Kids・IoT・ゲストWi-Fiを無理なく分ける【OpenWrt集中連載010】

家庭のWi-Fiって、気づくと何でも混ざります。

親のPC。  
スマートフォン。  
子どものタブレット。  
ゲーム機。  
テレビ。  
スマートスピーカー。  
ロボット掃除機。  
見守りカメラ。  
プリンター。  
NAS。  
そしてゲストWi-Fi。

最初は全部同じSSIDでいいんです。

むしろ、そのほうが楽です。  
家族にも説明しやすいし、端末もすぐつながります。

でも、ある日こう思うことがあります。

「子ども用端末だけDNSフィルタリングしたい」  
「スマート家電を親のPCやNASと同じ場所に置きたくない」  
「ゲスト端末からプリンターやNASが見えるのはちょっと嫌」  
「IoT機器が増えて、どれがどこにつながっているか分からない」

ここでいきなり強い制限を入れると、だいたい家の中がネットワーク相談窓口になります。

「プリンターが見えない」  
「学校用PCだけつながらない」  
「スマート家電アプリが動かない」  
「ゲストWi-FiはつながるけどWebが開けない」  
「結局どの設定を戻せばいいの？」

しんどいです。

なので、家庭内ネットワーク分離で最初にやることは、強い制限ではありません。

まずは **分けること** です。

親用。  
子ども用。  
IoT用。  
ゲストWi-Fi用。

この4つを「別の部屋」として考えると、DNSも通信ルールもかなり整理しやすくなります。

この記事では、Linksys Velop WRT Pro 7（LN6001-JP）で、家庭内の端末をKids、IoT、ゲストWi-Fiに分ける考え方と、LuCI / CLIでの確認ポイントをまとめます。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、家庭内ネットワークを完璧に分離することではありません。

まずは、ここまで分かればOKです。

- 家族用、子ども用、IoT用、ゲストWi-Fi用を分ける考え方が分かる
- 子ども用ネットワークへFamily DNSを配る流れが分かる
- IoT用ネットワークをメインLANから分ける理由が分かる
- firewall zoneで「インターネットはOK、LANは原則NG」を作るイメージが分かる
- DHCP予約で重要端末のIPを固定する意味が分かる
- ルーター側でできることと、端末側に任せることを分けられる
- 設定前にバックアップを取る理由が分かる

最初から全部を作らなくて大丈夫です。

まずKidsだけ。  
次にIoT。  
必要になったらゲストWi-Fi。

この順番で十分です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-01.png)

## 先にざっくり結論

家庭内ネットワークは、最初からガチガチに作り込まなくて大丈夫です。

まずは、この4分類で考えるとかなり分かりやすくなります。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-01.png)

最初の一歩としては、**子ども用SSIDだけ分ける** だけでも十分です。

その次にIoT。  
ゲストWi-Fiは必要になった時に追加。

家庭では、このくらいの段階的な進め方のほうが長続きします。

```txt
Mainを安定させる
  ↓
Kidsを分ける
  ↓
IoTを分ける
  ↓
Guestを分ける
  ↓
必要な機器だけ例外許可する
```

いきなり全部を分けるより、1つずつ作って確認してバックアップ。

これがいちばん安全です。

## この記事の前提

この記事では、LN6001-JPをルーターモードで使っている前提で進めます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-02.png)

AP Bridge、ブリッジモード、WDS子機として使っている場合は、この記事と同じ手順にはなりません。

その場合は、DHCPとfirewallをどの機器が担当しているかを先に確認してください。

この記事の構成は、LN6001-JPがDHCPとfirewallを担当する前提です。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-02.png)

## こういう家庭に向いています

この記事は、次のような家庭に向いています。

- 子ども用タブレットやゲーム機だけDNS方針を変えたい
- スマート家電やカメラを、親のPCやNASから分けたい
- ゲストWi-Fiを家庭内LANから分けたい
- IoT機器が増えて、どれがどこにつながっているか分からなくなってきた
- DNS広告ブロックやFamily DNSをネットワークごとに使い分けたい
- 端末ごとに細かく制御する前に、まず大きく分類したい
- 家族に説明できるWi-Fi構成にしたい

逆に、次のような場合は、最初からここまで分けなくても大丈夫です。

- 端末が少ない
- ゲストWi-Fiを出していない
- IoT機器がほとんどない
- 子ども用端末も親が直接管理できている
- 家族全員が同じルールで問題ない

ネットワーク分離は、必要になってから足せばOKです。

## 最初に言葉だけそろえる

家庭内ネットワーク分離で出てくる言葉を、ざっくり整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-03.png)

ここで大事なのは、firewall zoneを難しく考えすぎないことです。

最初は「部屋分け」でOKです。

```txt
lan    = 家族の部屋
kids   = 子ども用の部屋
iot    = スマート家電の部屋
guest  = ゲストの部屋
wan    = インターネット
```

どの部屋からどの部屋へ行っていいかを決める。

それがfirewall zoneの最初の理解です。

## まず端末を分類する

最初にやることは、制限ルールを作ることではありません。

どの端末をどこへ置くかを決めることです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-04.png)

家庭では、全部を `MyHome` に入れてしまいがちです。

それでも最初は動きます。

でも、子ども用端末やIoT機器だけでも分けておくと、あとからDNS、firewall、Adblock、VLANを整理しやすくなります。

## 設計の全体像

最終的には、こんな形を目指せます。

```txt
MyHome
  → lan
  → 192.168.1.0/24
  → 家族用・親用・NAS・プリンター

MyHome_Guest
  → guest
  → 192.168.2.0/24
  → ゲストWi-Fi
  → インターネットのみ

MyHome_Kids
  → kids
  → 192.168.3.0/24
  → Family DNS
  → LANへは原則入れない

MyHome_IoT
  → iot
  → 192.168.4.0/24
  → IoT機器
  → LANへは原則入れない
```

全部を一度に作る必要はありません。

おすすめはこの順番です。

```txt
lan + kids
  ↓
iot
  ↓
guest
  ↓
必要な例外ルール
```

子ども用SSIDだけ分けても、十分意味があります。

IoT機器が増えたらIoT用SSIDを足す。

ゲストWi-Fiを出したくなったらGuestを足す。

それくらいで大丈夫です。

## DNS方針を分ける

ネットワークを分けると、DNS方針も分けやすくなります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-05.png)

DNSフィルタリングは便利です。

でも、強くしすぎると必要なサービスまで止めることがあります。

とくに次の端末は注意です。

- 学校用PC
- ゲーム機
- スマートテレビ
- スマート家電
- 見守りカメラ
- プリンター
- 家族がよく使うアプリ

「強いほどよい」ではありません。

家庭では、

```txt
壊れない範囲で、少し減らす
```

くらいがちょうどいいです。

## Family DNSの例

子ども用ネットワークでは、Cloudflare 1.1.1.1 for FamiliesのようなFamily DNSが使いやすいです。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-06.png)

この記事では、子ども用ネットワークに次を配る例で進めます。

```txt
1.1.1.3
1.0.0.3
```

Family DNSも万能ではありません。

DoH、VPN、モバイル回線、アプリ内通信、同一ドメイン配信などには効きにくいことがあります。

なので、Family DNSは「全部を止める壁」ではなく、

```txt
危ない入口を少し減らすフィルター
```

として使うのが現実的です。

## 設定前にバックアップを取る

Kids、IoT、Guestを分ける設定では、`network`、`wireless`、`dhcp`、`firewall` をまとめて触ります。

ここでバックアップなしは、ちょっと怖いです。

まずLuCIでバックアップを取ります。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-family-network-split-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/family-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

cat /tmp/dhcp.leases > "$BACKUP_DIR/dhcp-leases.txt"
wifi status > "$BACKUP_DIR/wifi-status.json"
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、SSID、Wi-Fiパスワード、DNS設定、端末情報が含まれることがあります。

SNS、公開リポジトリ、記事のスクリーンショットへそのまま出さないでください。

設定で一番強い人は、全部暗記している人ではありません。

変更前に戻れる人です。

## 子ども用ネットワークを作る

まずは、子ども用の `kids` ネットワークを作ります。

ここでは、`192.168.3.0/24` を使います。

### ステップ1: kids用ブリッジデバイスを作る

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-07.png)

5. **Save** をクリックする

Wi-Fiだけで子ども用SSIDを作る場合、Bridge portsは空のままで進めます。

有線ポートも子ども用に分けたい場合は、VLAN設計が必要になります。

最初はWi-Fiだけで大丈夫です。

### ステップ2: kidsインターフェースを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-08.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-09.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-10.png)

9. **Save** をクリックする

これで、子ども用端末へ `192.168.3.100` から `192.168.3.149` あたりのIPアドレスを配る準備ができます。

## 子ども用SSIDを作る

次に、子ども用端末が接続するSSIDを作ります。

1. **Network** → **Wireless** を開く
2. 5GHz radioの **Add** をクリックする
   - LN6001-JPでは、Linksys公式手順上は `wifi1` が5GHz radioです
3. **Interface Configuration** の **General Setup** で次を設定する

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-11.png)

4. **Wireless Security** タブを開く
5. 次のように設定する

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-12.png)

6. **Save** をクリックする

ここで大事なのは、Networkを必ず `kids` にすることです。

`MyHome_Kids` というSSID名でも、Networkが `lan` なら、中身はメインLANです。

名前だけKidsになってしまいます。

これはゲストWi-Fiでもよくある失敗です。

SSID名ではなく、Networkの紐づきを見ます。

## kids firewall zoneを作る

子ども用ネットワークからインターネットへは出られるようにしつつ、メインLANへは入れないようにします。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-13.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-14.png)

6. **Save** をクリックする

この設定では、kidsからwanへは出られます。

一方で、kidsからlanへのforwardingは作りません。

つまり、子ども用端末から親のPC、NAS、プリンター、LuCI管理画面へは原則届かない構成になります。

## DHCPを許可する

kids zoneのInputを `REJECT` にすると、子ども用端末からルーター自身への通信は原則拒否されます。

そのため、IPアドレス取得に必要なDHCPだけを明示的に許可します。

1. **Network** → **Firewall** → **Traffic Rules** を開く
2. **Add** をクリックする
3. 次のように設定する

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-15.png)

4. **Save** をクリックする

このあと、子ども用端末へFamily DNSを直接配ります。

その場合、子ども用端末のDNS問い合わせはインターネット側のFamily DNSへ向かいます。

## kidsへFamily DNSを配布する

kidsネットワークの端末へ、Cloudflare Family DNSを配ります。

1. **Network** → **Interfaces** を開く
2. `kids` の **Edit** をクリックする
3. **DHCP Server** → **Advanced Settings** を開く
4. **DHCP Options** に次を追加する

```txt
6,1.1.1.3,1.0.0.3
```

5. **Save** をクリックする
6. 画面上部の保留中の変更を確認し、**Save & Apply** をクリックする

`6,1.1.1.3,1.0.0.3` の `6` は、DHCP Option 6です。

つまり、

```txt
このネットワークの端末には、このDNSを使ってね
```

と配る設定です。

設定後は、子ども用端末のWi-Fiを一度切断して再接続してください。

端末によっては、再起動したほうが早いこともあります。

## DNSの抜け道を少し減らす

子ども用端末が手動で `8.8.8.8` や `1.1.1.1` のような別DNSを指定すると、Family DNSを迂回できます。

これを少し減らすには、kidsから外部DNSへ出る通信を制限します。

基本方針は次です。

1. kidsからCloudflare Family DNSへのTCP/UDP 53を許可する
2. kidsからその他のTCP/UDP 53を拒否する
3. DoHやVPNは別問題として扱う

LuCIでは、**Network** → **Firewall** → **Traffic Rules** で次のようなルールを作ります。

### Family DNSを許可する

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-16.png)

### その他DNSを拒否する

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-17.png)

ここで大事なのは、ルールの順番です。

`Allow-Kids-Family-DNS` が `Block-Kids-Other-DNS` より上にあることを確認します。

順番が逆だと、Family DNSまで拒否してしまう可能性があります。

## IoT用ネットワークを作る

次に、IoT用ネットワークを作ります。

IoT機器は、親のPCやNASへアクセスする必要がないものが多いです。

テレビ、スマートスピーカー、見守りカメラ、スマート家電などは、IoT用SSIDにまとめておくと整理しやすくなります。

ここでは `iot` ネットワークとして `192.168.4.0/24` を使います。

### IoT用ブリッジデバイスを作る

1. **Network** → **Interfaces** を開く
2. **Devices** タブを開く
3. **Add device configuration** をクリックする
4. 次のように設定する

![表画像 table-18](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-18.png)

5. **Save** をクリックする

### IoT用インターフェースを作る

1. **Network** → **Interfaces** を開く
2. **Interfaces** タブで **Add new interface** をクリックする
3. 次のように設定する

![表画像 table-19](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-19.png)

4. **Create interface** をクリックする
5. **General Settings** で次を設定する

![表画像 table-20](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-20.png)

6. **DHCP Server** タブを開く
7. DHCP Serverが未設定なら **Setup DHCP Server** をクリックする
8. 次のように設定する

![表画像 table-21](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-21.png)

9. **Save** をクリックする

## IoT用SSIDを作る

IoT機器は2.4GHzしか対応していないことも多いです。

そのため、IoT用SSIDはまず2.4GHz側に作ると扱いやすいです。

1. **Network** → **Wireless** を開く
2. 2.4GHz radioの **Add** をクリックする
   - LN6001-JPでは、Linksys公式手順上は `wifi0` が2.4GHz radioです
3. **Interface Configuration** の **General Setup** で次を設定する

![表画像 table-22](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-22.png)

4. **Wireless Security** タブを開く
5. 次のように設定する

![表画像 table-23](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-23.png)

6. **Save** をクリックする

古いIoT機器では、WPA3や日本語SSID、記号の多いSSIDで相性が出ることがあります。

最初は英数字中心のSSIDと、互換性を重視した暗号化設定にしておくと切り分けしやすくなります。

IoTは速さより、まず「つながること」が大事です。

## IoT firewall zoneを作る

IoT用ネットワークも、メインLANから分けます。

1. **Network** → **Firewall** を開く
2. **Zones** タブを開く
3. **Add** をクリックする
4. 次のように設定する

![表画像 table-24](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-24.png)

5. **Inter-Zone Forwarding** で次を設定する

![表画像 table-25](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-25.png)

6. **Save** をクリックする

この構成では、IoT機器はインターネットへ出られます。

一方で、IoT機器からメインLAN側のPC、NAS、プリンター、LuCI管理画面へは入れない構成になります。

## IoT用DHCPとDNSを許可する

IoT zoneのInputも `REJECT` にするため、DHCPとDNSを明示的に許可します。

IoT用ネットワークでは、ルーター自身をDNSとして使わせる構成が分かりやすいです。

その場合、次の2つを許可します。

### DHCPを許可する

![表画像 table-26](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-26.png)

### DNSを許可する

![表画像 table-27](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-27.png)

IoT機器は、メーカーのクラウドやアプリ連携に依存していることが多いです。

DNSブロックを強くしすぎると、アプリから見えない、通知が来ない、ファームウェア更新に失敗する、といった問題が出ることがあります。

IoT用は、最初は軽めに始めるのがおすすめです。

## ゲストWi-Fiは別記事で作る

ゲストWi-Fiは、家庭内LANから分けてインターネットだけ使わせる用途です。

基本的な考え方は、次の通りです。

![表画像 table-28](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-28.png)

ゲストWi-Fiは、007の記事で詳しく扱っています。

この010では、家庭内ネットワーク全体の分類として、ゲストWi-Fiは「親用LANに入れない一時利用端末」として位置付けます。

ここで大事なのは、SSIDを作るだけではゲストWi-Fiは完成しないことです。

`MyHome_Guest` という名前でも、Networkが `lan` なら中身はメインLANです。

ゲストWi-Fiは、SSID、network、DHCP、firewall zoneまでセットで作ります。

## DHCP予約で重要端末を固定する

ネットワークを分けたら、重要な端末だけ固定IPにしておくと管理しやすくなります。

たとえば、次のような機器です。

- NAS
- プリンター
- 見守りカメラ
- テレビ
- Home AssistantやミニPC
- 子ども用ゲーム機
- 管理用PC

LuCIでは、次の手順で設定します。

1. **Network** → **DHCP and DNS** を開く
2. **Static Leases** タブを開く
3. **Add** をクリックする
4. 次の項目を入力する

![表画像 table-29](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-29.png)

5. **Save & Apply** をクリックする

端末のMACアドレスは、接続中であれば次でも確認できます。

```sh
cat /tmp/dhcp.leases
```

スマートフォンやタブレットでは、MACアドレスのランダム化機能が有効になっている場合があります。

固定DHCP割り当てを使う場合は、対象SSIDに対して端末側のプライベートアドレス設定も確認してください。

## 通信ルールの考え方

家庭向けでは、まず次の通信ルールで十分です。

![表画像 table-30](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-30.png)

ポイントは、**kids / iot / guestからlanへ広く許可しないこと**です。

プリンターやNASを使わせたい場合は、lan全体を許可するのではなく、必要な機器だけ個別に許可します。

```txt
kids → lan 全部OK
```

ではなく、

```txt
kids → printerだけOK
```

のほうが安全です。

## プリンターだけ許可したい場合

子ども用ネットワークやIoT用ネットワークから、家庭内プリンターだけ使わせたいことがあります。

この場合、kids → lan全体を許可するのではなく、プリンターのIPアドレスだけを宛先にしたルールを作ります。

例:

![表画像 table-31](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-31.png)

ただし、プリンターは機種によって必要な通信が違います。

IPP、AirPrint、mDNS、独自アプリなどが絡むこともあります。

最初は無理に例外を増やさず、本当に必要な機器だけ検証しながら許可するのがおすすめです。

家庭では、印刷が必要な時だけメインSSIDへ切り替える運用でも十分な場合があります。

## UCIでkidsとiotを作る例

ここからは上級者向けです。

最初はLuCIで作るほうが安全です。

UCIで作る場合は、必ず現在の設定とradio名を確認してください。

LN6001-JPでは、Linksys公式手順上、`wifi0` が2.4GHz、`wifi1` が5GHz、`wifi2` が6GHzです。

ただし、実機では必ず確認します。

```sh
uci show wireless | grep '=wifi-device'
uci show wireless | grep -E 'device|ssid|network'
```

以下は、5GHzにKids、2.4GHzにIoTを作る例です。

SSIDとパスワードは必ず変更してください。

```sh
BACKUP_DIR="/root/family-uci-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless dhcp firewall; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

KIDS_RADIO="wifi1"
IOT_RADIO="wifi0"

KIDS_SSID="MyHome_Kids"
KIDS_KEY="CHANGE_ME_KIDS_PASSWORD"

IOT_SSID="MyHome_IoT"
IOT_KEY="CHANGE_ME_IOT_PASSWORD"

# kids bridge/interface
uci -q delete network.kids_dev
uci set network.kids_dev="device"
uci set network.kids_dev.type="bridge"
uci set network.kids_dev.name="br-kids"

uci -q delete network.kids
uci set network.kids="interface"
uci set network.kids.proto="static"
uci set network.kids.device="br-kids"
uci set network.kids.ifname="br-kids"
uci set network.kids.ipaddr="192.168.3.1"
uci set network.kids.netmask="255.255.255.0"

# iot bridge/interface
uci -q delete network.iot_dev
uci set network.iot_dev="device"
uci set network.iot_dev.type="bridge"
uci set network.iot_dev.name="br-iot"

uci -q delete network.iot
uci set network.iot="interface"
uci set network.iot.proto="static"
uci set network.iot.device="br-iot"
uci set network.iot.ifname="br-iot"
uci set network.iot.ipaddr="192.168.4.1"
uci set network.iot.netmask="255.255.255.0"

# kids SSID
uci -q delete wireless.kids
uci set wireless.kids="wifi-iface"
uci set wireless.kids.device="$KIDS_RADIO"
uci set wireless.kids.mode="ap"
uci set wireless.kids.network="kids"
uci set wireless.kids.ssid="$KIDS_SSID"
uci set wireless.kids.encryption="psk2+ccmp"
uci set wireless.kids.key="$KIDS_KEY"

# iot SSID
uci -q delete wireless.iot
uci set wireless.iot="wifi-iface"
uci set wireless.iot.device="$IOT_RADIO"
uci set wireless.iot.mode="ap"
uci set wireless.iot.network="iot"
uci set wireless.iot.ssid="$IOT_SSID"
uci set wireless.iot.encryption="psk2+ccmp"
uci set wireless.iot.key="$IOT_KEY"

# DHCP
uci -q delete dhcp.kids
uci set dhcp.kids="dhcp"
uci set dhcp.kids.interface="kids"
uci set dhcp.kids.start="100"
uci set dhcp.kids.limit="50"
uci set dhcp.kids.leasetime="4h"
uci add_list dhcp.kids.dhcp_option="6,1.1.1.3,1.0.0.3"

uci -q delete dhcp.iot
uci set dhcp.iot="dhcp"
uci set dhcp.iot.interface="iot"
uci set dhcp.iot.start="100"
uci set dhcp.iot.limit="100"
uci set dhcp.iot.leasetime="12h"

# firewall zones
uci -q delete firewall.kids
uci set firewall.kids="zone"
uci set firewall.kids.name="kids"
uci set firewall.kids.network="kids"
uci set firewall.kids.input="REJECT"
uci set firewall.kids.output="ACCEPT"
uci set firewall.kids.forward="REJECT"

uci -q delete firewall.iot
uci set firewall.iot="zone"
uci set firewall.iot.name="iot"
uci set firewall.iot.network="iot"
uci set firewall.iot.input="REJECT"
uci set firewall.iot.output="ACCEPT"
uci set firewall.iot.forward="REJECT"

# forward to wan
uci -q delete firewall.kids_wan
uci set firewall.kids_wan="forwarding"
uci set firewall.kids_wan.src="kids"
uci set firewall.kids_wan.dest="wan"

uci -q delete firewall.iot_wan
uci set firewall.iot_wan="forwarding"
uci set firewall.iot_wan.src="iot"
uci set firewall.iot_wan.dest="wan"

# DHCP allow rules
uci -q delete firewall.kids_dhcp
uci set firewall.kids_dhcp="rule"
uci set firewall.kids_dhcp.name="Allow-Kids-DHCP"
uci set firewall.kids_dhcp.src="kids"
uci set firewall.kids_dhcp.proto="udp"
uci set firewall.kids_dhcp.src_port="68"
uci set firewall.kids_dhcp.dest_port="67"
uci set firewall.kids_dhcp.family="ipv4"
uci set firewall.kids_dhcp.target="ACCEPT"

uci -q delete firewall.iot_dhcp
uci set firewall.iot_dhcp="rule"
uci set firewall.iot_dhcp.name="Allow-IoT-DHCP"
uci set firewall.iot_dhcp.src="iot"
uci set firewall.iot_dhcp.proto="udp"
uci set firewall.iot_dhcp.src_port="68"
uci set firewall.iot_dhcp.dest_port="67"
uci set firewall.iot_dhcp.family="ipv4"
uci set firewall.iot_dhcp.target="ACCEPT"

# IoT uses router DNS
uci -q delete firewall.iot_dns
uci set firewall.iot_dns="rule"
uci set firewall.iot_dns.name="Allow-IoT-DNS"
uci set firewall.iot_dns.src="iot"
uci set firewall.iot_dns.proto="tcp udp"
uci set firewall.iot_dns.dest_port="53"
uci set firewall.iot_dns.target="ACCEPT"

# Kids Family DNS
uci -q delete firewall.kids_family_dns
uci set firewall.kids_family_dns="rule"
uci set firewall.kids_family_dns.name="Allow-Kids-Family-DNS"
uci set firewall.kids_family_dns.src="kids"
uci set firewall.kids_family_dns.dest="wan"
uci set firewall.kids_family_dns.proto="tcp udp"
uci add_list firewall.kids_family_dns.dest_ip="1.1.1.3"
uci add_list firewall.kids_family_dns.dest_ip="1.0.0.3"
uci set firewall.kids_family_dns.dest_port="53"
uci set firewall.kids_family_dns.family="ipv4"
uci set firewall.kids_family_dns.target="ACCEPT"

uci -q delete firewall.kids_other_dns
uci set firewall.kids_other_dns="rule"
uci set firewall.kids_other_dns.name="Block-Kids-Other-DNS"
uci set firewall.kids_other_dns.src="kids"
uci set firewall.kids_other_dns.dest="wan"
uci set firewall.kids_other_dns.proto="tcp udp"
uci set firewall.kids_other_dns.dest_port="53"
uci set firewall.kids_other_dns.family="ipv4"
uci set firewall.kids_other_dns.target="REJECT"

uci changes network
uci changes wireless
uci changes dhcp
uci changes firewall

uci commit network
uci commit wireless
uci commit dhcp
uci commit firewall

/etc/init.d/network restart
/etc/init.d/dnsmasq restart
/etc/init.d/firewall restart
wifi reload
```

反映後、端末を接続して状態を確認します。

```sh
cat /tmp/dhcp.leases
ifstatus kids
ifstatus iot
uci show dhcp | grep -E 'kids|iot|dhcp_option'
uci show firewall | grep -E 'kids|iot|Allow-Kids|Allow-IoT|Block-Kids'
```

CLI例は、理解のためのものです。

実運用では、最初はLuCIで作るほうが安全です。

## 設定後の確認

設定後は、実際の端末で確認します。

LuCIの画面だけ見て安心したくなりますが、最後は端末側で確認します。

## Kids端末

子ども用SSIDへ接続し、IPアドレスとDNSを確認します。

期待する状態です。

```txt
IP address: 192.168.3.x
Gateway: 192.168.3.1
DNS: 1.1.1.3 / 1.0.0.3
```

Family DNSの動作確認には、CloudflareのテストURLも使えます。

```txt
https://malware.testcategory.com/
https://nudity.testcategory.com/
```

## IoT端末

IoT用SSIDへ接続し、IPアドレスを確認します。

```txt
IP address: 192.168.4.x
Gateway: 192.168.4.1
```

スマート家電アプリ、テレビ、カメラなどが正常にクラウドへ接続できるかも確認します。

IoTは、つながるだけでなく、いつものアプリから見えるかまで確認します。

## LANへ届かないことを確認する

KidsやIoTにつないだ端末から、メインLAN側へアクセスしてみます。

```txt
https://192.168.1.1
http://192.168.1.x
ssh root@192.168.1.1
```

期待する状態は、LuCI、NAS、親のPCへ届かないことです。

ただし、プリンターや特定のNASだけ使わせたい場合は、必要な通信だけ例外ルールを作ります。

```txt
届くべきところに届く
届かないべきところに届かない
```

この両方を確認します。

## CLIで状態を読む

メインLAN側からSSHでLN6001-JPへ入り、読み取り確認します。

```sh
echo "### interfaces"
ifstatus lan
ifstatus kids 2>/dev/null || true
ifstatus iot 2>/dev/null || true

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### DHCP DNS options"
uci show dhcp | grep -E 'kids|iot|dhcp_option|dns'

echo "### firewall family zones"
uci show firewall | grep -E 'kids|iot|guest|Allow-Kids|Allow-IoT|Block-Kids'

echo "### wireless mapping"
uci show wireless | grep -E 'ssid|network|kids|iot|guest'

echo "### dnsmasq logs"
logread | grep -i dnsmasq | tail -n 80
```

CLIは、最初は設定変更より「今どうつながっているか」を確認する用途で使うくらいで十分です。

## DNSだけでできること、できないこと

DNSフィルタリングで制御できるのは、名前解決が中心です。

期待しすぎないために、「DNSでできること」と「端末側でやるべきこと」を分けて考えます。

![表画像 table-32](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/table-32.png)

DNSフィルタリングは、「全部を制御する機能」ではありません。

危険サイトや不要な入口を減らす仕組みとして使い、細かい利用時間やアプリ制御は端末側のペアレンタルコントロールと組み合わせるのが現実的です。

## DoH、VPN、モバイル回線への注意

子ども用ネットワークにFamily DNSを配っても、次のような経路では効きにくくなります。

- ブラウザやアプリがDoHを使う
- VPNアプリで外部へ抜ける
- スマートフォンがモバイル回線へ切り替わる
- 端末側に固定DNSが設定されている

このあたりは、ルーターだけで完全に管理するのは難しいです。

家庭では、次の順番で考えるのがおすすめです。

1. 子ども用SSIDを分ける
2. Family DNSを配る
3. 端末側のDNSやDoH設定を確認する
4. 端末側のファミリー機能を併用する
5. 必要ならbanIPなどのDoH対策を検討する
6. VPNやモバイル回線は家庭内ルールとして話す

技術で全部を封じようとすると、家庭内がだいぶギスギスします。

ネットワーク設計は、家庭内ルールを補助するものとして使うほうが長続きします。

## IPv6も忘れない

日本のIPoE環境では、IPv6が有効になっていることが多いです。

IPv4だけFamily DNSを配っても、IPv6側のDNSやDoHで抜け道ができる場合があります。

Cloudflare Family DNSにはIPv6アドレスもあります。

```txt
2606:4700:4700::1113
2606:4700:4700::1003
```

ただし、OpenWrtで子ども用ネットワークへIPv6をどう配るかは、prefix delegation、RA、DHCPv6、firewall zoneが関係します。

最初はIPv4で動作確認を行い、IPv6を子ども用ネットワークへ配る場合は、IPv6の落とし穴の記事と合わせて設計するのがおすすめです。

中途半端にIPv6だけ抜け道になる構成は避けます。

## よくある失敗

### 子ども用SSIDにつながるがIPアドレスが取れない

次を確認します。

- 子ども用SSIDのNetworkが `kids` になっているか
- `kids` interfaceでDHCP Serverが有効か
- `Allow-Kids-DHCP` ルールがあるか
- 端末をWi-Fiから切断して再接続したか

CLIでは次を見ます。

```sh
uci show dhcp.kids
logread | grep -i dnsmasq | tail -n 50
cat /tmp/dhcp.leases
```

### Family DNSが効かない

次を確認します。

- kidsのDHCP Optionsに `6,1.1.1.3,1.0.0.3` が入っているか
- 端末が新しいDHCP情報を受け取っているか
- 端末側に固定DNSが入っていないか
- DoHが有効になっていないか
- VPNやモバイル回線を使っていないか
- IPv6側が抜け道になっていないか

Macなら次でDNSを確認できます。

```sh
scutil --dns | grep nameserver
```

Windows PowerShellなら次を使います。

```powershell
Get-DnsClientServerAddress
```

### IoT機器が動かない

IoT用SSIDへ分けると、一部のスマート家電やテレビが不安定になることがあります。

よくある原因です。

- 2.4GHzにしか対応していない
- WPA3に対応していない
- アプリ初期設定時だけ同じLANにいる必要がある
- mDNSやローカル探索が必要
- クラウド接続先がDNSやfirewallで止まっている

IoT機器は、最初から強く絞りすぎないほうが安全です。

まずIoT用SSIDへ接続できること、インターネットへ出られることを確認し、その後でLAN制限やDNS制御を調整します。

### プリンターが見えなくなった

プリンターをLANに置いたままKidsやIoTから使わせたい場合、ネットワーク分離によって見えなくなることがあります。

これは分離としては正常です。

必要なら、プリンターのIPアドレスを固定し、必要な通信だけ例外許可します。

ただし、家庭では「印刷が必要な時だけ親用SSIDに切り替える」くらいの運用でも十分な場合があります。

### SSIDを増やしすぎた

SSIDを増やしすぎると、家族に説明しにくくなります。

最初は次の3〜4個くらいで十分です。

```txt
MyHome
MyHome_Kids
MyHome_IoT
MyHome_Guest
```

役割を説明できないSSIDは、作らないほうが管理しやすいです。

## 運用メモのテンプレート

設定したら、次のようなメモを残しておくと便利です。

```txt
家族用SSID:
  MyHome
  network: lan
  IP: 192.168.1.0/24

子ども用SSID:
  MyHome_Kids
  network: kids
  IP: 192.168.3.0/24
  DNS: 1.1.1.3 / 1.0.0.3
  kids -> wan: allow
  kids -> lan: reject

IoT用SSID:
  MyHome_IoT
  network: iot
  IP: 192.168.4.0/24
  DNS: router / Adblock
  iot -> wan: allow
  iot -> lan: reject

ゲストWi-Fi:
  MyHome_Guest
  network: guest
  IP: 192.168.2.0/24
  guest -> wan: allow
  guest -> lan: reject

バックアップ:
  backup-LN6001-before-family-network-split-20260621.tar.gz
```

家庭内ネットワークでも、こうしたメモがあるだけで、あとからかなり助かります。

半年後の自分は、だいたい設定内容を忘れています。

## まとめ

家庭内ネットワーク分離は、最初から強い制限を入れることではありません。

まず、端末を役割ごとに分けます。

1. 家族用 / 親用は `lan`
2. 子ども用は `kids`
3. IoT機器は `iot`
4. ゲストWi-Fiは `guest`

そのうえで、DNSと通信ルールを分けます。

- kidsにはFamily DNSを配る
- IoTは軽めに始める
- guestはインターネットだけ
- kids / iot / guestからlanへは原則入れない
- 必要な機器だけ例外許可する
- 端末側のファミリー機能も併用する

LN6001-JPはOpenWrtベースなので、SSID、DHCP、DNS、firewall zoneを組み合わせた家庭内ネットワーク分離を作れます。

最初はKidsだけ。

次にIoT。

必要になったらゲストWi-Fi。

このくらいの順番で進めると、家庭でも無理なく運用できます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/010/diagram-03.png)

## 次に読むなら

この次は、目的に合わせて進みます。

- [ゲストWi-Fiの作り方](https://note.com/ikmsan/n/nacbd5d573d67)
- [DNS広告ブロックの始め方](https://note.com/ikmsan/n/n4759cf81d0e1)
- [VLANで社内・ゲストネットワークを分ける](https://note.com/ikmsan/n/n516c42447850)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [家庭向け構成例](https://note.com/ikmsan/n/n73a7fed30751)

ゲストWi-Fiをまだ分けていない人は、ゲストWi-Fiの記事へ。

IoTやKidsを有線ポートやVLANまで含めて整理したい人は、VLANとfirewall zoneの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

無線の国設定、送信出力、DFS関連の値は変更しません。

ファームウェア更新で画面名やコマンドの出力が変わることがあります。

記事の内容と実際の画面が少し違う場合は、まずバックアップを取り、Linksys公式サポートの最新情報も確認してください。

## よくある質問

### 家庭では最初からKids、IoT、Guestを全部分けるべき？

全部を最初から作る必要はありません。

最初はKidsだけでも十分です。

次にIoT、必要になったらゲストWi-Fiという順番で増やすほうが、家庭では運用しやすくなります。

### 子ども用Wi-FiのフィルタリングはDNSだけで十分？

十分ではありません。

DNSは危険サイトや成人向けカテゴリの入口を減らすには役立ちますが、利用時間、アプリ制限、課金制限、モバイル回線、VPNまでは管理しきれません。

端末側のペアレンタルコントロール機能も併用するのがおすすめです。

### スマート家電やカメラは家族用Wi-Fiと分けたほうがいい？

分けたほうが管理しやすいです。

IoT機器は、親のPCやNASへアクセスする必要がないものが多いです。

IoT用SSIDへ分け、必要な通信だけ許可するほうが整理しやすくなります。

### IoT用SSIDは2.4GHzで作るべき？

多くの場合、最初は2.4GHzで作るとつながりやすいです。

古いIoT機器やスマート家電は、5GHzや6GHzに対応していないことがあります。

### Cloudflare Family DNSを使えば安全？

完全に安全になるわけではありません。

Family DNSはマルウェアや成人向けカテゴリの一部をDNS段階で減らす仕組みです。

DoH、VPN、モバイル回線、アプリ内通信、IPv6側の抜け道などには注意が必要です。

### ゲストWi-FiとKidsは同じでいい？

分けたほうがよいです。

ゲストWi-Fiは一時利用端末向けです。

Kidsは家庭内の子ども用端末向けで、DNSや端末管理の方針が違います。

同じSSIDにすると、後からルールを分けにくくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Velop WRT Pro 7 OpenWrt ルーターのWiFi設定の変更方法: https://support.linksys.com/kb/article/7037-jp/
- OpenWrt ルーターの複数SSIDセグメント分けをする方法 Velop WRT Pro 7: https://support.linksys.com/kb/article/7046-jp/
- Velop WRT Pro 7 OpenWrt ルーターにAdblockを追加してセットアップする方法: https://support.linksys.com/kb/article/7050-jp/
- Cloudflare 1.1.1.1 for Families: https://developers.cloudflare.com/1.1.1.1/setup/
- Google SafeSearchを管理対象デバイスやネットワークでロックする方法: https://support.google.com/websearch/answer/186669
- OpenWrt Wiki - Ad blocking: https://openwrt.org/docs/guide-user/services/ad-blocking

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
