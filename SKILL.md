---
name: skills-center-client
description: "公司技能中心的 agent 侧入口:列出技能、装到本平台、保持更新、连技能中心、替业务技能取凭证、没权限时代提申请。触发例句:'列出可用技能''把 <技能名> 装给我''我装的技能该更新了''我能用哪些技能''帮我连一下技能中心''帮我取 <技能名> 的凭证''我没有权限用 <技能名>'。owner 侧操作(开权限、凭证登记/轮换、上架)不在本技能——让 TA 的 AI 读「技能中心·owner 手册」:https://xtalpi.feishu.cn/docx/TD8Gd4Bl7oKnPwxuTsicuwmpn9d。不做:自动上架、代用户确认连接、打印或保存凭证值、未经 owner 指令开权限。"
维护人: 顾钰良
team: 平台组
domain: [数据与通用]
version: 1.2.0
status: 可用
risk: 只读查询
credential: 无
URL: https://github.com/koyurion/skills-center
metadata:
  requires:
    bins: ["lark-cli", "npx"]
---

# 技能中心客户端

同事侧的日常动词:列出、装、更新、连接、取值、没权限时提申请。owner 侧(开权限、凭证登记、上架)由「技能中心·owner 手册」承载,本技能只在末尾指路。

## 何时用(触发例句)

| 同事会说 | 做什么 |
|---|---|
| 「列出可用技能」 | 拉技能清单,过滤后报告 |
| 「把 <技能名> 装给我」 | 按来源走「安装 / 更新」 |
| 「我装的技能该更新了」 | 同安装,覆盖旧版 |
| 「我能用哪些技能?」 | 按身份分层,固定三行回答 |
| 「帮我连一下技能中心」 | 发起配对,用户点一次链接 |
| 「帮我取 <技能名> 的凭证」 | 连技能中心取值,值只交给进程 |
| 「我没有权限用 <技能名>」 | 主动提出替用户向负责人提申请 |

## 准备

- 本技能正源是 GitHub 仓库(frontmatter 的 URL 字段),更新走 `npx skills update -y`。注意是 `skills`,不是 `skill`——后者是无关的 npm 包。
- 「列出」和「装飞书文档源的技能」需要 lark-cli(技能文档的阅读走用户的飞书身份;默认授权不含搜索,本技能按 URL 直接拉文档,不受影响):

```
npx @larksuite/cli@latest install -y   # 装 lark-cli
lark-cli auth login                     # 设备码,用户自己扫
lark-cli auth status                    # agent 代跑自检
```

- 本技能要求 agent 能访问公司内网;豆包工作·云电脑模式到不了内网,凭证技能在那边如实说不可用(不是故障)。

## 列出

```
lark-cli docs +fetch --doc https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb --doc-format markdown --as user
```

这就是技能清单(正源)。导出的表格可能是 Markdown 表也可能是 HTML table,都直接解析;地址列按约定是纯文本完整网址,原样用;表格里 @文档/@人 的 cite 标签按转换规则第 7 条重建网址。查本地装了什么:`npx skills list`。

## 我能用哪些(固定问法,三行输出)

用户问「我能用哪些技能」时,先看清单,再按身份分层。已连接技能中心的,问一次:

```
curl -s http://10.254.21.214:8901/creds -H "Authorization: Bearer <令牌>"
```

- **已连接**:GET /creds 的 usable 里是能取到的凭证技能,加上清单里状态「可用」且使用说明没提凭证的技能,都算「能直接用」;清单里提到凭证但不在 usable 里的归「需要申请」。
- **未连接**:状态「可用」的技能里,使用说明没提凭证的直接能用;提到凭证的归「需要申请」,并建议先连接技能中心。
- 「需要申请」的负责人姓名:从技能清单那一行的使用说明、或技能文档 frontmatter 的维护人里找(取值被拒的 403 响应里也有)。

回答固定三行,不要自由发挥:

```
能直接用:N 个(名字)
需要申请:N 个(名字+负责人姓名)
想全看一遍:https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb
```

## 安装 / 更新

**From feishu.cn(装和更新同一条路,更新=版本号变了就重跑)**:

1. 取文档:lark-cli docs +fetch --doc <文档地址> --doc-format markdown --as user
2. 按下方转换规则组装 SKILL.md(临时目录,用绝对地址)
3. 安装:npx skills add <SKILL.md 所在目录> -y(项目级,默认;用户说「装到全局」则命令加 -g。传目录不传文件——传 SKILL.md 文件路径会报 No valid skills found)
4. 附件 zip(多脚本技能):lark 拿不到附件,提醒用户从文档附件另存,解压到 SKILL.md 同目录
5. 装完清理临时文件(rm -rf 被权限策略拦截时,改用 rm 文件 + rmdir 目录)

**转换规则(文档 → SKILL.md,agent 照此执行)**:

