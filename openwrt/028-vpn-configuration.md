<!-- mirror-source: articles/028-vpn-configuration.md -->

# VPNは“全部入れる鍵”じゃない｜WireGuard・Tailscaleで必要な場所だけ開ける構成例【OpenWrt集中連載028】

外出先から自宅や店舗へ入りたい。

NASを開きたい。  
ミニPCへSSHしたい。  
店舗の録画機を確認したい。  
LuCIでルーターの状態を見たい。  
家にいる時と同じように、でも外から安全に入りたい。

こういう時、VPNはかなり便利です。

でも、ここでひとつ大事な話があります。

VPNは、

```txt
つながったら勝ち
```

ではありません。

むしろ大事なのは、そのあとです。

```txt
つないだあと、どこまで入れるようにするか
```

です。

外から入れる入口を作ったからといって、家じゅう・店じゅうを全部歩き回れるようにする必要はありません。

NASだけ見たいなら、NASだけ。  
LuCIだけ確認したいなら、ルーターだけ。  
店舗の録画機だけ見たいなら、録画機だけ。  
管理用ミニPCへSSHしたいなら、そのミニPCだけ。

最初はそのくらいでいいです。

VPNは“全部入れる鍵”ではなく、**必要な場所だけへ通す専用通路**として考えます。

LN6001-JPでは、Linksys公式のVPN Assistantモジュールを使って、WireGuardとTailscaleを扱えます。

WireGuardはシンプルで、自分で入口を作るタイプ。  
Tailscaleは固定IPなし・ポート開放なしでも始めやすいタイプ。

どちらも便利です。

ただし、どちらを使う場合でも、

```txt
誰が
どの端末から
どこへ
どの範囲で
入れるのか
```

を決めてから設定するのがおすすめです。

この記事では、WireGuardとTailscaleの細かい導入手順をもう一度なぞるのではなく、家庭・小さなオフィス・店舗でよくあるVPN構成例を整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、VPN設定を全部暗記することではありません。

まずは、ここまで分かればOKです。

- WireGuardとTailscaleをどう選ぶか分かる
- 固定IP、DDNS、IPoE環境で判断するポイントが分かる
- VPN Assistantを使う基本的な流れが分かる
- 自宅NAS、LuCI、店舗録画機、小さなオフィス管理PCへの構成例が分かる
- WireGuardの `AllowedIPs` を絞る理由が分かる
- TailscaleのサブネットルーターとExit Nodeを分けて考えられる
- VPNからLAN全体へ広げすぎない設計が分かる
- 鍵、QRコード、Tailscale端末管理の注意点が分かる
- つながらない時に見るCLIコマンドが分かる

VPNは、入れるだけなら意外と簡単です。

でも、運用で差が出るのはここです。

```txt
入れる範囲を決める
不要になった端末を消す
必要以上に広げない
```

ここまで考えられると、かなり安心して使えます。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-01.png)

## 先にざっくり結論

VPN方式は、まずこう選ぶと分かりやすいです。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-01.png)

日本の家庭回線では、IPoE / IPv4 over IPv6、MAP-E、DS-Lite、HGW配下、二重ルーターなどが絡みます。

その結果、WireGuardのUDPポートを外から届けにくいことがあります。

この場合は、Tailscaleから始めるのが現実的です。

一方で、固定IPやDDNSがあり、ポート開放も管理できるなら、WireGuardはシンプルで強い選択肢です。

ただし、どちらでも最初からLAN全体へ入れる必要はありません。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-02.png)

VPNは、外から入る入口です。

入口の先を、最初から全部屋開放にしなくて大丈夫です。

## この記事の位置づけ

この連載では、VPNまわりを複数の記事に分けています。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-03.png)

この記事は、013と018の続きです。

主役は、**実際にどんな構成で使うか** です。

VPN Assistantのボタンをどこで押すかより、

```txt
自分の環境では、どの範囲をVPNで見せるべきか
```

を考える記事です。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-02.png)

## こういう人向けです

この記事は、次のような人向けです。

- 自宅NASへ外出先から入りたい
- ルーター管理画面を外部公開せずに使いたい
- 小さなオフィスのミニPCへSSHしたい
- 店舗の録画機やカメラを安全に確認したい
- WireGuardとTailscaleのどちらを選ぶか迷っている
- IPoE環境でポート開放できるか不安
- VPNを入れたあと、LAN全体へ広げすぎない設計を知りたい
- 家族やスタッフの端末をどう管理するか整理したい
- TailscaleのExit Nodeを使うべきか迷っている
- 保守担当へ一時的にアクセスを渡す構成を考えたい

