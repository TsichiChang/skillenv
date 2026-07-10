# skillenv

像 pyenv / nvm 管理语言版本一样，按仓库管理 AI Agent 的 skill。

进入目录 → 自动安装 `.skillsrc` 声明的 skill、按需屏蔽全局 skill；离开目录 → 一切恢复原样。同一份声明对 Claude Code、Codex、OpenCode 等所有支持 [Agent Skills 标准](https://code.claude.com/docs/en/skills)的工具生效。

单文件 bash 脚本，唯一依赖是 [direnv](https://direnv.net)。

## 安装（推荐：交给你的 Agent）

把下面这句话发给你的 Claude Code / Codex：

> 请阅读 https://raw.githubusercontent.com/TsichiChang/skillenv/main/README.md，按其中「Agent 安装指引」一节在我的机器上安装 skillenv，完成后执行验证步骤并汇报结果。

安装完成后，若想让 Agent 把机器上已有的 skill 也管理起来，再发：

> 按照 skillenv README 中「Agent 接管指引」一节，盘点我本地已安装的所有 skill，给出分类建议，经我确认后完成迁移和各仓库配置。

或者手动安装：

```bash
git clone https://github.com/TsichiChang/skillenv.git ~/.skillenv/repo
ln -s ~/.skillenv/repo/skillenv /usr/local/bin/skillenv
brew install direnv && echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc && source ~/.zshrc
```

之后升级只需 `skillenv update`（等价于在安装仓库里 `git pull`，symlink 安装下即时生效）。

## Agent 安装指引

> 本节写给执行安装的 AI Agent。按顺序执行，每步都有验证方式和失败分支。

**步骤 1 — 选择 bin 目录**：按顺序取第一个存在且可写的目录：`/opt/homebrew/bin`（Apple Silicon）→ `/usr/local/bin` → `~/.local/bin`（若用此项，需确认其在 PATH 中，不在则追加到 shell rc）。下文称 `$BIN`。

**步骤 2 — 安装 skillenv**（git clone + symlink，保证后续可用 `skillenv update` 升级）：

```bash
git clone https://github.com/TsichiChang/skillenv.git ~/.skillenv/repo
ln -s ~/.skillenv/repo/skillenv "$BIN/skillenv"
skillenv help   # 应输出用法说明
```

若 `~/.skillenv/repo` 已存在，改为 `git -C ~/.skillenv/repo pull` 更新后重建 symlink。若环境无法 git clone GitHub，才退化为 curl 下载单文件（此方式升级需重新下载）。

**步骤 3 — 安装 direnv**：先 `command -v direnv` 检查，已有则跳过。首选 `brew install direnv`；若 brew 不可用、报错或卡住超过 2 分钟（公司网络访问 formulae.brew.sh 可能极慢），改用二进制直装：

```bash
# macOS Apple Silicon 用 darwin-arm64；Intel 用 darwin-amd64；Linux 用 linux-amd64
curl -fsSL -o "$BIN/direnv" https://github.com/direnv/direnv/releases/latest/download/direnv.darwin-arm64
chmod +x "$BIN/direnv"
direnv version   # 应输出版本号
```

**步骤 4 — 挂 shell hook**：检测用户 shell（`echo $SHELL`），检查对应 rc 文件中是否已有 `direnv hook`，没有则追加（zsh → `~/.zshrc` 加 `eval "$(direnv hook zsh)"`；bash → `~/.bashrc` 加 `eval "$(direnv hook bash)"`；fish → `~/.config/fish/config.fish` 加 `direnv hook fish | source`）。此步骤修改用户 shell 配置，执行前告知用户。

**步骤 5 — 验证安装**：

```bash
command -v skillenv && command -v direnv && grep -c "direnv hook" ~/.zshrc
```

三项都有输出即安装完成。提醒用户：需要开新终端（或 `source` rc 文件）hook 才生效。

**步骤 6 — 在仓库启用（可选，用户要求时执行）**：

```bash
cd <目标仓库>
skillenv init        # 生成 .skillsrc 模板 + .envrc
# 编辑 .skillsrc 后：
skillenv allow && direnv allow
```

若仓库已带 `.skillsrc`（队友共享的仓库），跳过 init，直接进入下一条。

**Agent 必须遵守的安全规则**：`skillenv allow` 和 `direnv allow` 是信任门。执行前必须把 `.skillsrc`（和非模板生成的 `.envrc`）的完整内容展示给用户并获得确认——尤其是仓库自带、来源不明的清单。不要静默 allow。

## Agent 接管指引：盘点并管理已有 skill

> 本节写给 AI Agent：用户要求"把本地已安装的 skill 用 skillenv 管理起来"时，按以下流程执行。

**步骤 1 — 盘点**：在用户的常用仓库目录下运行 `skillenv scan`，得到全局（claude/codex/agents/opencode）与项目级 skill 的完整清单及各自的 description；另读取 `~/.claude/plugins/installed_plugins.json` 了解插件及其 scope。对 description 缺失或截断的 skill，读它的 `SKILL.md` 补全理解。

**步骤 2 — 分类并请用户确认**：把每个 skill 归入三类，以表格形式向用户呈现建议，**必须经用户确认后才能动手**：

- **通用 → 保留全局**：任何仓库都可能用到的（文档处理、通用工作流类）；
- **领域性 → 移入中央库，按仓库启用**：绑定某业务域/某类项目的（判断依据：description 中出现特定业务、框架、团队名）。中央库默认建议 `~/.skillenv/store/`，用户要团队共享则用 git 仓库；
- **已就位 → 不动**：已在各仓库 `.claude/skills/` 等目录的手写项目级 skill。

**步骤 3 — 迁移（硬性规则）**：

- **只 `mv` 不 `rm`**：领域性 skill 移入中央库；重复副本、被淘汰的目录一律移到 `~/.skillenv/backup/`，永不删除；
- Claude 插件 skillenv 管不了：需要按仓库关闭时，写该仓库 `.claude/settings.local.json` 的 `enabledPlugins`（已有文件只合并不覆盖）；
- 含公司内部信息的 skill 绝不能提交/推送到公开仓库。

**步骤 4 — 铺仓库配置**：为用户指定的每个仓库生成 `.skillsrc`（领域 skill 用 `skill <中央库路径>` 启用；要屏蔽的全局 skill 用 `disable`；需要完全纯净则 `isolation strict`）和 `.envrc`。含机器绝对路径的 `.skillsrc` 不要提交——git 仓库中加入 `.git/info/exclude`。

**步骤 5 — 信任与激活**：逐仓库向用户展示 `.skillsrc` 内容，确认后执行 `skillenv allow` 与 `direnv allow`，再运行 `skillenv activate` 完成首次物化。

**步骤 6 — 验证并汇报**：抽查一个启用仓库（skill 已物化到 `.claude/skills/` 等目录）和一个屏蔽仓库（`direnv export zsh` 输出 `CLAUDE_CONFIG_DIR`/`CODEX_HOME`，shadow 的 `skills/` 内容符合预期），然后向用户汇报最终布局：全局剩什么、中央库有什么、每个仓库启用/屏蔽了什么、备份在哪。

## 原理

```
.skillsrc（提交进 git，团队共享）
    │  进目录时 direnv 触发 skillenv activate
    ├─► 物化 skill 到 .claude/skills/ .codex/skills/ …（像 node_modules，本地生成）
    └─► 构建 shadow 配置目录并导出 CLAUDE_CONFIG_DIR / CODEX_HOME
          ~/.skillenv/shadow/<repo>/claude/
          ├── settings.json → ~/.claude/settings.json   (登录态、配置全部 symlink 回真实目录)
          └── skills/       ← 唯一受控的部分：strict=空；merge=全局 skill 减去 disable 的
```

隔离是**进程级**的（只影响从该 shell 启动的 agent），不改动任何全局文件——其他终端、其他仓库完全不受影响，离开目录时 direnv 自动 unset，环境原样恢复。

## .skillsrc 参考

```bash
# 管理哪些 agent：auto = 检测本机装了哪些（推荐——后续新装 agent 进目录即自动纳管）
# 也可显式指定：claude / codex / opencode / agents（通用 .agents/skills）；默认 claude codex
agents auto

# merge（默认）：全局 skill + 仓库 skill 叠加
# strict：屏蔽所有全局 skill，只保留 manifest 声明的
isolation strict

# skill 来源：github 仓库子目录（@ref 为分支或 tag），或本地路径
skill github:anthropics/skills/document-skills/pdf@main
skill ./skills/my-local-skill

# merge 模式下屏蔽某个全局 skill（按目录名）
disable some-global-skill
```

## 命令

| 命令 | 作用 |
|---|---|
| `skillenv init` | 在当前仓库生成 `.skillsrc` + `.envrc` |
| `skillenv allow` | 信任当前 `.skillsrc`（哈希记录在 `~/.skillenv/trust/`） |
| `skillenv activate` | 同步 skill 并输出 env export（由 `.envrc` 调用，也可手动跑） |
| `skillenv status` | 查看信任状态、生效模式与当前环境 |
| `skillenv scan` | 盘点本机所有已安装的 skill（全局 + 当前仓库项目级，含托管标记与描述） |
| `skillenv prompt` | 输出提示符用的紧凑状态（见下节），无 `.skillsrc` 时输出为空 |
| `skillenv update` | 升级 skillenv 本体（git 安装则 `git pull`，curl 安装则重新下载） |

## 静音与提示符集成

默认 cd 时 direnv 会打印 `loading` / `export +VAR` 行。想完全静音、把状态改放进提示符：

**1. 静音 direnv**——写 `~/.config/direnv/direnv.toml`：

```toml
[global]
hide_env_diff = true
log_filter = "^$"
```

skillenv 的「.skillsrc 未信任」安全警告走 `.envrc` 的 stderr，不受此过滤影响，仍会正常显示。

**2. 提示符显示状态**——`skillenv prompt` 输出形如 `merge -4`（merge 模式、屏蔽 4 个全局 skill）、`strict +2`（strict 模式、声明 2 个 skill）、`!untrusted`（清单待确认）。starship 用户在 `~/.config/starship.toml` 的 `format` 中加入 `${custom.skillenv}`，并追加：

```toml
[custom.skillenv]
command = "skillenv prompt"
when = "test -f .skillsrc"
symbol = "⛭"
style = "bold cyan"
format = "[$symbol $output]($style) "
```

powerlevel10k 用户在 `~/.p10k.zsh` 中加 custom segment：

```zsh
function prompt_skillenv() {
  local out
  out=$(skillenv prompt 2>/dev/null) || return
  [[ -n $out ]] || return
  p10k segment -f cyan -i '⛭' -t "$out"
}
# 再把 skillenv 加进 POWERLEVEL9K_LEFT_PROMPT_ELEMENTS 或 RIGHT_PROMPT_ELEMENTS
```

其他框架同理：任何能执行命令的 prompt 段（zsh `precmd` 拼 `RPROMPT`、oh-my-posh custom segment）调 `skillenv prompt` 即可，单次执行约 70ms。

输出状态一览：空（无 `.skillsrc`）、`merge -4`（merge 模式、屏蔽 4 个）、`strict +2`（strict 模式、声明 2 个）、`!untrusted`（清单变更待 `skillenv allow` 确认）。注意 prompt 反映的是 skillenv 的信任门状态；`.envrc` 未 `direnv allow` 时环境不会加载，但 prompt 仍按 `.skillsrc` 显示。

## 安全模型

自动安装/自动改 agent 配置根目录是标准的供应链攻击面（Codex 曾因 `.env` 重定向 `CODEX_HOME` 出过 RCE）。skillenv 的对策：

1. **双重信任门**：`.envrc` 由 `direnv allow` 把关；`.skillsrc` 内容哈希由 `skillenv allow` 把关——manifest 任何变更都会拒绝执行，直到你审阅后重新 allow。
2. shadow 目录只生成在 `~/.skillenv/` 下，绝不使用仓库内路径。
3. 物化的 skill 目录带 `.managed-by-skillenv` 标记，skillenv 只删除/覆盖自己管理的目录，手工放置的 skill 永远不碰。

## 已知限制

- **OpenCode 的全局 skill 无法屏蔽**：它硬编码读取 `~/.claude/skills` 和 `~/.agents/skills`，配置目录重定向拦不住；仓库级 skill 不受影响。
- **Claude Code 的插件 skill 不被 strict 屏蔽**（shadow 会保留 plugins 配置以免破坏登录态等功能）；按仓库关插件请用 `.claude/settings.json` 的 `enabledPlugins`。
- Claude Code 的 **VS Code 扩展不读取 `CLAUDE_CONFIG_DIR`**，隔离只对终端启动的 CLI 生效。
- `@ref` 仅支持分支/标签（`git clone --branch`），不支持 commit SHA；skill 路径不能含空格；远程源暂只支持 GitHub。
- 已在运行的 agent session 不受进出目录影响（环境变量在进程启动时读取——这也是符合预期的语义）。
