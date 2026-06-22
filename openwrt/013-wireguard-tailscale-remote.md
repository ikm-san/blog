<!-- mirror-source: articles/013-wireguard-tailscale-remote.md -->

# ポート開放の前にVPNを考える｜WireGuardとTailscaleで外から安全に入る【OpenWrt集中連載013】

外出先から自宅や小さなオフィスへ入りたい時、まず悩むのがここです。

NASを見たい。  
ミニPCへSSHしたい。  
店舗の録画機を確認したい。  
LuCIでルーターの状態を見たい。  
家にいる時と同じように、外から安全にアクセスしたい。

こういう時、昔ながらの発想だと「ポート開放」が出てきます。

でも、ちょっと待ってください。

ポート開放は便利です。  
ただし、言い方を変えると、

```txt
インターネット側から入れるドアを1つ増やす
```

設定です。

NASの管理画面を開ける。  
カメラの管理画面を開ける。  
SSHを開ける。  
LuCIを開ける。

できることは増えますが、外から見える入口も増えます。

自分だけが使いたいなら、まずVPNでよくない？  
というのが、この記事の出発点です。

LN6001-JPでは、Linksys公式のVPN Assistantモジュールを使って、WireGuardとTailscaleを扱えます。

固定IPやポート開放を自分で管理できるならWireGuard。  
固定IPなし、IPoE環境、NAT越えを楽にしたいならTailscale。

ざっくり言うと、そんな選び方です。

でも、VPNで一番大事なのは「どちらが強いか」ではありません。

**つないだあと、どこまで入れるようにするか** です。

VPNは、安全な専用通路です。  
でも、その通路の先を家じゅう全部に開ける必要はありません。

まずはLuCIだけ。  
NASだけ。  
管理用ミニPCだけ。

小さく始めるほうが安全です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、VPNを完璧に設計することではありません。

まずは、ここまで分かればOKです。

- WireGuardとTailscaleのざっくりした違いが分かる
- 固定IPやIPoE環境で、どちらを選ぶと楽か判断できる
- VPN Assistantを使う流れが分かる
- WireGuardの `AllowedIPs` でアクセス範囲を絞る意味が分かる
- TailscaleのサブネットルーターとExit Nodeの違いが分かる
- VPNでLAN全体をいきなり開かなくてよいと分かる
- QRコード、秘密鍵、Tailscale端末管理の注意点が分かる
- つながらない時に見るコマンドが分かる

VPNは「つながった！」で終わりではありません。

```txt
誰が
どの端末から
どこへ
どの範囲で
入れるのか
```

ここまで決めて、ようやく運用しやすくなります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-01.png)

## 先にざっくり結論

最初は、この分け方で十分です。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-01.png)

迷ったら、まずTailscaleから試すのが入りやすいです。

特に日本の家庭回線では、IPoE / IPv4 over IPv6、ホームゲートウェイ、MAP-E、DS-Liteなどが絡みます。

その結果、

```txt
外から自宅へWireGuardのUDPポートを届ける
```

ところで詰まる場合があります。

一方で、固定IPやDDNSがあり、ポート開放も自分で管理できるなら、WireGuardはシンプルで扱いやすい選択肢です。

ただし、どちらを使う場合も、最初からLAN全体を丸ごと開く必要はありません。

最初はこれくらいで十分です。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-02.png)

VPNは「外から安全に入る入口」です。

でも入口を作ったからといって、家の中や事務所の中を全部歩き回れるようにする必要はありません。

## こういう人向けです

この記事は、次のような人向けです。

- 外出先から自宅NASへ入りたい
- 小さなオフィスの管理端末へSSHしたい
- 店舗の録画機やカメラを見たい
- LuCIやSSHをインターネットへ直接公開したくない
- 固定IPがないけれどリモート接続したい
- IPoE / IPv4 over IPv6環境でWireGuardが使えるか不安
- WireGuardとTailscaleのどちらから始めるべきか迷っている
- VPNでLAN全体を開けすぎない設計を知りたい

逆に、

```txt
VPNの理論を全部理解したい
大規模拠点間VPNを設計したい
業務用ゼロトラスト設計を細かく作り込みたい
```

という人には、この記事は入口寄りです。

