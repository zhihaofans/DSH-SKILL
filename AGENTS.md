# AGENTS.md

给 agent 的操作说明。这里只写验证过的事实。

## 仓库是什么

DSH 技能合集：每个子目录是一份可复制的技能，格式与 `~/.dsh/skills/` 相同，拷过去即可被 DSH 载入。

## 目录

```text
<skill-name>/
  SKILL.md          # 入口，front matter: name（= 目录名）+ description（写触发时机）
  references/*.md   # 长文、清单、实测记录，按需加载
README.md           # 技能索引 + 安装方式
TODO.md             # 未完成事项
CHANGELOG.md        # 用户可感知的变化
```

## 常用命令

```bash
# 校验仓库内技能与已安装版本是否一致
for s in */SKILL.md; do diff -r "$(dirname "$s")" "$HOME/.dsh/skills/$(dirname "$s")"; done

# 安装（拷贝）
cp -R <skill-name> ~/.dsh/skills/

# 安装（软链，注意先删掉同名目录）
ln -s "$PWD/<skill-name>" ~/.dsh/skills/<skill-name>
```

## 约束

- 不提交凭据、Cookie、Token、`.env`、个人绝对路径；引用真实工程只写相对路径。
- `SKILL.md` 的 `name` 必须等于目录名；`description` 决定何时载入，改之前先说明影响。
- 长内容进 `references/`，`SKILL.md` 保持一屏内能读完的规则密度。
- 新增技能：建目录 → 写 SKILL.md → 更新 README 索引 → 记 CHANGELOG `[Unreleased]`。

## 提交

Conventional Commits：`feat: / fix: / docs: / chore:` + 一句话。一次提交只做一件事。
