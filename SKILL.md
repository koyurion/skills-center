---
name: skills-center
description: "Skills Center 客户端(Skill的商店):使用者从清单发现、安装、更新Skill;开发者写Skill文档、上架。触发例句:'列出可用Skill''把 <Skill名> 装给我''我装的Skill该更新了''我能用哪些Skill''帮我上架一个Skill'。"
metadata:
  version: 6.8.0
  requires:
    bins: ["lark-cli", "npx"]
---

# Skills Center 客户端

Skill的**商店**：发现、安装、更新、上架。凭证（钥匙柜）由姊妹 skill **skills-center-credentials** 承担——Skill 运行要取值、授权、轮换凭证时走它。

## 准备&背景知识

1. 一个运行时身份层：lark-cli（装好并确认登录）

- 安装：`npx @larksuite/cli@latest install -y`；已可用（如平台预装）就跳过
- 自检：`lark-cli whoami`——`tokenStatus: ready` 即可用

2. 两个关键资源

| 资源 | 地址 |
| --- | --- |
| Skill清单 | https://xtalpi.feishu.cn/docx/C59MdSl9Zo5rKFxTUaecSv8Jnx5 |
| Skill规范指引及示例 | https://xtalpi.feishu.cn/drive/folder/XCrxfCSWRlOJHldm1ZtcXOGBnPe |

3. 两个权限分类：Skill文档权限与凭证权限

- Skill文档的权限 = 谁能装这个Skill；凭证表行的权限 = 谁能取到这个Skill的钥匙。两者通过「Skill名」关联，互不拖累。

| 权限分开的理由 | 理由解释 |
| --- | --- |
| 受众天然不同 | Skill文档是说明书，希望越多人能读越好（发现→学会→装上）；凭证是钥匙，希望越少人拿到越好。绑在一起只有两个坏结局：为了传播把钥匙发宽了，或者为了保钥匙把说明书藏起来。 |
| 生命周期不同 | 文档改的是用法说明，凭证换的是值本身。若把凭证授权绑在文档分享设置上，轮换凭证和整理文档会互相牵动。 |
| 粒度对不上 | 一个Skill可能有多份凭证（如：只读一份、写入一份），受众各不相同；文档只有一份。 |

## 使用者

### 列出Skill

```
lark-cli docs +fetch --doc https://xtalpi.feishu.cn/docx/C59MdSl9Zo5rKFxTUaecSv8Jnx5 --doc-format markdown --as user
```

清单正源，表格按转换规则解。查本地已装：`npx skills list`。

### 我能用哪些

- 以用户自身权限逐一访问清单里的Skill链接：能打开 = 能用（能读 = 能装）；其中「需要凭证吗」声明了注入键的，凭证可用性按 skills-center-credentials「取凭证（含可用性判定）」走
- 打不开（403）= 没有文档权限。要申请的，经用户确认后跑 `lark-cli drive +apply-permission --token <Skill文档地址> --perm view --as user` 向文档 Owner 发申请卡片（view 即够）；被拒或文档不接受申请，让用户直接联系 Owner，不重试

### Skill安装与更新（Agent看）

**From feishu.cn**（装和更新同一条路，更新 = 版本号变了就重跑）：

1. 取文档：`lark-cli docs +fetch --doc <文档地址> --doc-format markdown --as user`
2. 按转换规则组装 SKILL.md（临时目录）
3. 安装：`npx skills add <SKILL.md 所在目录> -y`（项目级默认，全局加 -g；传目录不传文件，传文件路径报 No valid skills found）
4. 附件 zip（多脚本Skill）：lark 拿不到附件，提醒用户从文档附件另存，解压到 SKILL.md 同目录；完事清理临时文件（rm -rf 被权限策略拦截时，改用 rm 文件 + rmdir 目录）
5. **转换规则**（飞书文档 → SKILL.md，agent 照此执行）：

| 导出样式 | 处理 |
| --- | --- |
| 开头 YAML 代码块 | 去掉 markdown 围栏成真 frontmatter（块内自带首尾 ---，原样保留别再包一层）；收尾 --- 前追加一行 `URL: <文档地址>`。正文最前的 title 元素不算内容，yaml 块仍是第一个内容块 |
| 标题 / 列表 | 第一个标题转正文 H1；修复被重置的有序列表编号 |
| 硬换行 | 飞书按屏宽断行，合并成段 |
| 表格 | Markdown 表和 HTML table 都直接解；地址列是纯文本完整网址，原样用 |
| @文档 / @人（cite 标签） | 带 doc-id 的按类型重建网址（docx 走 `https://xtalpi.feishu.cn/docx/<id>`，wiki 走 `https://xtalpi.feishu.cn/wiki/<id>`）；带 user-id 的只当人名读，不是链接 |

**From git 或本地**：`npx skills add <仓库地址> -y`（项目级；全局加 -g）；更新 `npx skills update -y`。文档源Skill没有 update——重跑安装流程，版本号没变的跳过。

## 开发者

### 写Skill文档

- 参考「Skill规范指引及示例」 https://xtalpi.feishu.cn/drive/folder/XCrxfCSWRlOJHldm1ZtcXOGBnPe 进行Skill编写

- Skill文档权限：基于飞书云文档的高级权限做ACL（Access Control List，访问控制列表）

| 级别 | 谁 | 能做什么 |
| --- | --- | --- |
| 可阅读 | 由开发者自定：开链接分享（组织内获得链接的人可阅读，外部分享保持关闭）= 组织内可读；不开 = 仅协作者可读；协作者加不加、加谁都由开发者决定 | 读文档、装Skill——**能读文档 = 能装** |
| 可编辑 | 维护人（协作者添加） | 改内容、发新版；装与更新都从文档拉取，改完下次安装即生效 |
| 可管理 | 文档所有者 | 删文档、转让所有权、改链接分享设置 |

### 上架·登记清单（Agent 代办）

1. 拉Skill清单表格 XML，tbody 末尾追加一行（7 列照表头：名称、标签、地址、使用说明、版本、最后更新、状态），整表 block_replace 写回
2. **回读确认自己的行在表里才算上架完成**；不在，或写入报「未产生变更」——并发时后写者失败而非覆盖，先写者的行不丢——重拉最新 XML 重试（至多 2 次），仍失败如实报告
3. 新行状态填「草稿」，改「可用」需维护人明确确认

## 不做什么

- 不自动上架——清单加行、状态改「可用」只在维护人明确指令时做
- 文档侧无权限（403/打不开）不重试、不绕过；代发权限申请需用户确认

## 需要凭证吗

本Skill自身不需要。