ここでは、家庭・小さなオフィス・店舗でまず安全に使い始めることを優先します。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-02.png)

## 最初に言葉だけそろえる

VPNまわりで出てくる言葉を、ざっくり整理します。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-03.png)

最初は、これだけで大丈夫です。

```txt
WireGuard = 自分で入口を作る
Tailscale = 入口作りやNAT越えをかなり楽にする
AllowedIPs = VPNでどこへ行くか決める
Exit Node = 全通信をその出口へ流す
```

## ポート開放とVPNの違い

ポート開放は、外からLAN内の機器へ直接届ける設定です。

たとえば、NASの管理画面、カメラのWeb UI、SSH、ゲームサーバーなどを外から見たい時に使われます。

でも、ポート開放は外向きの入口を増やす設定です。

もちろん、適切に管理すれば使えます。

ただ、家庭や小さなオフィスで「自分だけが外から入りたい」なら、まずVPNを考えたほうが安全に設計しやすいです。

イメージとしてはこうです。

```txt
ポート開放:
  外から特定サービスの玄関を直接開ける

VPN:
  まず専用通路に入る
  そのあと必要な機器だけ見る
```

NASを外に直接出すより、VPNで入ってからNASを見る。  
LuCIを直接公開するより、VPNで入ってからLuCIを見る。  
店舗の録画機を外に出すより、VPNで入ってから録画機を見る。

このほうが、入口を管理しやすくなります。

## VPNで最初に決めるべきこと

VPN設定で最初に決めるのは、方式ではありません。

まず、目的です。

```txt
誰が
どの端末から
どこへ
何のために
どの範囲で
入るのか
```

これを決めずにVPNを作ると、だいたい広げすぎます。

## どこへ入りたいのか

目的ごとに、アクセス範囲を小さく考えます。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-04.png)

`192.168.1.0/24` 全体へ入れる設定は便利です。

でも、目的がNASだけなら、NASのIPアドレスだけでも十分です。

最初は狭く。  
必要になったら広げる。

VPNはこの順番が安全です。

## どの端末から入るのか

接続元も決めます。

- 自分のスマートフォン
- 自分のノートPC
- 管理用PC
- 家族の端末
- スタッフの端末
- 保守用端末

VPNに入れる端末は、信頼できる端末だけにします。

特に小さなオフィスや店舗では、退職者の端末、紛失端末、使わなくなったスマートフォンを放置しないことが大事です。

VPNは便利ですが、入口でもあります。

使わなくなった入口は閉じます。

## WireGuardとTailscaleの違い

WireGuardとTailscaleは、どちらもVPNとして使えます。

ただし、運用の考え方が少し違います。

## WireGuard

WireGuardは、自宅や小さなオフィス側にVPN入口を作り、外出先端末からそこへ接続するイメージです。

構成はシンプルです。

```txt
外出先スマホ / PC
  ↓ WireGuard
LN6001-JP
  ↓
NAS / LuCI / ミニPC
```

WireGuardに向いているのは、次のような環境です。

- 固定IPがある
- DDNSを使える
- 上位HGWやルーターでポート開放できる
- IPoE環境でも外からUDPポートを届けられる
- 自分でPeerや鍵を管理できる
- 外部サービス依存を減らしたい

WireGuardはシンプルで速いです。

ただし、外からLN6001-JPへ到達できる入口を自分で用意する必要があります。

ここがハマりポイントです。

## Tailscale

Tailscaleは、WireGuardをベースにしつつ、NAT越えや端末管理をかなり楽にしてくれるサービスです。

イメージとしてはこうです。

```txt
外出先スマホ / PC
  ↓ Tailscale
Tailnet
  ↓
LN6001-JP
  ↓
NAS / LuCI / ミニPC
```

Tailscaleに向いているのは、次のような環境です。

- 固定IPがない
- IPoE / IPv4 over IPv6でポート開放が難しい
- HGW配下や二重ルーターで、外からの着信が分かりにくい
- スマートフォンやPCなど複数端末を管理したい
- まず確実につながる状態を作りたい
- Tailscale管理コンソールで端末を管理したい

Tailscaleは始めやすいです。

ただし、Tailscaleアカウント、管理コンソール、ルート承認、不要端末の削除なども運用に含まれます。

便利なぶん、管理コンソールを放置しないことが大事です。

