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
  extra-cli-tools.yml      # CLI ツール（sudo, bash-completion, bind-tools 等）
  extra-account.yml        # sudo 設定・ユーザー作成
  extra-sound.yml          # サウンド（pipewire + wireplumber）
  extra-bluetooth.yml      # Bluetooth（bluez + blueman、ハード検出付き）
  extra-wayland.yml        # Wayland デスクトップ（labwc + waybar + foot + wofi）
  extra-ime.yml            # IME（fcitx5 + SKK、gentoo-zh overlay）
  extra-gui-tools.yml      # GUI ツール（Google Chrome, VSCode, yazi、GURU overlay）
  extra-remote-desktop.yml # リモートデスクトップ（wayvnc + noVNC + Apache HTTPS + OTP）
  extra-waydroid.yml       # Android コンテナ（Waydroid、GURU overlay）
  extra-waydroid-arm.yml   # ARM translation セットアップスクリプト配置（libndk）

group_vars/target.yml      # 共通設定値
host_vars/<IP>.yml         # ホスト固有設定（disk, hostname 等）
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
192.168.77.11 ansible_user=root
192.168.77.26 ansible_user=root
```

ホスト固有の設定（ディスク名・ホスト名）は `host_vars/<IP>.yml` に記述する：

```bash
vi host_vars/192.168.77.26.yml
```

```yaml
default_hostname: zfs-gentoo-desktop
default_target_disk: sda
second_disk: "sdb"
ccache_dir: "/mnt/sdb/ccache"
```

共通設定は `group_vars/target.yml` に記述する：

```bash
vi group_vars/target.yml
```

```yaml
# Disk
default_target_disk: sda
second_disk: ""
ccache_dir: ""

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

# WiFi (optional)
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

カーネルビルドに 1〜2 時間かかる（ccache 有効時は短縮）。

### 3. 拡張セットアップ（reboot 後）

```bash
ansible-playbook -i inventory.ini site-extra.yml --ask-pass
```

実行順序：

1. portage を git sync に切り替え・eix インストール
2. システムサービス（cronie）
3. CLI ツール（sudo, bash-completion, bind-tools 等）
4. ユーザーアカウント・sudo 設定
5. サウンド（pipewire + wireplumber）
6. Wayland デスクトップ（labwc + waybar + foot + wofi）
7. IME（fcitx5 + SKK、gentoo-zh overlay）
8. GUI ツール（Google Chrome, VSCode, yazi、GURU overlay）
9. リモートデスクトップ（wayvnc + noVNC）
10. Android コンテナ（Waydroid、GURU overlay）

### 4. 個別実行

```bash
ansible-playbook -i inventory.ini base/base-05-kernel.yml -e ansible_ssh_pass=rescue
ansible-playbook -i inventory.ini extra/extra-wayland.yml --ask-pass
ansible-playbook -i inventory.ini extra/extra-waydroid.yml --ask-pass
```

## ccache

- `FEATURES="ccache"` を make.conf に設定済み
- デフォルトは `/var/cache/ccache`（max 20G）
- `second_disk` を設定した場合は `ccache_dir` を合わせて指定すると2台目ディスク上に配置できる

```yaml
second_disk: "sdb"
ccache_dir: "/mnt/sdb/ccache"
```

## Bluetooth

- `extra-bluetooth.yml` 実行時にまず BT ハードウェアを検出（`lspci` / `lsusb` / `rfkill`）
- ハードが見つからない場合はメッセージを出して後続タスクをすべてスキップ（playbook は正常終了）
- ハードが検出された場合: `bluez` + `blueman` をインストールし `bluetooth.service` を有効化
- `blueman-applet` が labwc autostart で常駐し、waybar トレイにアイコンが表示される
- クリックで `blueman-manager` が起動（Wayland ネイティブ、XWayland 不要）
- 初回は rfkill でブロックされている場合があるため `rfkill unblock bluetooth` または blueman-manager から有効化すること

## WiFi

- WiFi NIC（`wlan*`, `wlp*`）が検出された場合のみ `iwd` をインストール・設定
- `wifi_ssid` と `wifi_password` を設定すると起動時に自動接続
- WiFi NIC が存在する場合、カーネルの crypto オプション（`CONFIG_CRYPTO_*`）を有効化してビルドする
- パスワードは `ansible-vault` で暗号化推奨：

```bash
ansible-vault encrypt_string 'your_ssid' --name 'wifi_ssid'
ansible-vault encrypt_string 'your_password' --name 'wifi_password'
```

## リモートデスクトップ

- `extra-remote-desktop.yml` で wayvnc + noVNC + Apache（HTTPS リバースプロキシ + OTP 認証）をインストール
- labwc autostart で `wayvnc` が自動起動し、`novnc.service` が `127.0.0.1:6080` でブリッジを提供
- Apache が HTTPS(443) で公開し、OTP 認証（TOTP）を前段で実施する
- ブラウザから `https://<host>/` でアクセス（自己署名証明書のため初回警告あり）
- noVNC は `127.0.0.1:6080` にバインドされ、外部から直接アクセス不可

### OTP ユーザー登録

playbook 実行後、ターゲットホストで以下を実行する：