逆に、

```txt
不特定多数にWebサービスを公開したい
公開ゲームサーバーを運用したい
本格的な拠点間VPNを細かく設計したい
```

という場合は、この記事だけでは足りません。

その場合は、公開設計、証明書、ログ監視、ルーティング、firewall、運用ルールまで別途考えます。

この記事では、家庭・小さなオフィス・店舗の現実的なVPN構成を扱います。

## 用語ミニ解説

VPNまわりの言葉を、ざっくり整理しておきます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-04.png)

最初は、これだけで十分です。

```txt
WireGuard = 自分で入口を作るVPN
Tailscale = 入口作りと端末管理を楽にするVPN
AllowedIPs = WireGuardでどこへ行けるか
サブネットルーター = TailscaleからLAN内機器へ入る中継役
Exit Node = 全通信の出口
```

## VPN設計で最初に決めること

VPN設定で最初に決めるのは、方式ではありません。

まず、目的を決めます。

```txt
誰が
どこから
どの機器へ
どの範囲で
いつ使うのか
```

これを決めるだけで、構成がかなり見えます。

## 目的を具体化する

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-05.png)

目的がNASだけなら、LAN全体は不要です。

目的が録画機だけなら、Cameraネットワーク全体を開く必要はないかもしれません。

最初は目的を絞ったほうが安全です。

## VPN端末を決める

VPNに入れる端末は、信頼できる端末だけにします。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-06.png)

VPNに入れる端末を増やすほど、管理対象も増えます。

「VPNに入れる端末は少なめ」が基本です。

便利なので増やしたくなりますが、入口が増えるということでもあります。

## まず“VPN不要”のケースもある

VPNの記事でいきなりこんなことを言うのも変ですが、VPNが不要なケースもあります。

たとえば、次のような場合です。

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-07.png)

VPNは便利ですが、何でもVPNにする必要はありません。

外から入る必要がある時に使います。

## 構成例1: 自宅NASだけ使う

一番よくある構成です。

外出先から自宅NASへ入りたい。  
でもNASをインターネットへ直接公開したくない。

この場合、VPNがかなり向いています。

```txt
外出先スマートフォン / PC
  ↓ VPN
LN6001-JP
  ↓ LAN
NAS 192.168.1.10
```

## 推奨方式

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-08.png)

固定IPやDDNSがあり、ポート開放も管理できるならWireGuard。

固定IPなし、IPoE環境、HGW配下、二重ルーターなどで外部着信が不安ならTailscale。

この分け方でかなり判断しやすくなります。

## アクセス範囲

NASだけなら、最初はNASのIPアドレスだけにします。

```txt
192.168.1.10/32
```

WireGuardならクライアント側の `AllowedIPs` を次のように絞ります。

```ini
AllowedIPs = 192.168.1.10/32
```

Tailscaleなら、サブネットルートをNASだけに絞る考え方です。

```txt
192.168.1.10/32
```

ただし、Tailscaleの運用上、LAN全体 `192.168.1.0/24` を広告したほうが楽な場面もあります。

その場合でも、Tailscale ACL / grantsやOpenWrt firewallで必要な範囲へ絞れるか検討します。

## 先にNASのIPを固定する

NASへVPNで入るなら、NASのIPアドレスは固定しておきます。

端末側固定IPより、LN6001-JP側のDHCP予約がおすすめです。

例:

```txt
nas-home → 192.168.1.10
```

確認します。

```sh
echo "### NAS static lease"
uci show dhcp | grep -E 'nas|192.168.1.10|@host|\\.ip='

echo "### DHCP leases"
cat /tmp/dhcp.leases
```

NASのIPが変わると、VPNのアクセス範囲も壊れます。

VPN設定の前に、まずDHCP予約です。

地味ですが、かなり効きます。

## 構成例2: LuCIだけ外出先から確認する

ルーターの状態確認だけしたい場合です。

LuCIやSSHをインターネットへ直接公開するのは避けます。

VPN経由でアクセスします。

```txt
外出先PC
  ↓ VPN
LN6001-JP
  ↓
LuCI https://192.168.1.1
```

## 推奨方式

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-09.png)

## アクセス範囲

LuCIだけなら、最初はルーターのLAN IPだけで十分です。

```txt
192.168.1.1/32
```

WireGuardの例です。

```ini
AllowedIPs = 192.168.1.1/32
```