## 選び方の目安

迷ったら、次で決めます。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-05.png)

最初から両方を同時に入れる必要はありません。

まず片方で動く構成を作る。

ログを見る。  
通信範囲を見る。  
切断方法を確認する。  
不要端末の消し方を確認する。

そこまで分かってから、必要ならもう片方を検討します。

## 設定前にバックアップを取る

VPN設定は、network、firewall、route、DNS、パッケージ状態に関係します。

設定前にバックアップを取ります。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-vpn-assistant-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

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
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
uci show firewall > "$BACKUP_DIR/firewall-full.uci.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

このバックアップには、内部IP、firewall設定、VPN関連情報が含まれることがあります。

公開しないようにしてください。

また、VPNのQRコード、秘密鍵、認証URL、設定ファイルは特に注意です。

スクリーンショットをSNSやチャットへ貼らないでください。

## 回線条件を確認する

VPN Assistantを入れる前に、回線条件を見ます。

特にWireGuardを使いたい場合は重要です。

```sh
echo "### WAN status"
ifstatus wan
ifstatus wan6

echo "### routes"
ip route show
ip -6 route show

echo "### current firewall WAN rules"
uci show firewall | grep -Ei 'wan|51820|wireguard|tailscale|vpn' || true
```

WireGuardでは、外からLN6001-JPへUDPポートを届ける必要があります。

次を確認します。

- 固定IP契約があるか
- DDNSを使えるか
- 上位にホームゲートウェイや別ルーターがあるか
- 上位ルーター側でポート開放できるか
- IPoE / IPv4 over IPv6の方式は何か
- DS-LiteのようにIPv4外部着信が難しい方式ではないか
- MAP-Eで使えるポート範囲に制約がないか

ここで「よく分からない」となったら、Tailscaleから始めるのがかなり現実的です。

VPNを始めたいのに、回線方式の沼で力尽きるのはもったいないです。

## VPN Assistantをインストールする

LN6001-JPでは、Linksys公式サポートでWireGuard / Tailscale向けのVPN Assistantモジュールが案内されています。

最初は、一般的なOpenWrt向け手順をそのままコピペするより、公式VPN Assistantを使うほうが安全です。

## LuCIでインストールする

1. Linksys公式サポートページから、LN6001-JP用のVPN Assistantモジュール `.ipk` をPCへダウンロードする
2. LuCIへログインする
3. **System** → **Software** を開く
4. **Update lists** をクリックする
5. 完了後、**Upload Package** をクリックする
6. ダウンロードしたVPN Assistantの `.ipk` を選択する
7. **Upload** をクリックする
8. 確認画面で **Install** をクリックする
9. 完了後、ルーターを再起動する
10. LuCIへ再ログインし、**Services** に **VPN Assistant** が追加されていることを確認する

インストール後、CLIでも確認できます。

```sh
echo "### VPN related packages"
opkg list-installed | grep -Ei 'vpn|wireguard|tailscale|luci-app-wireguard|kmod-wireguard' || true

echo "### services"
ls /etc/init.d | grep -Ei 'wireguard|tailscale|vpn' || true

echo "### recent logs"
logread | grep -Ei 'vpn|wireguard|tailscale|opkg' | tail -n 100
```

メニューが出ない場合は、LuCIを再読み込みするか、再ログイン、再起動を試します。

## WireGuardを使う

WireGuardは、自分でVPN入口を管理する方式です。

固定IPやDDNS、ポート開放を管理できるなら、かなり分かりやすい構成になります。

## WireGuardに向いている条件

WireGuardは、次の条件がそろっていると使いやすいです。

- 固定IPまたはDDNSがある
- 上位ルーターやホームゲートウェイでポート開放できる
- LN6001-JPへ外からUDPポートを届けられる
- 自分でPeerや鍵を管理できる
- 外部サービス依存を減らしたい

IPoE / IPv4 over IPv6環境では、プロバイダーや方式によってポート開放の可否が変わります。

ここで詰まる場合は、Tailscaleのほうが楽です。

## VPN AssistantでWireGuard Serverを有効にする

