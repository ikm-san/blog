# ikm-san 技術ブログ

OpenWrt、ネットワーク、Linksys Velop WRT Pro 7（LN6001-JP）まわりの実践記事をまとめています。

家庭、小規模オフィス、店舗で使うネットワークを、できるだけ現実の運用に寄せて整理するためのブログです。

## OpenWrt / Linksys Velop WRT Pro 7

Wi-Fi 7ルーター Linksys Velop WRT Pro 7（LN6001-JP）を題材に、OpenWrtベースのルーター運用を基礎から実践までまとめています。

- [OpenWrt連載トップ](openwrt/)
- [LN6001-JPで始めるOpenWrtベースルーター実践ガイド](openwrt/000-hub-openwrt-guide.md)

## まず読む記事

- [OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](openwrt/001-openwrt-intro.md)
- [日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](openwrt/002-japan-technical-conformity.md)
- [LN6001-JPとは何か: OpenWrtベースWi-Fi 7ルーターの立ち位置](openwrt/003-ln6001-product-positioning.md)
- [LN6001-JP初期設定チェックリスト](openwrt/004-initial-setup-checklist.md)
- [LuCIとSSHの基本](openwrt/005-luci-ssh-basics.md)

## 目的別に読む

### 家庭で使う

- [LN6001-JPのWi-Fi設定](openwrt/006-wifi-band-settings.md)
- [ゲストWi-Fiの作り方](openwrt/007-guest-wifi.md)
- [DNS広告ブロック](openwrt/008-dns-adblock.md)
- [家族向けフィルタリング](openwrt/010-family-filtering.md)
- [自宅での設定例](openwrt/027-home-configuration.md)

### 小規模オフィス・店舗で使う

- [VLANで社内・ゲストネットワークを分ける](openwrt/011-vlan-office-guest.md)
- [固定IPとDHCP予約](openwrt/012-dhcp-reservations.md)
- [監視とログ](openwrt/015-monitoring-logs.md)
- [firewall zonesの考え方](openwrt/016-firewall-zones.md)
- [小規模店舗での設定例](openwrt/029-store-configuration.md)

### VPN・リモート接続

- [WireGuard/Tailscaleでリモート接続](openwrt/013-wireguard-tailscale-remote.md)
- [Tailscaleの使いどころ](openwrt/018-tailscale-zerotier-use.md)
- [VPN設定まとめ](openwrt/028-vpn-configuration.md)

### 回線・IPv6・IPoE

- [NTT IPoEとIPv4 over IPv6](openwrt/014-ipoe-ipv4-over-ipv6.md)
- [IPv6の落とし穴](openwrt/019-ipv6-pitfalls.md)
- [つながらない時の切り分け](openwrt/021-troubleshooting-connectivity.md)

### 運用・復旧

- [ファームウェア更新運用](openwrt/020-firmware-update.md)
- [リセットと復旧](openwrt/022-reset-recovery.md)
- [バックアップと復元](openwrt/023-backup-restore.md)
- [パッケージ管理](openwrt/024-package-management.md)
- [SSH基本コマンド集](openwrt/025-ssh-basic-commands.md)

## 記事一覧

- [000: LN6001-JPで始めるOpenWrtベースルーター実践ガイド](openwrt/000-hub-openwrt-guide.md)
- [001: OpenWrtルーターは一般的な市販Wi-Fiルーターとは何が違うのか](openwrt/001-openwrt-intro.md)
- [002: 日本でOpenWrt系Wi-Fiルーターを使う時に技適をどう考えるか](openwrt/002-japan-technical-conformity.md)
- [003: LN6001-JPとは何か](openwrt/003-ln6001-product-positioning.md)
- [004: LN6001-JP初期設定チェックリスト](openwrt/004-initial-setup-checklist.md)
- [005: LuCIとSSHの基本](openwrt/005-luci-ssh-basics.md)
- [006: LN6001-JPのWi-Fi設定](openwrt/006-wifi-band-settings.md)
- [007: ゲストWi-Fiの作り方](openwrt/007-guest-wifi.md)
- [008: DNS広告ブロック](openwrt/008-dns-adblock.md)
- [009: LN6001-JPが快適な理由](openwrt/009-sqm-latency.md)
- [010: 家族向けフィルタリング](openwrt/010-family-filtering.md)
- [011: VLANで社内・ゲストネットワークを分ける](openwrt/011-vlan-office-guest.md)
- [012: 固定IPとDHCP予約](openwrt/012-dhcp-reservations.md)
- [013: WireGuard/Tailscaleでリモート接続](openwrt/013-wireguard-tailscale-remote.md)
- [014: NTT IPoEとIPv4 over IPv6](openwrt/014-ipoe-ipv4-over-ipv6.md)
- [015: 監視とログ](openwrt/015-monitoring-logs.md)
- [016: firewall zonesの考え方](openwrt/016-firewall-zones.md)
- [017: ポート開放の注意点](openwrt/017-port-forwarding-caution.md)
- [018: Tailscaleの使いどころ](openwrt/018-tailscale-zerotier-use.md)
- [019: IPv6の落とし穴](openwrt/019-ipv6-pitfalls.md)
- [020: ファームウェア更新運用](openwrt/020-firmware-update.md)
- [021: つながらない時の切り分け](openwrt/021-troubleshooting-connectivity.md)
- [022: リセットと復旧](openwrt/022-reset-recovery.md)
- [023: バックアップと復元](openwrt/023-backup-restore.md)
- [024: パッケージ管理](openwrt/024-package-management.md)
- [025: SSH基本コマンド集](openwrt/025-ssh-basic-commands.md)
- [026: ハードウェア概要](openwrt/026-hardware-overview.md)
- [027: 自宅での設定例](openwrt/027-home-configuration.md)
- [028: VPN設定まとめ](openwrt/028-vpn-configuration.md)
- [029: 小規模店舗での設定例](openwrt/029-store-configuration.md)
- [030: LN6001-JPレビュー](openwrt/030-ln6001-review-fit.md)
- [031: MLOセットアップ](openwrt/031-mlo-setup.md)
- [032: APモード・ブリッジ・WDS](openwrt/032-ap-bridge-wds.md)
- [033: WindowsとMacからSSHログイン](openwrt/033-ssh-login-windows-mac.md)
