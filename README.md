# DSH 破甲实操方法（网页版 dsh web + dsh-purge 1.1.71）

> 记录在本机把 **网页版 DeepSeek Harness（`dsh web`）** 破甲并跑通的完整方法。
> 亲测环境：DSH `0.2.0-rc.2` + `dsh-purge 1.1.71`，Windows，Node 22.x。
>
> ⚠️ 仅供已授权环境下的红队评测与鲁棒性研究。请遵守 [dsh-purge 的合规声明](https://github.com/YuJunZhiXue/dsh-purge)。

## 核心认知：破甲是三层，不是一句话

很多人以为往系统提示词里塞一段越狱文案就完事了。实际 DSH 的拦截在**三个层次**上，只改提示词会被另外两层吃掉：

| 层 | 位置 | 作用 | 不改的后果 |
|---|---|---|---|
| **① 宿主补丁** | `@deepseek-ai/*` 包文件 + shim | 改审批门、沙箱、提示词拼接逻辑 | 工具调用被审批门拦住 |
| **② 提示词注入** | `$DSH_HOME/prompt-inject.md` | 真正决定模型身份与规则 | 模型仍按官方人格回答 |
| **③ 工作区 baseline** | DSH 自动拼的 workspace 段 | 每轮往系统提示词追加策略 | 冲淡注入，触发拦截 |

**dsh-purge 的价值就是把这三层一起处理**，并且可还原。

## 版本矩阵（最重要的一条）

`dsh-purge` 的 `dshTarget` 必须与 DSH 版本严格匹配，对不上就「待应用/跳过」，不会乱改文件：

| DSH 版本 | 用哪个 purge |
|---|---|
| `0.2.x`（含官方桌面 0.2.0-rc.2、官方 `dsh web`） | **1.1.40+**（本例 1.1.71） |
| `0.1.7`（rc.1 / rc.2） | **1.1.39 及更早** |

`dsh-purge 1.1.71` 的 `package.json` 里写死：

```json
"dshTarget": "0.2.0-rc.2"
```

所以 **DSH 必须是 0.2.0-rc.2**。这是本例能跑通的前提。

## 完整步骤

### 1. 确认宿主是 dsh 0.2.0-rc.2

```powershell
node .\node_modules\@deepseek-ai\dsh\lib\bin.js --version
# 期望输出：0.2.0-rc.2
```

> 官方桌面 EXE 与官方 `dsh web` **分开装、分开点「应用」**。本文只讲 `dsh web`。

### 2. 装插件到 web profile

```sh
dsh plugin --profile web add dsh-purge
```

`dsh` 不在 PATH，或 registry 上不去时：

```sh
dsh plugin --profile web add https://github.com/YuJunZhiXue/dsh-purge/archive/refs/heads/master.tar.gz
```

装完验证（会在 `profiles/web/node_modules` 下出现 `dsh-purge`）：

```powershell
Get-Content .\profiles\web\node_modules\dsh-purge\package.json | Select-String 'version|dshTarget'
```

### 3. 在 cordis.patch.yml 注册（关键）

`dsh-purge` 不在默认 bundles 里，**必须用 `insert:` 挂载**，否则宿主不会加载它：

```yaml
- insert:
    - id: dsh-purge
      name: 'dsh-purge'
      inject: [ systemPrompt, tools ]
      config:
        enabled: true
        autoApplyOnStart: false     # 启动不自动改宿主，安全
        autoUpdateOnStart: false
        autoRevertOnMissing: false
        injectOnce: false           # 每轮都注入，不是只注一次
        stripMnemon: true
        verbose: false
        postPrompt: ""
        postPromptOrder: 5100
    # 演练台三个子行（只做破甲可省略）
    - id: dsh-purge-redteam
      name: dsh-purge/redteam
    - id: dsh-purge-redteam-store
      name: dsh-purge/redteam/store
      config:
        root: !!js dshHomePath('redteam')
    - id: dsh-purge-redteam-ui
      name: dsh-purge/redteam/ui
```

**几个参数为什么这么设：**

- `inject: [ systemPrompt, tools ]` —— 同时改系统提示词和工具清单（不改 tools 的话工具上限与审批门仍是默认）；
- `autoApplyOnStart: false` —— 别让它在启动时偷偷改包文件，必须人工点「应用」，出问题能立刻停；
- `injectOnce: false` —— 每轮都重新注入，防止多轮对话后注入被上下文冲掉；
- `postPromptOrder: 5100` —— 注入位置排在 workspace baseline 之后，保证优先级。

同时 `package.json` 的 `dsh.profile.bundles` 里也要有 `dsh-purge`，两处缺一不可。

### 4. 重启宿主

```powershell
# 完全退出旧进程再起，热重载不加载新插件
node .\node_modules\@deepseek-ai\dsh\lib\bin.js web --port 3085
```

### 5. 点「应用」—— 真正打补丁的一步

只写进 profile **不会**改 `@deepseek-ai` 包。必须：

1. 打开 `http://127.0.0.1:3085`；
2. 点**会话标题旁的 dsh-purge** 按钮（注意：**不在设置页里**，设置页找不到）；
3. 进「清洗」页 → 点 **「应用」**；
4. 成功后会自动重启一次，补丁进入当前进程。

应用过程做四件事（自动备份 `.dshpurge.bak`）：

```text
备份原件 → 按补丁列表替换文件 → 覆盖 shim → 注入加密默认提示词
```

验证是否真的生效：

```text
在聊天里输入  /purge status
```

应能打出 `DSH_HOME` 路径和补丁列表，路径必须是你正在用的那一份。

### 6. 写提示词注入

这是破甲的**内容层**。三种来源，优先级从高到低：

```text
prompt-inject.md（规则库写入） > 勾选的 AGENTS.md/CLAUDE.md > 插件加密默认值
```

文件位置（本例实测，24KB）：

```text
D:\dsh-purge\runtime\dsh-home\prompt-inject.md
```

编辑方式：

- 面板里「提示词」框直接改（保存即写 `prompt-inject.md`）；
- 或 `/purge edit`；
- 或 `/rules use <id>` 切换整套规则集。

可换模型换提示词——`dsh-purge` 支持**按模型绑定不同注入**，国模用默认「小码酱」那套，其他模型可另配。

### 7. 处理 workspace baseline（第三层）

即使注入写好了，DSH 每轮还会往系统提示词追加一段 workspace baseline，把注入冲淡。面板里的「工作区策略」三档：

| 策略 | 行为 | 何时用 |
|---|---|---|
| `strict` | **无条件删掉** DSH baseline 段 | 破甲最彻底，推荐 |
| `fallback`（默认） | 有用户注入时才删 | 平衡 |
| `always` | 保留 baseline | 不破甲 |

选了 `strict` 或 `fallback` 且已有注入，baseline 才会被去掉。

### 8. 新开一轮对话

补丁和注入都只在**新会话**生效。老会话里已经发出的系统提示词不会变，别在旧对话里测，会误判成「没生效」。

## 常见坑

**Q：面板里找不到 dsh-purge 按钮？**
在**会话标题旁**，不在设置页。装完必须**完全退出**宿主再开，热重载不加载新插件。

**Q：点了「应用」但模型还是拒绝？**
按顺序排查：① `/purge status` 路径是否指向你在用的宿主；② `prompt-inject.md` 是否非空；③ 工作区策略是否设成 `strict`；④ **是否新开了对话**（最常见）；⑤ `injectOnce` 是否被误设成 `true`。

**Q：桌面端改了 web 没改 / 反之？**
两个宿主独立，`apply` 只作用于点它的那个。不要在一个宿主里点另一个宿主。

**Q：升级插件后破甲失效？**
`npm install` 覆盖 `node_modules` 会冲掉补丁。重新在「清洗」里点「应用」，或用 `autoApplyOnStart: true`（不推荐，启动静默改文件）。

**Q：极简/PTC 模式被拦，标准模式正常？**
通常是 preset 里 `run_code` 或 plan 拦截句没洗净，或内置 minimal 缺 `agent-instructions`。升到 1.1.61+，**完全退出 → 应用 → 自动重启 → 新开一轮**。只换 preset 不重应用，旧进程里的补丁不会更新。

**Q：想还原？**
```sh
/purge revert     # 用 .dshpurge.bak 覆盖回去并删备份
```
`prompt-inject.md` 是用户文件，还原时**保留**。这样反复试验不会丢你自己写的提示词。

## 一句话总结

破甲 = **版本对上（0.2.0-rc.2 / purge 1.1.71）+ 注册对（cordis.patch.yml 用 insert）+ 点应用（会话标题旁，不是设置页）+ 注入非空 + baseline strict + 新开会话**。
六条缺一条，表现都是「模型还是拒绝」。

---

**署名与致谢**

本项目的方法与提示词均来自 [YuJunZhiXue/dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)（MIT）。按其署名要求，借用其名称、思路、代码或提示词必须署名并标明来源仓库。

个人实验记录，非盈利，仅供技术参考与授权环境下的研究。