1. **Services** → **VPN Assistant** を開く
2. **WireGuard Server** タブを開く
3. **Enable WireGuard VPN** を有効にする
4. 必要な項目を入力する
5. **Save & Apply** をクリックする
6. 設定反映後、必要に応じてルーターを再起動する
7. **Client Peers** で接続端末を作成する
8. **Show QR** からスマートフォン用QRコード、またはテキスト設定を取得する

最初はスマートフォン1台だけ登録し、モバイル回線から接続確認するのがおすすめです。

自宅Wi-Fiにつながったままだと、本当に外から入れているか分かりにくいです。

## WireGuardのAllowedIPsを狭くする

WireGuardで大事なのが `AllowedIPs` です。

これは、どの宛先をVPNへ流すかを決める設定です。

最初は、必要な範囲だけに絞ります。

### LuCIだけへ入る

```ini
AllowedIPs = 192.168.1.1/32
```

LN6001-JPの管理画面だけ確認したい場合です。

### NASだけへ入る

```ini
AllowedIPs = 192.168.1.10/32
```

NASだけ使いたい場合です。

### ミニPCへSSHする

```ini
AllowedIPs = 192.168.1.20/32
```

管理用ミニPCへSSHしたい場合です。

### Staff LAN全体へ入る

```ini
AllowedIPs = 192.168.1.0/24
```

便利ですが、範囲は広くなります。

### 全通信をVPNへ流す

```ini
AllowedIPs = 0.0.0.0/0
```

これはフルトンネルです。

外出先の通信をすべて自宅やオフィス経由にしたい場合に使います。

ただし、最初からこれにする必要はありません。

まずはスプリットトンネルで、必要なLANだけ通すほうが切り分けしやすいです。

## WireGuard設定例

以下は考え方の例です。

実際の鍵やエンドポイントはVPN Assistantで生成したものを使います。

```ini
[Interface]
PrivateKey = <クライアントの秘密鍵>
Address = 10.0.0.2/24
DNS = 192.168.1.1

[Peer]
PublicKey = <LN6001-JP側の公開鍵>
Endpoint = <固定IPまたはDDNSホスト名>:51820
AllowedIPs = 192.168.1.0/24
PersistentKeepalive = 25
```

QRコードを使う場合、スマートフォンへの取り込みはかなり簡単です。

ただし、QRコードや設定ファイルには秘密鍵が含まれます。

これを人に渡すということは、そのVPN入口の合鍵を渡すのに近いです。

扱いはかなり慎重にしてください。

## WireGuard用firewallを確認する

VPN Assistantを使う場合は、まずVPN Assistantが作った設定を確認します。

手動でfirewall ruleを追加する場合は、設定が重複しないように注意します。

確認用コマンドです。

```sh
echo "### WireGuard firewall hints"
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true

echo "### WireGuard interfaces"
ip addr show | grep -A5 -Ei 'wg|wireguard' || true

echo "### WireGuard command"
if command -v wg >/dev/null; then
  wg show
else
  echo "wg command is not installed"
fi

echo "### logs"
logread | grep -Ei 'wireguard|wg|51820|vpn' | tail -n 100
```

手動でUDP 51820を開ける例は次の通りです。

VPN Assistantがすでに作っている場合は重複させないでください。

```sh
cp /etc/config/firewall /etc/config/firewall.backup.before-wg-rule.$(date +%Y%m%d-%H%M)

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'

uci changes firewall
uci commit firewall
/etc/init.d/firewall restart
```

## WireGuardの動作確認

スマートフォンやPCからVPN接続したあと、まず狭い範囲で確認します。

```sh
echo "### WireGuard status"
wg show 2>/dev/null || echo "wg command is not available"

echo "### routes"
ip route show

echo "### firewall logs"
logread | grep -Ei 'wireguard|wg|51820' | tail -n 80
```

外出先端末側では、次を確認します。

- VPNが接続状態になる
- `192.168.1.1` へpingできる
- LuCIを開ける
- NASやミニPCなど、許可した機器へだけ届く
- 許可していないネットワークへ届かない

WireGuardでつながらない時は、最初に「UDPポートが外から届いているか」を疑います。

特に、ホームゲートウェイ配下やIPoE環境ではここで詰まりやすいです。

## Tailscaleを使う

Tailscaleは、固定IPやポート開放をあまり気にせず始めやすい方式です。

