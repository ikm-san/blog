<!-- mirror-source: articles/033-ssh-login-windows-mac.md -->

# SSHは怖くない｜WindowsとMacからLN6001-JPへ入る最初の一歩【OpenWrt集中連載033】

SSHって、最初はちょっと怖く見えます。

黒い画面。  
英語のメッセージ。  
謎のfingerprint。  
パスワードを打っても何も表示されない。  
そして、画面には `root@192.168.1.1`。

いかにも「間違えたら壊れそう」な雰囲気があります。

でも、大丈夫です。

最初にやることは、設定変更ではありません。

まずは **読むだけ** です。

WANは取れている？  
IPv6側はどう見えている？  
端末にIPアドレスは配れている？  
DNSだけおかしい？  
ログにエラーは出ている？

このあたりを見るだけなら、SSHはかなり便利です。

LN6001-JPは、WindowsでもMacでもSSHで入れます。

まずは、これだけです。

```sh
ssh root@192.168.1.1
```

ここまでできるだけで、OpenWrtベースルーターの見え方がかなり変わります。

LuCIの画面で見る。  
SSHで状態を読む。  
困った時にログを見る。

この2つを行き来できるようになると、トラブル対応がかなり落ち着きます。

SSHは、いきなりルーターを改造するためのものではありません。

最初は、ルーターの体温を測るための道具です。

> この連載では、Linksys Velop WRT Pro 7（LN6001-JP）を使いながら、家庭・小さなオフィス・店舗でOpenWrtベースルーターを現実的に使う方法をまとめています。  
> 連載目次: [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](https://note.com/ikmsan/n/ndf7569fea475)

## 今日のゴール

この記事のゴールは、SSHを完璧に使いこなすことではありません。

まずは、ここまでできればOKです。

- WindowsまたはMacからLN6001-JPへSSH接続できる
- 初回fingerprint確認で何を聞かれているか分かる
- パスワード入力時に文字が出なくても焦らない
- SSHへ入ったあと、読み取りコマンドをいくつか実行できる
- `Host key verification failed` が出た時に対処できる
- 慣れてきたら鍵認証へ進める
- SSHをWAN側へ直接公開しない理由が分かる
- 診断ログや鍵を共有する時の注意点が分かる

最初からCLIで設定変更しなくて大丈夫です。

SSHは、まず「状態を見るための入口」として使います。

```txt
SSHに入る
  ↓
状態を見る
  ↓
ログを見る
  ↓
必要ならLuCIで設定する
```

最初はこのくらいで十分です。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-01.png)

## 先にざっくり結論

最初の接続コマンドはこれです。

```sh
ssh root@192.168.1.1
```

LAN IPを変更している場合は、変更後のIPを使います。

```sh
ssh root@192.168.10.1
```

SSHへ入れたら、まずこの4つだけ覚えれば十分です。

```sh
ubus call system board
ifstatus wan
cat /tmp/dhcp.leases
logread | tail -n 50
```

この4つで、次が見えます。

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-01.png)

IPoEやIPv6環境では、あとでこれも足します。

```sh
ifstatus wan6
ip -6 route show
```

鍵認証は便利ですが、最初から必須ではありません。

まずはパスワードで入る。  
SSHに慣れる。  
そのあと鍵認証へ進む。

この順番で大丈夫です。

## こういう人向けです

この記事は、次のような人向けです。

- WindowsまたはMacからLN6001-JPへ初めてSSH接続する
- LuCIだけでは見えない状態を確認したい
- つながらない時にログを見たい
- 最初はパスワード認証で入りたい
- 慣れてきたら鍵認証へ移りたい
- `Host key verification failed` の意味を知りたい
- SSHを安全に使う最低限のルールを知りたい
- 025のSSH基本コマンド集へ進む前に、まず接続だけ成功させたい

逆に、すでにSSH鍵管理やOpenWrtのCLI運用に慣れている人には、かなり基本寄りです。

でも、最初の1回でつまずくとSSHが嫌いになりがちなので、ここは丁寧にいきます。

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-02.png)

## 最初に言葉だけそろえる

