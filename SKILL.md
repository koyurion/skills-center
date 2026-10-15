---
name: skills-center-client
description: "公司技能中心客户端,一个技能两种角色:使用者对 AI 说自然语言,列出、安装、更新技能,连技能中心取凭证,没权限时代提申请;开发者开文档权限,凭证登记、轮换、吊销,查账、上架。触发例句:'列出可用技能''把 <技能名> 装给我''我能用哪些技能''帮我连一下技能中心''帮我取 <技能名> 的凭证''给张三开 <技能> 的权限''把这条凭证登记进技能中心''轮换 <技能名> 的凭证''查一下这条凭证的流水''帮我上架一个技能'。连接 = 飞书授权点一次:会话凭证由飞书签发(约 2 小时),只留会话内存不落盘。凭证值不打印、不保存:登记与轮换的值命令由开发者亲手跑。"
metadata:
  version: 5.1.0
  requires:
    bins: ["lark-cli", "npx"]
---

# 技能中心客户端

一个技能,两种角色:使用者——列出、装、更新、连接、取值、没权限时提申请;开发者——开权限、登记、轮换、吊销、查账、上架。说人话,命令现成。

## 何时用(触发例句)

| 说 | 做什么 |
|---|---|
| 「列出可用技能」 | 拉技能清单,过滤后报告 |
| 「把 <技能名> 装给我」「我装的技能该更新了」 | 按来源走「装与更新」 |
| 「我能用哪些技能?」 | 按身份分层,固定三行回答 |
| 「帮我连一下技能中心」 | 发起连接,用户在飞书点一次「同意」 |
| 「帮我取 <技能名> 的凭证」 | 连技能中心取值,值只交给进程 |
| 「我没有权限用 <技能名>」 | 主动提出替用户向开发者提申请 |
| 「给张三开 <技能> 的权限」「把研发一部加进 <技能> 文档」 | 按「开权限」加协作者 |
| 「这篇文档关掉链接分享」 | 关链接可读,凭证授权收回协作者名单 |
| 「把这条凭证登记进技能中心」 | 照「凭证」代办(值命令开发者亲手跑) |
| 「轮换 <技能名> 的凭证」「把 <技能名> 的凭证吊销」 | 照「凭证」轮换/吊销 |
| 「查一下这条凭证的流水」 | 代查审计台账 |
| 「帮我上架一个技能」 | 指路三步:模板写文档→开权限→清单登记 |

## 准备(一次性)

- 本技能正源是 GitHub 仓库（https://github.com/koyurion/skills-center）,更新走 `npx skills update -y`。注意是 `skills`,不是 `skill`——后者是无关的 npm 包。
- 「列出」和「装飞书文档源的技能」需要 lark-cli(技能文档的阅读走用户的飞书身份;默认授权不含搜索,本技能按 URL 直接拉文档,不受影响):

```
npx @larksuite/cli@latest install -y   # 装 lark-cli
lark-cli auth login                     # 设备码,用户自己扫
lark-cli auth status                    # agent 代跑自检
```

- 本技能要求 agent 能访问 xtalpi.feishuapp.com(公网 HTTPS)——不再要求公司内网,豆包工作·云电脑模式也能用凭证技能了。
- 连接与取值的每个请求都要带网关 API key(平台管理员下发的应用级凭证,与个人身份无关):存本地私密文件(如 `~/.sc-api-key`,单行内容,`chmod 600`),命令里注入——**不进对话、不落仓库**;下文写作 `<api_key>`。

## 连接技能中心(飞书授权,每会话一次)

agent 发起,用户只在飞书点一次「同意」:

```
curl -s -X POST https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/auth/pair \
  -H "Authorization: Bearer <api_key>" -H "Content-Type: application/json" \
  -d '{"poll_secret":"<agent生成的随机串>"}'
# → {"pair_id":"...","user_code":"K7QM-3XP9","pair_url":"https://xtalpi.feishuapp.com/app/app_17fzk95wm91/pair?pair_id=..."}
```

把 pair_url 原样给用户,配这段话:

> 「点开这个链接,页面会直接跳到飞书,用你自己的飞书账号点一次「同意」就完成连接。授权完回来,页面上会显示配对码,请和我这里显示的一致:K7QM-3XP9;对不上就点页面上的「不一致——作废并告警」。本次连接(约 2 小时)内有效,新会话要用时再点一次。」

用户点「同意」后,agent 轮询取 session_token(poll_secret 防抢跑,一次性取走即清):