Tailscaleの場合、LN6001-JP自身のTailscale IPへ直接アクセスできる場合もあります。

```txt
https://<LN6001-JPのTailscale IP>
```

ただし、tailnetに参加できる端末がLuCIへ近づけるという意味なので、Tailscale端末管理をしっかり行います。

## 注意点

- LuCI / SSHをWANへ直接公開しない
- VPNへ入れる端末を絞る
- QRコードやWireGuard設定ファイルを共有しない
- Tailscale管理コンソールで不要端末を削除する
- 管理者パスワードを強いものに変更する
- 家族やスタッフ全員にLuCIを見せる必要はない

LuCIはルーターの管理画面です。

便利だからといって、VPN参加者全員へ開ける必要はありません。

## 構成例3: 小さなオフィスの管理PCへSSHする

小さなオフィスにある管理用ミニPCやNASへ、外出先からSSHしたい場合です。

```txt
外出先ノートPC
  ↓ VPN
LN6001-JP
  ↓ Staff LAN
管理PC 192.168.1.20
```

## 推奨方式

![表画像 table-10](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-10.png)

## アクセス範囲

管理PCだけなら、次で十分です。

```txt
192.168.1.20/32
```

WireGuardの例です。

```ini
AllowedIPs = 192.168.1.20/32
```

小さなオフィスのStaff LAN全体へ入る必要がある場合は、次です。

```ini
AllowedIPs = 192.168.1.0/24
```

ただし、範囲が広がります。

最初は管理PCだけに絞るほうが安全です。

## firewallの考え方

VPN専用zoneを作る場合は、次のように考えます。

![表画像 table-11](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-11.png)

最初から `vpn → lan` を広く許可すると楽です。

でも、アクセス範囲は広がります。

小さなオフィスでは、管理対象だけ許可する設計も検討します。

## 構成例4: 店舗の録画機・カメラを確認する

店舗では、録画機やカメラを確認したい場面があります。

ただし、POSや決済端末は別扱いにします。

```txt
外出先管理PC
  ↓ VPN
LN6001-JP
  ↓ Device / Camera network
録画機 192.168.3.20
```

## 推奨方式

![表画像 table-12](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-12.png)

## アクセス範囲

録画機だけなら、録画機のIPだけにします。

```txt
192.168.3.20/32
```

Cameraネットワーク全体へ入れる場合は、次です。

```txt
192.168.3.0/24
```

ただし、Cameraネットワーク全体へ入れる必要が本当にあるか確認します。

録画機経由でカメラを見られるなら、録画機だけ許可するほうが安全です。

## 店舗での注意点

- POSや決済端末をVPNから触る設計は慎重にする
- ベンダー要件やサポート条件を優先する
- 保守担当のアクセスは期限や端末を管理する
- Tailscaleなら不要端末を管理コンソールで削除する
- 録画機やカメラの管理パスワードを初期値にしない
- 直接ポート開放せず、VPN経由にする
- 店舗名や住所が分かる端末名を公開しない

店舗では「つながること」より「必要なものだけへ入れること」が重要です。

つながるから触る、は危ないです。

## 構成例5: 保守担当に一時アクセスを渡す

小さなオフィスや店舗では、外部の保守担当に一時的にアクセスしてもらいたいことがあります。

たとえば、録画機の設定確認、ミニPCのログ確認、NASの設定確認などです。

この場合、常設の広いVPNアクセスは避けます。

```txt
保守担当PC
  ↓ VPN
LN6001-JP
  ↓
対象機器だけ
```

## おすすめ方針

![表画像 table-13](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-13.png)

## 運用メモ例

```txt
一時保守アクセス:
  日付: 2026-06-22
  目的: 店舗録画機の設定確認
  方式: Tailscale
  許可端末: vendor-support-laptop
  許可先: 192.168.3.20/32
  Exit Node: 無効
  作業後:
    Tailscale端末削除
    サブネットルート確認
    firewall rule削除
```

一時アクセスのつもりが、半年残る。

これ、かなり危ないです。

作業後に消すところまでが設定です。

## 構成例6: 外出先の通信を自宅経由にする

TailscaleのExit Nodeを使う構成です。

```txt
外出先スマートフォン
  ↓ Tailscale
LN6001-JP
  ↓ 自宅回線
インターネット
```

## 使いどころ

- 外出先のフリーWi-Fiから自宅回線経由で出たい
- 海外から日本の自宅回線経由で一部サービスへアクセスしたい
- 検証のために自宅回線からのアクセスにしたい
- すべての通信を特定拠点へ寄せたい