SSHまわりで出てくる言葉を、ざっくりそろえておきます。

![表画像 table-02](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-02.png)

ざっくり言うと、こうです。

```txt
公開鍵 = ルーターに渡してよい
秘密鍵 = 絶対に外へ出さない
```

ここだけは覚えておくと安全です。

秘密鍵は、家の鍵そのものです。

チャットに貼る。  
SNSに貼る。  
GitHubに上げる。  
記事のスクリーンショットへ出す。

全部だめです。

## SSHを使う時の基本ルール

SSHは便利ですが、最初はこのルールで使うのがおすすめです。

```txt
最初は読み取りだけ
設定変更はLuCI中心
変更前にバックアップ
SSHはLAN側またはVPN経由だけ
WAN側へ直接公開しない
```

ここ、かなり大事です。

SSHは管理者用の入口です。  
便利だからといって、インターネット側へ直接開けるのはおすすめしません。

外出先から使いたい場合は、TailscaleやWireGuardなどのVPN経由にします。

```txt
外出先PC
  ↓ VPN
LN6001-JP
  ↓ SSH / LuCI
```

最初は、同じLAN内から使うだけで十分です。

「SSHできた！」の次にやることは、WAN側へ開けることではありません。

ログを見ることです。

## 接続前に確認すること

SSH接続前に、まずここを見ます。

![表画像 table-03](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-03.png)

LuCIでは、SSH関連の設定が次のような場所にあります。

```txt
System → SSH Access
System → Administration → SSH Access
System → SSH-Keys
```

画面名はファームウェアやLuCIの表示によって少し違うことがあります。

迷ったら、LuCIのSystemメニュー周辺でSSH AccessやSSH-Keysを探します。

## 最初は有線LANがおすすめ

最初のSSH接続は、有線LANのPCから行うと安心です。

```txt
PC
  ↓ 有線LAN
LN6001-JP LANポート
```

Wi-FiでもSSH接続できます。

ただ、SSID変更や暗号化方式変更をしている途中だと、Wi-Fiが一時的に切れることがあります。

SSHの初回確認や鍵登録は、有線LANのほうが切り分けしやすいです。

家庭でも店舗でも、困ったら有線です。

地味ですが、かなり強いです。

## PC側から疎通確認する

SSHする前に、PCからLN6001-JPへ届くか見ます。

### Mac

```sh
ping -c 3 192.168.1.1
nc -vz 192.168.1.1 22
```

`ping` が通れば、IPとしては届いています。

`nc -vz` で22番ポートへ接続できれば、SSHポートが開いている可能性が高いです。

### Windows PowerShell

```powershell
ping 192.168.1.1
Test-NetConnection 192.168.1.1 -Port 22
```

`TcpTestSucceeded : True` が出れば、22番ポートへ届いています。

届かない場合は、ここを確認します。

- PCがLN6001-JP配下のIPを取れているか
- ゲストWi-Fiへつないでいないか
- LAN IPを変更していないか
- SSH Accessが有効か
- firewallでLANからSSHを止めていないか

## `192.168.1.1` へ入れない時はDefault Gatewayを見る

「SSHできない」と思った時、実は接続先IPが違うだけのことがあります。

LAN IPを `192.168.10.1` などへ変更していると、当然 `192.168.1.1` では入れません。

PC側のDefault Gatewayを確認します。

### Windows PowerShell

```powershell
ipconfig
```

`Default Gateway` を見ます。

### Mac

```sh
route -n get default | grep gateway
```

または:

```sh
netstat -rn | grep default
```

例:

```txt
gateway: 192.168.10.1
```

この場合、SSH接続先は次です。

```sh
ssh root@192.168.10.1
```

「壊れた」のではなく、「管理画面の住所が変わっただけ」。

OpenWrt系では、これがけっこうあります。

```txt
192.168.1.1にいない
  ↓
ルーターが壊れた

ではなく、

192.168.10.1へ引っ越しただけ
```

こう考えると落ち着けます。

## MacからSSH接続する

Macには、標準でSSHクライアントが入っています。  
追加ソフトは不要です。

### ターミナルを開く

