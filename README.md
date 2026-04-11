# Gentoo Linux ZFS Install Playbook

Ansible playbook で Gentoo Linux を ZFS root にインストールする。

## 前提条件

- ターゲットマシンが ZFS 対応のライブ環境（SystemRescue 等）で起動済み
- ライブ環境で SSH + ZFS が利用可能
- コントロールノードに `ansible` と `sshpass` がインストール済み

## 構成

```
site.yml              # 全工程を一括実行（00-07）
site-extra.yml        # 拡張セットアップ（reboot 後に実行）
00-preflight.yml      # SSH 疎通・ZFS コマンド確認
01-disk-setup.yml     # パーティション作成・ZFS pool/dataset 作成
02-stage3.yml         # stage3 ダウンロード・展開・chroot 準備
03-portage.yml        # Portage 設定・make.conf 生成
04-system.yml         # timezone/locale/hostname/network/fstab
05-kernel.yml         # カーネルビルド・ZFS モジュール・dracut initramfs
06-bootloader.yml     # GRUB インストール・grub.cfg 生成
07-finalize.yml       # root パスワード・SSH・unmount・reboot
group_vars/target.yml # 全設定値
inventory.ini         # ターゲットホスト定義
extra/                # 拡張 playbook（wayland 等）
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

GRUB は rpool のデータセットから直接カーネルを読み込む：

```
set root=(hd0,gpt3)
linux /ROOT/gentoo@/boot/kernel-<version> root=rpool/ROOT/gentoo boot=zfs
initrd /ROOT/gentoo@/boot/initramfs-<version>.img
```

- `/ROOT/gentoo@/` — GRUB の ZFS ドライバが `rpool/ROOT/gentoo` データセットのルートを参照する記法
- `boot=zfs` — dracut に ZFS root であることを伝える
- initramfs が ZFS モジュールをロードし、rpool を import してから root をマウントする

## 使い方

### 1. 設定を編集

```bash
vi group_vars/target.yml
```

```yaml
# Disk
default_target_disk: sda
default_confirm_destroy: "yes"

# System
default_timezone: Asia/Tokyo
default_hostname: zfs-gentoo

# Portage
use_flags: "zfs -X -wayland -gtk -qt5 -qt6"

# Root password
default_root_password: root
```

### 2. 一括実行

```bash
ansible-playbook -i inventory.ini site.yml -e ansible_ssh_pass=rescue
```

最初に確認プロンプトが出る。`yes` で全工程がノンストップで実行される。
カーネルビルドに 1〜2 時間かかる。

### 3. 個別実行

```bash
ansible-playbook -i inventory.ini 05-kernel.yml -e ansible_ssh_pass=rescue
```

各 playbook は冪等性があり、完了済みの工程はスキップされる。

## 冪等性

| playbook | スキップ条件 |
|----------|-------------|
| 01 | rpool が既に存在する |
| 02 | `/usr/bin/emerge` が存在する |
| 03 | portage tree が存在する、make.conf が同一 |
| 04 | 各設定ファイルが同一 |
| 05 | カーネルイメージ + ZFS モジュール + initramfs が存在する |
| 06 | GRUB が MBR/EFI にインストール済み、grub.cfg が同一 |
| 07 | 毎回実行（パスワード設定・reboot） |

## 注意事項

- `default_confirm_destroy: "yes"` の状態で実行するとディスクが破壊される
- ライブ環境のカーネル config（`/proc/config.gz`）をベースに `localmodconfig` でカーネルを最小化している。異なるライブ環境で再実行すると config が変わりカーネルが再ビルドされる
- `group_vars/target.yml` に root パスワードが平文で保存される。必要に応じて `ansible-vault` で暗号化すること
