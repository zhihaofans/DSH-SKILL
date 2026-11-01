# DSH-SKILL

DSH 会话里沉淀下来的可复用技能，一个技能一个目录，目录格式与 `~/.dsh/skills/` 一致：
`SKILL.md`（入口，含 front matter）+ `references/`（长文、清单、实测记录）。

## 技能索引

| 技能 | 用途 | 内容 |
|---|---|---|
| [agent-development-handbook](agent-development-handbook/SKILL.md) | 开发流程手册：三件套、git 写操作先问、方案先行、版本号先问、写 → 审 → 定 | 规则正文 + `references/templates.md`（三件套骨架、短方案、提交话术） |
| [swift-app-development](swift-app-development/SKILL.md) | Swift / SwiftUI 的「静默失败」经验，来自 DeviceStatus、SwiftUtils、zoclip、SafaQrcode、KidToys | `references/experience.md`（按病根分类的完整案例） |

## 安装到本机

技能放在 `~/.dsh/skills/<目录名>/` 才会被 DSH 载入。可以拷贝，也可以软链（改仓库即生效）：

```bash
# 拷贝
cp -R agent-development-handbook ~/.dsh/skills/

# 或软链
ln -s "$PWD/agent-development-handbook" ~/.dsh/skills/agent-development-handbook
```

两份技能都会出现在会话的技能目录里，描述命中任务时由 agent 主动载入。

## 新增一个技能

1. 建目录 `新技能名/`，`SKILL.md` 的 front matter 至少写 `name`（与目录名一致）和 `description`（写清「什么时候该载入」，不是简介）。
2. 正文只留决策规则和最短路径；长案例、平台实测、模板放 `references/`，在 `SKILL.md` 里按需引用。
3. 更新本 README 的索引表，在 `CHANGELOG.md` 的 `[Unreleased]` 记一条。

## 约定

- 不提交凭据、Cookie、Token、个人路径；references 里引用真实工程时只写相对路径。
- 技能描述会直接影响 agent 是否载入，改 `description` 前先在会话里说明影响。
