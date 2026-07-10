# skillenv

像 pyenv / nvm 管理语言版本一样，按仓库管理 AI Agent 的 skill。

进入目录 → 自动安装 `.skillsrc` 声明的 skill、按需屏蔽全局 skill；离开目录 → 一切恢复原样。同一份声明对 Claude Code、Codex、OpenCode 等所有支持 [Agent Skills 标准](https://code.claude.com/docs/en/skills)的工具生效。

单文件 bash 脚本，唯一依赖是 [direnv](https://direnv.net)。

## 安装（推荐：交给你的 Agent）

把下面这句话发给你的 Claude Code / Codex：

> 请阅读 https://raw.githubusercontent.com/TsichiChang/skillenv/main/README.md，按其中「Agent 安装指引」一节在我的机器上安装 skillenv，完成后执行验证步骤并汇报结果。

或者手动安装：

```bash
curl -fsSL https://raw.githubusercontent.com/TsichiChang/skillenv/main/skillenv -o /usr/local/bin/skillenv && chmod +x /usr/local/bin/skillenv
brew install direnv && echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc && source ~/.zshrc
```

## Agent 安装指引

> 本节写给执行安装的 AI Agent。按顺序执行，每步都有验证方式和失败分支。

**步骤 1 — 选择 bin 目录**：按顺序取第一个存在且可写的目录：`/opt/homebrew/bin`（Apple Silicon）→ `/usr/local/bin` → `~/.local/bin`（若用此项，需确认其在 PATH 中，不在则追加到 shell rc）。下文称 `$BIN`。

**步骤 2 — 安装 skillenv**（单文件，无需 clone）：

```bash
curl -fsSL https://raw.githubusercontent.com/TsichiChang/skillenv/main/skillenv -o "$BIN/skillenv"
chmod +x "$BIN/skillenv"
skillenv help   # 应输出用法说明
```

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
# 管理哪些 agent 的目录（默认 claude codex；agents = 通用 .agents/skills）
agents claude codex opencode

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