外出先から自宅や小さなオフィスへ入りたいけれど、回線方式やポート開放で悩みたくない場合にかなり便利です。

## Tailscaleに向いている条件

Tailscaleは、次のような場合に向いています。

- 固定IPがない
- IPoE / IPv4 over IPv6環境でポート開放が難しい
- ホームゲートウェイ配下で外からの着信が分かりにくい
- スマートフォン、PC、複数拠点を管理したい
- まず確実につながる構成を作りたい
- 不要端末を管理コンソールから削除したい

一方で、Tailscaleアカウントと管理コンソールに依存します。

家庭なら楽に始められます。

小さなオフィスや店舗で使うなら、アカウント管理、退職者端末、不要端末の削除まで含めて考えます。

## VPN AssistantでTailscaleを有効にする

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブを開く
3. **Enable Tailscale Service** を有効にする
4. 必要に応じて **Advertise as Exit Node** を選ぶ
5. 認証用URLが表示されたらブラウザで開く
6. Tailscaleアカウントでログインし、LN6001-JPを登録する
7. LuCI側へ戻り、完了操作を行う
8. 必要に応じてルーターを再起動する

最初はExit Nodeを有効にしなくて大丈夫です。

まずはTailscale上でLN6001-JPが見えること、外出先端末からLN6001-JPへ届くことを確認します。

## サブネットルーターとして使う

Tailscaleが入っていないLAN内機器へアクセスしたい場合は、LN6001-JPをサブネットルーターとして使います。

たとえば、次のような用途です。

- 外出先スマートフォンからNASへアクセスする
- ノートPCから事務所のミニPCへSSHする
- Tailscaleが入っていないプリンターやカメラへ接続する

サブネットルーターでは、LN6001-JPが、

```txt
このLANへは自分経由で入れます
```

とTailscaleへ広告します。

例:

```txt
192.168.1.0/24
```

ただし、Tailscaleではルートを広告したあと、管理コンソール側で承認が必要です。

Tailnetへ参加できていても、サブネットルートが承認されていないとLAN内機器へ届きません。

ここはかなりハマりやすいです。

## サブネットルートをCLIで確認する

VPN Assistantで設定した内容を確認する時は、まず読み取りから始めます。

```sh
echo "### Tailscale status"
tailscale status 2>/dev/null || echo "tailscale command is not available or not authenticated"

echo "### Tailscale IP"
tailscale ip 2>/dev/null || true

echo "### Tailscale debug prefs"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|Shields' || true

echo "### logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

CLIでルート広告を設定する場合は、VPN Assistantの設定と衝突しないように注意します。

例:

```sh
tailscale set --advertise-routes=192.168.1.0/24
```

設定後、Tailscale管理コンソールで該当ルートを承認します。

## Exit Nodeは最初から不要

Exit Nodeは、外出先端末のインターネット通信をLN6001-JP経由にするための機能です。

便利な場面はあります。

たとえば、外出先のフリーWi-Fiから、自宅やオフィス回線経由でインターネットへ出したい場合です。

ただし、最初から有効にする必要はありません。

Exit Nodeを使うと、次の点を考える必要があります。

- 外出先端末の全通信がLN6001-JP経由になる
- 自宅やオフィス回線の上り帯域を使う
- ルーター側の負荷が増える
- DNSや広告ブロックの見え方が変わる
- Tailscale管理コンソール側でも設定が必要になる
- クライアント側でもExit Nodeを選ぶ必要がある

最初は、サブネットルーターとしてLANへ入るところまでで十分です。

Exit Nodeは、必要になってからでOKです。

## Tailscaleの動作確認

Tailscaleで接続確認する時は、次を見ます。

```sh
echo "### service"
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale status
fi

echo "### status"
tailscale status 2>/dev/null || true

echo "### ip"
tailscale ip 2>/dev/null || true

echo "### routes"
ip route show
ip -6 route show

echo "### logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

外出先端末側では、次を確認します。

- TailscaleアプリでLN6001-JPが見える
- LN6001-JPのTailscale IPへpingできる
- サブネットルートを承認後、`192.168.1.1` へ届く
- NASやミニPCなど、目的の機器へ届く
- 使わないネットワークへ広く入れる設定になっていない

## WireGuardとTailscaleを同時に使うべきか

