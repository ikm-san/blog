<!-- mirror-source: articles/018-tailscale-zerotier-use.md -->

# 固定IPなしでも家に帰れる｜TailscaleでNAS・店舗へ安全に入るVPN設計【OpenWrt集中連載018】

外出先から自宅や小さなオフィスへ入りたい。

NASを見たい。  
ミニPCへSSHしたい。  
店舗の録画機を確認したい。  
LuCIでルーターの状態を見たい。  
でも、固定IPはない。  
ポート開放もよく分からない。  
IPoE / IPv4 over IPv6環境で、外からWireGuardへ届くのかも自信がない。

こういう時に、かなり使いやすいのが **Tailscale** です。

Tailscaleは、ざっくり言うと、

```txt
固定IPなし
ポート開放なし
でも、自分の端末同士を安全につなげる
```

ためのVPNサービスです。

中身としてはWireGuardをベースにしつつ、NAT越え、端末管理、認証、サブネットルーター、Exit Nodeなどを扱いやすくしてくれます。

LN6001-JPでは、Linksys公式のVPN Assistantモジュールを使って、WireGuardとTailscaleを扱えます。

ただし、Tailscaleで大事なのは「つながった！」で終わらせないことです。

つながるのは便利です。  
でも、便利すぎて広げすぎることがあります。

LAN全体。  
カメラ用ネットワーク。  
店舗ネットワーク。  
NAS。  
LuCI。  
SSH。  
録画機。  
POS周辺。

全部を一気にTailscaleから見えるようにすると、あとで管理が大変になります。

Tailscaleは、安全な“どこでもドア”みたいなものです。

でも、ドアの先を家じゅう全部にする必要はありません。

まずはNASだけ。  
まずはLuCIだけ。  
まずは管理用ミニPCだけ。

小さく始めるのが安全です。

この記事では、Tailscaleを「なんとなく便利なVPN」としてではなく、**固定IPなし・ポート開放なしで、どこまで安全に入れるようにするかを設計する道具**として整理します。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、Tailscaleを完璧に設計することではありません。

まずは、ここまで分かればOKです。

- Tailscaleが向いている回線・用途が分かる
- WireGuardとTailscaleの使い分けが分かる
- LN6001-JPでVPN Assistantを使う流れが分かる
- サブネットルーターとExit Nodeの違いが分かる
- 最初からLAN全体を広告しなくてよい理由が分かる
- Tailscale管理コンソールで見るべき項目が分かる
- ACL / grants、タグ、端末削除、鍵期限の考え方が分かる
- LuCIとCLIでTailscale状態を確認できる

Tailscaleは、導入そのものはかなり簡単です。

でも、運用で大事なのはここです。

```txt
誰が
どの端末から
どこへ
どの範囲で
入れるのか
```

ここを決めておくと、かなり安全に使いやすくなります。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-01.png)

## 先にざっくり結論

Tailscaleは、次のような人に向いています。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-01.png)

一方で、Tailscaleは外部サービスに依存します。

次のような人は、WireGuardも検討します。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-02.png)

迷ったら、まずTailscaleから始めるのが現実的です。

特に家庭や小さなオフィスでは、最初からWireGuardのポート開放で悩むより、Tailscaleで「外から安全に入れる状態」を作るほうが切り分けしやすいです。

ただし、最初からLAN全体を広告しなくて大丈夫です。

おすすめはこれです。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-03.png)

Tailscaleは、広げようと思えばかなり広げられます。

だからこそ、最初は小さく始めます。

## この記事で扱わないこと

この記事では、Tailscaleの基本設計と、LN6001-JPでの使いどころを扱います。

WireGuard / Tailscaleの導入比較やVPN Assistant全体の設定は、013の記事でも扱っています。

また、ZeroTierも便利なメッシュVPNの選択肢です。

ただし、LN6001-JP向けにLinksys公式モジュールとして案内されている主軸はWireGuard / Tailscaleです。

そのため、この018ではZeroTierを深掘りせず、比較対象として軽く触れる程度にします。

この連載では、LN6001-JPで実際に始めやすいTailscaleを中心に進めます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-02.png)

## こういう人向けです

この記事は、次のような人向けです。

- 固定IPなしで外出先から自宅NASへ入りたい
- IPoE環境でWireGuardのポート開放が難しい
- 店舗や小さなオフィスへ一時的に管理アクセスしたい
- 家族やスタッフの端末をVPNへ参加させたい
- ルーター管理画面を外部公開せずに使いたい
- TailscaleでLAN全体を広告してよいか迷っている
- Exit Nodeを有効にすべきか分からない
- Tailscaleの端末管理やアクセス制御をどう考えるか知りたい

逆に、次のような人にはWireGuardや別構成も検討対象になります。

- 外部サービス依存を極力減らしたい
- 固定IPがあり、自前VPN入口をきちんと管理できる
- 大規模な業務ネットワークで集中管理や監査が必要
- 不特定多数へ公開サービスを提供したい

