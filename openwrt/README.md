# OpenWrt / Linksys Velop WRT Pro 7 連載

Linksys Velop WRT Pro 7（LN6001-JP）を題材に、OpenWrtベースのルーター運用をまとめた記事シリーズです。

家庭、小規模オフィス、店舗で使うWi-Fi、VLAN、VPN、IPoE、広告ブロック、バックアップ、トラブル対応を、実際の運用に近い順番で読めるように整理しています。

## 読み始めにおすすめ

- [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](000-hub-openwrt-guide.md)
- [OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](001-openwrt-intro.md)
- [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](002-japan-technical-conformity.md)
- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](003-ln6001-product-positioning.md)
- [LN6001-JP初期設定チェックリスト](004-initial-setup-checklist.md)

## 目的別ガイド

### 基本操作

- [LuCIとSSHの基本](005-luci-ssh-basics.md)
- [SSH基本コマンド集](025-ssh-basic-commands.md)
- [WindowsとMacからSSHログイン](033-ssh-login-windows-mac.md)

### Wi-Fiと家庭向け設定

- [LN6001-JPのWi-Fi設定](006-wifi-band-settings.md)
- [ゲストWi-Fiの作り方](007-guest-wifi.md)
- [DNS広告ブロック](008-dns-adblock.md)
- [家族向けフィルタリング](010-family-filtering.md)
- [自宅での設定例](027-home-configuration.md)

### 小規模オフィス・店舗

- [VLANで社内・ゲストネットワークを分ける](011-vlan-office-guest.md)
- [固定IPとDHCP予約](012-dhcp-reservations.md)
- [監視とログ](015-monitoring-logs.md)
- [firewall zonesの考え方](016-firewall-zones.md)
- [ポート開放の注意点](017-port-forwarding-caution.md)
- [小規模店舗での設定例](029-store-configuration.md)

### VPN・リモート接続

- [WireGuard/Tailscaleでリモート接続](013-wireguard-tailscale-remote.md)
- [Tailscaleの使いどころ](018-tailscale-zerotier-use.md)
- [VPN設定まとめ](028-vpn-configuration.md)

### 回線・IPv6・IPoE

- [NTT IPoEとIPv4 over IPv6](014-ipoe-ipv4-over-ipv6.md)
- [IPv6の落とし穴](019-ipv6-pitfalls.md)
- [つながらない時の切り分け](021-troubleshooting-connectivity.md)

### 運用・復旧

- [ファームウェア更新運用](020-firmware-update.md)
- [リセットと復旧](022-reset-recovery.md)
- [バックアップと復元](023-backup-restore.md)
- [パッケージ管理](024-package-management.md)

### ハードウェア・レビュー・応用

- [ハードウェア概要](026-hardware-overview.md)
- [LN6001-JPレビュー](030-ln6001-review-fit.md)
- [MLOセットアップ](031-mlo-setup.md)
- [APモード・ブリッジ・WDS](032-ap-bridge-wds.md)

## 全記事一覧

- [000: LN6001-JPで始めるOpenWrtベースルーター実践ガイド](000-hub-openwrt-guide.md)
- [001: OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](001-openwrt-intro.md)
- [002: 日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](002-japan-technical-conformity.md)
- [003: LN6001-JPとは何か](003-ln6001-product-positioning.md)
- [004: LN6001-JP初期設定チェックリスト](004-initial-setup-checklist.md)
- [005: LuCIとSSHの基本](005-luci-ssh-basics.md)
- [006: LN6001-JPのWi-Fi設定](006-wifi-band-settings.md)
- [007: ゲストWi-Fiの作り方](007-guest-wifi.md)
- [008: DNS広告ブロック](008-dns-adblock.md)
- [009: LN6001-JPが快適な理由](009-sqm-latency.md)
- [010: 家族向けフィルタリング](010-family-filtering.md)
- [011: VLANで社内・ゲストネットワークを分ける](011-vlan-office-guest.md)
- [012: 固定IPとDHCP予約](012-dhcp-reservations.md)
- [013: WireGuard/Tailscaleでリモート接続](013-wireguard-tailscale-remote.md)
- [014: NTT IPoEとIPv4 over IPv6](014-ipoe-ipv4-over-ipv6.md)
- [015: 監視とログ](015-monitoring-logs.md)
- [016: firewall zonesの考え方](016-firewall-zones.md)
- [017: ポート開放の注意点](017-port-forwarding-caution.md)
- [018: Tailscaleの使いどころ](018-tailscale-zerotier-use.md)
- [019: IPv6の落とし穴](019-ipv6-pitfalls.md)
- [020: ファームウェア更新運用](020-firmware-update.md)
- [021: つながらない時の切り分け](021-troubleshooting-connectivity.md)
- [022: リセットと復旧](022-reset-recovery.md)
- [023: バックアップと復元](023-backup-restore.md)
- [024: パッケージ管理](024-package-management.md)
- [025: SSH基本コマンド集](025-ssh-basic-commands.md)
- [026: ハードウェア概要](026-hardware-overview.md)
- [027: 自宅での設定例](027-home-configuration.md)
- [028: VPN設定まとめ](028-vpn-configuration.md)
- [029: 小規模店舗での設定例](029-store-configuration.md)
- [030: LN6001-JPレビュー](030-ln6001-review-fit.md)
- [031: MLOセットアップ](031-mlo-setup.md)
- [032: APモード・ブリッジ・WDS](032-ap-bridge-wds.md)
- [033: WindowsとMacからSSHログイン](033-ssh-login-windows-mac.md)
