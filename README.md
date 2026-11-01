# DSH-SKILL

DSH 会话里沉淀下来的可复用技能，一个技能一个目录，目录格式与 `~/.dsh/skills/` 一致：
`SKILL.md`（入口，含 front matter）+ `references/`（长文、清单、实测记录）。

## 技能索引

| 技能 | 用途 | 内容 |
|---|---|---|
| [agent-development-handbook](agent-development-handbook/SKILL.md) | 开发流程手册：三件套、git 写操作先问、方案先行、版本号先问、写 → 审 → 定 | 规则正文 + `references/templates.md`（三件套骨架、短方案、提交话术） |
| [swift-app-development](swift-app-development/SKILL.md) | Swift / SwiftUI 的「静默失败」经验，来自 DeviceStatus、SwiftUtils、zoclip、SafaQrcode、KidToys | `references/experience.md`（按病根分类的完整案例） |

## 安装教程

### 装到哪里

DSH 从下面几个根目录发现技能：一个技能 = 一个子目录（`<name>/SKILL.md`）或一个平铺文件（`<name>.md`）。

| 层级 | 路径 | 生效范围 |
|---|---|---|
| 项目 | `<repo>/.dsh/skills/` | 只在该仓库，项目根是最近的含 `.git` 的祖先目录 |
| 项目 | `<repo>/.agents/skills/` | 同上 |
| 用户 | `~/.dsh/skills/` | 所有会话 |
| 用户 | `~/.agents/skills/` | 同上 |

同名技能项目级覆盖用户级；`~/.dsh/skills/.system/` 下的内容会被跳过，可以忽略。

### 方式一：拷贝到用户级（推荐，最不容易出岔子）

```bash
mkdir -p ~/.dsh/skills
cp -R agent-development-handbook ~/.dsh/skills/
cp -R swift-app-development ~/.dsh/skills/
```

目标目录已存在时，`cp -R` 会把源目录**塞进**已有目录里（变成 `agent-development-handbook/agent-development-handbook/`）。重装前先删：

```bash
rm -rf ~/.dsh/skills/agent-development-handbook
cp -R agent-development-handbook ~/.dsh/skills/
```

### 方式二：软链（仓库是唯一源，`git pull` 即生效）

```bash
ln -s "$PWD/agent-development-handbook" ~/.dsh/skills/agent-development-handbook
```

目标名已存在会报 `File exists`。删旧链接用不带 `-r` 的 `rm`：

```bash
rm ~/.dsh/skills/agent-development-handbook     # 只删链接
rm -rf ~/.dsh/skills/agent-development-handbook  # ⚠️ 会顺着链接删掉仓库里的文件
```

### 方式三：装进某个项目

技能可以跟着仓库走，团队里每个人 clone 下来就有：

```bash
cd /path/to/你的项目
mkdir -p .dsh/skills
cp -R /path/to/DSH-SKILL/swift-app-development .dsh/skills/
```

注意别装到没有 `.git` 的目录：项目根会退化成当前工作目录，换个目录打开会话就看不到了。

### 验证装好了

```bash
# 1. 文件在正确位置
ls ~/.dsh/skills/agent-development-handbook/SKILL.md

# 2. frontmatter 的 name 与目录名一致（必须 kebab-case）
grep -m1 '^name:' ~/.dsh/skills/agent-development-handbook/SKILL.md
```

3. 开一个新会话，技能会出现在模型可见的技能目录里，描述命中任务时由 agent 主动载入。
   装好一般不用重启 DSH：发现器监视着技能根目录，新增 / 删除目录会自动刷新，`SKILL.md` 正文改动下次调用时重新读取。没出现就新开会话再看。

### 更新

- 拷贝安装：`git pull` 后重装（先删再拷，见方式一）。
- 软链安装：`git pull` 即生效，无需其他操作。

### 卸载

```bash
rm -rf ~/.dsh/skills/agent-development-handbook    # 拷贝安装
rm      ~/.dsh/skills/agent-development-handbook    # 软链安装，不要带 -r
```

## 排障：装完却看不到技能

按顺序查，多数是前两条：

1. **目录名与 `name` 不一致**，或 `name` 不是 kebab-case（`my_skill`、`MySkill` 都不行）。
2. **层级多了**：只认 `<root>/<name>/SKILL.md`，`<root>/a/b/SKILL.md` 不会被发现。
3. **frontmatter 不对**：文件第一行必须是 `---`；必须有 `name` 和 `description`，缺 `description` 整条被跳过。
4. **被调用策略藏起来了**：`disable-model-invocation: true` 让模型看不到（只能用户手动调用）；`user-invocable: false` 则从用户命令里隐藏。这两个键写成驼峰（`disableModelInvocation`）会导致整条技能被丢弃。
5. **重名**：项目级和用户级有同名技能时，只有项目级那份生效。

## 新增一个技能

1. 建目录 `新技能名/`，`SKILL.md` 的 front matter 至少写 `name`（与目录名一致）和 `description`（写清「什么时候该载入」，不是简介）。
2. 正文只留决策规则和最短路径；长案例、平台实测、模板放 `references/`，在 `SKILL.md` 里按需引用。
3. 更新本 README 的索引表，在 `CHANGELOG.md` 的 `[Unreleased]` 记一条。

## 约定

- 不提交凭据、Cookie、Token、个人路径；references 里引用真实工程时只写相对路径。
- 技能描述会直接影响 agent 是否载入，改 `description` 前先在会话里说明影响。
