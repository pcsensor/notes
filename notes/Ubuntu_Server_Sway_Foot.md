# Ubuntu Server：Sway + Foot 速记

**目的**：在物理机上全屏使用 Foot + Zsh + Starship，显示 Maple Mono NF CN 和 Nerd Font 图标；不安装完整桌面。

## 1. 安装

```bash
sudo apt install sway foot dbus-user-session fontconfig
fc-match 'Maple Mono NF CN'
```

字体已放在 `/usr/local/share/fonts/truetype/MapleMono/`；新装字体后运行 `fc-cache -f -v`。

## 2. Foot

```bash
mkdir -p ~/.config/foot
micro ~/.config/foot/foot.ini
```

```ini
[main]
font=Maple Mono NF CN:size=18
shell=/usr/bin/zsh
login-shell=yes
pad=12x12
```

## 3. Sway

```bash
mkdir -p ~/.config/sway
cp -n /etc/sway/config ~/.config/sway/config
micro ~/.config/sway/config
```

确认终端设置：

```text
set $term foot
```

在文件末尾添加（不要重复添加）：

```text
xwayland disable
exec foot
for_window [app_id="foot"] fullscreen enable
```

**在物理 TTY 中**测试（不是 SSH）：

```bash
dbus-run-session -- sway
```

进 Foot 后测试 `micro`、↑ 历史记录和 Starship 图标；退出：`swaymsg exit`。

## 4. 登录后自动启动（推荐）

编辑 Zsh 登录配置：

```bash
micro ~/.zprofile
```

末尾加入（这里指定 **tty1**；若实际登录的是 tty3，则改为 `/dev/tty3`）：

```zsh
if [[ "$(tty)" == "/dev/tty1" && -z "$SSH_CONNECTION" && -z "$WAYLAND_DISPLAY" ]]; then
    dbus-run-session -- sway
fi
```

开机后在 tty1 **手动登录一次**，即自动进入 Sway + 全屏 Foot；SSH 不受影响。

## 5. 免登录开机自启（可选，不推荐公共环境）

如果想开机后**不输入密码**就进入 Foot，还需要 tty1 自动登录（任何能接触物理机的人都能进入账户）：

```bash
sudo systemctl edit getty@tty1.service
```

填入：

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin pcsensor --noclear %I $TERM
```

```bash
sudo systemctl daemon-reload
systemctl cat getty@tty1.service
sudo reboot
```

启动链路：开机 → tty1 自动登录 → `.zprofile` → Sway → Foot。

## 6. 恢复 / 排错

- **退出 Sway**：`swaymsg exit`
- **取消登录后自启**：从 `~/.zprofile` 删除上述 `if ... fi`。
- **取消免密码登录**：`sudo systemctl revert getty@tty1.service`（注意：也会撤销该单元的其他自定义覆盖配置）。
- **查启动日志**（本地 TTY 执行）：

```bash
dbus-run-session -- sway -d 2> ~/sway.log
tail -n 100 ~/sway.log
```

**备注**：Sway + Foot、Micro、Zsh 历史记录已实测正常；开机自启需重启验证。出现问题优先用 SSH 或 `Ctrl + Alt + F3` 恢复。