次のどちらかで開きます。

```txt
Launchpad → ターミナル
Spotlight（Cmd + Space）→ ターミナル
```

### SSH接続コマンド

```sh
ssh root@192.168.1.1
```

LAN IPを変更している場合は、変更後のIPを使います。

```sh
ssh root@192.168.10.1
```

接続できない時は、詳細ログを出します。

```sh
ssh -v root@192.168.1.1
```

`-v` は、SSHのどこで止まっているかを見るためのオプションです。

普段は不要ですが、エラー時には便利です。

## Macで初回接続する時

初回接続時には、次のような確認が出ます。

```txt
The authenticity of host '192.168.1.1 (192.168.1.1)' can't be established.
ED25519 key fingerprint is SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

自分のLAN内で、接続先がLN6001-JPだと分かっている場合は、`yes` と入力します。

```txt
yes
```

その後、rootパスワードを入力します。

パスワード入力中は、画面に文字が表示されません。

何も出なくても入力されています。

ここで「あれ、打ててない？」と不安になりますが、ちゃんと打てています。

入力したらEnterです。

## WindowsからSSH接続する

Windows 10 / 11では、PowerShellやWindows TerminalからSSH接続できます。

### PowerShellを開く

次のどちらかで開きます。

```txt
スタートメニュー → PowerShell と検索
Windows Terminal を開く
```

### SSH接続コマンド

```powershell
ssh root@192.168.1.1
```

LAN IPを変更している場合は、変更後のIPを使います。

```powershell
ssh root@192.168.10.1
```

接続できない時は、詳細ログを出します。

```powershell
ssh -v root@192.168.1.1
```

SSHポート確認は次です。

```powershell
Test-NetConnection 192.168.1.1 -Port 22
```

`TcpTestSucceeded : True` が出れば、少なくともPCから22番ポートへは届いています。

## Windowsで初回接続する時

Windowsでも、初回接続時にfingerprint確認が出ます。

```txt
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

接続先がLN6001-JPであることを確認し、`yes` と入力します。

その後、rootパスワードを入力します。

Macと同じく、パスワード入力中は文字が表示されません。

何も出ないのが正常です。

パスワードを打ってEnter。

この黒い画面の無口さに、最初はちょっとびっくりします。

## PuTTYを使う場合

WindowsではPowerShellだけで十分です。

ただ、普段からPuTTYに慣れている人は、PuTTYでも接続できます。

1. PuTTYを起動する
2. **Host Name** に `192.168.1.1` を入力する
3. **Port** に `22` を入力する
4. **Connection type** は `SSH`
5. **Open** をクリックする
6. 初回のセキュリティ警告でfingerprintを確認し、**Accept** する
7. `login as:` に `root` と入力する
8. rootパスワードを入力する

最初はPowerShellで試し、必要ならPuTTYを使うくらいで大丈夫です。

この記事では、Windows標準のOpenSSHを前提に進めます。

## fingerprint確認は何をしているのか

SSHのfingerprintは、「接続先が前回と同じSSHサーバーか」を確認するためのものです。

初回接続時は、PC側に記録がありません。

そのため、

```txt
この機器を信用して接続していい？
```

と聞かれます。

一度 `yes` で受け入れると、PC側の `known_hosts` に記録されます。

次回からは、同じIPアドレスに同じSSHサーバーがいれば、警告は出ません。

## fingerprint警告が出る代表例

次のような時に、fingerprint警告が出ることがあります。

- LN6001-JPを初期化した
- ファームウェア更新でSSHホスト鍵が変わった
- 別のルーターを同じIPアドレスで使っている
- LAN IPを使い回している
- 本当に別の機器へ接続している

初期化直後なら自然なことがあります。

ただし、身に覚えがない時は、接続先が本当にLN6001-JPか確認してください。

fingerprint警告は、ただの面倒なメッセージではありません。

「前と違う相手かもしれないよ」という、SSHなりの警告です。

## `Host key verification failed` が出た時

ルーター初期化後などに、次のようなエラーが出ることがあります。

```txt
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
Host key verification failed.
```