## 注意点

Exit Nodeは便利ですが、最初から有効にする必要はありません。

次の点を考えます。

- 外出先端末の全通信がLN6001-JP経由になる場合がある
- 自宅や店舗回線の上り帯域を使う
- ルーター側の負荷が増える
- DNSやAdblockの見え方が変わる
- Tailscale管理コンソール側でExit Node利用を許可する必要がある
- クライアント側でもExit Nodeを選択する必要がある
- ローカルLANアクセスの扱いが変わることがある

最初はExit Nodeを使わず、サブネットルーターとしてLAN内機器へ入れるところまでで十分です。

Exit Nodeは、必要になってからでOKです。

## WireGuardを選ぶ時の構成

WireGuardは、自分でVPN入口を管理する構成です。

```txt
外出先端末
  ↓ UDP 51820など
LN6001-JP WireGuard Server
  ↓
LAN内機器
```

## WireGuardに向く条件

- 固定IPがある
- DDNSを使える
- UDPポート開放できる
- 上位HGWやルーター側の転送設定を管理できる
- IPoE環境でも外部着信の条件を理解している
- 外部サービス依存を減らしたい
- Peerや鍵を自分で管理できる

## VPN Assistantでの流れ

1. Linksys公式サポートからVPN Assistantモジュールを入手する
2. LuCIの **System** → **Software** で **Update lists** を実行する
3. **Upload Package** から `.ipk` をアップロードしてインストールする
4. ルーターを再起動する
5. **Services** → **VPN Assistant** を開く
6. **WireGuard Server** タブを開く
7. **Enable WireGuard VPN** を有効にする
8. サーバー設定を入力して **Save & Apply** する
9. 必要に応じてルーターを再起動する
10. **Client Peers** の **Show QR** からクライアント設定を取得する

QRコードや設定テキストには秘密鍵が含まれます。

スクリーンショットをチャットやSNSへ貼らないでください。

## WireGuardのAllowedIPs例

![表画像 table-14](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-14.png)

最初は `0.0.0.0/0` を使わないほうが切り分けしやすいです。

必要なLANだけ通すスプリットトンネルから始めます。

## WireGuardの確認コマンド

```sh
echo "### WireGuard status"
wg show 2>/dev/null || echo "wg command is not available"

echo "### WireGuard config hints"
uci show network | grep -Ei 'wireguard|wg|vpn' || true

echo "### firewall hints"
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true

echo "### logs"
logread | grep -Ei 'wireguard|wg|51820|vpn' | tail -n 100
```

`wg show` でlatest handshakeが更新されない場合、外部から通信が届いていない可能性があります。

その場合は、固定IP、DDNS、上位ルーターのポート開放、IPoE方式、firewallを確認します。

## Tailscaleを選ぶ時の構成

Tailscaleは、固定IPやポート開放を前提にしないで始めやすい方式です。

```txt
外出先端末
  ↓ Tailscale
LN6001-JP
  ↓ LAN
NAS / LuCI / 管理PC / 録画機
```

## Tailscaleに向く条件

- 固定IPがない
- IPoE / IPv4 over IPv6でポート開放が不安
- スマートフォンやPCを複数管理したい
- 外出先からまず確実につなぎたい
- 端末削除や権限管理を管理コンソールで扱いたい
- 家庭や小さなオフィスで簡単に始めたい

## VPN Assistantでの流れ

1. VPN Assistantをインストールする
2. **Services** → **VPN Assistant** を開く
3. **Tailscale** タブを開く
4. **Enable Tailscale Service** を有効にする
5. 認証URLをブラウザで開く
6. TailscaleアカウントでLN6001-JPを登録する
7. LuCI側へ戻り、完了操作を行う
8. ルーター再起動後、Tailscale管理コンソールで端末を確認する

最初はExit Nodeを有効にしなくて大丈夫です。

まずLN6001-JPがtailnetに参加し、スマートフォンやPCから見えることを確認します。

## サブネットルーターの考え方

Tailscaleが入っていないLAN内機器へアクセスしたい場合、LN6001-JPをサブネットルーターとして使います。

例:

```txt
192.168.1.0/24
```

または、NASだけに絞るなら:

```txt
192.168.1.10/32
```

Tailscaleでは、ルートを広告したあと、管理コンソール側で承認が必要です。

ルーターがtailnetに見えていても、サブネットルートが承認されていなければLAN内機器へ届きません。

ここは本当にハマりやすいです。