```
curl -s https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/auth/pair/<pair_id> \
  -H "Authorization: Bearer <api_key>" -H "X-Sc-Token: <poll_secret>"
# → {"status":"ok","session_token":"...","name":"...","email":"..."}
```

- session_token 里装的就是**飞书签发的那张 user_access_token**(约 2 小时),只放 agent 会话内存——**不写文件、不落盘、不进对话**;此后所有请求带 `X-Sc-Token: <session_token>`(Authorization 头装的是网关 API key)
- 拿到后先 GET /me(带齐上面两个头)向用户确认一句「我用的是 <姓名> 的授权,对吗?」——本地与云端平台同一条规矩,防共享主机串号
- 请求收到 401(授权已过期):同一会话超过约 2 小时会这样,重新走一遍本节流程即可——系统里没有任何长期令牌,这是设计不是故障
- 轮询若回 status `not_mine`:用户点了「不一致——作废并告警」——**停下来问清楚**,不要当成技术故障重试
- 断开:让用户去飞书「设置 → 应用授权」取消本应用,立即生效(下一次请求就 401);不操作则约 2 小时后自然失效——系统里没有任何长期令牌,这是设计不是故障

## 使用者

### 列出

```
lark-cli docs +fetch --doc https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb --doc-format markdown --as user
```

这就是技能清单(正源)。导出的表格可能是 Markdown 表也可能是 HTML table,都直接解析;地址列按约定是纯文本完整网址,原样用;表格里 @文档/@人 的 cite 标签按转换规则第 7 条重建网址。查本地装了什么:`npx skills list`。

### 我能用哪些(固定问法,三行输出)

用户问「我能用哪些技能」时,先看清单,再按身份分层。本会话已授权的,问一次:

```
curl -s https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds \
  -H "Authorization: Bearer <api_key>" -H "X-Sc-Token: <session_token>"
```

- **本会话已授权**:GET /creds 的 usable 里是能取到的凭证技能,加上清单里状态「可用」且使用说明没提凭证的技能,都算「能直接用」;清单里提到凭证但不在 usable 里的归「需要申请」。
- **未授权(本会话)**:状态「可用」的技能里,使用说明没提凭证的直接能用;提到凭证的归「需要申请」,并建议先连一下技能中心(每会话一次,见「连接技能中心」)。
- 「需要申请」的开发者姓名:从技能清单那一行的使用说明、或技能文档 frontmatter 的维护人里找(取值被拒的 403 响应里也有)。

回答固定三行,不要自由发挥:

```
能直接用:N 个(名字)
需要申请:N 个(名字+开发者姓名)
想全看一遍:https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb
```

### 装与更新

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

### 取值(替业务技能代办)

先拿数字 id。GET /creds 的 usable 里每行是 {id, skill_doc, target},对法两步:从技能清单那行拿技能文档地址,取 wiki/ 或 docx/ 后面那串 token;在 usable 里找 **skill_doc 等于这串 token、target 等于技能文档里写的凭证名**(文档没写凭证名就是 default)的行,拿它的数字 id:

```
curl -s https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds \
  -H "Authorization: Bearer <api_key>" -H "X-Sc-Token: <session_token>"
```

再取值(URL 里是 usable 返回的数字 id;不在 usable 里=还没权限,走「没有权限时」):

```
curl -s -X POST https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds/<数字id>/value \
  -H "Authorization: Bearer <api_key>" -H "X-Sc-Token: <session_token>"
# → {"skill_doc":"...","target":"...","secret":"...","note":"值只供当次使用,要用再取..."}
```

**硬约束:值只允许直接交给需要它的进程或当次运行的环境变量;不得打印、不得写入对话、不得存文件。**要用就取,值只供当次使用;取到的是真值,有效期由发它的系统决定——万一哪里存了旧值,轮换后接口自然报错,重取一次就好。用户全程不接触值。

### 没有权限时(取值被拒)

取值被拒的 403 响应里有开发者姓名(owner_name,不含邮箱)和一句 hint。授权跟着绑定文档的分享设置走——被拒多半是因为用户不是这篇文档的协作者。agent 主动说:

> 「我替你向开发者提个申请,好吗?」

- agent 替用户向该技能文档的开发者申请阅读权限,用户身份执行——

```
lark-cli drive +apply-permission --token <技能文档token> --type docx --perm view --as user
```

