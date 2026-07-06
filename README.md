# Offerdao Skills

[Offer岛](https://offerdao.ai) 的官方 Agent Skill 集合，让 AI agent（Claude Code、Cursor、Codex、Gemini CLI 等）直接调用 Offer岛开放 API。

当前包含的 skill：

| Skill | 说明 |
|---|---|
| [`offerdao`](skills/offerdao/SKILL.md) | AI 方向求职与行业情报的一站式数据源：搜索 & 发布岗位、查 AI 公司目录、找面经攻略、看每日行业资讯。 |

## 能力速览

| 能力 | 说明 | API key |
|---|---|---|
| 搜索岗位 | 按公司 / 岗位 / 地点 / 工作类型 / 时间等筛选已审核岗位 | 需要 |
| 岛上精选 | 随机推荐几条管理员精选的优质岗位 | 需要 |
| 发布岗位 | 把 JD 原文或帖子链接（小红书 / 公众号等）解析成结构化岗位提交，审核通过后公开 | 需要（个人 key） |
| 公司目录 | AI 公司的简介 / 团队 / 融资 / 产品 / 在招岗位 | 无需 |
| 面经攻略 | 精选面经与求职攻略 | 无需 |
| 行业资讯 | 每日 AI 资讯，按日期和类型（资讯 / 产品 / 模型 / 播客 / 论文 / 博文）查询 | 无需 |

完整参数、示例与注意事项见 [`skills/offerdao/SKILL.md`](skills/offerdao/SKILL.md)。

## 安装

以下三种方式任选其一。安装后该 skill 会出现在 agent 的 `skills/` 目录，被自动识别。

### 方式一：`npx skills add`（推荐）

用 [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI 一键安装，无需手动拷贝：

```bash
# 安装到当前项目（默认，写入 ./.claude/skills/）
npx skills add offerdaoai/skills

# 安装到全局（写入 ~/.claude/skills/，所有项目可用）
npx skills add offerdaoai/skills -g
```

仓库里有多个 skill 时，可只挑选 `offerdao`：

```bash
npx skills add offerdaoai/skills --skill offerdao
```

也可以直接指向子目录：

```bash
npx skills add https://github.com/offerdaoai/skills/tree/main/skills/offerdao
```

常用管理命令：`npx skills list` 查看已装、`npx skills update` 更新、`npx skills remove offerdao` 卸载。

### 方式二：下载到本地再 `cp`

适合内网 / 离线，或想手动放置的场景。先把仓库拉下来，再把 `offerdao` 目录拷进 agent 的 skills 目录：

```bash
git clone https://github.com/offerdaoai/skills.git
cd skills

# 全局安装（所有项目可用）
mkdir -p ~/.claude/skills
cp -r skills/offerdao ~/.claude/skills/

# 或仅安装到当前项目
mkdir -p .claude/skills
cp -r skills/offerdao .claude/skills/
```

完成后应存在 `~/.claude/skills/offerdao/SKILL.md`（或项目下的 `.claude/skills/offerdao/SKILL.md`）。

> 其他 agent 的 skills 目录路径不同（如 Cursor / Codex 为各自的 `.../skills/`），按对应工具的约定放置即可。

### 方式三：让 agent 自己安装

直接对你的 AI agent 说（把下面这句发给它）：

> 安装 offerdao skill，源地址：https://github.com/offerdaoai/skills/tree/main/skills/offerdao 。安装完成后验证它能正常加载。

agent 会自动拉取并放进 skills 目录，无需你手动操作。

## API key：仅搜索与发布需要

公司目录、面经攻略、行业资讯完全公开，装好即可用，无需任何配置。只有**搜索**和**发布**接口需要 API key（形如 `sk-...`）。

**快速体验（免注册）**：agent 可以自助领取一把开放测试 key，直接开始搜索：

```bash
curl -s https://offerdao.ai/api/agent/test-key
```

测试 key 仅有查询权限（不能发布岗位），且有有效期和每日调用上限，仅供体验。

**正式使用（个人 key）**：

1. 浏览器打开 [offerdao.ai](https://offerdao.ai)，用邮箱或 Google 登录 / 注册。
2. 右上角**头像 → 我的设置 → API keys → 新建 key**，复制保存。
3. 把 key 交给 agent（或存进环境变量 `OFFERDAO_API_KEY`），后续请求复用。

> key 与账号绑定：用谁的 key 发布，岗位就归属到谁的「我的发布」。请使用你本人的 key，不要外泄。

## 验证

安装后新开一个会话，问 agent：

> 帮我搜一下 Offer 岛最近 30 天北京的 AI infra 正式岗位

如果 agent 触发了 `offerdao` skill 并调用 Offer岛 API（没有 key 时它会先自助领取测试 key，或向你要 key），即说明安装成功。

也可以试试无需 key 的能力：

> 最近 AI 圈有什么新模型发布？

## License

[MIT](LICENSE) © Offerdao