最初は同時に使わないほうが分かりやすいです。

両方を有効にすると、次のような混乱が起きやすくなります。

- どちらのVPN経由で通信しているか分からない
- DNSの経路が分かりにくい
- firewallルールの切り分けが難しい
- LANへ届かない時の原因が増える
- 端末ごとの接続管理が散らばる

まずTailscaleで動かす。

または、まずWireGuardで動かす。

片方で接続、ログ、アクセス範囲、切断方法まで理解してから、必要があればもう片方を検討します。

## アクセス範囲を絞る設計

VPNで最も大事なのは、アクセス範囲です。

## LuCIだけ使う

```txt
192.168.1.1/32
```

ルーター管理だけ確認したい場合です。

最初の検証にも向いています。

## NASだけ使う

```txt
192.168.1.10/32
```

NASだけへアクセスしたい場合です。

バックアップやファイル確認だけなら、この範囲で十分なことがあります。

## Staff LANだけ使う

```txt
192.168.1.0/24
```

小さなオフィスのStaffネットワークへ入る場合です。

便利ですが、アクセス範囲は広くなります。

## Device / Cameraへも入りたい

```txt
192.168.1.0/24
192.168.3.0/24
```

カメラ用ネットワークや録画機も管理したい場合です。

ただし、Device / CameraネットワークへVPNから入れる必要が本当にあるかは確認します。

店舗では、POSや決済端末へVPNから入れる設計は慎重に扱ってください。

## firewall zoneとの関係

VPNを作ったあと、どのzoneに入れるかが重要です。

よくある考え方は次の通りです。

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/table-06.png)

VPNを `lan` と同じ扱いにすると楽です。

でも、アクセス範囲は広くなります。

小さなオフィスや店舗では、VPN専用zoneを作って、必要な宛先だけ許可する設計も検討します。

## 鍵・QRコード・アカウント管理

VPNは、設定後の管理がかなり大事です。

## WireGuardの場合

WireGuardでは、Peerごとに鍵を管理します。

- 端末ごとにPeerを分ける
- スマートフォンとPCで同じ設定を使い回さない
- 端末を紛失したら、そのPeerを削除する
- QRコードをスクリーンショットで放置しない
- 秘密鍵をチャットやSNSへ貼らない
- 退職者や不要端末のPeerを残さない

「誰のどの端末がどのPeerか」をメモしておくと運用しやすくなります。

例:

```txt
wg-peer-iphone-ikm
wg-peer-macbook-ikm
wg-peer-admin-laptop
```

## Tailscaleの場合

Tailscaleでは、管理コンソールで端末を管理します。

- 使っていない端末を削除する
- 紛失端末を無効化する
- 管理者アカウントを適切に管理する
- 業務利用では退職者のアカウント・端末を確認する
- サブネットルートの承認状態を確認する
- Exit Nodeを有効にしている端末を把握する

Tailscaleは便利ですが、端末を増やしやすいぶん、不要端末も残りやすいです。

管理コンソールをたまに見る習慣を作っておくと安心です。

## VPN運用メモ

VPNを設定したら、次のようなメモを残しておくと便利です。

```txt
VPN方式:
  Tailscale

用途:
  外出先からNASとLuCIへアクセス

許可範囲:
  192.168.1.1/32
  192.168.1.10/32

Tailscale:
  Device name: ln6001-home
  Subnet route: 192.168.1.0/24
  Exit Node: disabled
  管理コンソールでルート承認済み

WireGuard:
  未使用

バックアップ:
  backup-LN6001-before-vpn-assistant-20260621.tar.gz

注意:
  QRコード、秘密鍵、認証URLは公開しない
```

VPN設定は、半年後に見返した時に「何のために入れたのか」が分からなくなりがちです。

目的と許可範囲をメモしておくと、あとからかなり助かります。

## よくある問題と対処

## WireGuardで接続できない

まず見るべきポイントは、外からLN6001-JPへ届いているかです。

確認します。

- 固定IPまたはDDNSが正しいか
- Endpointのホスト名やIPアドレスが正しいか
- UDPポート番号が一致しているか
- ホームゲートウェイ側でポート開放されているか
- LN6001-JP側firewallでWireGuardポートを許可しているか
- IPoE / IPv4 over IPv6でそのポートが使える条件か
- クライアントのAllowedIPsが正しいか