Tailscaleは、閉じたメンバーだけが使うリモートアクセスに向いています。

誰でもアクセスできる公開サーバー用途とは少し違います。

## 最初に言葉だけそろえる

Tailscaleまわりで出てくる言葉を、ざっくり整理します。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-04.png)

最初は、次だけで大丈夫です。

```txt
tailnet = 自分専用のVPN空間
subnet router = LAN内機器へ入るための中継役
Exit Node = インターネット通信の出口
ACL / grants = 誰がどこへ入れるかのルール
```

## Tailscaleが便利な理由

Tailscaleの大きな価値は、固定IPやポート開放を前提にしなくても始めやすいことです。

従来の自前VPNでは、外から自宅やオフィスへ入るために、次のような条件を整える必要があります。

- グローバルIPv4アドレス
- 固定IPまたはDDNS
- ポート開放
- 上位ホームゲートウェイの転送設定
- firewall設定
- VPNサーバーの公開ポート
- 鍵やクライアント設定の管理

WireGuardはシンプルで高速です。

ただし、自宅側へUDPポートを届ける必要があります。

日本のIPoE / IPv4 over IPv6環境では、MAP-E、DS-Lite、IPIP、HGW配下、二重ルーターなどが絡みます。

その結果、

```txt
WireGuard自体は分かる
でも外からルーターへ届かない
```

というところで詰まることがあります。

Tailscaleは、この入口の複雑さをかなり減らしてくれます。

自宅側ルーターも、外出先のスマートフォンやPCもTailscaleへ参加させ、Tailscale上のプライベートIPで通信する形になります。

つまり、ポート開放の前に試しやすいVPNです。

## Tailscaleが向いている用途

## 自宅NASへ外から入る

NASをインターネットへ直接公開するのは、あまりおすすめしません。

NASの管理画面やファイル共有をポート開放すると、外部から見つけられる入口になります。

Tailscaleを使えば、NAS自体にTailscaleを入れるか、LN6001-JPをサブネットルーターにしてNASへ到達できます。

構成例です。

```txt
外出先スマートフォン
  ↓ Tailscale
LN6001-JP
  ↓ LAN
NAS 192.168.1.10
```

この場合、NASの管理画面をインターネットへ直接出さずに済みます。

自分だけが使うNASなら、まずこの形がかなり現実的です。

## 小さなオフィスの管理端末へ入る

小さなオフィスでは、外出先から管理用PC、NAS、プリンター、ミニPCへ入りたいことがあります。

Tailscaleなら、まず管理者のPCとLN6001-JPだけをtailnetへ参加させ、必要になったらサブネットルートを広告できます。

ここで大事なのは、最初からStaff LAN全体を広く開かないことです。

目的がNASだけならNASだけ。  
管理PCだけなら管理PCだけ。  
ルーター管理だけならLN6001-JPだけ。

このように範囲を絞るほうが安全です。

## 店舗の状態を確認する

店舗では、録画機、カメラ、予約端末、POS周辺機器などがあります。

Tailscaleで店舗へ入れると、遠隔確認がかなり楽になります。

ただし、POSや決済端末はベンダー要件があるため、Tailscaleで雑に入れる対象にしないほうが安全です。

店舗でTailscaleを使うなら、まずは次のような用途が向いています。

- ルーターの状態確認
- 監視用ミニPCへのSSH
- 録画機の管理画面
- 店舗内NAS
- 一時的な保守アクセス

POSや決済端末へアクセスする場合は、必ずベンダー要件を確認します。

「つながるから触る」は危ないです。

店舗では、つながることより、サポート条件を守ることのほうが大事な場合があります。

## ルーター管理画面を外部公開しない

LuCIやSSHをインターネットへ直接公開するのは避けます。

外出先から管理したい場合は、Tailscale経由にします。

```txt
外出先PC
  ↓ Tailscale
LN6001-JPのTailscale IP
  ↓
LuCI / SSH
```

この場合も、Tailscaleに参加できる端末を管理することが前提です。

TailscaleでLuCIへ入れるということは、tailnetへ参加できる端末がルーター管理画面へ近づけるということです。

便利ですが、端末管理はかなり大事です。

## Tailscaleが向かない、または慎重にしたい用途

Tailscaleは便利ですが、万能ではありません。

## 外部サービス依存を避けたい

TailscaleはTailscaleアカウントと管理コンソールに依存します。

外部サービスへ依存したくない、完全に自前でVPN入口を持ちたい、という場合はWireGuardを検討します。

## 不特定多数へサービス公開したい

Tailscaleは、tailnetに参加した端末同士をつなぐ用途に向いています。

不特定多数にWebサービスやゲームサーバーを公開したい場合は、通常の公開設計、ポート開放、クラウド、リバースプロキシなどを検討します。

Tailscaleは、

```txt
閉じたメンバーだけが使う入口
```

と考えると分かりやすいです。

## 大人数の業務運用

Tailscaleはチーム運用もできます。