これは、PC側に残っている古いfingerprintと、今のLN6001-JPのfingerprintが違うという意味です。

初期化や機器交換をした直後なら、古い記録を削除して再接続します。

### Mac

```sh
ssh-keygen -R 192.168.1.1
ssh root@192.168.1.1
```

LAN IPを変えている場合:

```sh
ssh-keygen -R 192.168.10.1
ssh root@192.168.10.1
```

### Windows PowerShell

```powershell
ssh-keygen -R 192.168.1.1
ssh root@192.168.1.1
```

LAN IPを変えている場合:

```powershell
ssh-keygen -R 192.168.10.1
ssh root@192.168.10.1
```

削除後、再接続すると新しいfingerprint確認が出ます。

接続先がLN6001-JPであることを確認して `yes` を入力します。

身に覚えがないfingerprint変更なら、反射で削除しないでください。

まず接続先を確認します。

## SSH接続ショートカットを作る

毎回 `ssh root@192.168.1.1` と打つのが面倒なら、SSH configを作ります。

これを作っておくと、次だけで接続できます。

```sh
ssh router
```

### Mac

```sh
mkdir -p ~/.ssh

cat >> ~/.ssh/config << 'EOF'

Host router
    HostName 192.168.1.1
    User root
    Port 22
EOF

chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
```

LAN IPを変更している場合は、`HostName` を変更後のIPにします。

```txt
HostName 192.168.10.1
```

接続します。

```sh
ssh router
```

### Windows PowerShell

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh"

Add-Content -Path "$env:USERPROFILE\.ssh\config" -Value @"

Host router
    HostName 192.168.1.1
    User root
    Port 22
"@
```

接続します。

```powershell
ssh router
```

Windowsでも、次の場所にSSH configを置けます。

```txt
C:\Users\<ユーザー名>\.ssh\config
```

ルーターのLAN IPを変更したら、このSSH configも忘れずに直します。

```txt
ルーターの住所を変えた
  ↓
SSH configのHostNameも変える
```

ここを忘れると、また `192.168.1.1` に突撃して迷子になります。

## SSHへ入ったらまず見るコマンド

SSHへ入れたら、最初は読み取りだけで十分です。

```sh
echo "### system"
ubus call system board

echo "### WAN"
ifstatus wan

echo "### WAN6"
ifstatus wan6

echo "### DHCP leases"
cat /tmp/dhcp.leases

echo "### DNS test"
nslookup www.google.co.jp

echo "### Wi-Fi status"
wifi status

echo "### recent logs"
logread | tail -n 50
```

見るポイントです。

![表画像 table-04](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-04.png)

ここまでは設定変更ではありません。

「見るだけ」なので、最初のSSH練習にちょうどいいです。

## SSHへ入った時にやらないこと

SSHへ入れると、急に何でもできそうな気分になります。

でも、最初はやらないほうがいいこともあります。

- 意味が分からない `uci set` を貼り付ける
- バックアップなしで `/etc/config/network` を編集する
- `opkg upgrade` を実行する
- WAN側へSSHを公開する
- firewallを一気に複数変更する
- VLANを有線管理手段なしで変更する
- ネット上のスクリプトを `curl | sh` で即実行する
- 秘密鍵や設定ファイルをSNSへ貼る

SSHは、最初は“読むため”に使います。

設定変更は、LuCIでできるならLuCIで大丈夫です。

CLIで変更するのは、バックアップと戻し方が分かってからで十分です。

## 設定変更前に簡易バックアップする

CLIで設定変更する前には、対象設定をバックアップします。

最初は、主要ファイルだけで十分です。

```sh
BACKUP_DIR="/root/ssh-before-change-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"

for cfg in network wireless firewall dhcp dropbear; do
  if [ -f "/etc/config/$cfg" ]; then
    cp "/etc/config/$cfg" "$BACKUP_DIR/$cfg"
    uci show "$cfg" > "$BACKUP_DIR/$cfg.uci.txt" 2>/dev/null || true
  fi
done

opkg list-installed > "$BACKUP_DIR/opkg-list-installed.txt"
logread | tail -n 200 > "$BACKUP_DIR/logread-tail.txt"