CLIでは次を確認します。

```sh
wg show 2>/dev/null || true
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true
logread | grep -Ei 'wireguard|wg|51820' | tail -n 100
```

`wg show` でlatest handshakeが更新されない場合、そもそも通信が届いていない可能性があります。

## Tailscaleで認証できない

次を確認します。

- LN6001-JPがインターネットへ出られるか
- 時刻が大きくずれていないか
- Tailscale認証URLを正しく開いたか
- Tailscaleアカウントでログインできるか
- LuCI側で完了操作を行ったか
- tailscaledが起動しているか

CLIでは次を確認します。

```sh
tailscale status 2>/dev/null || true
/etc/init.d/tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

## TailscaleでLANへ届かない

Tailnet上でLN6001-JPが見えていても、LANへ届かないことがあります。

よくある原因です。

- サブネットルートを広告していない
- Tailscale管理コンソールでルート承認していない
- クライアント側でサブネットルートを使っていない
- firewallでVPNからLANへの通信が許可されていない
- LAN内機器側のfirewallが応答していない

確認します。

```sh
tailscale status 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|Route' || true
ip route show
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

Tailscale管理コンソールでSubnetsの承認状態も確認します。

## VPN接続は成功するがNASへ入れない

次を確認します。

- NASのIPアドレスが変わっていないか
- DHCP予約しているか
- VPN側AllowedIPsやサブネットルートにNASのIPが含まれているか
- NAS側firewallやアクセス制限があるか
- NASが別VLANやDeviceネットワークにいるか
- DNS名ではなくIPアドレスで接続できるか

DNS名でつながらないだけの場合、VPN経由のDNS設定が原因のことがあります。

まずIPアドレスで確認します。

```txt
http://192.168.1.10
ssh user@192.168.1.10
```

## VPN接続後にインターネットが遅い

フルトンネルやExit Nodeを使っている可能性があります。

確認するポイントです。

- WireGuardのAllowedIPsが `0.0.0.0/0` になっていないか
- TailscaleでExit Nodeを使っていないか
- DNSが遠回りしていないか
- 自宅やオフィス回線の上り帯域が足りているか

最初は、必要なLANだけ通すスプリットトンネルのほうが切り分けしやすいです。

## VPN経由でLuCIが開けてしまうのが不安

VPNの目的がLuCI管理なら問題ありません。

ただし、VPN利用者全員にLuCIを開かせる必要はありません。

次を検討します。

- VPN接続できる端末を絞る
- VPNからLuCIへのアクセスを管理端末だけにする
- VPN専用zoneを作り、必要な宛先だけ許可する
- 不要なPeerやTailscale端末を削除する

VPNは便利ですが、管理画面へ入れる端末を増やしすぎないほうが安全です。

## WireGuardを無効化する

使わなくなった場合は、VPN AssistantでWireGuardを無効化します。

LuCIから行うのが安全です。

1. **Services** → **VPN Assistant** を開く
2. **WireGuard Server** タブを開く
3. **Enable WireGuard VPN** を無効にする
4. **Save & Apply** をクリックする
5. 不要なPeerやfirewall ruleが残っていないか確認する

CLIで確認します。

```sh
uci show firewall | grep -Ei 'wireguard|wg|51820|vpn' || true
wg show 2>/dev/null || true
logread | grep -Ei 'wireguard|wg' | tail -n 80
```

## Tailscaleを無効化する

Tailscaleを使わなくなった場合は、LuCIのVPN Assistantで無効化します。

あわせて、Tailscale管理コンソール側でもLN6001-JPの端末を無効化または削除します。

CLIでサービスを止める場合は次です。

```sh
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale stop
  /etc/init.d/tailscale disable
fi

tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 80
```

Tailscaleは、ルーター側を止めるだけでなく、管理コンソール側の端末管理も忘れないようにします。

## まとめ

LN6001-JPでは、Linksys公式のVPN Assistantモジュールを使って、WireGuardとTailscaleを扱えます。

選び方はシンプルです。

- 固定IP、DDNS、ポート開放を管理できるならWireGuard
- 固定IPなし、IPoE環境、NAT越えを楽にしたいならTailscale
- まず確実につながる構成を作りたいならTailscaleから始める
- 外部サービス依存を減らしたいならWireGuardを検討する