- 然后告诉用户:「已申请,开发者同意后跟我说『再试一次』。」
- 例外:hint 里写着「关掉互联网链接」时,申请没有用——是文档开了互联网链接分享,凭证被服务一律挡住。如实转告用户:「请开发者在文档分享设置里把互联网链接关掉,只对公司开或关闭,这条凭证就能恢复。」

## 开发者

### 开权限

文档的分享设置就是凭证的授权面板:把人加进文档(个人、部门或群),TA 就能用对应的凭证技能;移走、换组、退群即失效。链接分享的公司开关就是凭证的「全公司开关」:开着,任何连接过技能中心的使用者都能取;关掉,立即回到只有协作者能取。开权限默认静默(不推通知),想让对方知道就在命令末尾加 --need-notification:

```
# 加个人(high-risk-write,必须 --yes)
lark-cli drive +member-add --token <t> --type docx --member-type email --member-id someone@xtalpi.com --perm view --yes
# 加部门(必须 --as user;bot 不支持部门授权)
lark-cli drive +member-add --token <t> --type docx --member-type opendepartmentid --member-id od-xxx --perm view --as user --yes
# 把「技能中心」应用加为协作者(完全访问;绑了凭证的文档必须,判定靠它读状态和名单,重复执行无妨)
lark-cli drive +member-add --token <t> --type docx --member-type appid --member-id cli_aa48e775a5f89beb --perm full_access --as user --yes
# 关链接可读(链接开着=拿到就能读、能装;绑了凭证的,还等于凭证全公司可取)
lark-cli drive permission.public patch --token <t> --type docx --data '{"link_share_entity":"closed"}' --yes
# 看现在谁有权限
lark-cli drive +member-list --token <t> --type docx --fields '*'
```

### 凭证(登记、轮换、换绑、吊销、查账)

agent 代办流程:确认字段、起草命令、代查结果。**唯一不代办的是值:登记和轮换的命令由开发者在自己终端里亲手跑,值由 TA 亲手填(read -s 不回显);agent 不打印、不转述、不代填。**代查结果、台账时,agent 走「连接技能中心」(开发者身份);值命令所在的终端要单独做一次终端授权(下框,约 2 小时,过期重跑;配对码只印在这个终端里,值不经过对话;服务是公网 HTTPS,云电脑终端也能跑):

**终端会话授权**(值命令所在的终端里跑一次;网关 API key 同「准备」节,存 ~/.sc-api-key):

```
KEY=$(cat ~/.sc-api-key)   # 网关 API key,本地私密文件
PS=$(openssl rand -hex 24); PAIR=$(curl -s -X POST https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/auth/pair -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" -d "{\"poll_secret\":\"$PS\"}")
echo "$PAIR" | python3 -c 'import json,sys;d=json.load(sys.stdin);print("浏览器打开:",d["pair_url"],"\n页面配对码应为:",d["user_code"])'
# 打开链接、在飞书点一次「同意」;回来核对页面上的配对码与上面打印的一致(对不上就点「作废并告警」),再跑这一行:
SC_SESS=$(curl -s "https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/auth/pair/$(echo "$PAIR" | python3 -c 'import json,sys;print(json.load(sys.stdin)["pair_id"])')" -H "Authorization: Bearer $KEY" -H "X-Sc-Token: $PS" | python3 -c 'import json,sys;print(json.load(sys.stdin)["session_token"])')
echo 授权完成,本终端约2小时内可用
```

**登记**(新凭证进服务)。字段逐项和开发者确认:

- **skill_doc**:技能文档的 token(清单地址 wiki/ 或 docx/ 后面那串,原样贴,不带域名不带斜杠)——它和凭证名一起就是这条凭证的身份
- **target**:凭证名,技能文档里「凭证名: xxx」声明的那份,没写就是 default;同一技能要多份凭证(如一份只读、一份写入),文档里写多个凭证名,同一 skill_doc 按凭证名各登记一行
- **doc_token**(可选):绑定哪篇文档做授权,缺省=绑技能文档本身(也可以绑另一篇私有文档,只把该看的人加为协作者)
- **doc_type**:文档载体,默认 docx,地址 wiki/ 开头必须写 wiki——载体写错,取值判定读不到状态
- **scope**:一句话用途;**skill_name**(可选,技能名,仅给人看)

登记前先把「技能中心」应用加成绑定文档的完全访问协作者(「开权限」的 appid 那条;漏了它,谁取值都会被拒)。字段确认后,把这组命令给开发者,让 TA 在已做终端授权的终端里跑:

