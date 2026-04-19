# tools

起動済みの Gentoo システムに対して実行するメンテナンス用 playbook。

`base/` や `extra/` と異なり、rescue 環境や chroot は不要。SSH でログインできる状態であれば実行可能。

## 一覧

| playbook | 説明 |
|----------|------|
| `update-kernel.yml` | カーネルアップデート |

## update-kernel.yml

### 概要

- `eix-update` でパッケージDBを更新し、現行より新しい `gentoo-sources` を番号付きで表示
- 選択したバージョンをビルドし、ZFS モジュール・initramfs・grub.cfg を更新して reboot

### 実行

```bash
# 対象全ホスト
ansible-playbook -i inventory.ini tools/update-kernel.yml --ask-pass

# 特定ホストのみ
ansible-playbook -i inventory.ini tools/update-kernel.yml --ask-pass --limit 192.168.77.221

# バージョンを事前指定（選択プロンプトをスキップ）
ansible-playbook -i inventory.ini tools/update-kernel.yml --ask-pass -e kernel_target_version=6.18.22
```

### 変数

| 変数 | デフォルト | 説明 |
|------|-----------|------|
| `kernel_target_version` | `""` | 対象バージョン。空の場合は対話選択 |
| `grub_keep_generations` | `2` | grub.cfg に残す世代数 |

### 動作フロー

1. `eix-update` 実行（失敗時はエラー終了）
2. 現行より新しいバージョンを一覧表示（stable / ~amd64 ラベル付き）
3. 番号で選択
4. `~amd64` 版を選択した場合は `package.accept_keywords` に自動追加
5. kernel sources を `emerge`
6. `/proc/config.gz` をベースに `olddefconfig` でビルド
7. `@module-rebuild` で ZFS モジュールを再ビルド
8. `dracut` で initramfs 生成（zpool.cache 込み）
9. `/boot` をスキャンして grub.cfg を再構築（新しい順に `grub_keep_generations` 世代）
10. kernel / initramfs / zfs.ko / hostid をレポート
11. reboot 確認

### 注意事項

- **起動済みの Gentoo 上で実行すること**（rescue 環境からは実行しない）
- grub.cfg は `/boot` の実態から毎回再構築される。古い kernel ファイルを `/boot` から削除すれば自動的にエントリも消える
- `eix` がインストールされていない場合は事前に `emerge app-portage/eix` を実行すること
