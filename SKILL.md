---
name: skills-center-client
description: "公司技能中心的 agent 侧入口:列出技能、装到本平台、保持更新、连技能中心、替业务技能取凭证、权限问题代提申请、替 owner 开文档权限。触发例句:'列出可用技能''把 <技能名> 装给我''我装的技能该更新了''我能用哪些技能''帮我连一下技能中心''帮我取 <技能名> 的凭证''我没有权限用 <技能名>''给 <同事/部门> 开 <技能> 权限'(owner 说)。owner 侧凭证登记/轮换/收回:照「凭证管理服务」文档代办(https://xtalpi.feishu.cn/docx/Sjnud9TWFopMc0x0f15chbkRnnh)。不做:自动上架、代用户确认连接、打印或保存凭证值、未经 owner 指令开权限、代跑含凭证值的命令。"
owner: 顾钰良
team: 平台组
domain: [数据与通用]
version: 1.1.0
status: 可用
risk: 只读查询
credential: 无
URL: https://github.com/koyurion/skills-center
metadata:
  requires:
    bins: ["lark-cli", "npx"]
---

# 技能中心客户端

装好后一句话列出全公司技能、装到本平台、保持更新;要连技能中心取凭证,你引导用户点一次链接;取值由你代办,值不经过对话。本技能正源是 GitHub 仓库(frontmatter 的 URL 字段),更新走 `npx skills update`;owner 侧凭证管理在「凭证管理服务」文档。

## 何时用(触发例句)

| 同事会说 | 做什么 |
|---|---|
| 「列出可用技能」 | 拉技能清单,按标签/关键词过滤后报告 |
| 「把 <技能名> 装给我」 | 按下方 From feishu.cn 规则转换安装 |
| 「我装的技能该更新了」 | 对比版本,重装更新 |
| 「我能用哪些技能?」 | 按身份分层回答(见下,三行格式) |
| 「帮我连一下技能中心」 | 发起配对,引导用户点链接授权(只需一次) |
| 「帮我取 <技能名> 的凭证」 | 连技能中心取值,值只交给需要的进程,不进对话 |
| 「我没有权限用 <技能名>」 | 取值被拒时,主动提出替用户向负责人提申请 |
| 「给 <同事/部门> 开 <技能> 权限」(owner 说) | 执行开权限命令(仅 owner 指令才执行) |
| 「把这条凭证登记进技能中心」等(owner 说) | 照「凭证管理服务」文档代办(见「owner 凭证管理」节) |

## 前置(一次性)

装本技能本身不需要 lark-cli;「列出」「装飞书文档源的技能」「开权限」需要——技能文档的阅读走用户的飞书身份:

```
npx @larksuite/cli@latest install -y   # lark-cli,读飞书文档用
lark-cli auth login                     # 设备码,用户自己扫
lark-cli auth status                    # 确认已登录(agent 代跑自检)
```

(登录授权默认不全勾:搜索类操作需另勾 search:docs:read,不勾只影响搜索;按 URL 直接拉文档不受影响。)本技能要求 agent 能访问公司内网(云电脑模式除外,见平台矩阵)。

## 列出

```
lark-cli docs +fetch --doc https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb --doc-format markdown --as user
```

这就是技能清单(正源)。导出的表格可能是 Markdown 表也可能是 HTML table,都直接解析;地址列按约定是纯文本完整网址,原样用;表格里如遇 @文档/@人 丢地址的 cite 标签,按转换规则第 7 条重建网址。查本地装了什么用 `npx skills list`。

## 我能用哪些(固定问法,三行输出)

用户问「我能用哪些技能」时,先看清单,再按身份分层:

```
curl -s http://10.254.21.214:8901/creds -H "Authorization: Bearer <令牌>"
```

- **已连接技能中心**:GET /creds 的 usable 里是能取到的凭证技能,加上清单里状态「可用」且使用说明没提凭证的技能,都算「能直接用」;清单里提到凭证但不在 usable 里的归「需要申请」。
- **未连接**:状态「可用」的技能里,使用说明没提凭证的直接能用;提到凭证的归「需要申请」,并建议用户先连接技能中心——连接后才能按身份精确分层。
- 「需要申请」的负责人姓名:从技能清单那一行的使用说明、或技能文档 frontmatter 的维护人里找(取值被拒的 403 响应里也有)。

回答固定三行,不要自由发挥:

```
能直接用:N 个(名字)
需要申请:N 个(名字+负责人姓名)
想全看一遍:https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb
```

## 安装其他技能(From feishu.cn)

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

## 更新

- git 源技能(含本技能):npx skills update -y(在装了它的项目里跑;全局装的加 -g)
- 文档源技能:对 skills-lock.json 里每个技能重新执行安装流程(同规则覆盖);版本号没变的跳过

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

**硬约束:值只允许直接交给需要它的进程或当次运行的环境变量;不得打印、不得写入对话、不得存文件。**每次借用 4 小时,用完即弃、要用再取;借到的是真值,有效期由发它的系统决定——万一哪里存了旧值,轮换后接口自然报错,重取一次就好(重取只是再说一句话,不用重新登录)。用户全程不接触值。