```txt
Tailscale上では見える
でもNASへ届かない
```

という時は、まずサブネットルート承認を確認します。

## Tailscaleの確認コマンド

```sh
echo "### Tailscale status"
tailscale status 2>/dev/null || echo "tailscale command is not available or not authenticated"

echo "### Tailscale IP"
tailscale ip 2>/dev/null || true

echo "### Tailscale preferences"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|DNS' || true

echo "### logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

見るポイントです。

- LN6001-JPがtailnetに参加しているか
- Tailscale IPがあるか
- サブネットルートを広告しているか
- Exit Nodeを意図せず有効にしていないか
- ログに認証エラーがないか

## VPNからのアクセス範囲を表にする

設定前に、アクセス範囲を表にします。

![表画像 table-15](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-15.png)

この表があると、設定が広がりすぎにくくなります。

VPNは、あとから便利なので広げたくなります。

最初に表を作っておくと、不要な許可を減らせます。

## VPN専用zoneを作るかどうか

OpenWrt側でVPN専用zoneを作ると、VPNからLANへのアクセスをfirewallで細かく制御できます。

ただし、最初から必須ではありません。

家庭なら、Tailscale側の管理やWireGuardの `AllowedIPs` だけで十分な場合もあります。

小さなオフィスや店舗では、VPN専用zoneを作って、必要な宛先だけTraffic Ruleで許可する設計も検討します。

## 考え方

![表画像 table-16](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-16.png)

## Tailscaleをvpn zoneとして扱う例

ここからは上級者向けです。

Tailscaleの `tailscale0` をOpenWrtの `vpn` zoneとして扱う例です。

まず読み取り確認します。

```sh
ip addr show | grep -A5 -Ei 'tailscale|tailscale0'
uci show network | grep -Ei 'tailscale|vpn' || true
uci show firewall | grep -Ei 'tailscale|vpn' || true
```

必要な場合だけ設定します。

```sh
BACKUP_DIR="/root/vpn-zone-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

uci -q delete network.tailscale
uci set network.tailscale='interface'
uci set network.tailscale.proto='none'
uci set network.tailscale.device='tailscale0'
uci set network.tailscale.ifname='tailscale0'

uci -q delete firewall.vpn
uci set firewall.vpn='zone'
uci set firewall.vpn.name='vpn'
uci set firewall.vpn.network='tailscale'
uci set firewall.vpn.input='ACCEPT'
uci set firewall.vpn.output='ACCEPT'
uci set firewall.vpn.forward='REJECT'

uci changes network
uci changes firewall

uci commit network
uci commit firewall
/etc/init.d/network reload
/etc/init.d/firewall restart
```

この状態では、vpnからlanへ広くforwardingしていません。

必要な宛先だけTraffic Ruleで許可します。

例: VPNからNASのHTTPSだけ許可する。

```sh
uci -q delete firewall.vpn_to_nas_https
uci set firewall.vpn_to_nas_https='rule'
uci set firewall.vpn_to_nas_https.name='Allow-VPN-to-NAS-HTTPS'
uci set firewall.vpn_to_nas_https.src='vpn'
uci set firewall.vpn_to_nas_https.dest='lan'
uci set firewall.vpn_to_nas_https.dest_ip='192.168.1.10'
uci set firewall.vpn_to_nas_https.proto='tcp'
uci set firewall.vpn_to_nas_https.dest_port='443'
uci set firewall.vpn_to_nas_https.target='ACCEPT'

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

VPN Assistantの設定と衝突しないように、まずは読み取り確認から始めます。

家庭利用なら、ここまで細かくしなくてもよい場合があります。

## 設定前にバックアップを取る

VPNは、network、firewall、route、DNSに関係します。

設定前にバックアップします。

LuCIでは次です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` をPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-vpn-config-20260622.tar.gz
```

SSHでも状態メモを残します。

```sh
BACKUP_DIR="/root/vpn-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

VPNのQRコード、WireGuard秘密鍵、Tailscale認証情報、設定ファイルは公開しないでください。

VPN設定は、合鍵そのものに近い情報を含みます。

## VPN Assistantの導入確認

VPN Assistantは、Linksys公式サポートから `.ipk` を入手してLuCIでインストールします。

1. Linksys公式サポートページからVPN AssistantモジュールをPCへダウンロードする
2. LuCIの **System** → **Software** を開く
3. **Update lists** をクリックする
4. **Upload Package** から `.ipk` をアップロードする
5. **Install** をクリックする
6. 完了後、ルーターを再起動する
7. **Services** → **VPN Assistant** が出ることを確認する

CLIでは次を確認します。

```sh
echo "### VPN packages"
opkg list-installed | grep -Ei 'vpn|wireguard|tailscale' || true