ls -l "$BACKUP_DIR"
```

PCへコピーする場合は、PC側から実行します。

### Mac

```sh
scp -r root@192.168.1.1:/root/ssh-before-change-* ~/Downloads/
```

### Windows PowerShell

```powershell
scp -r root@192.168.1.1:/root/ssh-before-change-* "$env:USERPROFILE\Downloads\"
```

大きな変更前は、LuCIでもバックアップを取ります。

```txt
System → Backup / Flash Firmware → Generate archive
```

SSHで一番強い人は、コマンドを全部覚えている人ではありません。

変更前にバックアップを取っている人です。

## 鍵認証へ移る前に

鍵認証は便利です。

でも、いきなりパスワード認証を無効化しないでください。

安全な順番はこれです。

```txt
1. パスワード認証でSSH接続できる
2. 公開鍵をルーターへ登録する
3. 鍵認証で接続できることを確認する
4. 別ターミナルから再接続テストする
5. 問題なければ、必要に応じてパスワード認証を見直す
```

ここで焦ると、SSHへ入れなくなることがあります。

鍵認証は、接続に慣れてからで大丈夫です。

まずはパスワードで入れること。

次に鍵認証。

そのあと、必要ならパスワード認証を見直す。

この順番が安全です。

## SSHキーペアを作る

PC側で秘密鍵と公開鍵のペアを作ります。

### Mac

```sh
ssh-keygen -t ed25519 -C "ln6001-router-key"
```

保存先は、最初はEnterでデフォルトのままで大丈夫です。

```txt
~/.ssh/id_ed25519
```

パスフレーズは任意です。

より安全にするなら設定します。

### Windows PowerShell

```powershell
ssh-keygen -t ed25519 -C "ln6001-router-key"
```

保存先は、最初はEnterでデフォルトのままで大丈夫です。

```txt
C:\Users\<ユーザー名>\.ssh\id_ed25519
```

作成されるファイルは次です。

![表画像 table-05](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-05.png)

秘密鍵をチャット、メール、SNS、記事、公開リポジトリへ貼らないでください。

公開してよいのは `.pub` が付く公開鍵です。

## LuCIで公開鍵を登録する

一番分かりやすいのは、LuCIのSSH-Keysタブから公開鍵を登録する方法です。

1. PC側で公開鍵を表示する
2. 公開鍵の1行をコピーする
3. LuCIへログインする
4. **System** → **SSH Access** または **System** → **Administration** → **SSH Access** を開く
5. **SSH-Keys** タブを開く
6. 公開鍵を貼り付ける
7. **Save & Apply** をクリックする
8. 別ターミナルから鍵認証で接続確認する

### Macで公開鍵を表示する

```sh
cat ~/.ssh/id_ed25519.pub
```

### Windows PowerShellで公開鍵を表示する

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

公開鍵は、次のような1行です。

```txt
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... ln6001-router-key
```

行全体をコピーします。

途中で改行しないようにします。

## CLIで公開鍵を登録する

LuCIで登録するほうが分かりやすいですが、SSHからCLIで登録することもできます。

### Mac

```sh
cat ~/.ssh/id_ed25519.pub | ssh root@192.168.1.1 "umask 077; mkdir -p /etc/dropbear; cat >> /etc/dropbear/authorized_keys; chmod 600 /etc/dropbear/authorized_keys"
```

LAN IPを変更している場合:

```sh
cat ~/.ssh/id_ed25519.pub | ssh root@192.168.10.1 "umask 077; mkdir -p /etc/dropbear; cat >> /etc/dropbear/authorized_keys; chmod 600 /etc/dropbear/authorized_keys"
```

### Windows PowerShell

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | ssh root@192.168.1.1 "umask 077; mkdir -p /etc/dropbear; cat >> /etc/dropbear/authorized_keys; chmod 600 /etc/dropbear/authorized_keys"
```