ただし、ユーザー管理、アクセス制御、タグ、端末管理、退職者対応、ログ、権限設計が必要になります。

家庭や小さなオフィスでは簡単に始められます。

業務で使う場合は、

```txt
誰が管理者か
誰がどの端末へ入れるか
退職者の端末をどう消すか
```

まで最初に決めます。

VPNは入口です。

入口を増やすなら、鍵の管理もセットです。

## WireGuard / Tailscale / ZeroTierの考え方

似た用途の名前として、WireGuard、Tailscale、ZeroTierがあります。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-05.png)

LN6001-JPでは、Linksys公式VPN AssistantでWireGuardとTailscaleが案内されています。

そのため、この連載では、実機で始めやすいTailscaleとWireGuardを中心に扱います。

ZeroTierを使う場合は、別途パッケージ、互換性、管理方法、firewall、アップデートの扱いを確認する必要があります。

## 方式の選び方

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-06.png)

最初はTailscale。

必要になったらWireGuard。

ZeroTierは、明確な理由がある場合に検討。

この順番が、LN6001-JPでは扱いやすいと思います。

## Tailscaleで最初に決めること

Tailscaleを有効化する前に、次を決めます。

## 何にアクセスしたいか

![表画像 table-07](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-07.png)

最初から `192.168.1.0/24` を広告しなくても構いません。

NASだけ必要なら、NASだけ届く設計から始めるほうが安全です。

## 誰がアクセスするか

Tailscaleでは、tailnetに参加できるユーザーや端末を管理します。

家庭なら、自分のスマートフォンとノートPCだけで十分なこともあります。

小さなオフィスなら、管理者、スタッフ、保守担当で権限を分ける必要があります。

まず次を決めます。

- 管理者は誰か
- どのメールアカウントでTailscaleを管理するか
- 家族やスタッフを招待するか
- 退職者や不要端末をどう削除するか
- 2FAを有効にするか
- ACL / grantsを使うか

家庭なら簡単に始められます。

業務利用なら、アカウント管理もVPN設計の一部です。

## Exit Nodeを使うか

Exit Nodeは、外出先端末のインターネット通信を、LN6001-JPや別のTailscale端末経由で出す機能です。

たとえば、外出先のフリーWi-Fiから、自宅回線経由でインターネットへ出るような使い方です。

ただし、最初から有効にする必要はありません。

最初は次を優先します。

```txt
外出先端末 → LN6001-JP → LAN内NAS
```

Exit Nodeは次の段階です。

```txt
外出先端末 → LN6001-JP → インターネット
```

Exit Nodeを有効にすると、外出先端末の全通信がLN6001-JP経由になる場合があります。

通信量、回線速度、DNS、プライバシー、業務ポリシーも関係します。

必要になってから設定するのがおすすめです。

## 設定前にバックアップを取る

VPN Assistantを入れる前に、現在の状態を保存します。

LuCIでは次の手順です。

1. **System** → **Backup / Flash Firmware** を開く
2. **Generate archive** をクリックする
3. `.tar.gz` ファイルをPCへ保存する
4. ファイル名に日付と状態を入れる

例:

```txt
backup-LN6001-before-tailscale-20260621.tar.gz
```

SSHで状態メモも残すなら、次を実行します。

```sh
BACKUP_DIR="/root/tailscale-before-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network firewall dhcp; do
  cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
  uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt"
done

ubus call system board > "$BACKUP_DIR/system-board.json"
ifstatus wan > "$BACKUP_DIR/ifstatus-wan.json" 2>/dev/null || true
ifstatus wan6 > "$BACKUP_DIR/ifstatus-wan6.json" 2>/dev/null || true
ip addr show > "$BACKUP_DIR/ip-addr.txt"
ip route show > "$BACKUP_DIR/route-v4.txt"
ip -6 route show > "$BACKUP_DIR/route-v6.txt"
opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

Tailscale設定には、端末名、認証状態、サブネットルート、Exit Node設定、管理コンソール情報が関係します。

ログや設定を共有する場合は、個人名、拠点名、端末名、IPアドレスなどを必要に応じて伏せてください。

## VPN Assistantをインストールする

LN6001-JPでは、Linksys公式サポートでVPN Assistantモジュールが案内されています。

Tailscaleを使う場合も、まずVPN Assistantを入れます。

一般的なOpenWrt向けに `opkg install tailscale` とするのではなく、LN6001-JP向けには公式VPN Assistantを使う方針にします。

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

インストール後、CLIでも状態を確認します。

```sh
echo "### VPN-related packages"
opkg list-installed | grep -Ei 'vpn|tailscale|wireguard' || true

echo "### init scripts"
ls /etc/init.d | grep -Ei 'tailscale|wireguard|vpn' || true

