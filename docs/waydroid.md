# Waydroid メモ

## ウィンドウモードで起動する

デフォルトの `waydroid show-full-ui` はフルスクリーンで起動する。
ウィンドウモードにするには以下の2つの設定が必要。

### 1. multi_windows を有効化

`/var/lib/waydroid/waydroid.cfg` の `[properties]` セクションに追加：

```ini
[properties]
persist.waydroid.multi_windows = true
```

または起動中のセッションに対して：

```bash
waydroid prop set persist.waydroid.multi_windows true
```

> セッション再起動後に反映される。

### 2. labwc の windowRules でサイズと位置を指定

`~/.config/labwc/rc.xml` に追加：

```xml
<windowRules>
  <windowRule identifier="Waydroid" matchType="exact">
    <action name="SetMaximized" maximized="no"/>
    <action name="SetSize" width="1152" height="648"/>
    <action name="MoveTo" x="0" y="0"/>
  </windowRule>
</windowRules>
```

- `identifier` は大文字の `Waydroid`（app_id に一致させる）
- `SetSize` の値は画面解像度の 60% 程度が目安（1920x1080 の場合 1152x648）
- `MoveTo x="0" y="0"` で左上に配置

### 補足

- `persist.waydroid.width` / `persist.waydroid.height` は Android 内部の解像度設定であり、ウィンドウサイズとは無関係
- app_id の確認には `wlrctl toplevel list` が使える（`gui-apps/wlrctl` を drkhsh-overlay からインストール）

```bash
eselect repository add drkhsh git https://git.drkhsh.at/overlay/
emaint sync -r drkhsh-overlay
echo 'gui-apps/wlrctl **' > /etc/portage/package.accept_keywords/wlrctl
emerge --noreplace '=gui-apps/wlrctl-9999'

XDG_RUNTIME_DIR=/run/user/1000 WAYLAND_DISPLAY=wayland-0 wlrctl toplevel list
```