## 没有权限时(取值被拒)

取值被拒的 403 响应里有负责人姓名(owner_name,不含邮箱)。agent 主动说:

> 「我替你向负责人提个申请,好吗?」

- 凭证档是「文档协作者」:agent 替用户向该技能文档的 owner 申请阅读权限,用户身份执行——

```
lark-cli drive +apply-permission --token <技能文档token> --type docx --perm view --as user
```

- 然后告诉用户:「已申请,owner 同意后跟我说『再试一次』。」
- 凭证档是「指定的人」:没有可点的申请入口——agent 以用户身份给负责人发飞书私信代为申请(lark-cli im +messages-send --as user),内容写明技能名和用户姓名。

## 开权限(owner 指令才执行)

owner 说「给张三开 <技能> 的权限」「把研发一部加进去」「这篇文档关掉链接分享」时,agent 执行:

```
# 加个人(high-risk-write,必须 --yes)
lark-cli drive +member-add --token <t> --type docx --member-type email --member-id someone@xtalpi.com --perm view --yes
# 加部门(必须 --as user;bot 不支持部门授权)
lark-cli drive +member-add --token <t> --type docx --member-type opendepartmentid --member-id od-xxx --perm view --as user --yes
# 把「技能中心」应用加为协作者(完全访问;凭证档=「文档协作者」的技能必须,判定名单靠它读)
lark-cli drive +member-add --token <t> --type docx --member-type appid --member-id cli_aa48e775a5f89beb --perm full_access --as user --yes
# 关链接可读(不打算全公司都能用的技能;链接开着=拿到就能读、能装)
lark-cli drive permission.public patch --token <t> --type docx --data '{"link_share_entity":"closed"}' --yes
# 看现在谁有权限
lark-cli drive +member-list --token <t> --type docx --fields '*'
```

把谁加进文档(个人、部门或群),谁就能用对应的凭证技能;移走、换组、退群即失效。不打算让全公司都能用的技能,关掉链接分享——链接开着,任何拿到链接的人都能读、能装。

凭证档选「文档协作者」的技能,名单靠「技能中心」应用读。这步你主动做,不用 owner 记:开权限或登记凭证时,顺手把应用加成完全访问协作者(上面 appid 那条命令,重复执行无妨);漏了它,谁取值都会被拒(报「协作者名单暂不可读」)。

开权限默认静默:上面的命令不会给对方推任何飞书通知。想让对方知道被授权了,在 member-add 命令末尾加 --need-notification(仅 --as user 身份有效)。

## owner 凭证管理(登记、轮换、收回)

登记、轮换、改档与收回、吊销、查账的完整流程在「凭证管理服务」文档,owner 说「把这条凭证登记进技能中心」「轮换 <技能名> 的凭证」时,fetch 它照做:

https://xtalpi.feishu.cn/docx/Sjnud9TWFopMc0x0f15chbkRnnh

要点:你代办流程与代查,「值」那半条命令由 owner 在自己终端里亲手跑;凭证档=「文档协作者」的,先替 owner 把「技能中心」应用加成绑定文档的完全访问协作者(命令见「开权限」节)。

## 平台矩阵(如实说)

| 平台 | 连接方式 | 凭证可用 |
|---|---|---|
| Cursor / Claude Code(本地) | 长期连接(存本机) | 可用 |
| 豆包工作·本地电脑模式 | 长期连接(存本机) | 可用(2026-10-08 实测内网可达) |
| agentScope(内网) | 会话连接(只存内存) | 可用 |
| 豆包工作·云电脑模式 | — | **不可用**:云主机到不了内网;技能安装走文档内容粘贴降级,凭证技能如实说不可用 |

## 坑表(安装侧)

| 坑 | 正确做法 |
|---|---|
| 表格里的链接不可点/丢地址 | 地址列本来就是纯文本网址,原样用;cite 标签按转换规则第 7 条重建 |
| 有序列表编号重置 | 转换时重排 |
| 一句被断成多行 | 合并成段 |
| 附件 zip 拿不到 | 提醒用户从文档附件另存,解压到 SKILL.md 同目录 |
| 装了未授权的带凭证技能 | 装得上,取值时被拒——走「没有权限时」剧本 |
| 云电脑模式连不上技能中心 | 见平台矩阵;不是故障,如实告知 |

写文档侧的坑(链接进单元格等)见模板 Skill:https://xtalpi.feishu.cn/docx/SqmJdOPZboewphxO6CNcA2E2nTg

## 不做什么

- 不上架、不代改技能清单——新增技能由 owner 自己走上架三步
- 不代用户确认连接——配对链接必须用户本人点,对话里的口头「可以」不算数
- 不打印、不保存、不代填凭证值——值只在进程与环境变量里存在;登记/轮换的命令由 owner 亲手跑
- 未经 owner 指令不执行任何开权限命令

## 需要凭证吗

本技能自身不需要。取值是替业务技能代办,连接的也是用户的身份。