echo "### init scripts"
ls /etc/init.d | grep -Ei 'tailscale|wireguard|vpn' || true

echo "### logs"
logread | grep -Ei 'vpn|wireguard|tailscale|opkg' | tail -n 100
```

最初は、公式UI手順で動く状態を作ることを優先します。

CLIは状態確認に使います。

## DNS設計も忘れない

VPN接続後に、IPアドレスでは届くのに名前では届かないことがあります。

たとえば、次のような状態です。

```txt
http://192.168.1.10 は開く
http://nas-home.lan は開かない
```

この場合、VPN経由のDNSを確認します。

## 確認コマンド

```sh
echo "### router DNS"
nslookup nas-home 127.0.0.1
nslookup nas-home.lan 127.0.0.1

echo "### dnsmasq config"
uci show dhcp | grep -Ei 'domain|local|dns|server'

echo "### VPN DNS hints"
tailscale debug prefs 2>/dev/null | grep -Ei 'DNS|MagicDNS|Route' || true
wg show 2>/dev/null || true
```

WireGuardでは、クライアント設定の `DNS = 192.168.1.1` が関係することがあります。

Tailscaleでは、MagicDNSや管理コンソール側のDNS設定が関係することがあります。

最初は、名前解決より先にIPアドレスで届くか確認します。

名前解決は、そのあとで大丈夫です。

## IPv6とVPN

VPNでIPv6をどう扱うかも、最初に考えすぎると難しくなります。

最初はIPv4 LANへのアクセスだけで十分なことが多いです。

```txt
192.168.1.0/24
```

WireGuardで全IPv6通信までVPNへ流す場合は、`::/0` などの設定が関係します。

TailscaleでExit Nodeを使う場合も、IPv4 / IPv6とDNSの経路が変わります。

最初からIPv6フルトンネルを目指すより、まず目的のLAN内機器へIPv4で届くことを確認します。

IPv6を本格的に扱う場合は、019のIPv6の落とし穴の記事と合わせて確認してください。

## よくあるトラブル

## WireGuardでhandshakeが発生しない

確認します。

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true
logread | grep -Ei 'wireguard|wg|51820' | tail -n 120
```

よくある原因です。

- EndpointのIPアドレスやDDNS名が違う
- UDPポート番号が違う
- 上位HGWやルーターでポート開放していない
- IPoE / DS-Lite / MAP-E環境で外部着信できない
- firewallでWireGuardポートを許可していない
- クライアントの鍵やPeer設定が違う

DS-Lite / MAP-E環境では、IPv4外部着信が使えない場合があります。

その場合は、IPv6でのWireGuard、固定IPサービス、またはTailscaleを検討します。

## WireGuardでhandshakeはあるがLANへ入れない

確認します。

```sh
wg show 2>/dev/null || true
ip route show
uci show firewall | grep -Ei 'wireguard|wg|vpn|lan|forward|rule'
```

よくある原因です。

- クライアント側 `AllowedIPs` にLANセグメントがない
- ルーター側firewallでVPNからLANへ許可していない
- LAN内機器側firewallが応答しない
- 接続先IPが変わっている
- DNS名だけ解決できていない

まずIPアドレスで確認します。

```txt
https://192.168.1.1
http://192.168.1.10
ssh user@192.168.1.20
```

## Tailscaleで認証できない

確認します。

```sh
ifstatus wan
ifstatus wan6
date
tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled|auth|login' | tail -n 120
```

よくある原因です。

- ルーターがインターネットへ出られていない
- 時刻が大きくずれている
- 認証URLを最後まで完了していない
- Tailscaleアカウント側の認証が止まっている
- VPN Assistant有効化直後で、まだ接続試行中

Tailscale有効化直後は、数分待ってから確認します。

## TailscaleでLN6001-JPは見えるがLAN内機器へ届かない

確認します。

```sh
tailscale status 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|Route' || true
ip route show
uci show firewall | grep -Ei 'tailscale|vpn|lan|forward|rule' || true
```

よくある原因です。

- サブネットルートを広告していない
- Tailscale管理コンソールでルート承認していない
- クライアント側がサブネットルートを使っていない
- OpenWrt側firewallで止めている
- LAN内機器側のfirewallが応答していない
- 接続先IPが変わっている