LAN IPを変更している場合:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | ssh root@192.168.10.1 "umask 077; mkdir -p /etc/dropbear; cat >> /etc/dropbear/authorized_keys; chmod 600 /etc/dropbear/authorized_keys"
```

登録後、ルーター側で確認します。

```sh
tail -n 5 /etc/dropbear/authorized_keys
ls -l /etc/dropbear/authorized_keys
```

`authorized_keys` に入れるのは公開鍵だけです。

秘密鍵は絶対に入れないでください。

## 鍵認証で接続確認する

別のターミナルやPowerShellを開き、接続します。

```sh
ssh root@192.168.1.1
```

SSH configを作っている場合:

```sh
ssh router
```

鍵ファイルを明示したい場合:

```sh
ssh -i ~/.ssh/id_ed25519 root@192.168.1.1
```

Windows PowerShellでは:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519" root@192.168.1.1
```

パスワードを聞かれずに入れれば、鍵認証が動いています。

パスフレーズを設定した場合は、秘密鍵のパスフレーズを聞かれます。

これはrootパスワードではなく、秘密鍵に付けたパスフレーズです。

## ルーター側でDropbear設定を見る

```sh
uci show dropbear
/etc/init.d/dropbear status
logread | grep -i dropbear | tail -n 80
```

鍵認証に成功している場合、ログに公開鍵認証の記録が出ることがあります。

ログの見え方はファームウェアやDropbearのバージョンで変わります。

最初は、鍵認証で入れることを確認できれば十分です。

## パスワード認証を無効化する前に

鍵認証が動いたら、パスワード認証を無効化したくなるかもしれません。

ただし、ここは慎重に進めます。

最低限、次を確認します。

```txt
□ 鍵認証で接続できた
□ 別のターミナルから再接続できた
□ PC再起動後も接続できた
□ 秘密鍵を安全に保管している
□ 予備PCやLuCIなど復旧手段がある
□ 現在のSSHセッションを閉じずにテストした
```

パスワード認証を無効化する場合は、LuCIのSSH Access画面で設定します。

設定後も、現在のSSHセッションは閉じません。

別ターミナルで再接続テストを行い、接続できることを確認してから古いセッションを閉じます。

これ、地味ですが大事です。

SSH締め出し事故は、だいたい「確認前にセッションを閉じた」ところから始まります。

## よくある接続エラーと対処

![表画像 table-06](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-06.png)

## `Connection refused`

SSHポートへ到達しているけれど、SSHサーバーが受け付けていない状態です。

LuCIのSSH Accessで、LAN側からのSSHが有効か確認します。

もし別経路でルーターへ入れるなら、次を見ます。

```sh
/etc/init.d/dropbear status
uci show dropbear
```

## `Connection timed out`

接続先へ届いていない状態です。

確認します。

```sh
ping 192.168.1.1
```

Windows:

```powershell
Test-NetConnection 192.168.1.1 -Port 22
```

よくある原因です。

- LAN IPを変更している
- ゲストWi-Fiへつないでいる
- PCが別ネットワークにいる
- firewallで止めている
- ルーターが起動中

## `Permission denied (publickey)`

鍵認証のみ許可されていて、使っている鍵が合っていない可能性があります。

詳細ログで確認します。

```sh
ssh -v root@192.168.1.1
```

秘密鍵を明示します。

```sh
ssh -i ~/.ssh/id_ed25519 root@192.168.1.1
```

Windows:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519" root@192.168.1.1
```

ルーター側の `/etc/dropbear/authorized_keys` に、対応する公開鍵が入っているか確認します。

## PCを買い替えた時

PCを買い替えた場合、古いPCの秘密鍵は新しいPCへ自動では移りません。

選択肢は2つです。

1. 新しいPCで新しいSSH鍵を作り、公開鍵をLN6001-JPへ追加する
2. 古いPCの秘密鍵を安全に移行する

おすすめは、新しいPCで新しい鍵を作ることです。

```sh
ssh-keygen -t ed25519 -C "new-pc-ln6001-key"
```

使わなくなった古いPCの公開鍵は、LuCIのSSH-Keysまたは `/etc/dropbear/authorized_keys` から削除します。

古い鍵を残しっぱなしにしない。

これも地味に大事です。

## SSH鍵を整理する

登録済み公開鍵を確認します。

```sh
nl -ba /etc/dropbear/authorized_keys
```

`nl -ba` は行番号付きで表示します。

不要な鍵を削除する場合は、LuCIのSSH-Keysから消すのが安全です。

CLIで編集する場合は、先にバックアップします。

```sh
cp /etc/dropbear/authorized_keys /etc/dropbear/authorized_keys.backup.$(date +%Y%m%d-%H%M)
vi /etc/dropbear/authorized_keys
```

鍵のコメントに端末名を入れておくと管理しやすくなります。

例:

```txt
ssh-ed25519 AAAA... macbook-ikm-202606
ssh-ed25519 AAAA... windows-adminpc-202606
```

## SSH診断メモ

トラブル時に、次のようなメモを残すと相談しやすくなります。

```txt
SSH診断メモ:

