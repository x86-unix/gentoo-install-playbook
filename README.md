# Gentoo Linux ZFS Install Playbook

Ansible playbook で Gentoo Linux を ZFS root にインストールする。

## 前提条件

- ターゲットマシンが ZFS 対応のライブ環境（SystemRescue 等）で起動済み
- ライブ環境で SSH + ZFS が利用可能
- コントロールノードに `ansible` と `sshpass` がインストール済み

## 構成

```
site-base.yml              # ベースインストール一括実行
site-extra.yml             # 拡張セットアップ（reboot 後に実行）

base/
  base-00-preflight.yml    # SSH 疎通・ZFS コマンド確認
  base-01-disk-setup.yml   # パーティション作成・ZFS pool/dataset 作成・2台目ディスク
  base-02-stage3.yml       # stage3 ダウンロード・展開・chroot 準備
  base-03-portage.yml      # Portage 設定・make.conf 生成・ccache（20G）
  base-04-system.yml       # timezone/locale/hostname/network/fstab/WiFi(iwd)
  base-05-kernel.yml       # カーネルビルド・ZFS モジュール・dracut initramfs
  base-06-bootloader.yml   # GRUB インストール・grub.cfg 生成（BIOS/UEFI 自動）
  base-07-finalize.yml     # root パスワード・SSH・unmount・reboot

extra/
  extra-portage-git.yml    # portage sync を git に切り替え・eix インストール
  extra-services.yml       # システムサービス（cronie）
  extra-cli-tools.yml      # CLI ツール（vim, sudo, bash-completion 等）
  extra-account.yml        # sudo 設定・ユーザー作成
  extra-wayland.yml        # Wayland デスクトップ（labwc）
  extra-sound.yml          # サウンド（pipewire + wireplumber）
  extra-ime.yml            # IME（fcitx5 + SKK、ソースビルド）
  extra-gui-tools.yml      # GUI ツール（Google Chrome, VSCode、GURU overlay）

group_vars/target.yml      # 全設定値
inventory.ini              # ターゲットホスト定義
```

## パーティション構成

| # | BIOS モード | UEFI モード | 用途 |
|---|-------------|-------------|------|
| 1 | 2M (EF02)   | 512M (EF00) | BIOS boot / ESP |
| 2 | swap        | swap        | メモリ同等（最大 8G） |
| 3 | ZFS (BF00)  | ZFS (BF00)  | rpool（compatibility=grub2） |

## ZFS レイアウト

```
rpool                  mountpoint=none    canmount=off
rpool/ROOT             mountpoint=none    canmount=off
rpool/ROOT/gentoo      mountpoint=/       canmount=noauto
rpool/home             mountpoint=/home   canmount=on
```

## GRUB の起動パス

GRUB は `search --label` で rpool を自動検出する（BIOS/UEFI 共通）：

```
search --no-floppy --label --set=root rpool
linux /ROOT/gentoo@/boot/kernel-<version> root=zfs:rpool/ROOT/gentoo
initrd /ROOT/gentoo@/boot/initramfs-<version>.img
```

- `search --label rpool` — BIOS/UEFI 両対応でディスク番号に依存しない
- initramfs が ZFS モジュールをロードし、rpool を import してから root をマウントする

## 使い方

### 1. 設定を編集

```bash
vi inventory.ini
```

```ini
[target]
192.168.77.26 ansible_user=root
```

```bash
vi group_vars/target.yml
```

```yaml
# Disk
default_target_disk: sda       # インストール先ディスク（例: sda, nvme0n1）
second_disk: ""                 # 2台目物理ディスク名。設定すると ZFS pool を作成し /mnt/<disk> にマウント
ccache_dir: ""                  # ccache ディレクトリ。空の場合は /var/cache/ccache を使用

# System
default_timezone: Asia/Tokyo
default_hostname: zfs-gentoo

# Portage
gentoo_mirror: https://distfiles.gentoo.org
use_flags: "zfs wayland -X -gtk -qt5 -qt6"

# Locale
default_locales:
  - en_US.UTF-8 UTF-8
  - ja_JP.UTF-8 UTF-8
default_lang: ja_JP.UTF-8
default_l10n: "en ja"

# Root password
default_root_password: root

# User
default_user: gentoo
default_user_password: gentoo
sudo_nopasswd: true

# WiFi (optional) - WiFi NIC が検出された場合のみ使用
wifi_ssid: ""
wifi_password: ""
```