echo "### logs"
logread | grep -Ei 'vpn|tailscale|wireguard|opkg' | tail -n 100
```

メニューが出ない場合は、ブラウザ更新、LuCI再ログイン、ルーター再起動を試します。

## Tailscaleを有効化する

VPN Assistantが入ったら、Tailscaleを有効化します。

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブを開く
3. **Enable Tailscale Service** を有効にする
4. 必要に応じて **Advertise as Exit Node** を選ぶ
5. 認証用URLが表示されたらブラウザで開く
6. Tailscaleアカウントでログインし、LN6001-JPを登録する
7. LuCI側へ戻り、完了操作を行う
8. 必要に応じてルーターを再起動する
9. 再起動後、Tailscale管理コンソールでLN6001-JPが見えることを確認する

最初はExit Nodeを有効にしなくて大丈夫です。

まずは、LN6001-JPがtailnetに参加し、自分のスマートフォンやPCから見えることを確認します。

ここで一気にサブネットルートやExit Nodeまで広げないほうが、切り分けしやすいです。

## 最初の接続確認

SSHでLN6001-JPへ入り、状態を確認します。

```sh
echo "### service"
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale status
fi

echo "### tailscale status"
tailscale status 2>/dev/null || echo "tailscale command is not available or not authenticated"

echo "### tailscale IP"
tailscale ip 2>/dev/null || true

echo "### tailscale logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 100
```

見るポイントです。

- `tailscale status` が通るか
- LN6001-JP自身がtailnet上に見えているか
- Tailscale IPが表示されるか
- 他のTailscale端末が見えているか
- ログに認証エラーが出ていないか

外出先端末側では、TailscaleアプリからLN6001-JPが見えることを確認します。

最初は、スマートフォン1台だけで確認すると切り分けしやすくなります。

## LN6001-JP自身へアクセスする

Tailscaleに参加しただけなら、まずLN6001-JP自身のTailscale IPへアクセスできます。

Tailscale IPを確認します。

```sh
tailscale ip -4
```

表示されたTailscale IPv4アドレスへ、Tailscale参加済みのPCやスマートフォンからアクセスします。

```txt
https://<LN6001-JPのTailscale IP>
```

LuCIが開けるか確認します。

ただし、Tailscale経由でLuCIへ入れるということは、tailnetに参加できる端末がルーター管理画面へ近づけるということです。

tailnetへ参加させる端末は、信頼できる端末に限定します。

## サブネットルーターとして使う

Tailscaleが入っていないLAN内機器へアクセスしたい場合、LN6001-JPをサブネットルーターとして使います。

たとえば、次のような用途です。

```txt
外出先ノートPC
  ↓ Tailscale
LN6001-JP
  ↓ LAN
NAS 192.168.1.10
```

この場合、LN6001-JPが `192.168.1.0/24` への経路をTailscaleへ広告します。

ただし、広告しただけでは使えません。

Tailscale管理コンソールで、広告されたルートを承認する必要があります。

ここがかなりハマりやすいです。

```txt
tailscale statusでは見える
でもNASへ届かない
```

この場合、サブネットルート未承認のことがあります。

## サブネットルートの範囲を絞る

最初からLAN全体を広告する必要はありません。

NASだけ必要なら、こうです。

```txt
192.168.1.10/32
```

Staff LAN全体が必要なら、こうです。

```txt
192.168.1.0/24
```

カメラ用ネットワークも必要なら、追加で広告します。

```txt
192.168.3.0/24
```

ただし、カメラやPOS、Deviceネットワークまで広く広告する必要があるかは慎重に考えます。

特に店舗では、POSや決済端末をTailscale経由で触る設計は、ベンダー要件を確認してからにします。

サブネットルートは、便利なので広げたくなります。

でも、まずは狭く始めます。

## VPN Assistantでサブネットルートを設定する

VPN Assistant上でサブネットルート広告の項目がある場合は、LuCIから設定します。

画面名や項目は、VPN Assistantやファームウェアのバージョンで変わる可能性があります。

基本的な流れは次の通りです。

1. **Services** → **VPN Assistant** を開く
2. **Tailscale** タブを開く
3. サブネットルート広告に関する項目を確認する
4. 例: `192.168.1.0/24` または `192.168.1.10/32` を指定する
5. **Save & Apply** をクリックする
6. Tailscale管理コンソールで該当ルートを承認する
7. Tailscale参加済み端末からLAN内機器へアクセス確認する

CLIで状態を確認します。

```sh
echo "### Tailscale prefs"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route' || true

echo "### Tailscale status"
tailscale status 2>/dev/null || true

