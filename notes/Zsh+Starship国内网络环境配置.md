在无法访问 GitHub 的中国大陆 Ubuntu 云服务器上，围绕 **Starship** 构建 Zsh 环境，最干净、极速的落地方式。

---

### 1. 安装基础环境与 APT 核心插件

Ubuntu 的 `universe` 软件源已经内置了 Zsh 的三大核心插件中的两个（语法高亮、自动建议）以及绝大多数现代 CLI 工具，直接通过 `apt` 安装即可：

```bash
# 1. 更新索引并安装 Zsh、核心插件及现代命令行替代工具
sudo apt update
sudo apt install -y zsh git curl wget \
    zsh-autosuggestions \
    zsh-syntax-highlighting \
    fzf zoxide bat fd-find ripgrep

# 2. 尝试安装 eza（Ubuntu 24.04+ 直接支持，22.04 若无此包会自动跳过，稍后可用 Cargo 装）
sudo apt install -y eza 2>/dev/null || true

# 3. 补充 zsh-completions 增强补全库
mkdir -p ~/.zsh/plugins
git clone --depth=1 https://ghfast.top/https://github.com/zsh-users/zsh-completions.git ~/.zsh/plugins/zsh-completions

# 4. 将当前用户的默认 Shell 切换为 Zsh
sudo chsh -s $(which zsh) $USER

```

*验证方式：* 执行 `zsh --version` 能正常输出版本号，且 `~/.zsh/plugins/zsh-completions` 目录已生成。

---

### 2. 在大陆无代理环境下安装 Starship

由于 `starship.rs/install.sh` 官方脚本默认从 GitHub Releases 拉取二进制包，在大陆机器上直接运行会超时。请按你的 Ubuntu 版本选择以下**任意一种**国内安装方式：

#### 方式 A：APT 直接安装（仅限 Ubuntu 24.04 及以上版本）

Ubuntu 24.04 (Noble) 及后续版本的官方源已直接收录 Starship：

```bash
sudo apt install -y starship

```

#### 方式 B：使用 GitHub 国内镜像代下预编译包（全版本通用，最快）

适合 Ubuntu 20.04 / 22.04，无需编译，直接借助国内反向代理拉取官方静态编译好的 musl 二进制文件（10秒内完成）：

```bash
# 自动识别架构 (x86_64 或 aarch64) 并通过国内镜像下载解压至 /usr/local/bin
ARCH=$(uname -m)
curl -fsSL "https://ghfast.top/https://github.com/starship/starship/releases/latest/download/starship-${ARCH}-unknown-linux-musl.tar.gz" | sudo tar -xz -C /usr/local/bin

# 验证是否安装成功
starship --version

```

*(注：如果 `ghfast.top` 偶发波动，可将 URL 中的 `[https://ghfast.top/](https://ghfast.top/)` 替换为 `[https://gh-proxy.com/](https://gh-proxy.com/)`。)*

#### 方式 C：通过字节跳动 Rsproxy 镜像使用 Cargo 编译（稳妥备选）

如果服务器上本来就需要 Rust 环境，或者你想同时编译最新版 `eza`，可配置字节跳动的 `rsproxy.cn` 镜像源直接安装：

```bash
# 1. 使用中科大/字节镜像安装 Rust 工具链
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh -s -- -y
source "$HOME/.cargo/env"

# 2. 配置 Cargo 字节跳动 sparse 镜像源
mkdir -p ~/.cargo
cat << 'EOF' > ~/.cargo/config.toml
[source.crates-io]
replace-with = 'rsproxy-sparse'
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"
[net]
git-fetch-with-cli = true
EOF

# 3. 编译安装 starship（如果前面 apt 没装上 eza，这里可以加上 eza 一起装）
sudo apt install -y build-essential pkg-config libssl-dev
cargo install starship --locked

```

---

### 3. 写入 Starship 专属 `.zshrc` 配置文件

直接执行以下整块命令，生成针对 **Ubuntu 路径 + Starship** 深度优化的 `~/.zshrc`：

