# dotman 使用文档

dotman 是一个自用的 dotfiles 管理脚本，采用 **Git 仓库保存真实文件 + 本机软链引用** 的模型。它很轻量，只依赖 `git` 和 `python3`。

---

## 核心模型

- 所有被管理的文件/目录的真实副本保存在一个 Git 仓库中（默认 `~/.dotman/dotfiles`）。
- 本机对应位置只保留指向仓库内副本的**符号链接（symlink）**。
- 通过 TOML 文件记录每个设备上应该建立哪些软链。
- 修改映射表或新增文件后会**自动提交并推送**到远程仓库。

```
本机路径              软链指向
~/.zshrc     ──────▶   ~/.dotman/dotfiles/.zshrc
~/.config/nvim  ───▶   ~/.dotman/dotfiles/.config/nvim
```

---

## 默认目录结构

```
~/.dotman/                          # dotman 工作目录
├── dotfiles/                       # dotfiles Git 仓库
│   ├── config.toml                 # 仓库地址 + 当前设备名
│   ├── links.common.toml           # 所有设备共用的软链
│   ├── links.<hostname>.toml      # 当前设备特有的软链
│   └── .zshrc / .config/nvim/ ...  # 真实文件

~/.config/dotman/config.toml        # 本地指针：记录仓库在本机的路径
```

### 可配置项

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `DOTMAN_HOME` | `~/.dotman` | dotman 工作目录 |
| `DOTMAN_REPO` | `~/.dotman/dotfiles` | dotfiles Git 仓库路径 |

### config.toml

```toml
repo_url = "https://github.com/yourname/dotfiles.git"
hostname = "macbook-pro"
```

### links.common.toml

```toml
[links]
".zshrc" = "~/.zshrc"
".config/nvim" = "~/.config/nvim"
".gitconfig" = "~/.gitconfig"
```

### links.<hostname>.toml

```toml
[links]
".hammerspoon" = "~/.hammerspoon"
".config/karabiner" = "~/.config/karabiner"
```

**合并规则**：`links.common.toml` 先加载，设备特定文件覆盖同名目标。设备特定链接优先级更高。

---

## 快速开始

### 方式一：直接运行（推荐）

```bash
dotman
```

如果仓库尚未初始化，会提示你是否立即创建本地仓库。之后进入交互式界面，顶部显示 ASCII logo 和当前软链状态，下方是操作菜单。

### 方式二：从远程仓库初始化

```bash
dotman init https://github.com/yourname/dotfiles.git
```

会把远程仓库 clone 到 `~/.dotman/dotfiles`。

### 交互式界面操作

```
    ____   ____  _______   ____    __  ___
   / __ \ / __ \/ ____/ | / / /   /  |/  /
  / / / // / / / __/ /  |/ / /   / /|_/ /
 / /_/ // /_/ / /___/ /|  / /___/ /  / /
/_____/ \____/_____/_/ |_/_____/_/  /_/

自用 dotfiles 管理器 | 版本 0.2.0

设备：macbook-pro   仓库：/Users/you/.dotman/dotfiles

[i] 设备名：macbook-pro
[i] 仓库：/Users/you/.dotman/dotfiles

.zshrc                           -> ~/.zshrc
  正常
.config/nvim                     -> ~/.config/nvim
  正常

[i] 总计: 2 | 正常: 2 | 异常: 0 | 缺失: 0

操作菜单：
  1. 初始化/克隆远程仓库        dotman init <git-url>
  2. 拉取远程更新               dotman pull
  3. 推送本地改动               dotman push [message]
  4. 添加本机文件到仓库         dotman add <src> [dst]
  5. 移除并恢复文件             dotman remove <dst>
  6. 建立软链                   dotman link <repo-path> <dst>
  7. 解除软链                   dotman unlink <dst>
  8. 检查链接状态               dotman status
  9. 一键修复链接               dotman fix
  10. 综合诊断                  dotman doctor
  0. 退出

请选择操作 [0-10]:
```

### 把本机已有的 dotfile 纳入管理

```bash
# 把 ~/.zshrc 移入仓库，并在原位置建立软链
dotman add ~/.zshrc

# 把 ~/.config/nvim 移入仓库，软链到原位置
dotman add ~/.config/nvim

# 指定不同的目标位置
dotman add ~/my-theme.zsh-theme ~/.oh-my-zsh/custom/themes/my-theme.zsh-theme
```

`add` 会：
1. 把源文件/目录移动到仓库中对应位置。
2. 在原位置创建指向仓库的软链。
3. 把映射写入 `links.<hostname>.toml`。
4. 自动 `git commit` + `git push`。