echo "### routes"
ip route show
ip -6 route show
```

CLIで直接設定する場合は、VPN Assistantの設定と衝突しないように注意します。

例:

```sh
tailscale set --advertise-routes=192.168.1.0/24
```

設定後、Tailscale管理コンソール側でルートを承認します。

## サブネットルーターでSNATをどう考えるか

Tailscaleのサブネットルーターでは、デフォルトでSNATが使われることがあります。

この場合、LAN内機器から見ると、外出先端末のIPではなく、サブネットルーターであるLN6001-JPから通信が来たように見えます。

家庭や小さなオフィスでは、まずデフォルトのままで十分なことが多いです。

ただし、次のような場合は注意します。

- LAN内機器側で接続元IPを厳密に見たい
- 監査ログで外出先端末ごとのIPを識別したい
- firewallルールを接続元ごとに分けたい

この場合は、TailscaleのSNAT設定やLAN側ルーティングを理解してから設計します。

最初からSNAT無効化を目指すより、まずサブネットルーターとして安定して使えることを優先します。

## Exit Nodeはいつ使うか

Exit Nodeは、外出先端末のインターネット通信を、指定したTailscale端末経由にする機能です。

たとえば、外出先のフリーWi-Fiから自宅回線経由でインターネットへ出るような使い方です。

```txt
外出先スマートフォン
  ↓ Tailscale
LN6001-JP
  ↓ 自宅回線
インターネット
```

これは便利です。

でも、最初から使わなくて大丈夫です。

## Exit Nodeが向く場面

- 外出先のWi-Fiを使う時に通信を自宅経由へ寄せたい
- 海外から日本の自宅回線経由で一部サービスへアクセスしたい
- すべての通信を管理された出口へ通したい
- 検証で特定拠点からのアクセスに見せたい

## Exit Nodeで注意すること

- 外出先端末の全通信がLN6001-JP経由になる場合がある
- 自宅や店舗回線の上り帯域を使う
- ルーター側の負荷が増える
- DNSやAdblockの見え方が変わる
- Tailscale管理コンソール側でExit Node承認が必要
- 各クライアント端末側でもExit Node利用を選ぶ必要がある
- ローカルLANアクセスの扱いが変わることがある

Tailscaleでは、Exit Nodeを使う側の端末で、Exit Nodeを明示的に選択します。

また、Exit Nodeを広告する側も、管理コンソール側で許可が必要です。

最初はExit Nodeなしで、LAN内機器へアクセスできるところまでを確認するのがおすすめです。

## ACL / grantsでアクセス範囲を絞る

Tailscaleは、tailnet内のアクセスを管理コンソール側のアクセス制御で絞れます。

以前からよく使われている表現としてACLがあります。  
現在のTailscaleドキュメントでは、新規のアクセス制御には **grants** の利用も案内されています。

細かい構文はTailscale公式ドキュメントを見ながら設定してください。

この記事では、まず考え方だけ押さえます。

![表画像 table-08](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-08.png)

家庭の個人利用なら、最初は初期設定でも十分なことがあります。

小さなオフィスや店舗で使う場合は、少なくとも次を決めます。

```txt
誰がTailscaleに入れるのか
誰がLANへ入れるのか
誰が録画機やNASへ入れるのか
退職者や不要端末をどう消すのか
```

これは技術設定というより、運用ルールです。

でもVPNではかなり大事です。

## ACLの考え方例

以下は考え方の例です。

実際の構文や推奨方式は、Tailscale公式ドキュメントの最新内容を確認してください。

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["group:admins"],
      "dst": ["tag:router:*", "192.168.1.10:22,443"]
    },
    {
      "action": "accept",
      "src": ["group:family"],
      "dst": ["192.168.1.10:443"]
    }
  ],
  "tagOwners": {
    "tag:router": ["group:admins"]
  }
}
```

最近の構成では、grantsを使う設計も検討します。

ただし、家庭や小規模運用では、最初から複雑なポリシーを書かなくても大丈夫です。

まずは端末数を絞る。  
サブネットルートを絞る。  
不要端末を消す。

ここからで十分です。

## タグを使う

Tailscaleでは、端末へタグを付けて管理できます。

LN6001-JPのようなルーターやサブネットルーターには、ユーザー所有の個人端末としてではなく、役割付きの端末としてタグを付けると管理しやすくなります。

例:

```txt
tag:router
tag:office-router
tag:shop-router
```

タグを使うと、アクセス制御や自動承認の設計がしやすくなります。

ただし、タグの所有者を誰にするか、誰がタグを付けられるかも管理が必要です。

家庭利用では必須ではありません。

小さなオフィスや店舗で複数人が使う場合は、タグ管理を検討します。

## 端末管理で見るべきこと

Tailscaleは、使い始めるのは簡単です。

ただし、使わなくなった端末を放置すると、tailnetが散らかります。

Tailscale管理コンソールで、定期的に次を確認します。

- 使っていない端末が残っていないか
- 紛失したスマートフォンが残っていないか
- 退職者や不要ユーザーの端末が残っていないか
- LN6001-JPの端末名が分かりやすいか
- サブネットルートを広告している端末はどれか
- Exit Nodeを広告している端末はどれか
- 鍵期限が切れそうな端末はないか
- ACL / grantsが想定どおりか

端末名は分かりやすくしておきます。

例:

```txt
ln6001-home
ln6001-office
ln6001-shop
```

店舗や住所が特定できる名前を付けるかどうかは、運用方針に合わせて判断します。

## 鍵期限と再認証

Tailscale端末には鍵期限があります。