接続元:
  MacBook / Windows PC
  有線LAN / Main Wi-Fi

接続先:
  192.168.1.1

結果:
  ssh root@192.168.1.1 OK / NG

確認:
  ping: OK / NG
  port 22: OK / NG
  fingerprint warning: あり / なし
  password auth: OK / NG
  key auth: OK / NG

ルーター状態:
  ubus call system board: 取得済み
  ifstatus wan: up / down
  ifstatus wan6: up / down
  logread tail: 取得済み
```

ログや設定を共有する場合は、SSID、MACアドレス、IPv6 prefix、VPN情報、Wi-Fiパスワード、公開鍵コメントの個人名などを必要に応じて伏せてください。

公開鍵は秘密ではありません。

でも、コメントに端末名や個人名が入っていることはあります。

そこは少し気にしておくと安心です。

## セキュリティ上の注意

SSHを使う時は、次を守ります。

- WAN側へSSHを直接公開しない
- 初期パスワードのまま長く使わない
- rootパスワードは強いものにする
- 鍵認証を使う場合、秘密鍵を外へ出さない
- 公開鍵は端末ごとに分ける
- 使わなくなった端末の公開鍵を削除する
- パスワード認証を無効化する時は、鍵認証の動作確認後に行う
- SSHログや設定ファイルをSNSへ貼らない
- 外出先から管理する場合はVPN経由にする

SSHは管理者用の入口です。

便利ですが、扱いは少し丁寧にしたほうが安全です。

## SSH運用メモ

最後に、こんなメモを残しておくと便利です。

```txt
LN6001-JP SSHメモ:

接続先:
  ssh root@192.168.1.1

SSH config:
  Host router
  HostName 192.168.1.1
  User root
  Port 22

認証:
  パスワード認証: 有効 / 無効
  鍵認証: 有効 / 無効

登録済み鍵:
  macbook-ikm-202606
  windows-adminpc-202606

注意:
  SSHはLAN側のみ
  外出先からはTailscale / WireGuard経由
  秘密鍵は公開しない
  PC買い替え時は新しい公開鍵を登録