Tailscaleでは、広告したルートを管理コンソールで承認することを忘れがちです。

ここは毎回見ます。

## VPN接続後にインターネットが遅い

確認します。

```sh
ip route show
ip -6 route show
tailscale debug prefs 2>/dev/null | grep -Ei 'ExitNode|Route|DNS' || true
wg show 2>/dev/null || true
```

よくある原因です。

- WireGuardの `AllowedIPs` が `0.0.0.0/0` になっている
- TailscaleでExit Nodeを使っている
- 自宅や店舗回線の上り帯域が足りない
- DNSが遠回りしている
- VPN経由の全通信になっている

LAN内機器へ入りたいだけなら、フルトンネルやExit Nodeは不要です。

スプリットトンネルに戻すと改善することがあります。

## VPN経由でDNS名だけ解決できない

IPアドレスでは届くのに、名前だけ失敗する場合です。

```sh
nslookup nas-home 192.168.1.1
nslookup nas-home.lan 192.168.1.1
tailscale debug prefs 2>/dev/null | grep -Ei 'DNS|MagicDNS|Route' || true
```

見るポイントです。

- WireGuardクライアントにDNSを指定しているか
- TailscaleのDNS / MagicDNS設定がどうなっているか
- LN6001-JPのdnsmasqにホスト名があるか
- DHCP予約にhostnameが入っているか
- AdblockやFamily DNSが関係していないか

まずIPで届くことを確認します。

名前解決はそのあとです。

## 鍵・QRコード・端末管理

VPNの安全性は、設定後の管理に大きく左右されます。

## WireGuard

- 端末ごとにPeerを分ける
- スマートフォンとPCで同じ設定を使い回さない
- 端末を紛失したら該当Peerを削除する
- QRコードや設定ファイルを共有しない
- 秘密鍵をバックアップする場合は安全に保管する
- 退職者や不要端末のPeerを残さない
- Peer名に誰のどの端末か分かる名前を付ける

例:

```txt
wg-peer-iphone-ikm
wg-peer-macbook-ikm
wg-peer-admin-laptop
```

## Tailscale

- 管理コンソールで不要端末を削除する
- 紛失端末を無効化する
- 管理アカウントに二要素認証を設定する
- サブネットルートを広告している端末を把握する
- Exit Nodeを広告している端末を把握する
- 業務利用ではACL / grantsやタグを検討する
- 退職者や不要ユーザーの端末を残さない

Tailscaleは簡単に端末を追加できます。

そのぶん、使わなくなった端末を消す運用が大事です。

## VPN運用メモ

設定したら、次のようなメモを残します。

```txt
VPN構成:
  Tailscale

用途:
  外出先からLuCIとNASへアクセス

ルーター:
  ln6001-home

アクセス元:
  自分のスマートフォン
  自分のノートPC

許可範囲:
  192.168.1.1/32
  192.168.1.10/32

サブネットルート:
  192.168.1.10/32
  管理コンソールで承認済み

Exit Node:
  無効

WireGuard:
  未使用

バックアップ:
  backup-LN6001-before-vpn-config-20260622.tar.gz

注意:
  QRコード、秘密鍵、認証URLは公開しない
```

半年後に見ると、VPN設定はかなり忘れます。

目的、方式、許可範囲、端末、バックアップをメモしておくと復旧しやすくなります。

## 定期点検すること

月1回、または設定変更時に次を確認します。

![表画像 table-17](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/table-17.png)

VPNは、作ったあと放置しないほうが安全です。

「誰がどこへ入れるか」を定期的に見るだけでも、かなり違います。

## まとめ

VPNは、外出先から自宅や小さなオフィスへ安全に入るための便利な入口です。

LN6001-JPでは、Linksys公式VPN Assistantを使って、WireGuardとTailscaleを扱えます。

選び方はシンプルです。

```txt
固定IPやDDNS、ポート開放を管理できる
  → WireGuard

固定IPなし、IPoE環境、NAT越えを楽にしたい
  → Tailscale
```

迷うなら、まずTailscaleで小さく始めるのが現実的です。

外部サービス依存を減らしたい場合は、WireGuardを検討します。

ただし、方式選びより大事なのはアクセス範囲です。

最初からLAN全体へ入れる必要はありません。

```txt
LuCIだけ
NASだけ
管理PCだけ
録画機だけ
```

このように狭く始めるほうが安全です。

VPNは「つながること」がゴールではありません。

