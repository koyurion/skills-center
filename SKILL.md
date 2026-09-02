---
name: skills-center
version: 0.2.0
description: "私有的 skills-center 仓库，具有一个统一的索引，可以通过这个索引安装各种 skill。Skill 库支持飞书文档和 git 两种模式"
metadata:
  requires:
    bins: ["lark-cli", "npx"]
    skills: ["lark-shared", "lark-doc", "lark-wiki"]
---

# Skill Center

## 准备

```
{SKILLS_CENTER_INDEX_URL} = https://xtalpi.feishu.cn/wiki/ZTp7wM7Kiik2tLkXpFUcyyE3nnh
```

- lark-cli 未安装时：`npx @larksuite/cli@latest install -y`。
- 操作前用 `lark-cli auth status` 确认登录状态，user identity 不可用就引导用户登录。
- 安装/更新用的是 `skills` CLI（`npx skills ...`）。注意是 `skills` 不是 `skill`——后者是 npm 上另一个不相干的包。

## 列出 skill 列表

如果用户说列出 skills-center 的 skill 列表，通常是说查询远程索引，而并不是查询本地安装情况。查询远程索引可以通过 lark-cli 获取索引内容，然后再显示给用户：

```
lark-cli docs +fetch --doc "{SKILLS_CENTER_INDEX_URL}" --doc-format markdown --as user
```

索引是一个表格（名称 | 标签 | 地址）。地址有两种形态：

- 普通 URL（如 git 仓库）：原样使用。
- 飞书文档：markdown 中表现为 `<cite doc-id="RV9Jd0q4..." file-type="docx" title="..." type="doc">`，没有明文 URL。展示用 `title`，取地址时用 doc-id 拼出 `https://xtalpi.feishu.cn/docx/{doc-id}`。

查看本地已安装情况用 `npx skills list`。

## 安装 / 更新 skill (From feishu.cn)

安装和更新是同一套流程：`skills add` 会覆盖同名旧版本并刷新 skills-lock.json 的 hash。`npx skills update` 只对 git 来源的 skill 有效；飞书文档来源安装后是 local 类型，更新必须重走本流程。

第一步，从索引表格中找到目标 skill 的飞书文档（取 doc-id 拼出地址），用 lark-cli 获取内容：

```
lark-cli docs +fetch --doc "https://xtalpi.feishu.cn/docx/{doc-id}" --doc-format markdown --as user
```

第二步，整理内容并保存为 SKILL.md：

- 飞书文档不支持 YAML frontmatter，开头的 YAML code block（如果有）要转换成真正的 frontmatter。
- 在 frontmatter 中增加 `URL: {文档地址}` 字段，方便后续更新比对。
- 文档标题（`<title>` 标签）转为正文一级标题 `# {title}`。
- 正文里被硬换行拆开的句子、编号被重置的有序列表是 docx 排版产物，整理时再合并/修正。
- 文档中的其余代码块（如工具实现代码）属于 skill 内容，原样保留。

第三步，创建临时文件夹（用 skill 名称命名，注意非法字符的处理），把 SKILL.md 放进去，以本地文件夹形式安装：

```
npx skills add {临时文件夹} -y
```

第四步，清理临时文件夹。若 `rm -rf` 被权限策略拦截，改用 `rm {文件}` + `rmdir {目录}`。

## 安装 / 更新 skill (From 其他)

直接使用 `npx skills` 系列命令安装或更新