```

未来の自分は、だいたい細かいことを忘れます。

メモは未来の自分への置き手紙です。

## まとめ

WindowsとMacのどちらからでも、LN6001-JPへSSH接続できます。

最初はこれだけで十分です。

```sh
ssh root@192.168.1.1
```

初回fingerprint確認で `yes` を入力し、rootパスワードで入れれば成功です。

SSHへ入ったら、まずは読み取りコマンドから始めます。

```sh
ubus call system board
ifstatus wan
ifstatus wan6
cat /tmp/dhcp.leases
logread | tail -n 50
```

鍵認証は、接続に慣れてからで大丈夫です。

公開鍵をLuCIのSSH-Keysへ登録し、別ターミナルで鍵認証できることを確認してから、必要に応じてパスワード認証を見直します。

SSHは、設定を壊すためのものではありません。

状態を見て、落ち着いて切り分けるための道具です。

まずは「接続できる」「ログを見られる」「WAN状態を確認できる」。

それだけでも、OpenWrtベースルーターの運用はかなり楽になります。

黒い画面は、敵ではありません。

ただ、最初からそこで戦わなくていいだけです。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-03.png)

## 次に読むなら

SSH接続に慣れたら、次の記事も合わせて読むと運用しやすくなります。

- [SSH基本コマンド集](https://note.com/ikmsan/n/n99897dd3cae6)
- [設定バックアップと復元](https://note.com/ikmsan/n/n3c25190a8e94)
- [つながらない時の切り分け](https://note.com/ikmsan/n/n1ab2d47c6d10)
- [パッケージ管理](https://note.com/ikmsan/n/n2bc5e8447e77)
- [監視とログ](https://note.com/ikmsan/n/n8571cacdde40)

SSHで何を見ればよいかを増やしたい人は、SSH基本コマンド集へ。

設定変更前の戻し方を固めたい人は、設定バックアップと復元へ進むと読みやすいです。

## CLI例の前提

この記事のCLI例は、LN6001-JPのOpenWrtベースのver 1.2.0.15ファームウェアを前提にしています。

LuCIの画面名、SSH Access / SSH-Keysの表示、Dropbear設定、WindowsやmacOSのSSHクライアント挙動は、OSやファームウェア更新で変わることがあります。

記事の内容と実際の画面や出力が違う場合は、まずLinksys公式サポートやOpenWrt / OpenSSHの最新情報を確認してください。

無線の国設定、送信出力、DFS関連の値は変更しません。

## よくある質問

### WindowsでもLN6001-JPへSSH接続できますか？

できます。

Windows 10 / 11では、PowerShellやWindows TerminalからOpenSSHクライアントを使って接続できます。

```powershell
ssh root@192.168.1.1
```

### Macでは追加ソフトが必要ですか？

通常は不要です。

ターミナルから標準のSSHクライアントで接続できます。

```sh
ssh root@192.168.1.1
```

### 初回fingerprint確認は `yes` で進めていいですか？

自分のLAN内で、接続先が本当にLN6001-JPだと分かっている場合は進めて大丈夫です。

身に覚えがないfingerprint変更が出た場合は、接続先を確認してください。

### ルーター初期化後に `Host key verification failed` が出ます

初期化でSSHホスト鍵が変わった可能性があります。

PC側の古い記録を削除します。

```sh
ssh-keygen -R 192.168.1.1
```

その後、再接続して新しいfingerprintを確認します。

### 鍵認証は最初から設定したほうがいいですか？

最初はパスワード認証で大丈夫です。

SSH接続に慣れてから鍵認証へ移るほうが、トラブル時に切り分けしやすいです。

### パスワード認証を無効化してもいいですか？

鍵認証で確実に接続できることを確認してからにしてください。

別ターミナルや別PCから再接続できることを確認するまで、現在のSSHセッションは閉じないほうが安全です。

### SSHはWAN側へ公開してもいいですか？

おすすめしません。

外出先から管理したい場合は、TailscaleやWireGuardなどのVPN経由でSSHへアクセスしてください。

### 公開鍵と秘密鍵の違いは？

公開鍵はルーターへ登録する鍵です。

秘密鍵はPC側に保管する鍵で、絶対に外へ出してはいけません。

### `ssh-copy-id` はMacで使えますか？

環境によります。

入っていないMacもあります。

この記事では、`cat ~/.ssh/id_ed25519.pub | ssh ...` のように、標準コマンドだけで公開鍵を登録する方法も紹介しています。

### パスワードを入力しても画面に何も出ません

正常です。

SSHでは、パスワード入力中に文字も `*` も表示されません。

そのまま入力してEnterを押してください。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 WRT Pro 7 FAQs: https://support.linksys.com/kb/article/217-en/?section_id=175
- Velop WRT Pro 7 OpenWrt ルーターのWEB管理画面へアクセスする方法: https://support.linksys.com/kb/article/7038/
- OpenSSH公式サイト: https://www.openssh.com/
- Microsoft OpenSSH documentation: https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse
- Microsoft OpenSSH key management: https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement
- OpenWrt Wiki - SSH access for newcomers: https://openwrt.org/docs/guide-quick-start/sshadministration
- OpenWrt Wiki - Dropbear public key authentication: https://openwrt.org/docs/guide-user/security/dropbear.public-key.auth

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
