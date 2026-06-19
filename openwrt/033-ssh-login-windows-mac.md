<!-- mirror-source: articles/033-ssh-login-windows-mac.md -->

> Original note.com article: [WindowsとMacからSSHログイン: 初回接続から鍵認証まで【OpenWrt集中連載033】](https://note.com/ikmsan/n/na751d7336b87)

# WindowsとMacからSSHログイン: 初回接続から鍵認証まで【OpenWrt集中連載033】

OpenWrtのように細かくカスタマイズできるWi-Fiルーターは面白そうだけど、「日本のIPoE回線で本当に普通に使えるの？」「設定を触りすぎて壊れない？」と不安になる人も多いと思います。

この連載では、OpenWrtベースのWi-Fi 7ルーター「Linksys Velop WRT Pro 7」を使いながら、家庭・小規模オフィス・店舗向けに、“実用的なOpenWrt運用” をわかりやすく紹介していきます。

LN6001-JPは、日本向けに技適や法令へ対応したモデルで、一般的なWi-Fiルーターのように最初からセットアップ済み。OpenWrt系ルーターとしてはかなり始めやすい部類です。

このシリーズでは、実機開発にも関わった立場から、LuCI・SSH・VLAN・VPN・IPoE/IPv6まわりまで、「結局どう設定するのが現実的なのか？」を、画面操作とCLIの両方でまとめていきます。

## 要約

LN6001-JPはSSHサーバー（Dropbear）が動作しており、Mac、WindowsどちらのPCからもSSH接続できます。

LuCIでは見えにくい状態確認やログ確認、バックアップ作業などを行いたい時にSSHがかなり役立ちます。

ただし、最初からCLIで全部設定変更しなくても大丈夫です。

まずは「SSHへ入れる」「状態確認コマンドを打てる」くらいから始めるだけでも、かなり運用しやすくなります。

この記事では、Mac、Windows両方からのSSH接続手順、初回fingerprint確認の意味、あとから鍵認証へ移行する流れを整理します。

![よくある悩み](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-01.png)

## この記事でわかること

- WindowsとMacからLN6001-JPへSSH接続する方法
- 初回fingerprint確認の意味
- パスワード認証から鍵認証へ移る流れ
- SSH接続後にまず確認したい基本コマンド

## こんな人に向いています

- WindowsやMacからLN6001-JPへ初めてSSH接続する
- LuCIだけでは見えない情報を確認したい
- 最初はパスワード認証で入り、あとから鍵認証へ移りたい

![考え方はシンプル](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-02.png)

## まずはここまでで十分

SSHも、最初から難しく考えなくて大丈夫です。

まずは次の3つで十分です。

1. 同じネットワークから `ssh root@192.168.1.1` で入る
2. 初回fingerprint確認を理解して受け入れる
3. パスワード認証で接続できることを確認する

最初は「接続できること」がかなり重要で、鍵認証やconfig整理はあとからで問題ありません。

---

最初は、有線LAN接続したPCから確認するほうが切り分けしやすくなります。

## 接続前の確認

- LN6001-JPのLAN IP（デフォルト: `192.168.1.1`）
- 管理者パスワード（LuCIのrootパスワードと同じ）
- PCとLN6001-JPが同じネットワークへ接続されていること（有線推奨）

Linksys公式FAQでは、SSHを有効化する場所として LuCI の **System** → **SSH Access** が案内されています。

鍵を追加する場合は **SSH-Keys** タブから登録できます。

接続前にPC側から疎通だけ確認します。

**Mac:**

```sh
ping -c 3 192.168.1.1
nc -vz 192.168.1.1 22
```

**Windows（PowerShell）:**

```powershell
ping 192.168.1.1
Test-NetConnection 192.168.1.1 -Port 22
```

ポート22が開いていなければ、LuCIのSSH Access設定、有線接続、LAN IP変更の有無を先に確認します。

---

Macでは、最初からSSHクライアントが入っているため追加ソフトなしで始められます。

## MacからのSSH接続

Macには標準でSSHクライアントが入っています。

ターミナルアプリを使います。

### ターミナルを開く

- Launchpad → ターミナル
- または Spotlight（Cmd + Space）→ 「ターミナル」と検索

### SSH接続コマンド

```sh
ssh root@192.168.1.1
```

最初は「ssh root@192.168.1.1 を打てる」だけでもかなり役立ちます。

### 初回接続時の操作

初回接続時にルーターの fingerprint（公開鍵のハッシュ）が表示されます:

```
The authenticity of host '192.168.1.1 (192.168.1.1)' can't be established.
ED25519 key fingerprint is SHA256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes` と入力してEnterを押します。

次にパスワードを入力します。

### LAN IPを変更している場合

```sh
ssh root@192.168.X.X   # 変更後のLAN IPを使用

# 接続が不安定な時は詳細ログを表示
ssh -v root@192.168.X.X
```

最初は、「LAN IPを変更していないか」を確認するだけでもかなり切り分けしやすくなります。

---

Windows 10 / 11でも、追加ソフトなしでSSH接続できます。

## WindowsからのSSH接続

Windows 10 / 11にはOpenSSHクライアントが標準搭載されています。

### PowerShell（またはコマンドプロンプト）を開く

- スタートメニュー → 「PowerShell」または「cmd」と検索
- または Win + X → Windows PowerShell

### SSH接続コマンド

```powershell
ssh root@192.168.1.1
```

最初はPowerShellから試すほうが整理しやすくなります。

初回接続時に「yes」と入力してEnterを押し、次にパスワードを入力します。

接続できない時は、PowerShell側で詳細ログを出します。

```powershell
ssh -v root@192.168.1.1
Test-NetConnection 192.168.1.1 -Port 22
```

普段PowerShellを使わない場合は、PuTTYを使う方法もあります。

### PuTTYを使う場合（オプション）

PuTTY（https://www.putty.org/）を使う場合:

1. PuTTYをダウンロードしてインストール
2. Session画面: Host Name = `192.168.1.1`, Port = `22`, Connection type = `SSH`
3. **Open** をクリック
4. セキュリティ警告が出たら **Accept**
5. `login as:` に `root` と入力してEnter
6. パスワードを入力してEnter

最初は、「ログインできること」を確認できれば十分です。

---

SSHへ慣れてきたら、接続ショートカットを作るとかなり楽になります。

## SSH接続の設定ショートカット（~/.ssh/config）

**Mac/Linux（ターミナルで実行）:**

```sh
# SSH設定ファイルを編集（なければ自動作成）
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

最初は `ssh router` だけで接続できるようになるとかなり便利です。

**Windows（PowerShellで実行）:**

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh"
Add-Content -Path "$env:USERPROFILE\.ssh\config" -Value "`nHost router`n    HostName 192.168.1.1`n    User root`n    Port 22"
```

Windowsでも同じように接続ショートカットを作れます。

---

鍵認証は、「SSHへ慣れてから」少しずつ追加するくらいでも十分です。

## SSH鍵認証の設定（パスワードレスログイン）

パスワード認証よりセキュリティが高く、入力の手間も減らせます。

### ステップ1: SSHキーペアを生成する（PC側）

**Mac/Linux:**

```sh
# Ed25519鍵を生成（推奨）
ssh-keygen -t ed25519 -C "my-router-key"

# 保存先は Enter でデフォルト（~/.ssh/id_ed25519）を使用
# パスフレーズは任意（空でも可、設定するとよりセキュア）
```

最初は、デフォルト保存先（~/.ssh/id_ed25519）のままでも十分です。

**Windows（PowerShell）:**

```powershell
ssh-keygen -t ed25519 -C "my-router-key"
```

Windowsでも、基本的な流れはMacとほぼ同じです。

### ステップ2: 公開鍵をルーターに登録する

**Mac/Linux:**

```sh
# 公開鍵をルーターへ転送（パスワード認証で一度ログインが必要）
ssh-copy-id root@192.168.1.1

# または手動でコピー
cat ~/.ssh/id_ed25519.pub | ssh root@192.168.1.1 "mkdir -p /etc/dropbear && cat >> /etc/dropbear/authorized_keys && chmod 600 /etc/dropbear/authorized_keys"
```

`ssh-copy-id` は、まず最初に覚えておきたい便利コマンドです。

**Windows（PowerShell）:**

```powershell
# 公開鍵の内容を確認してコピー
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"

# SSHでルーターにログインして公開鍵を貼り付け
ssh root@192.168.1.1
```

ルーター上で:

```sh
# authorized_keysに公開鍵を追加
mkdir -p /etc/dropbear
echo "ssh-ed25519 AAAA...（公開鍵の内容）" >> /etc/dropbear/authorized_keys
chmod 600 /etc/dropbear/authorized_keys

# 登録内容を確認
tail -n 3 /etc/dropbear/authorized_keys
```

最初は、「authorized_keysへ登録できたか」を確認するだけでも十分役立ちます。

### ステップ3: 鍵認証の動作確認

```sh
# パスワードなしで接続できることを確認
ssh root@192.168.1.1

# またはショートカット設定済みの場合
ssh router

# ルーター側でDropbear設定を確認
uci show dropbear
/etc/init.d/dropbear status
```

最初は、「パスワードなしで再接続できるか」を確認できればかなり十分です。

---

SSHでは、「壊れている」のではなく、「IPアドレス違い」や「古いfingerprintが残っている」ケースもかなり多くあります。

## よくある接続エラーと対処

![表画像 table-01](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/table-01.png)

最初は、「LAN IP違い」か「fingerprint問題」かを分けるだけでもかなり切り分けしやすくなります。

---

ルーター初期化後は、fingerprint不一致が出ることがあります。

### fingerprint削除の詳細

ルーターをリセットした後に接続しようとすると `Host key verification failed` が出ます:

```sh
# Mac/Linux: known_hostsからエントリを削除
ssh-keygen -R 192.168.1.1

# その後再接続（新しいfingerprintを accept）
ssh root@192.168.1.1
```

**Windows（PowerShell）:**

```powershell
ssh-keygen -R 192.168.1.1
ssh root@192.168.1.1
```

最初は、「古いfingerprintを削除して再接続する」流れだけ覚えておけば十分です。

---

SSHへ入れたら、最初は「状態確認コマンド」を中心に覚えるほうが整理しやすくなります。

```sh
# システム情報確認
ubus call system board

# WAN接続状態確認
ifstatus wan | grep '"ipaddr"'
ifstatus wan6 | grep -E '"ip6addr"|"ip6prefix"'

# DHCPリース一覧
cat /tmp/dhcp.leases

# ログ確認
logread | tail -n 50

# 設定変更前の簡易バックアップ
BACKUP_DIR="/root/ssh-first-check-$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
cp /etc/config/network "$BACKUP_DIR/network"
cp /etc/config/firewall "$BACKUP_DIR/firewall"
cp /etc/config/dropbear "$BACKUP_DIR/dropbear" 2>/dev/null || true
```

最初は `ubus call system board`、`ifstatus wan`、`logread | tail -n 50` の3つを見られるだけでもかなり役立ちます。

サービス再起動や設定変更は、バックアップを取ってから必要な時だけ実行します。

---

## まとめ

最初の成功条件としては、次の3つを見られれば十分です。

- MacまたはWindowsからログインできる
- `ubus call system board` や `ifstatus wan` のような基本確認ができる
- ログアウト後も同じ手順で再接続できる

最初はパスワード認証でつながれば十分です。

接続に慣れてきたら `~/.ssh/config` や鍵認証を追加していくと、普段の確認作業がかなり楽になります。

SSHは設定変更だけでなく、「今どう動いているか」を落ち着いて確認する入口として使うと整理しやすくなります。

![まず見るところ](https://raw.githubusercontent.com/ikm-san/blog/main/openwrt/assets/033/diagram-03.png)

## よくある質問

### WindowsでもLN6001-JPへSSH接続できる？

できます。

Windows 10 / 11なら、PowerShellのOpenSSHクライアントでそのまま接続できます。

### LN6001-JPへSSHする時、初回fingerprint確認は進めて大丈夫？

自分のLAN内で、接続先が本当にLN6001-JPだと分かっているなら進めて問題ありません。

以後は、その鍵情報が保存されます。

### LN6001-JPのSSH鍵認証は最初から設定したほうがいい？

最初はパスワード認証で十分です。

接続に慣れてから鍵認証へ切り替えると、運用しやすさと安全性を両立しやすくなります。

## 参考リンク

- Linksys Velop WRT Pro 7 製品情報 ＆ FAQ: https://support.linksys.com/kb/article/6274-jp/
- Linksys MBE70 WRT Pro 7 FAQs: https://support.linksys.com/kb/article/217-en/?section_id=175

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