---

## 命令参考

| 命令 | 说明 |
|------|------|
| `dotman` | 进入交互式界面 |
| `dotman init <git-url>` | 从远程仓库初始化 `~/.dotman/dotfiles` |
| `dotman pull` | 拉取仓库更新 |
| `dotman push [message]` | 提交并推送本地改动，默认消息 `dotman update` |
| `dotman add <src> [dst]` | 把本机文件移入仓库并建立软链；`dst` 省略时等于 `src` |
| `dotman remove <dst>` | 解除软链，把真实文件从仓库拷回原位 |
| `dotman link <repo-path> <dst>` | 为仓库中已存在的路径建立软链（不移动文件） |
| `dotman unlink <dst>` | 解除软链，不删除仓库文件，仅从映射表删除 |
| `dotman status` | 显示所有软链状态 |
| `dotman fix` | 一键修复异常软链 |
| `dotman doctor` | 综合检查仓库和软链状态 |

---

## 多设备管理

dotman 通过 `hostname` 区分设备。

- 设备名优先读取 `config.toml` 中的 `hostname`；未配置时使用 `hostname -s` 的值。
- 通用配置放在 `links.common.toml`。
- 设备特定配置放在 `links.<hostname>.toml`。

例如：

```toml
# links.common.toml
[links]
".zshrc" = "~/.zshrc"
".gitconfig" = "~/.gitconfig"

# links.macbook-pro.toml
[links]
".hammerspoon" = "~/.hammerspoon"

# links.workstation.toml
[links]
".config/i3" = "~/.config/i3"
```

在 `macbook-pro` 上，`dotman status` / `dotman fix` 只会处理 `.zshrc`、`.gitconfig` 和 `.hammerspoon`。

---

## 备份策略

任何可能覆盖或删除本机文件的操作都会先自动备份：

- 备份命名格式：`<原路径>.<YYYYMMDDhhmmss>.bak`
- 触发场景：`add` 覆盖仓库内旧文件、`remove`/`link`/`unlink`/`fix` 覆盖目标位置时。
- 备份文件不会被 dotman 跟踪，也不会被自动清理。

建议定期手动清理旧备份：

```bash
find ~ -name "*.bak" -type f -mtime +30 -delete
```

---

## 依赖

- `git`
- `python3`，并且需要 TOML 解析库：
  - Python 3.11+ 自带 `tomllib`。
  - 旧版本请安装 `tomli`：`pip3 install tomli`

---

## 常见问题

### Q: 仓库不存在时运行 dotman 会怎样？

如果直接运行 `dotman` 且无仓库，会提示你是否立即初始化一个本地空仓库。选择是后，会创建 `~/.dotman/dotfiles` 并提交初始配置。

之后你可以：

```bash
dotman init https://github.com/yourname/dotfiles.git
```

来关联或覆盖到远程仓库。

### Q: 能否管理系统级文件（如 `/etc/hosts`）？

`dotman add` 只支持 `$HOME` 内的路径。对于系统级文件，请手动把它放入仓库，然后使用 `dotman link`：

```bash
sudo cp /etc/hosts ~/.dotman/dotfiles/etc/hosts
dotman link etc/hosts /etc/hosts
```

### Q: 我在 A 设备添加了一个文件，B 设备如何同步？

在 B 设备上：

```bash
dotman pull
dotman fix
```

如果该文件映射在 `links.common.toml` 中，B 设备会自动建立软链。如果只在 A 设备的 `links.<hostname>.toml` 中，B 设备不会自动建立，需要手动添加映射。

### Q: 为什么修改映射表后要重新运行 `fix`？

`links.*.toml` 只是声明了"应该存在哪些软链"。`status` 用于检查，`fix` 用于应用这些声明到本机文件系统。

### Q: 我不想每次都 push，可以只本地提交吗？

当前版本所有会修改仓库文件的操作都会自动 push。如果你需要更细粒度的控制，可以直接进入仓库目录手动使用 `git` 命令。

---

## 设计取舍

- **TOML 而非 JSON/YAML**：人类可读，注释友好，键值对适合路径映射。
- **软链而非复制**：保持单份真实来源，本机编辑即仓库编辑。
- **设备特定文件覆盖 common**：简单明确，避免复杂的条件表达式。
- **自动提交推送**：自用脚本，减少手动步骤，但意味着所有修改操作都会立即同步到远程。
- **交互式界面为默认入口**：无参数启动最符合日常查看状态 + 执行操作的流程。