ただし、方式選びより大事なのはアクセス範囲です。

どちらを選んでも、最初からLAN全体を広く許可する必要はありません。

まずは、LuCIだけ。  
NASだけ。  
管理用ミニPCだけ。

VPNは「外から安全に入る入口」です。

その入口から、どこまで入れるかを決めるのが設計です。

まずは1方式、1端末、1用途から始める。

これが、家庭や小さなオフィスでVPNを安全に運用する一番現実的な進め方だと思います。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/013/diagram-03.png)

## 次に読むなら

VPNの次は、アクセス先や通信範囲に応じて読み進めると整理しやすいです。

- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)
- [IPv6の落とし穴](https://note.com/ikmsan/n/n61235a13b478)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)

VPNでNASやミニPCへ入りたい人は、固定IPとDHCP予約の記事へ。

VPNからどのzoneへ通すかを整理したい人は、firewall zonesの記事へ。

WireGuard/Tailscaleの構成パターンをもう少し見たい人は、VPN向け構成例へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

VPN Assistantモジュール、WireGuard、Tailscale関連の画面やコマンド出力は、モジュールやファームウェア更新で変わることがあります。

また、IPoE / IPv4 over IPv6、ホームゲートウェイ、固定IP、DDNS、ポート開放の条件は、契約している回線やプロバイダーによって異なります。

記事の内容と実際の画面や挙動が違う場合は、まずバックアップを取り、Linksys公式サポート、Tailscale公式ドキュメント、契約中のプロバイダー情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### LN6001-JPではWireGuardとTailscaleのどちらがいい？

固定IPやDDNS、ポート開放を自分で管理できるならWireGuardが向いています。

固定IPがない、IPoE環境でポート開放が不安、まず簡単につなぎたいという場合はTailscaleが始めやすいです。

### IPoE / IPv4 over IPv6環境ではWireGuardは使えない？

必ず使えないわけではありません。

ただし、MAP-E、DS-Lite、IPIPなど方式やプロバイダー条件によって、外から任意ポートを届けられるかが変わります。

ポート開放で悩む場合は、Tailscaleを先に試すほうが現実的です。

### Tailscaleならポート開放はいらない？

多くの場合、固定IPや手動ポート開放を意識せず始めやすいです。

ただし、Tailscaleアカウント、端末管理、サブネットルート承認、Exit Node設定など、Tailscale側の管理は必要です。

### VPNを使えばLAN全体へ入れるようにしていい？

最初からLAN全体へ入れる必要はありません。

NASだけ、LuCIだけ、管理PCだけのように、必要な範囲へ絞るほうが安全です。

### WireGuardのAllowedIPsは何を入れればいい？

最初は狭くするのがおすすめです。

LuCIだけなら `192.168.1.1/32`、NASだけならNASのIPアドレス、Staff LAN全体なら `192.168.1.0/24` です。

`0.0.0.0/0` は全通信をVPNへ流すフルトンネルなので、最初から使う必要はありません。

### Tailscaleのサブネットルーターが効かない時は？

Tailscale管理コンソールでサブネットルートを承認しているか確認します。

ルーターがTailnetに参加していても、ルート承認が済んでいないとLAN内機器へ届きません。

### Exit Nodeは有効にしたほうがいい？

最初は不要です。

LAN内のNASやLuCIへ入りたいだけなら、サブネットルーターで十分なことが多いです。

外出先端末のインターネット通信をLN6001-JP経由にしたい場合に、Exit Nodeを検討します。

### VPNのQRコードをスクリーンショットで保存していい？

おすすめしません。

QRコードや設定ファイルには秘密鍵が含まれることがあります。

安全な場所に保管し、チャット、SNS、公開リポジトリには出さないでください。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- OpenWrt IPoE等インターネットサービスの設定方法: https://support.linksys.com/kb/article/6902-jp/
- Tailscale Subnet routers: https://tailscale.com/docs/features/subnet-routers
- Tailscale Configure a subnet router: https://tailscale.com/docs/features/subnet-routers/how-to/setup
- Tailscale Exit nodes: https://tailscale.com/docs/features/exit-nodes
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