| 変数 | 説明 |
|------|------|
| `default_target_disk` | インストール先ディスク（`sda`, `nvme0n1` 等） |
| `second_disk` | 2台目ディスク名。設定すると ZFS pool を作成し `/mnt/<disk>` にマウント |
| `ccache_dir` | ccache ディレクトリ。空の場合は `/var/cache/ccache` を使用 |
| `default_timezone` | システムのタイムゾーン |
| `default_hostname` | ホスト名 |
| `gentoo_mirror` | stage3 / distfiles のミラー URL |
| `use_flags` | Portage のグローバル USE フラグ |
| `default_locales` | `/etc/locale.gen` に書くロケール一覧 |
| `default_lang` | `LANG` 環境変数 |
| `default_l10n` | Portage の `L10N` |
| `default_root_password` | root パスワード（平文） |
| `default_user` | 作成するユーザー名 |
| `default_user_password` | ユーザーパスワード（平文） |
| `sudo_nopasswd` | `true` で NOPASSWD sudo を有効化 |
| `wifi_ssid` | WiFi SSID（WiFi NIC 検出時のみ使用、vault 推奨） |
| `wifi_password` | WiFi パスワード（WiFi NIC 検出時のみ使用、vault 推奨） |

### 2. ベースインストール

ライブ環境から実行する：

```bash
ansible-playbook -i inventory.ini site-base.yml -e ansible_ssh_pass=rescue
```

vault を使用している場合：

```bash
ansible-playbook -i inventory.ini site-base.yml -e ansible_ssh_pass=rescue --vault-password-file <(echo <vault_password>)
```

実行中に確認プロンプトが出る：

1. 開始確認 — インストールを開始してよいか
2. ディスク確認 — rpool が既に存在する場合のみ、破壊して再作成するか確認

最後に reboot 確認が出る。`yes` で再起動、それ以外でスキップ。

カーネルビルドに 1〜2 時間かかる。

### 3. 拡張セットアップ（reboot 後）

```bash
ansible-playbook -i inventory.ini site-extra.yml --ask-pass
```

実行順序：

1. portage を git sync に切り替え
2. システムサービス（cronie）
3. CLI ツール（vim, sudo, bash-completion 等）
4. ユーザーアカウント・sudo 設定
5. Wayland デスクトップ（labwc + waybar + foot + wofi）
6. サウンド（pipewire + wireplumber）
7. IME（fcitx5 + SKK、ソースビルド）
8. GUI ツール（Google Chrome, VSCode、GURU overlay 経由）

### 4. 個別実行

```bash
ansible-playbook -i inventory.ini base/base-05-kernel.yml -e ansible_ssh_pass=rescue
ansible-playbook -i inventory.ini extra/extra-wayland.yml --ask-pass
```

## ccache

- `FEATURES="ccache"` を make.conf に設定済み
- デフォルトは `/var/cache/ccache`（max 20G）
- `second_disk` を設定した場合は `ccache_dir` を合わせて指定すると2台目ディスク上に配置できる

2台目ディスクを使う場合の設定例（`sdb` の場合）：

```yaml
second_disk: "sdb"
ccache_dir: "/mnt/sdb/ccache"
```

## WiFi

- WiFi NIC（`wlan*`, `wlp*`）が検出された場合のみ `iwd` をインストール・設定
- `wifi_ssid` と `wifi_password` を設定すると起動時に自動接続
- 有線と WiFi NIC が共存していても WiFi NIC が存在すれば kernel の crypto オプション（`CONFIG_CRYPTO_*`）を有効化してビルドする
- 有線のみの環境では WiFi NIC がなければ何もしない
- パスワードは `ansible-vault` で暗号化推奨：

```bash
ansible-vault encrypt_string 'your_ssid' --name 'wifi_ssid'
ansible-vault encrypt_string 'your_password' --name 'wifi_password'
```

## 冪等性

| playbook | スキップ条件 |
|----------|-------------|
| base-01 | rpool が既に存在し、ユーザーが再作成を拒否した場合 |
| base-02 | `/usr/bin/emerge` が存在する |
| base-03 | portage tree が存在する、make.conf が同一 |
| base-04 | 各設定ファイルが同一 |
| base-05 | カーネルイメージ + ZFS モジュール + initramfs が存在する |
| base-06 | GRUB が MBR/EFI にインストール済み、grub.cfg が同一 |
| base-07 | 毎回実行（パスワード設定）。reboot は確認後のみ |

## 注意事項

- rpool が存在しないディスクに対しては確認なしでパーティションが作成される
- reboot を拒否した場合、ライブ環境の bind mount はそのまま維持される（再実行可能）
- ライブ環境のカーネル config（`/proc/config.gz`）をベースに `localmodconfig` でカーネルを最小化している。異なるライブ環境で再実行すると config が変わりカーネルが再ビルドされる
- `group_vars/target.yml` に root パスワードとユーザーパスワードが平文で保存される。必要に応じて `ansible-vault` で暗号化すること
- `extra-ime.yml` は fcitx5 + SKK をソースからビルドするため時間がかかる
- `extra-gui-tools.yml` は GURU overlay を使用する。初回は overlay の sync が走る