鍵が期限切れになると、端末が再認証を求められることがあります。

サブネットルーターとして使っているLN6001-JPが再認証待ちになると、外出先からLANへ入れなくなる可能性があります。

家庭では気づいた時に対応でもよい場合があります。

小さなオフィスや店舗で使うなら、次を決めておきます。

- 誰がTailscale管理コンソールを見るか
- LN6001-JPの鍵期限をどう扱うか
- 再認証が必要になった時、現地で対応できる人はいるか
- 重要な拠点ではタグや認証運用をどうするか

タグ付きデバイスでは鍵期限の扱いが通常端末と変わる場合があります。

ここはTailscale公式ドキュメントの最新内容を確認してください。

サブネットルーターやExit Nodeは、止まると影響が大きいです。

鍵期限も運用項目に入れておくと安心です。

## Tailscale状態確認コマンド

SSHでLN6001-JPへ入り、Tailscale状態を確認します。

```sh
echo "### tailscale version"
tailscale version 2>/dev/null || true

echo "### tailscale status"
tailscale status 2>/dev/null || true

echo "### tailscale IP"
tailscale ip 2>/dev/null || true

echo "### tailscale prefs"
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|ExitNode|Route|Shields' || true

echo "### tailscale routes"
ip route show
ip -6 route show

echo "### tailscale logs"
logread | grep -Ei 'tailscale|tailscaled' | tail -n 120
```

見るポイントです。

- `tailscale status` が成功するか
- LN6001-JP自身がtailnetに参加しているか
- Tailscale IPがあるか
- サブネットルートを広告しているか
- Exit Nodeが有効になっていないか
- ログに認証エラーがないか

## LAN内機器へ届くか確認する

Tailscale参加済みのPCやスマートフォンから、目的のLAN内機器へアクセスします。

例:

```txt
https://192.168.1.1
https://192.168.1.10
ssh user@192.168.1.20
```

届かない場合は、次を確認します。

- LN6001-JPがtailnetに参加しているか
- サブネットルートを広告しているか
- 管理コンソールでルート承認済みか
- クライアント側でルートを使っているか
- LAN内機器のIPが変わっていないか
- LAN内機器側firewallが拒否していないか
- LN6001-JP側firewallでVPNからLANへの通信を止めていないか

まずはIPアドレスで確認します。

DNS名でつながらない場合は、TailscaleのMagicDNSやローカルDNSの問題かもしれません。

名前解決は後回しで大丈夫です。

まずIPで届くか見ます。

## OpenWrt側firewallとの関係

Tailscaleはtailnet側のACL / grantsでも制御できますが、LN6001-JP側のfirewallも関係します。

最初はVPN Assistantの設定を優先し、必要になったらOpenWrt側firewallでもTailscale用zoneを作ることを検討します。

ただし、いきなり複雑にしないほうが安全です。

## Tailscaleインターフェースを確認する

```sh
ip addr show | grep -A5 -Ei 'tailscale|tailscale0'
```

`tailscale0` が見えている場合、firewallで扱うためのinterfaceとして設定することがあります。

## firewallで扱う場合の考え方

たとえば、Tailscaleを `vpn` zoneとして扱い、VPNからLANへ必要な通信だけ許可する設計です。

ただし、VPN Assistantの設定と衝突しないように、まず読み取り確認から始めます。

```sh
echo "### network tailscale hints"
uci show network | grep -Ei 'tailscale|vpn' || true

echo "### firewall vpn hints"
uci show firewall | grep -Ei 'tailscale|vpn|wg|wireguard' || true
```

必要になった場合のみ、次のようにzoneを作ります。