1. 开头的 YAML 代码块 → 真 frontmatter(去掉的是 markdown 代码围栏;块内自带的首尾 --- 就是 frontmatter 围栏,原样保留,别再包一层),并追加一行 `URL: <文档地址>`(加在收尾 --- 之前;导出正文最前的 title 元素不算内容——走第 2 条变正文 H1,yaml 块仍是第一个内容块)
2. 第一个标题 → 正文 H1
3. 硬换行的句子合并成段(飞书导出按屏宽断行)
4. 修复导出时重置的有序列表编号
5. 其余代码块原样保留(尤其 frontmatter 之外的部分)
6. 表格:Markdown 表和 HTML table 都直接解;表格里的地址列是纯文本完整网址,原样用;表格里如出现 @文档/@人 丢地址的情况,按第 7 条重建
7. @文档/@人 导出成 cite 标签时:带 doc-id 的按类型重建网址(docx → https://xtalpi.feishu.cn/docx/<id>,wiki → https://xtalpi.feishu.cn/wiki/<id>);带 user-id 的只当人名读,不是链接

**From git 或本地**:`npx skills add <仓库地址> -y`(项目级;全局加 -g)安装;更新 `npx skills update -y`。文档源技能没有 update——重跑上面的安装流程,版本号没变的跳过。

## 连接技能中心(配对,只需一次)

agent 发起,用户只在飞书点一次「确认连接」:

```
curl -s -X POST http://10.254.21.214:8901/auth/pair -d '{"mode":"device","poll_secret":"<agent生成的随机串>"}'
# → {"pair_id":"...","user_code":"K7QM-3XP9","pair_url":"http://10.254.21.214:8901/auth/..."}
```

把 pair_url 原样给用户,配这段话:

> 「点开这个链接授权,只需要一次。请核对页面上的配对码和我这里显示的一致:K7QM-3XP9——对得上再点确认。」

用户点确认后,agent 轮询取令牌(poll_secret 防抢跑,一次性取走即清):

```
curl -s http://10.254.21.214:8901/auth/pair/<pair_id> -H "Authorization: Bearer <poll_secret>"
# → {"token":"..."}
```

- 本地 agent(Cursor/Claude Code/豆包本地电脑):令牌存 ~/.skills-center/device-token(权限 0600),90 天没用自动失效;每次复用前先查一次身份,并向用户确认一句「我用的是 <姓名> 的连接,对吗?」(防云端共享主机串号)
- 内网 agent 平台(agentScope):令牌只存会话内存,12 小时,不落盘
- 解除连接:用户说「不再用了」时,agent 执行 POST http://10.254.21.214:8901/me/revoke(Bearer 令牌),删本地令牌文件

## 取值(替业务技能代办)

先拿数字 id——GET /creds 的 usable 里每行是 {id, skill_id, target, tier},按 skill_id 对上要取的技能:

```
curl -s http://10.254.21.214:8901/creds -H "Authorization: Bearer <令牌>"
```

再取值(URL 里是 usable 返回的数字 id,不是 SK-xxxxx;不在 usable 里=还没权限,走「没有权限时」):

```
curl -s -X POST http://10.254.21.214:8901/creds/<数字id>/value -H "Authorization: Bearer <令牌>"
# → {"lease_id":"...","skill_id":"...","target":"...","secret":"...","expires_at":"<租约到期时刻,取值后4小时>"}
```

**硬约束:值只允许直接交给需要它的进程或当次运行的环境变量;不得打印、不得写入对话、不得存文件。**每次借用 4 小时,用完即弃、要用再取;借到的是真值,有效期由发它的系统决定——万一哪里存了旧值,轮换后接口自然报错,重取一次就好。用户全程不接触值。

## 没有权限时(取值被拒)

取值被拒的 403 响应里有负责人姓名(owner_name,不含邮箱)。agent 主动说:

> 「我替你向负责人提个申请,好吗?」

- 凭证档是「文档协作者」:agent 替用户向该技能文档的 owner 申请阅读权限,用户身份执行——

```
lark-cli drive +apply-permission --token <技能文档token> --type docx --perm view --as user
```

- 然后告诉用户:「已申请,owner 同意后跟我说『再试一次』。」
- 凭证档是「指定的人」:没有可点的申请入口——agent 以用户身份给负责人发飞书私信代为申请(lark-cli im +messages-send --as user),内容写明技能名和用户姓名。

## 不做什么

- 不上架、不代改技能清单——owner 侧见下节指路
- 不代用户确认连接——配对链接必须用户本人点,对话里的口头「可以」不算数
- 不打印、不保存、不代填凭证值——值只在进程与环境变量里存在;登记/轮换的命令由 owner 亲手跑
- 未经 owner 指令不执行任何开权限命令(开权限的命令在 owner 手册技能里)

## owner 侧指路

owner 说「给某某开权限」「把这条凭证登记进技能中心」「轮换 <技能名> 的凭证」「帮我上架一个技能」时,不在本技能——让 TA 的 AI 读「技能中心·owner 手册」照做,或先装它:

https://xtalpi.feishu.cn/docx/TD8Gd4Bl7oKnPwxuTsicuwmpn9d

## 需要凭证吗

本技能自身不需要。取值是替业务技能代办,连接的也是用户的身份。