```bash
cat << 'EOF' > ~/.zshrc
# ==================== 1. 环境变量与 PATH ====================
export PATH="$HOME/.local/bin:$HOME/.cargo/bin:$PATH"
export EDITOR='vim'

# ==================== 2. 历史记录优化 ====================
HISTFILE=~/.zsh_history
HISTSIZE=50000
SAVEHIST=50000
setopt EXTENDED_HISTORY          # 记录命令执行时间戳
setopt HIST_EXPIRE_DUPS_FIRST    # 历史满时优先清理重复项
setopt HIST_IGNORE_DUPS          # 忽略连续重复的命令
setopt HIST_IGNORE_ALL_DUPS      # 新命令重复时删除旧记录
setopt HIST_IGNORE_SPACE         # 忽略以空格开头的命令（防敏感信息记录）
setopt INC_APPEND_HISTORY        # 实时追加历史而非等终端退出
setopt SHARE_HISTORY             # 多 SSH 会话之间实时共享历史

# ==================== 3. 补全系统初始化 ====================
# 加载 Gitee 克隆的 zsh-completions 补全库
[[ -d ~/.zsh/plugins/zsh-completions/src ]] && fpath=(~/.zsh/plugins/zsh-completions/src $fpath)

autoload -Uz compinit
# 每天仅做一次完整安全检查，其余时间直接读缓存，确保启动耗时 < 10ms
if [[ -n ${ZDOTDIR:-$HOME}/.zcompdump(#qN.mh+24) ]]; then
  compinit
else
  compinit -C
fi

# 补全菜单交互优化：方向键选择、不区分大小写、彩色补全列表
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}' 'r:|[._-]=* r:|=*' 'l:|=* r:|=*'
zstyle ':completion:*' list-colors "${(s.:.)LS_COLORS}"
zstyle ':completion:*:descriptions' format '%F{yellow}-- %d --%f'

# ==================== 4. 键盘快捷键绑定 ====================
bindkey -e
bindkey '^[[A' up-line-or-search                  # 上方向键：基于当前已输入前缀搜索历史
bindkey '^[[B' down-line-or-search                # 下方向键
bindkey '^[[1;5C' forward-word                    # Ctrl + 右方向键：按词向右跳转
bindkey '^[[1;5D' backward-word                   # Ctrl + 左方向键：按词向左跳转
bindkey '^[[3~' delete-char                       # Delete 键正常向后删除
bindkey '^[[H' beginning-of-line                  # Home 键跳到行首
bindkey '^[[F' end-of-line                        # End 键跳到行尾

# ==================== 5. 现代 CLI 工具别名与集成 ====================
# 修正 Ubuntu APT 包名防冲突带来的命令名差异
command -v batcat &> /dev/null && alias bat='batcat'
command -v fdfind &> /dev/null && alias fd='fdfind'

# 目录列表：若安装了 eza 则启用图标和树状视图，否则回退至彩色 ls
if command -v eza &> /dev/null; then
    alias ls='eza --icons=auto --group-directories-first'
    alias ll='eza -lh --icons=auto --group-directories-first --time-style=long-iso'
    alias la='eza -lah --icons=auto --group-directories-first --time-style=long-iso'
    alias lt='eza --tree --level=2 --icons=auto'
else
    alias ls='ls --color=auto'
    alias ll='ls -lh --color=auto'
    alias la='ls -lAh --color=auto'
fi

alias grep='grep --color=auto'
alias df='df -h'
alias du='du -h'
alias free='free -m'

# FZF 模糊搜索集成 (Ctrl+R 交互式历史搜索, Ctrl+T 快速查找文件)
[[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ]] && source /usr/share/doc/fzf/examples/key-bindings.zsh
[[ -f /usr/share/doc/fzf/examples/completion.zsh ]] && source /usr/share/doc/fzf/examples/completion.zsh

# Zoxide 智能目录跳转 (使用 z <目录关键词> 替代 cd)
if command -v zoxide &> /dev/null; then
    eval "$(zoxide init zsh)"
fi

# ==================== 6. 加载 Zsh 自动建议插件 ====================
if [[ -f /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh ]]; then
    source /usr/share/zsh-autosuggestions/zsh-autosuggestions.zsh
    ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE='fg=244'
    ZSH_AUTOSUGGEST_STRATEGY=(history completion)
fi

# ==================== 7. 启动 Starship 提示符 ====================
if command -v starship &> /dev/null; then
    export STARSHIP_CONFIG="$HOME/.config/starship.toml"
    eval "$(starship init zsh)"
fi

# ==================== 8. 加载语法高亮 (必须在文件最底部) ====================
if [[ -f /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh ]]; then
    source /usr/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
fi
EOF

```

---

### 4. 配置云端服务器专用 `starship.toml`

在云服务器上，默认配置有时不会常驻显示主机名（Hostname）或本地 IP，且在大仓库中扫描 Git 状态容易引起卡顿。下面这份配置专为**远程 SSH 服务器**调优（紧凑单行/双行可选、常驻显示 `用户名@服务器名`、显示命令耗时与后台任务、关闭缓慢的云平台自动探测）：

```bash
mkdir -p ~/.config
cat << 'EOF' > ~/.config/starship.toml
# 取消每次命令之间的额外空行，保持终端紧凑（如需空行改为 true）
add_newline = false
# 调高命令超时阈值，防止云服务器高负载或大 Git 仓库下报超时警告
command_timeout = 1000

# 自定义提示符左侧布局：用户名@主机名 -> 目录 -> Git状态 -> 命令耗时 -> 提示符
format = """
$username\
$hostname\
$directory\
$git_branch\
$git_status\
$cmd_duration\
$jobs\
$character"""

[username]
style_user = "bold cyan"
style_root = "bold red"
format = "[$user]($style)"
show_always = true

[hostname]
ssh_only = false
style = "bold green"
format = "@[$hostname]($style) "

[directory]
style = "bold blue"
truncation_length = 4
truncate_to_repo = false
read_only = " 󰌾"

[git_branch]
symbol = " "
style = "bold purple"
format = "on [$symbol$branch]($style) "

[git_status]
style = "bold yellow"
format = '([\[$all_status$ahead_behind\]]($style) )'

[cmd_duration]
min_time = 2000
style = "bold yellow"
format = "took [$duration]($style) "

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"
vimcmd_symbol = "[❮](bold green)"
EOF

```

---

### 5. 立即应用并测试性能

执行以下命令重载进入全新的 Zsh + Starship 环境：

```bash
exec zsh

```

*验证与日常使用提示：*

1. **测速检查**：可以在终端中运行 `time (zsh -i -c exit)`，正常情况下总耗时应在 **0.01s ~ 0.03s** 之间。
2. **快捷键验证**：按 `Ctrl + R` 可唤出 `fzf` 全屏历史命令模糊搜索；输入过一次的命令会以灰色字体自动浮现，按 `→`（右方向键）即可直接补全；使用 `bat <文件名>` 可带行号和语法高亮查看配置文件。