```
read -s SC_VALUE        # 回车后直接敲值:屏幕不显示,也不进 shell 历史
curl -s -X POST https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds -H "Authorization: Bearer $KEY" -H "X-Sc-Token: $SC_SESS" -H "Content-Type: application/json" -d @- <<EOF
{"skill_doc":"<技能文档token>","target":"<凭证名>","doc_token":"<绑定文档token,同一篇就删掉这个字段>","doc_type":"<docx|wiki>","scope":"<一句话用途>","skill_name":"<技能名,可省>","value":"$SC_VALUE"}
EOF
unset SC_VALUE
```

值经 stdin 进请求体(-d @-),不出现在命令行参数里;值含引号或反斜杠时改用 python3 组 JSON(变量同样来自 read -s)。跑完 agent 代查:GET /creds 的 mine 里应出现这一行,把 roster_summary(当前协作者名单摘要)念给开发者核对——把谁加进绑定文档,谁就能取值。若绑定文档的链接共享对公司开着,登记会被 409 拦下并回显当前状态——开发者想真开,原话说出「确认全公司可取」,agent 把这句短语原样放进 confirm_open 字段重发。409=已登记(转轮换),400=把提示照实转告。

**轮换**(值更新)。id 从 GET /creds 的 mine 里拿;新值同样由开发者亲手跑(终端授权过期就先重跑上面的授权块):

```
read -s SC_VALUE
curl -s -X PATCH https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds/<数字id> -H "Authorization: Bearer $KEY" -H "X-Sc-Token: $SC_SESS" -H "Content-Type: application/json" -d @- <<EOF
{"value":"$SC_VALUE"}
EOF
unset SC_VALUE
```

轮换后旧值即刻作废:哪里还存着旧值,接口自然报错,重取一次就好。纪律:每季度一次,敏感节点(离职/换岗/泄露疑虑)后加一次——万一哪里落过旧值,轮换让它们全部自然失效。

**换绑**(授权跟着换文档):PATCH {"doc_token":"<新文档token>"} 不含值,agent 可直接代办;换到链接对公司开的文档同样要 confirm_open 确认短语。想收紧授权,不用改登记——直接在绑定文档里移人、收链接,立即生效。

**吊销**(值下线)。开发者一条命令自办(终端授权过的终端),新取值一律拒绝;反悔=亲手跑一次轮换(带新值)即复活:

```
curl -s -X DELETE https://xtalpi.feishuapp.com/app/app_17fzk95wm91/openapi/creds/<数字id> -H "Authorization: Bearer $KEY" -H "X-Sc-Token: $SC_SESS"
```

**查账**:GET /creds 看自己名下与当前能用的;GET /creds/<数字id>/audit 看这条凭证的完整流水(一本账,不含值)。端点一览与错误语义见架构页《凭证管理服务架构》。

### 上架

1. 写:照「模板 Skill」的约定写技能文档 https://xtalpi.feishu.cn/docx/SqmJdOPZboewphxO6CNcA2E2nTg
2. 开权限:见「开权限」;不打算全公司都能用的,让 AI 关掉链接分享
3. 登记:在技能清单表末尾加一行——名称、标签、地址(纯文本网址)、使用说明,状态先填「草稿」 https://xtalpi.feishu.cn/docx/Rn1TdQRIIoemjmxOlg3cJKdNnVb

带凭证的技能,三步之外还有半步:值由开发者亲手登记进技能中心——就是上面的「凭证」节。

## 不做什么

- 不自动上架、不代改技能清单——开发者要上架,按「上架」三步走
- 不代用户点授权——飞书那条「同意」必须本人点,对话里的口头「可以」不算数
- 不打印、不保存、不代填凭证值;值命令(登记/轮换)只给开发者在自己终端亲手跑,不经过 AI
- session_token 只在会话内存,不写文件、不落盘、不进对话;网关 API key 只按本地私密文件存放,不进对话、不落仓库
- 未经开发者指令不执行任何开权限命令;「确认全公司可取」必须开发者原话说出才照发
- 401=会话过期(约 2 小时):agent 侧重新走「连接技能中心」,终端侧重跑终端授权块——系统里没有长期令牌,这是设计不是故障

## 需要凭证吗

本技能自身不需要。取值替业务技能代办,连接的是用户的身份;开发者的值由开发者亲手跑命令,不经过 AI。