```sh
BACKUP_DIR="/root/tailscale-firewall-before-$(date +%Y%m%d-%H%M)"
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

この時点では、vpnからlanへforwardingを作っていません。

必要な宛先だけTraffic Ruleで許可します。

例: VPNからNAS `192.168.1.10` のHTTPSだけ許可する。

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

家庭利用なら、ここまで細かくしなくてもよい場合があります。

小さなオフィスや店舗では、Tailscale側のアクセス制御とOpenWrt firewallの両方で「必要な範囲だけ」を意識します。

## つながらない時の切り分け

## Tailscaleに参加できない

確認します。

```sh
ifstatus wan
ifstatus wan6
date
tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled|auth|login' | tail -n 120
```

見るポイントです。

- LN6001-JPがインターネットへ出られるか
- 時刻が大きくずれていないか
- 認証URLを最後まで完了したか
- Tailscaleアカウントでログインできているか
- ルーター再起動後もサービスが起動しているか

## LN6001-JPは見えるがLAN内NASへ届かない

確認します。

```sh
tailscale status 2>/dev/null || true
tailscale debug prefs 2>/dev/null | grep -Ei 'AdvertiseRoutes|Route' || true
ip route show
uci show firewall | grep -Ei 'tailscale|vpn|lan|forward|NAS|192.168.1.10' || true
```

見るポイントです。

- サブネットルートを広告しているか
- Tailscale管理コンソールで承認しているか
- クライアント側がサブネットルートを使っているか
- NASのIPアドレスが変わっていないか
- NAS側firewallが拒否していないか
- OpenWrt側firewallで止めていないか

## Exit Nodeが使えない

確認します。

```sh
tailscale debug prefs 2>/dev/null | grep -Ei 'ExitNode|AdvertiseRoutes|Route' || true
tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled|exit' | tail -n 100
```

見るポイントです。

- LN6001-JPがExit Nodeとして広告しているか
- 管理コンソールでExit Nodeを承認しているか
- クライアント側でExit Nodeを選択しているか
- クライアント側でローカルLANアクセスが必要か
- 自宅や店舗回線の上り帯域が足りているか

最初はExit Nodeを使わず、サブネットルーターだけで確認したほうが切り分けしやすいです。

## DNS名でつながらない

IPアドレスでは届くが、名前では届かない場合です。

確認します。

```sh
nslookup nas-home 192.168.1.1
nslookup nas-home.lan 192.168.1.1
tailscale status 2>/dev/null || true
```

可能性です。

- MagicDNSの設定
- ローカルDNSの向き先
- DHCP予約のホスト名
- Tailscale側DNS設定
- AdblockやFamily DNSとの衝突

まずはIPアドレスで通信できるかを確認します。

名前解決はその後です。

## Exit Nodeを使ったらローカルLANへ届かない

Exit Node使用時は、クライアント側でローカルLANアクセスの扱いが変わることがあります。

Tailscaleクライアント側で、Exit Node利用時にローカルネットワークアクセスを許可する設定を確認します。

また、Exit Nodeを使うと通信経路が変わるため、DNSやルーティングの見え方も変わります。

最初はExit Nodeを無効にし、サブネットルートだけで目的の機器へ届くかを確認してください。

## Tailscaleを無効化する

使わなくなったら、Tailscaleを止めます。

LuCIでは、VPN AssistantのTailscaleタブから無効化します。

CLIでは次のようにします。

```sh
echo "### logout from tailnet"
tailscale logout 2>/dev/null || true

echo "### stop service"
if [ -x /etc/init.d/tailscale ]; then
  /etc/init.d/tailscale stop
  /etc/init.d/tailscale disable
fi

echo "### verify"
tailscale status 2>/dev/null || true
logread | grep -Ei 'tailscale|tailscaled' | tail -n 80
```

さらに、Tailscale管理コンソールでLN6001-JPの端末を削除または無効化します。

ルーター側で止めるだけではなく、管理コンソール側の端末整理も忘れないようにします。

## Tailscale運用メモ

設定後は、次のようなメモを残しておきます。

```txt
Tailscale運用メモ:

ルーター:
  ln6001-home

用途:
  外出先からLuCIとNASへアクセス

Tailscale:
  enabled: yes
  exit node: no
  subnet routes:
    192.168.1.10/32
  approved in admin console: yes

アクセス許可:
  自分のスマートフォン
  自分のノートPC

アクセス先:
  LN6001-JP LuCI
  NAS 192.168.1.10

Access controls:
  default / ACL / grants
  admins only / family allowed

確認:
  tailscale status OK
  スマートフォンからNASへアクセスOK
  不要なネットワークは広告していない

無効化手順:
  VPN AssistantでTailscaleを無効化
  管理コンソールから端末削除

バックアップ:
  backup-LN6001-before-tailscale-20260621.tar.gz