**必要な端末から、必要な機器へ、必要な範囲だけ入れる状態を作ること**がゴールです。

VPNは、全部屋のマスターキーではなく、必要な部屋へ行くための通行証。

そのくらいの距離感で使うと、家庭でも小さなオフィスでも店舗でも、かなり安心して運用できます。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/028/diagram-03.png)

## 次に読むなら

VPN構成を作るなら、次の記事も合わせて読むと整理しやすくなります。

- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [Tailscaleの使いどころ](https://note.com/ikmsan/n/n8d21425036a9)
- [ポート開放の注意点](https://note.com/ikmsan/n/n765ffea6ea82)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)

VPN Assistantの導入手順を詳しく見たい人は013の記事へ。

Tailscaleの運用やサブネットルーターを詳しく見たい人は018の記事へ。

ポート開放とVPNのどちらにするか迷う人は017の記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

VPN Assistant、WireGuard、Tailscale関連の画面名、コマンド出力、Tailscale管理コンソール、サブネットルート、Exit Node、ACL / grantsの仕様は、モジュールやサービス更新で変わることがあります。

IPoE / IPv4 over IPv6、PPPoE、固定IP、DDNS、ポート開放の条件は、契約している回線やプロバイダによって異なります。

記事の内容と実際の画面や挙動が違う場合は、まずバックアップを取り、Linksys公式サポート、Tailscale公式ドキュメント、WireGuard公式情報、契約中のプロバイダ情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPではWireGuardとTailscaleのどちらがいい？

固定IPやDDNS、ポート開放を自分で管理できるならWireGuardが向いています。

固定IPなし、IPoE環境、NAT越えを楽にしたいならTailscaleが始めやすいです。

### IPoE環境でもWireGuardは使える？

使える場合もあります。

ただし、MAP-EやDS-LiteなどのIPv4 over IPv6環境では、IPv4の外部着信やポート開放に制約がある場合があります。

その場合は、IPv6到達、固定IPサービス、またはTailscaleを検討します。

### Tailscaleならポート開放はいらない？

多くの場合、手動ポート開放を意識せず始めやすいです。

ただし、Tailscaleアカウント、端末管理、サブネットルート承認、Exit Node設定など、Tailscale側の運用は必要です。

### VPNでLAN全体へ入れるようにしていい？

最初からLAN全体へ入れる必要はありません。

NASだけ、LuCIだけ、管理PCだけのように、必要な範囲へ絞るほうが安全です。

### WireGuardのAllowedIPsは何を入れればいい？

目的に合わせて絞ります。

LuCIだけなら `192.168.1.1/32`、NASだけならNASのIP、LAN全体なら `192.168.1.0/24` です。

`0.0.0.0/0` は全通信をVPNへ流す設定なので、最初から使う必要はありません。

### Tailscaleのサブネットルーターが効かない時は？

Tailscale管理コンソールでサブネットルートを承認しているか確認します。

ルーターがtailnetに参加していても、ルート承認が済んでいないとLAN内機器へ届きません。

### Exit Nodeは有効にしたほうがいい？

最初は不要です。

LAN内NASやLuCIへ入りたいだけなら、サブネットルーターで十分なことが多いです。

Exit Nodeは、外出先端末のインターネット通信をLN6001-JP経由にしたい場合に検討します。

### VPNのQRコードを保存・共有していい？

おすすめしません。

WireGuardのQRコードや設定テキストには秘密鍵が含まれます。

スクリーンショットをチャット、SNS、公開リポジトリへ貼らないでください。

### 保守担当に一時的にVPNを渡してもいい？

必要な場合はできます。

ただし、対象機器、期間、端末、権限を絞り、作業後にPeerやTailscale端末を削除してください。

一時アクセスを常設にしないことが大事です。

### POSや決済端末へVPNで入ってもいい？

慎重に判断してください。

POSや決済端末は、ベンダー要件やサポート条件がある場合があります。

VPNで入れるからといって、勝手に触らないほうが安全です。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Tailscale Subnet routers: https://tailscale.com/docs/features/subnet-routers
- Tailscale Configure a subnet router: https://tailscale.com/docs/features/subnet-routers/how-to/setup
- Tailscale Exit nodes: https://tailscale.com/docs/features/exit-nodes
- Tailscale Use exit nodes: https://tailscale.com/docs/features/exit-nodes/how-to/setup
- Tailscale CLI `up`: https://tailscale.com/docs/reference/tailscale-cli/up
- WireGuard Quick Start: https://www.wireguard.com/quickstart/

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