```bash
novnc-otp-register <username>
```

出力された QR コード URL をブラウザで開き、Google Authenticator 等に登録する。

```bash
# 登録済みユーザー確認
cat /var/www/otp/users
```

登録後、ブラウザから `https://<host>/` にアクセスし、ユーザー名と 6 桁のワンタイムパスワードを入力する。認証通過後に noVNC 画面が表示される。

- OTP セッションは 8 時間有効（`OTPAuthMaxLinger 28800`）
- 証明書を Let's Encrypt 等に差し替える場合は `/etc/ssl/apache2/server.{crt,key}` を置き換えて `systemctl restart apache2`

### ポートバインド確認

```bash
ss -ltpn | grep -E ':(443|6080|5900)'
```

正常な状態：

```
0.0.0.0:443    ← Apache（外部公開、HTTPS + OTP）
127.0.0.1:6080 ← novnc（ローカルのみ）
127.0.0.1:5900 ← wayvnc（ローカルのみ）
```

`6080` や `5900` が `0.0.0.0` にバインドされている場合は外部から直接アクセス可能になるため危険。`systemctl restart novnc` で修正する。

## Waydroid

- `extra-waydroid.yml` で Android コンテナ環境をインストール（GURU overlay 使用）
- カーネルに `CONFIG_ANDROID_BINDER_IPC=y` / `CONFIG_ANDROID_BINDERFS=y` が必要
  - `base-05-kernel.yml` で自動的に有効化される
  - binder が未設定のカーネルが検出された場合、自動的にカーネルを再ビルドする
- インストール後に初期化が必要：

```bash
waydroid init
```

- 起動は `waydroid show-full-ui` またはアプリランチャーから
- ウィンドウモードで使う場合は、初期化後にセッション起動中に以下を実行：

```bash
waydroid prop set persist.waydroid.multi_windows true
waydroid prop set persist.waydroid.width 1080
waydroid prop set persist.waydroid.height 720
```

### ARM アプリを動かす（libndk ARM translation）

x86_64 ホスト上で ARM アプリを動かすには ARM トランスレーションレイヤーが必要。
`extra-waydroid-arm.yml` で `/usr/local/bin/waydroid-arm-setup` スクリプトを配置する。

前提条件：
- `waydroid init` 済み
- `waydroid session start` でコンテナが RUNNING 状態

スクリプト実行：

```bash
sudo waydroid-arm-setup
```

実行内容：
1. 前提条件チェック（init 済み・RUNNING）
2. libndk インストール（`/opt/waydroid_script` を使用）
3. session 再起動
4. ARM translation 有効確認（`ro.product.cpu.abilist` に `armeabi` が含まれれば OK）

## 冪等性

| playbook | スキップ条件 |
|----------|-------------|
| base-01 | rpool が既に存在し、ユーザーが再作成を拒否した場合 |
| base-02 | `/usr/bin/emerge` が存在する |
| base-03 | portage tree が存在する、make.conf が同一 |
| base-04 | 各設定ファイルが同一 |
| base-05 | カーネルイメージ + ZFS モジュール + initramfs が存在し、binder が有効 |
| base-06 | GRUB が MBR/EFI にインストール済み、grub.cfg が同一 |
| base-07 | 毎回実行（パスワード設定）。reboot は確認後のみ |

## rescue 環境から手動 chroot する場合

`zfs mount` はデフォルトで mountpoint プロパティ通りにマウントする。`rpool/ROOT/gentoo` の mountpoint は `/` なので、そのままマウントすると rescue の `/proc` `/sys` `/dev` を上書きし、systemd が ABRT でクラッシュ・freeze する。

必ず mountpoint を `/mnt/gentoo` に変えてから mount すること：

```bash
zpool import -f -N rpool
zfs set mountpoint=/mnt/gentoo rpool/ROOT/gentoo
zfs set mountpoint=/mnt/gentoo/home rpool/home
zfs mount rpool/ROOT/gentoo
zfs mount rpool/home

mount --rbind /dev  /mnt/gentoo/dev
mount --rbind /proc /mnt/gentoo/proc
mount --rbind /sys  /mnt/gentoo/sys

chroot /mnt/gentoo /bin/bash
```

作業後は mountpoint を元に戻す：

```bash
zfs set mountpoint=/ rpool/ROOT/gentoo
zfs set mountpoint=/home rpool/home
```

## 注意事項

- rpool が存在しないディスクに対しては確認なしでパーティションが作成される
- reboot を拒否した場合、ライブ環境の bind mount はそのまま維持される（再実行可能）
- ライブ環境のカーネル config（`/proc/config.gz`）をベースに `localmodconfig` でカーネルを最小化している。異なるライブ環境で再実行すると config が変わりカーネルが再ビルドされる
- `group_vars/target.yml` に root パスワードとユーザーパスワードが平文で保存される。必要に応じて `ansible-vault` で暗号化すること
- `extra-ime.yml` は fcitx5 + SKK を gentoo-zh overlay からインストールする
- `extra-gui-tools.yml` および `extra-waydroid.yml` は GURU overlay を使用する。初回は overlay の sync が走る