```

VPN設定は、あとから何のために入れたか忘れやすいです。

特にTailscaleは簡単に端末を追加できるので、誰がどこへ入れるかをメモしておくと安全です。

## 定期点検すること

Tailscaleを入れたあと、月1回くらいで次を確認すると運用が安定します。

![表画像 table-09](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/table-09.png)

Tailscaleは「入れるのが簡単」なぶん、使わなくなった端末の整理が大事です。

端末が増えると、便利さと同時に入口も増えます。

## まとめ

Tailscaleは、固定IPやポート開放なしで、自宅・小さなオフィス・店舗へ安全に入るための有力な選択肢です。

特に、IPoE / IPv4 over IPv6環境でWireGuardのポート開放が難しい場合、Tailscaleはかなり始めやすいです。

ただし、Tailscaleは「入れれば安全」というものではありません。

大事なのは次の点です。

1. 最初はスマートフォン1台とLN6001-JPだけで確認する
2. Exit Nodeは最初から有効にしない
3. サブネットルートは必要最小限にする
4. LAN全体を広告する前に、NASだけなど狭く始める
5. Tailscale管理コンソールでルート承認と端末管理を行う
6. 不要端末や紛失端末を削除する
7. 業務利用ではACL / grantsとタグを考える
8. POSや決済端末はベンダー要件を優先する

Tailscaleは、ポート開放の代わりとしてかなり便利です。

でも、アクセス範囲を広げすぎると、結局「外からLAN全体へ入れる入口」になります。

最初は小さく始めて、必要になったところだけ広げる。

これが、家庭・小さなオフィス・店舗でTailscaleを安全に使うための現実的な進め方です。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/018/diagram-03.png)

## 次に読むなら

Tailscaleの使いどころが見えてきたら、次の記事も合わせて読むと整理しやすいです。

- [WireGuard/Tailscaleでリモート接続](https://note.com/ikmsan/n/n1a83908a226c)
- [ポート開放の注意点](https://note.com/ikmsan/n/n765ffea6ea82)
- [firewall zonesの考え方](https://note.com/ikmsan/n/nfe0609ff7bd4)
- [固定IPとDHCP予約](https://note.com/ikmsan/n/ndd5ae853f033)
- [VPN向け構成例](https://note.com/ikmsan/n/nb0b4b7e63309)

WireGuardとの比較やVPN Assistant全体の設定を見たい人は、013の記事へ。

ポート開放とどちらを選ぶか迷っている人は、017の記事へ。

VPNからLAN内のどこまで入れるかを整理したい人は、firewall zonesの記事へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

VPN Assistant、Tailscale、WireGuard関連の画面名やコマンド出力は、モジュールやファームウェアの更新で変わることがあります。

Tailscaleの管理コンソール、ACL / grants、タグ、サブネットルート、Exit Node、鍵期限の仕様も変更される可能性があります。

記事の内容と実際の画面や挙動が違う場合は、まずバックアップを取り、Linksys公式サポートとTailscale公式ドキュメントの最新情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### Tailscaleは固定IPがなくても使える？

使えます。

固定IPやポート開放を前提にしなくても始めやすいのがTailscaleの大きな利点です。

IPoE / IPv4 over IPv6環境でWireGuardの外部着信が難しい場合にも、Tailscaleは有力な選択肢になります。

### TailscaleとWireGuardはどちらがいい？

固定IP、DDNS、ポート開放を管理できるならWireGuardもよい選択肢です。

固定IPがない、ポート開放が不安、複数端末を楽に管理したい場合はTailscaleが始めやすいです。

### TailscaleでLAN全体へ入れるようにしていい？

できますが、最初からLAN全体を広告する必要はありません。

まずはNASだけ、LuCIだけ、管理PCだけのように狭く始めるのがおすすめです。

### サブネットルーターとExit Nodeは何が違う？

サブネットルーターは、Tailscale端末からLAN内の非Tailscale機器へアクセスするための機能です。

Exit Nodeは、Tailscale端末のインターネット通信全体を指定端末経由で出す機能です。

NASへ入りたいだけなら、まずサブネットルーターを考えます。

### Exit Nodeは有効にしたほうがいい？

最初は不要です。

Exit Nodeを使うと外出先端末の通信がLN6001-JP経由になるため、回線速度、上り帯域、DNS、プライバシー、管理ポリシーに影響します。

必要になってから有効化するのがおすすめです。

### Tailscaleを使えばポート開放は不要？

自分や家族、スタッフだけが使うリモートアクセスなら、Tailscaleでポート開放を避けられることが多いです。

ただし、不特定多数へWebサービスやゲームサーバーを公開する用途では、別の公開設計が必要です。

### Tailscaleの端末をなくしたらどうする？

Tailscale管理コンソールから、その端末を削除または無効化します。

ルーター側の設定だけでなく、管理コンソール側の端末整理が重要です。

### ACLとgrantsはどちらを使えばいい？

Tailscaleでは従来のACLも使えますが、新しい設計ではgrantsが案内される場面があります。

実際の設定はTailscale公式ドキュメントの最新内容を確認してください。

この記事では、まず「誰がどこへ入れるかを絞る」という考え方を重視しています。

### ZeroTierは使わないの？

ZeroTierもメッシュVPNの選択肢ですが、LN6001-JP向けの公式VPN AssistantではWireGuardとTailscaleが案内されています。

この連載では、公式手順に沿って始めやすいTailscaleとWireGuardを中心に扱います。

## 参考リンク

- Linksys WireGuard/Tailscale公式モジュール設定手順: https://support.linksys.com/kb/article/8723-jp/
- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Tailscale Subnet routers: https://tailscale.com/docs/features/subnet-routers
- Tailscale Configure a subnet router: https://tailscale.com/docs/features/subnet-routers/how-to/setup
- Tailscale Exit nodes: https://tailscale.com/docs/features/exit-nodes
- Tailscale Use exit nodes: https://tailscale.com/docs/features/exit-nodes/how-to/setup
- Tailscale ACLs: https://tailscale.com/docs/features/access-control/acls
- Tailscale grants examples: https://tailscale.com/docs/reference/examples/grants
- Tailscale policy file syntax: https://tailscale.com/docs/reference/syntax/policy-file
- Tailscale tags: https://tailscale.com/docs/features/tags
- Tailscale key expiry: https://tailscale.com/docs/features/access-control/key-expiry
- Tailscale管理コンソール: https://login.tailscale.com/admin/

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
