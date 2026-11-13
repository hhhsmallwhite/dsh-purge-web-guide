# DSH 网页版破甲完整指南（隔离实例 + 完整闭环）

> 把网页版 DeepSeek Harness（`dsh web`）破甲，并且**破甲后能真正正常使用**的完整方法。
>
> 本文以本机实测跑通的**隔离实例**为基准：
> **`D:\dsh-purge\runtime\dsh-home`｜端口 3085｜DSH 0.2.0-rc.2｜dsh-purge 1.1.71**
>
> ⚠️ 仅供已授权环境下的红队评测与鲁棒性研究。遵循 [dsh-purge 合规声明](https://github.com/YuJunZhiXue/dsh-purge)。

---

## 为什么用隔离实例

破甲会改写 `@deepseek-ai/*` 包文件。如果直接对日常在用的宿主动手，一旦出问题就得重装整个 DSH。

隔离实例的做法是把 `DSH_HOME` 指到独立目录，**补丁、注入、会话全部落在沙盒里**，日常环境一点不受影响，出问题删掉目录重来即可。

| 项 | 日常环境 | 本文破甲实例 |
|---|---|---|
| DSH_HOME | `%USERPROFILE%\.dsh` | `D:\dsh-purge\runtime\dsh-home` |
| 端口 | 54498（桌面端） | **3085** |
| profile | desktop | **web** |
| 破甲状态 | 未装 purge | **已应用 1.1.71** |

两套实例的账号池、会话、插件互相不可见——这是隔离的预期行为，不是故障。

---

## 第一层认知：破甲是三层拦截，不是一段提示词

只往系统提示词塞越狱文案，会被另外两层吃掉：

| 层 | 位置 | 作用 | 不改的后果 |
|---|---|---|---|
| **① 宿主补丁** | `@deepseek-ai/*` 包内文件 + shim | 改审批门、沙箱、提示词拼接 | 工具调用被审批门拦住 |
| **② 提示词注入** | `$DSH_HOME/prompt-inject.md` | 决定模型身份与规则 | 模型仍按官方人格回答 |
| **③ workspace baseline** | DSH 每轮自动拼接 | 稀释注入 | 拦截复现 |

**dsh-purge 的价值在于三层一起处理，且可还原。**

---

## 版本矩阵（最先确认这条）

`dsh-purge` 的 `dshTarget` 必须与 DSH 版本严格匹配，对不上会显示「待应用/跳过」，不会乱改文件：

| DSH 版本 | 所需 purge |
|---|---|
| `0.2.x`（官方桌面 0.2.0-rc.2、官方 `dsh web`） | **1.1.40+** |
| `0.1.7`（rc.1/rc.2） | **1.1.39 及更早** |

本例实测（`package.json` 内写死）：

```json
"dshTarget": "0.2.0-rc.2"
```

```powershell
node .\node_modules\@deepseek-ai\dsh\lib\bin.js --version
# 期望：0.2.0-rc.2
```

---

## 第一步：建隔离实例

```powershell
mkdir D:\dsh-purge\runtime
cd D:\dsh-purge\runtime
npm init -y
npm install @deepseek-ai/dsh@0.2.0-rc.2
```

**三个必须遵守的隔离条件**（缺一就会污染日常环境）：

1. `DSH_HOME` 指向沙盒，绝不使用 `%USERPROFILE%\.dsh`；
2. `DSH_BASE` **必须清空**——它会让 purge 去改写路径错误的包文件；
3. `autoApplyOnStart: false`——启动时不静默改文件，必须人工点「应用」。

---

## 第二步：装 purge 进 web profile

```powershell
cd D:\dsh-purge\runtime\dsh-home\profiles\web
dsh plugin --profile web add dsh-purge
```

registry 不通时改用 tarball：

```powershell
dsh plugin --profile web add https://github.com/YuJunZhiXue/dsh-purge/archive/refs/heads/master.tar.gz
```

验证安装：

```powershell
Get-Content .\node_modules\dsh-purge\package.json | Select-String 'version|dshTarget'
# version: 1.1.71   dshTarget: 0.2.0-rc.2
```

---

## 第三步：注册到 cordis.patch.yml（关键）

purge **不在默认 bundles 里，必须 `insert:` 挂载**，否则宿主根本不加载它：

```yaml
- insert:
    - id: dsh-purge
      name: 'dsh-purge'
      inject: [ systemPrompt, tools ]   # 必须带 tools，否则审批门不变
      config:
        enabled: true
        autoApplyOnStart: false         # 启动不自动改宿主
        autoUpdateOnStart: false
        autoRevertOnMissing: false
        injectOnce: false               # 每轮都注入，防多轮后被冲掉
        stripMnemon: true
        verbose: false
        postPrompt: ""
        postPromptOrder: 5100           # 排在 baseline 之后
    - id: dsh-purge-redteam
      name: dsh-purge/redteam
    - id: dsh-purge-redteam-store
      name: dsh-purge/redteam/store
      config:
        root: !!js dshHomePath('redteam')
    - id: dsh-purge-redteam-ui
      name: dsh-purge/redteam/ui
```

**同时** `profiles/web/package.json` 的 `dsh.profile.bundles` 里也要有 `dsh-purge`。两处缺一不可。

---

## 第四步：启动

```powershell
cd D:\dsh-purge\runtime
$env:DSH_HOME = "D:\dsh-purge\runtime\dsh-home"
$env:DSH_PROFILE = "web"
$env:DSH_PROFILE_DIR = "D:\dsh-purge\runtime\dsh-home\profiles\web"
$env:DSH_BASE = ""
$env:DSH_WEB_URL = ""
$env:DSH_SESSION_ID = ""

node .\node_modules\@deepseek-ai\dsh\lib\bin.js web --port 3085 --no-open
```

打开 `http://127.0.0.1:3085`。

---

## 第五步：点「应用」——真正打补丁的一步

只写进 profile **不会**改 `@deepseek-ai`。必须：

1. 点**会话标题旁的 dsh-purge** 按钮（⚠️ **不在设置页里**，设置页永远找不到）；
2. 进「清洗」页 → 点 **「应用」**；
3. 成功后自动重启一次，补丁进入进程。

应用动作依次执行：

```text
备份原件(.dshpurge.bak) → 按补丁列表替换文件 → 覆盖 shim → 注入加密默认提示词
```

### 怎么确认真的应用了

本例的实盘证据（`$DSH_HOME/dsh-purge/applied.json`）：

```json
{
  "version": "1.1.71",
  "sha": "apply:1.1.71",
  "at": "2026-11-13T06:32:11.539Z"
}
```

```powershell
# 一条命令验证
Get-Content "$env:DSH_HOME\dsh-purge\applied.json"
Get-Content "$env:DSH_HOME\dsh-purge\installed-rev"   # apply:1.1.71
```

或在聊天里：

```text
/purge status        # 打出 DSH_HOME 与补丁列表，路径必须是你正在用的那份
```

---

## 第六步：配置提示词注入

三种来源，优先级从高到低：

```text
prompt-inject.md（规则库写入） > 勾选的 AGENTS.md/CLAUDE.md > 插件加密默认值
```

本例实测文件（**226 行 / 24KB**）：

```text
D:\dsh-purge\runtime\dsh-home\prompt-inject.md
```

编辑途径：

- 面板「提示词」框直接改（保存即写入 `prompt-inject.md`）；
- `/purge edit`；
- `/rules use <id>` 切换整套规则集；
- `/rules create <id>` 自建。

**插件支持按模型绑定不同提示词**——国模用内置「小码酱」那套，其他模型可另配，不必一套打天下。

> 本例 `override-state.json` 显示 `customized: false`，即**用的是官方自带加密默认值**，尚未自建。想定制就改这里。

---

## 第七步：三档 workspace baseline 策略

即使注入写好了，DSH 每轮还会追加 workspace baseline 稀释它。面板「工作区策略」：

| 策略 | 行为 | 适用 |
|---|---|---|
| `strict` | **无条件删掉** baseline 段 | 破甲最彻底 ✅ |
| `fallback`（默认） | 有用户注入时才删 | 平衡 |
| `always` | 保留 baseline | 不破甲 |

判断逻辑：注入非空 → 走 `fallback` 时才会去掉 baseline；选 `strict` 则永远去掉。

---

## 第八步：新开会话（最容易漏）

补丁和注入**只对新会话生效**。老会话的系统提示词已经发出去不会变，在旧对话里测必然误判成「没生效」。

---

## 完整闭环：破甲后怎么算「能正常使用」

破甲成功 ≠ 能正常使用。必须跑通这三件事：

### ① 接上模型

`cordis.patch.yml` 里指定默认 provider：

```yaml
- id: agent-default-model
  name: "@deepseek-ai/dsh-agent-default-model"
  config:
    provider: workbuddy2api
    model: deepseek-v4.1-flash
```

### ② 让工具调用不被审批门拦

这就是 `inject: [ systemPrompt, tools ]` 里 `tools` 的作用。只注入提示词不改 tools，模型愿意干活但工具调不动，表现为「说得好好的却不执行」。

### ③ 真跑一次工具调用验证

在新会话里让模型执行一个**真实文件操作**，例如：

```text
在当前目录建一个 hello.txt，写入 "purge ok"，然后读回来确认
```

三个都要成立才算通：

- ✅ 模型没拒绝、没加免责声明
- ✅ 工具真的执行了（文件真建出来了）
- ✅ 读回内容正确（不是幻觉）

### 本例的实盘佐证

3085 实例确有持续活跃的工作会话（15:31–15:36 连续写入），工作目录 `D:\AI-Agents-Workspace\harness\生图片`，说明破甲后实例在实际执行任务，不是空转。

---

## 常见坑

**面板里找不到 dsh-purge？**
在**会话标题旁**，不在设置页。装完必须完全退出宿主再开，热重载不加载新插件。

**点了「应用」模型还是拒绝？**
按序排查：① `/purge status` 路径对不对；② `prompt-inject.md` 是否非空；③ 工作区策略是否 `strict`；④ **是否新开会话**（最常见）；⑤ `injectOnce` 是否被误设 `true`；⑥ `inject` 里有没有 `tools`。

**模型愿意答但工具不执行？**
`inject` 缺少 `tools`，或者权限策略没放开。

**升级插件后失效？**
`npm install` 覆盖 `node_modules` 冲掉补丁。重新点「应用」。不建议用 `autoApplyOnStart: true`（启动静默改文件）。

**极简/PTC 被拦，标准模式正常？**
preset 里 `run_code` 或 plan 拦截句没洗净，或内置 minimal 缺 `agent-instructions`。升 1.1.61+，**完全退出 → 应用 → 自动重启 → 新开会话**。只换 preset 不重应用，旧进程补丁不更新。

**想还原？**

```text
/purge revert        # 用 .dshpurge.bak 覆盖回去并删备份
```

`prompt-inject.md` 是**用户文件，还原时保留**，反复试验不丢你自己写的提示词。

**两个实例账号池不一样？**
正常。网页登录的凭据按 `DSH_HOME` 存储不跨界；桌面端 auth 目录发现的账号才全局共享。要同步就复制 `.workbuddy2api-auth.<accountId>.json` 过去再刷新。

---

## 一句话总结

破甲 = **版本对上（0.2.0-rc.2 / purge 1.1.71）+ 注册对（insert + bundles 两处）+ 点应用（标题旁不是设置页）+ 注入非空 + baseline strict + 新开会话**，六条缺一表现都是「模型还是拒绝」。

能用 = **接上模型 + inject 带 tools + 真跑一次工具调用校验**。

---

**署名与致谢**

方法与提示词来自 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)（MIT）。按其署名要求，借用名称、思路、代码或提示词必须署名并标明来源仓库。

个人实验记录，非盈利，仅供技术参考与授权环境下的研究。
