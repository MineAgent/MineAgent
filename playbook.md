# Minecraft 通关操作手册（种子 `-818810680825819701`）

> 面向模型的操作手册。窗口固定 854×480（guiScale=2）。
> 接口根地址：`http://127.0.0.1:3420`；下面所有命令都直接在 shell 里执行。

---

## 1. 开工自检与观测对账

### 1.1 卡住先自检

先分清「没做成」还是「没读对」：

```sh
curl -s http://127.0.0.1:3420/                       # 接口索引；连不上 = 服务没起
for p in ctl aif op bt ka; do
  curl -s -o /dev/null -w "$p %{http_code}\n" -I "http://127.0.0.1:3420/$p/"
done
curl -s http://127.0.0.1:3420/aif/info                # 409 / no world loaded = 未进世界
curl -s http://127.0.0.1:3420/ka/state                # 自动攻击状态；enabled 默认 false
curl -s http://127.0.0.1:3420/aif/msg                 # 上一步的回显
```

* `GET /` 通但某个前缀不是 200：对应模组没挂上。
* `/aif/info` 返回 `no world loaded (still on a menu?)`：客户端在主菜单或加载界面。
* 语法拿不准时读权威全文：`GET /ctl/`、`GET /aif/`、`GET /op/`、`GET /bt/`、`GET /ka/`（**内容较长，按需读**）。

装了哪些模组，决定了能用哪些手段：

| 模组 | 没有它时的后果 |
| --- | --- |
| **MGHttpdProvider**（必需） | 端口 3420 上什么都没有，mcctl 和 AdvancedInfoFetcher 都挂不上 |
| **mcctl**（必需） | 什么都做不了 |
| AdvancedInfoFetcher | 只能靠截图判断背包、血量、聊天回显、声音、准星目标方块和光照 |
| cmdCraft | 合成/冶炼只能去点 GUI 格子（有 `/ctl/mouse` + `mouse goto` 兜底，但仍容易点错） |
| KillAura | 只能自己按 `mouse left` 砍怪 |
| unlockRecipe（服务端模组） | 配方书是空的：`craft` 只能做**已经解锁**的配方，其余一律 `400 找不到可以合成 <物品> 的配方（配方未解锁或不存在）` |
| Baritone + MGBaritoneConnector | 没装 Baritone 只能一步步按键走路、手动挖矿；只装 Baritone 而没有连接器时 `POST /bt` 直接 404，`bt` 命令完全不可用 |

`/bt` 由 MGBaritoneConnector 提供（请求体就是一条 Baritone 命令），它硬依赖 Baritone。

### 1.2 观测工具与读法

| 接口 | 用途 | 读法要点 |
| --- | --- | --- |
| `GET /aif/info` | 坐标/朝向/血量/饱食度/准星目标方块 | 判断“有没有真的在动”只能靠它；瞄准/放置前也用它确认落点 |
| `GET /aif/world` | 时间/天气/维度 | 时间过 12000 是黄昏，`thunder` 别在外面晃 |
| `GET /ctl/prtsc` | 界面/地形确认 | 可能有延迟，两张一样不代表命令失败；一屏能答的问题别连截三张 |
| `GET /aif/msg` | 指令输出、Baritone 输出、报错 | 不要靠截图猜 |
| `GET /aif/keysnd` | 判断挖到没有/附近有没有怪 | 读完即清空，不要连读 |
| `GET /aif/inventory` | 背包/容器对账 | 界面打开时会额外显示容器内容 |
| `GET /ctl/mouse` | 光标位置 | 只在必须手点 GUI 时用 |
| `GET /ka/state` | 自动攻击的目标与命中记录 | `target:null` = 范围内没有合法目标；`lastAttack` 为最近一次实际攻击 |

截图：

```sh
curl -s -o /tmp/mc.png http://127.0.0.1:3420/ctl/prtsc
```

**`GET /aif/info`** 的字段：

| 字段 | 说明 / 用途 |
| --- | --- |
| `玩家` | 用户名 |
| `坐标` | 精确坐标，2 位小数；记下基地、熔炉、矿洞口的坐标 |
| `方块` | 所在方块坐标（整数） |
| `目标方块` | 准星指向方块的完整命名空间 ID（如 `minecraft:oak_log`）；天空/实体/空气/命中结果未刷新时为 `未选中` |
| `光照` | 目标方块位置的原始光照 0–15（不做昼夜天空光衰减，不是相邻面）：方块光为 0 时 `光照：天空光 <n>（此方块不发光）`，否则 `光照：<最大值> (方块光 <n> / 天空光 <n>)`；`未选中` 时没有这一行 |
| `方位` / `yaw` / `pitch` | 朝向角度，1 位小数；pitch=90 是正下方，pitch=-90 是正上方 |
| `选中` | 快捷栏选中格 1–9 |
| `生命值` / `饱食度` / `饱和度` | 危险判断：血少就撤，饿了自己找吃的 |
| `效果：` | 仅有效果时出现；每行 `<效果ID> <等级> <剩余秒数>` |

死亡时 `生命值：0.0`，`坐标` 仍是**死亡地点**；要在重生前读一次并记下。

放置、右键、挖掘前先读 `目标方块`：`未选中` 说明准星不在方块上，用 `look` 调，不要靠截图猜。

**`GET /aif/world`** 的字段：

| 字段 | 说明 / 用途 |
| --- | --- |
| `维度` | 维度命名空间 ID；进下界/末地后看这个确认自己到了哪 |
| `时间` | 主世界时钟 `0`–`23999`：`0`=清晨、`6000`=正午、`12000`=黄昏、`18000`=午夜；**判断天黑没黑看这个** |
| `天数` | 已经过的天数（主世界计数） |
| `游戏刻` | 世界创建以来的总 tick 数（20 tick = 1 秒），两次读数可以算经过时间 |
| `天气` | `clear` / `rain` / `thunder` |

没有昼夜循环的维度（下界/末地）`时间` 返回 `不可用`；`天数`/`游戏刻` 仍是主世界的计数。

**`GET /aif/inventory`** 的段：

| 段 | 说明 |
| --- | --- |
| `背包：` | 主背包 + 快捷栏合并后按命名空间 ID 聚合（同一物品跨格相加），按 ID 排序 |
| `副手：` / `盔甲：` | 只有非空时才出现，空栏整段消失 |
| `原料：` / `燃料：` / `产物：` | 熔炉三个槽位；空槽输出 `空` |
| `燃烧：` / `烧炼：` | 燃料剩余比例 / 当前烧炼进度，`0.00`–`1.00` |
| `箱子：` | `类型：` 是菜单命名空间 ID，`容量：` 是格数（小箱子/木桶 27，大箱子 54） |
| `<槽位号>：` | 箱子槽位从 1 开始，空槽跳过 |

* `熔炉：` / `箱子：` **只有对应界面打开时才出现**；未打开时整段消失。
* **看不到工具耐久**，也看不到附魔。潜影盒不支持。
* 聚合可靠，可以做精确对账：动作前后各读一次，看材料少了多少、产物多了多少。

**`GET /aif/msg`**：只返回上一次请求之后新出现的聊天内容，读完即清空。

* 范围是聊天栏里的一切：玩家聊天、指令输出、Baritone 输出与报错、加入/退出/死亡提示。
* 空结果 = 没有新消息（`200` 空正文）。
* 节奏：发完动作 `sleep 1`–`2` 再读；读过的不会重复出现。
* `HEAD /aif/msg` 会白白吃掉回显，所以只允许 `GET`。

**`GET /aif/keysnd`**：只返回上一次请求之后新播放的**重要声音**，读完即清空。

* 保留方块破坏/放置、挖掘命中、怪物叫声、爆炸、开关门、拾取、受伤等。
* 过滤脚步、音乐、环境音、UI 音效、天气音效。
* 音量是请求音量，不随距离衰减，不能当“远近”用；判断距离看 `/aif/info`。
* 和 `/aif/msg` 一样是增量，`HEAD` 返回 405。

### 1.3 决策循环

```text
① 观察  GET /aif/info + /aif/inventory + /aif/world + /ctl/mouse（自动攻击时 + /ka/state）
② 决策  结合目标，判断当下情况
③ 执行  POST /ctl（按键/鼠标/聊天）、POST /bt（Baritone）、POST /op（合成/冶炼/存取/转向）、POST /ka（开关自动攻击）
④ 校验  POST /op 看 HTTP 响应；POST /ctl·/bt 后读 /aif/msg，再读一次状态对账
   ↺ 回到 ①
```

* **一次只做一个动作**，别把「挖矿 + 合成 + 冶炼」塞进一个请求里。
* 材料够不够先读 `/aif/inventory`，别凭记忆；目标优先级：活下来 ＞ 攒装备 ＞ 赶到传送门 ＞ 打龙。
* `POST /op` 返回的正文就是结果或原因，按原因补齐材料、界面状态再试。
* 拿不准的操作先发一次看回显（例如不确认物品 id），再决定下一步。

### 1.4 对账、回显与上下文纪律

* `bt mine` 没有数量参数：定期看 `/aif/inventory`，够了立刻 `POST /bt 'stop'`。
* 长时间任务定期 `GET /aif/info`；掉血、有怪、天黑就撤。
* 记坐标：基地、工作台、熔炉、矿洞口、传送门。
* 上下文预算：`/ctl/prtsc` 和 `GET /ctl/`、`GET /aif/`、`GET /op/`、`GET /bt/`、`GET /ka/` 全文都很大，按需取。
* 读 `/aif/msg`：指令输出、Baritone 输出、报错都在里面，不要靠截图猜。
* 读 `/aif/keysnd`：判断挖到没有/附近有没有怪，读完即清空，不要连读。
* 读 `/aif/world`：时间接近 12000 就该停止户外作业。

---

## 2. 操作规范

### 2.1 GUI 纪律

* 按钮一律用键盘：`POST /ctl 'TAB'` 移焦点、`POST /ctl 'ENTER'` 激活；主菜单/选择世界直接 `ENTER`。
* 格子操作用 `POST /op`，不要点格子。
* 界面打开时 `GET /aif/inventory` 会额外显示容器内容；`POST /ctl 'ESC'` 关闭。
* 除了 fov，不要擅自改游戏设置，特别是键位。

**输入通道实测结论**

| 类别 | 按键 | 通过 HTTP | 说明 |
| --- | --- | --- | --- |
| isDown（按住持续生效） | `W` `A` `S` `D`、`mouse left/right` | ✅ | 走路、挖掘、攻击 |
| consumeClick（点按触发一次） | `E`、`1`–`9`、`Q` | ✅ | 开背包 / 切格 / 丢弃 |
| 特判 | `esc`、`F3` | ✅ | `esc` 关界面，世界里开暂停菜单 |
| 滚轮 | `mouse scroll n` | ✅ 但一次只走一格 | 切格用 `POST /op 'inventory …'` 更省事 |

发文本一律用 `POST /ctl 'chat <文本>'`；Baritone 命令走 `POST /bt '<命令>'`。**不打开聊天框打字**。

聊天框开着时：

* `E` / `Q` / `1`–`9`：**自动关掉聊天框**，再执行开背包 / 丢弃 / 切格。
* 方向键 / `BACKSPACE` / `ENTER` / `TAB`：作用在聊天框（编辑 / 发送 / 补全）。
* `esc`：关掉聊天框。
* `W` `A` `S` `D` `SPACE` `SHIFT`：**不关聊天框**，直接控制游戏。
* `mouse move` / `left` / `right` / `scroll`：直接作用于世界。

**手点鼠标是兜底**（只在 `POST /op` 不可用时）：

```sh
curl -s http://127.0.0.1:3420/ctl/mouse
curl -s -X POST --data-binary 'mouse goto 427 240' http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'mouse left'          http://127.0.0.1:3420/ctl
```

* `光标` 是窗口像素坐标（和 `/ctl/prtsc`、下面坐标表同一坐标系）。
* `抓取：是` 表示鼠标被游戏锁住（在世界里），位置没有意义；`界面` 是当前界面类名。
* 若 `/ctl/mouse` 报 **`不在窗口内，请使用 mouse goto <x> <y>`**：GLFW 已不更新位置，旧坐标不作数；先 `mouse goto` 再点。
* 界面刚打开时光标在窗口中心（854×480 下是 `427 240`），之后可能已被移动，现读现用。
* `/ctl/prtsc` 截的是**游戏帧，不含系统光标**；想知道鼠标在哪就读 `/ctl/mouse`。
* `mouse move` 的相对位移会累积误差，不要用它点格子。
* 死亡画面的「重生」：`TAB` + `ENTER`；「选择世界」界面世界条目要**先选中**。

**手点兜底坐标表（854×480，仅在没有 cmdCraft 时）**

HUD：准星 `(427,240)`；快捷栏第 i 格 `x = 266 + 40×(i-1)`，`y = 458`。

工作台界面：

| 元素 | 坐标 |
| --- | --- |
| 3×3 合成格 | `(325,123) (361,123) (397,123)` / `(325,159) (361,159) (397,159)` / `(325,195) (361,195) (397,195)` |
| 输出槽 | `(513,159)` |
| 玩家 3×9 | `x = 281 + 36c`（c=0…8），`y = 257 / 293 / 329` |
| 界面内快捷栏 | `y = 373` |

熔炉界面：`raw (377,123)`、`fuel (377,195)`、`product (497,159)`。

转向优先用 `POST /op 'look …'`。只有在没有 cmdCraft 时才退回像素换算：`mouseSensitivity=0.5` 时每 1 像素 ≈ **0.15°**，转 θ 度用 `dx ≈ θ/0.15`；转完读 `/aif/info` 核对。

### 2.2 放置方块与开容器

站定 → `POST /op 'look pitch 40'` 瞄 2~3 格外地面 → `POST /ctl 'mouse right'` 放置 → 再 `mouse right` 一次打开 → `POST /op` 操作 → `ESC` 关闭。

```sh
curl -s -X POST --data-binary 'look pitch 40' http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
```

* 脚下那块方块上无法放置；目标格也不能被自己的碰撞箱占据。窄隧道里贴脸放常失败，先后退 1–2 格。
* 放完读 `GET /aif/keysnd` 确认：出现 `minecraft:block.stone.place`（或对应方块）就是放成功了；只有右键音、没有放置音就是没放上。
* 右键方块没反应时：先看 `GET /aif/info` 的 `目标方块`；`未选中` 就是准星没落在方块上，用 `look` 调 `pitch` / `yaw`。仍对不上再用 `F3` 调试屏截图。

### 2.3 Baritone 使用纪律

Baritone 命令一律 `POST :3420/bt`，请求体就是命令本身：

```sh
curl -s -X POST --data-binary 'stop'                  http://127.0.0.1:3420/bt
curl -s -X POST --data-binary 'goto 1697 -2 1124'     http://127.0.0.1:3420/bt
curl -s -X POST --data-binary 'mine hay_block'        http://127.0.0.1:3420/bt
```

下文把 `POST /bt '<命令>'` 简写成 `bt <命令>`（`bt stop` 就是 `POST /bt 'stop'`）。

* 一次请求只执行**一条**命令；多行请求返回 400，拆成多个请求。
* Baritone 有任务状态；换目标或死亡后先 `bt stop`，再看回显。
* 天黑后不要 `bt goto` 长途：Baritone 不还手，会被一路跟死。
* `bt mine <方块>` 的危险：`bt mine chest` 会认定地下箱子更近，直接挖竖井；`bt mine red_bed` 同样会往下钻。
* 只有确认目标在地表时才用 `bt mine`；使用时每 10 秒 `GET /aif/info`，y 掉超过 5 格立刻 `bt stop`。
* 找容器/床/矿石不用 `bt mine`，改成“走过去 + 右键 + `GET /aif/inventory` 读容器”。
* **无树区域不要用 `bt mine oak_log`**：它不会放弃，而是向下挖去找废弃矿井的支撑原木，又慢又危险；先在地表用 `bt goto <远处坐标>` 定向找森林/村庄。
* `bt mine` 的数量控制见 §1.4；`bt stop` 后必须对账。
* Baritone 的任务跨死亡存活；死亡/中断后第一件事就是 `bt stop` + `POST /ctl 'release'`。
* 本版本 Baritone **没有 `craft` 命令**；长距离 `bt goto` 会消耗大量饱食度，出发前备食物。

### 2.4 Kill Aura

Kill Aura 自动攻击 `range` 内的生物，没有游戏内按键，开关和配置全部走 `/ka`；只影响客户端，不修改服务端规则。

```sh
curl -s http://127.0.0.1:3420/ka/state                # 状态 JSON
curl -s -X POST http://127.0.0.1:3420/ka/on           # 开（幂等）
curl -s -X POST http://127.0.0.1:3420/ka/off          # 关（幂等）
curl -s -X POST http://127.0.0.1:3420/ka/toggle
curl -s -X POST http://127.0.0.1:3420/ka/reload       # 重读 config/killaura.json
curl -s -X POST -H 'Content-Type: application/json' \
     --data-binary '{"range":4.0,"denyTargets":["minecraft:creeper"]}' \
     http://127.0.0.1:3420/ka/config
```

状态 JSON 的字段：

| 字段 | 说明 |
| --- | --- |
| `enabled` / `inWorld` | 是否开启 / 是否已进世界 |
| `player` | 玩家名；未进世界时为 `null` |
| `config` | 当前生效的配置 |
| `target` | 现在会攻击的目标 `{name, id, distance, health}`；未开启、未进世界或没有合法目标时为 `null` |
| `lastAttack` | 最近一次实际攻击 `{name, agoMs}`；本次启动还没打过时为 `null` |

所有写操作成功后返回同一份状态。`target.name` / `lastAttack.name` 是本地化显示名。

配置字段（`config/killaura.json`，与 `POST /ka/config` 的字段同名）：

| 字段 | 默认 | 含义 |
| --- | --- | --- |
| `range` | `3.0` | 攻击距离，0.5–6.0（原版 3.0），按包围盒最近点算 |
| `fov` | `360.0` | 三维视野锥角，1–360；360 = 不限制 |
| `rotation` | `"visible"` | `silent` 只发转向包 / `visible` 直接改本地视角 / `none` 不转向 |
| `priority` | `"closest"` | 档位内排序：`closest` 距离 / `lowestHealth` 血量 / `angle` 偏角 |
| `minAttackCharge` | `0.0` | 蓄力 ≥ 该值才出手；1.0 = 只打满蓄力 |
| `targetPlayers` | `true` | 打玩家（旁观/创造除外） |
| `attackHostile` | `true` | 打敌意生物 |
| `attackFriendlyDrops` | `false` | 打友善且有掉落（牛、猪、鸡…） |
| `attackFriendlyNoDrops` | `false` | 打友善且无掉落（村民、蝙蝠…） |
| `denyTargets` | `[]` | 永不攻击的实体类型 ID |
| `alwaysTargetsHigh` / `Medium` / `Low` / `Lowest` | `[]` | 四档强制目标（高 → 低） |
| `targetArmorStands` | `false` | 打盔甲架 |
| `ignoreInvisible` | `false` | 忽略隐身目标 |
| `throughWalls` | `false` | 无视视线遮挡 |
| `pauseInScreens` | `true` | 打开界面时暂停 |
| `pauseWhileUsingItem` | `true` | 使用物品时暂停 |

`POST /ka/config` 只改出现的字段，全有或全无：任一字段非法返回 400，文件不写；`_help` 只读。

目标分为敌意 / 中立 / 友善有掉落 / 友善无掉落；`attackHostile` / `attackFriendlyDrops` / `attackFriendlyNoDrops`
只控制普通目标，中立生物只能由 `alwaysTargets*` 命中；蜘蛛/洞穴蜘蛛按光照在敌意与中立之间切换。

**是否攻击的判定顺序**：`denyTargets` → 任一 `alwaysTargets*` → `targetPlayers` / `targetArmorStands` /
`ignoreInvisible` / `throughWalls` → `attackHostile` / `attackFriendlyDrops` / `attackFriendlyNoDrops`。
强制目标绕过玩家/盔甲架/隐身/视线检查，但仍受 `range`、`fov`、暂停、蓄力限制；拒绝与强制同时命中时拒绝优先。

**多个合法目标的选择顺序**：`alwaysTargetsHigh` ＞ 普通敌意 ＞ `alwaysTargetsMedium` ＞ 普通友善有掉落 ＞
`alwaysTargetsLow` ＞ 普通友善无掉落 ＞ `alwaysTargetsLowest`。同一档内才按 `priority` 评分。
玩家/盔甲架没进 always 名单时按「友善无掉落」档排序。

配置在开启（`POST /ka/on`）时重读一次；运行中手改文件后 `POST /ka/reload` 生效。开关状态不持久化，重启后为关闭。

### 2.5 合成与错误处理

* 合成返回「配方未解锁」：服务端没装 `unlockRecipe` 时配方书要自己攒；先凑材料解锁。
* `POST /op 'craft …'` 报「请先放下鼠标光标上持有的物品」：合成产物卡在光标上，`ESC` 关界面再开一次。
* `POST /ctl` 发完没反应：命令排队执行；定时动作配 `sleep` 或 `delay`，或隔一步再对账。

五类 cmdCraft 命令，全部用 `POST :3420/op`，请求体就是命令本身（**不套 `chat`、不带 `/cmdop`**）：

| 命令 | 说明 |
| --- | --- |
| `craft <物品id> [数量]` | 用背包里的材料合成（走原版合成格） |
| `inventory <物品id> [1-9]` | 把物品换到快捷栏第 N 格 |
| `furnace put <raw/fuel> <物品id> [数量]` | 往打开着的熔炉里放 |
| `furnace get <raw/fuel/product> [数量]` | 从打开着的熔炉里取 |
| `chest put <物品id> [数量]` | 往打开着的箱子里放 |
| `chest get <物品id> [数量]` | 从打开着的箱子里取 |
| `look <yaw/pitch> <数值>` | 转视角（支持 `~` 相对值） |

**`craft`**

```sh
curl -s -X POST --data-binary 'craft stone_pickaxe'   http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'craft oak_planks 8'    http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'craft crafting_table'  http://127.0.0.1:3420/op
```

* `数量` 指**产物个数**，会自动向上取整到整数次合成；结果就是响应正文。
* 前置条件：**打开着合成界面**（工作台 3×3，或背包 2×2）；合成格空；光标空。
* **2×2 vs 3×3**：`POST /ctl 'E 50'` 开背包只能做 2×2。**镐/剑/斧/熔炉/铁砧是 3×3**，在背包里做会得到 `400 该配方需要 3×3 的工作台`。
* 工作台自己是 2×2 配方：`craft oak_planks 4` → `craft crafting_table`；把工作台放下去、右键打开，就能做 3×3。
* `craft` 会等合成真正做完才返回；连着发两条不用等间隔（撞上还在跑的合成会得到 `400 已有合成任务正在进行中。`）。
* 合成格里有东西会报「合成格不为空」——**关掉界面再打开**就清空了，不需要用鼠标一个个点回背包。
* 材料不够会报 `400 原料不足：<物品> 需要 N 个`——先读 `/aif/inventory` 备齐，别凭记忆（木棍常常是漏项）。
* 配方必须已在客户端配方书里（原版规则：拿到材料才解锁）。装了 `unlockRecipe` 的存档一进世界就是整本配方书；没装就只能自己先解锁，对未解锁的配方返回 `400 找不到可以合成 <物品> 的配方（配方未解锁或不存在）`。
* 任一检查不过就什么都不做，不会半途消耗材料。

**`inventory`**

```sh
curl -s -X POST --data-binary 'inventory torch 2'         http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'inventory iron_pickaxe 1'  http://127.0.0.1:3420/op
```

会把该格原有物品「换出」回主背包，响应会提示「已对调：快捷栏第 N 格 ← X ×N（换出 Y ×N）」。别不小心把正在用的工具换走。

**`furnace`**

```sh
curl -s -X POST --data-binary 'furnace put raw raw_iron 8'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'furnace put fuel coal 4'     http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'furnace get product'         http://127.0.0.1:3420/op
```

* 槽位名就是 `raw` / `fuel` / `product`；**必须开着熔炉界面**，否则报「请先右键打开一个熔炉」。
* `get raw` / `get fuel` 可以按数量拆；`get product` 不能拆，要多少都会整堆取出。
* 燃料：煤 8、木板/原木 1.5、木棍 0.5 个物品。

**`chest`**

```sh
curl -s -X POST --data-binary 'chest put cobblestone 64'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'chest get iron_ingot 16'   http://127.0.0.1:3420/op
```

* **必须开着箱子界面**（箱子/陷阱箱/大箱子/木桶）。
* `get` 是严格的：箱里不够要的数量就报错，不会只拿一部分。

**`look`**

```sh
curl -s -X POST --data-binary 'look yaw 90'    http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'look yaw ~-20'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'look pitch 40'  http://127.0.0.1:3420/op
```

* `90` = 设为 90°，`~0.1` = 当前值 +0.1，`~-20` = 当前值 −20，`~` = 保持不变。
* 改的就是 `/aif/info` 读到的 `yaw` / `pitch`，比 `mouse move` 换算像素精确；要转到确定方向就用它。
* `pitch` 会夹到 `-90..90`；`yaw` 按给定值存、可以超过 ±180。
* 响应直接给出实际值（`已转向 yaw：90.0（原 -60.9）`），发完读 `/aif/info` 核对。

**常用报错**

| 报错 | 处理 |
| --- | --- |
| `请先放下鼠标光标上持有的物品` | 产物卡在光标上：`ESC` 关界面再开 |
| `合成格不为空` | 关掉合成界面再打开，东西会自动退回背包 |
| `该配方需要 3×3 的工作台` | 先放工作台并右键打开 |
| `已有合成任务正在进行中` | 等上一条 `craft` 返回再发 |
| `配方未解锁或不存在` | 先凑材料解锁，或换别的做法 |
| `原料不足：…` | 读 `/aif/inventory` 补材料（木棍常漏） |
| `请先打开背包或工作台界面` | 用 `E 50` 开背包，或右键工作台 |
| `409` | 客户端还没进世界（主菜单/加载界面） |

---

## 3. 生存纪律

> 铁律：先保命，再推进度。血不满、没食物、没火把、没护甲，不下矿。

### 3.1 优先级

活下来 ＞ 重生点 ＞ 基础工具 ＞ 铁装备 ＞ 赶到传送门 ＞ 打龙

### 3.2 饱食度与回血

* 饱食度 < 18 不回血；掉血后第一件事是吃食物到 20（并攒饱和度）再干活。
* Baritone 冲刺约 40 格/1 点，满饱食度约 800 格；长途前先备食物，< 10 不做长距离移动。
* 面包（3 小麦）= 5 饱食度，是村庄里最容易量产的食物；平原上的牛/羊是稳定食物来源，羊毛留着做床。

### 3.3 光源起步

木头 → 木板 → 熔炉（8 圆石，需先有木镐）→ 原木 + 木板当燃料烧出木炭 → 火把（木炭 + 木棍）。

### 3.4 过夜

时间 > 12000 是黄昏/过夜节点，唯一真正的死因。

* 村民的床可直接右键。白天睡不了，但重生点会被记下。
* 天黑后不要再 `bt goto` 长途：Baritone 不还手，会被一路跟死。
* 如果村民已经睡上去了，可能无法正常设置重生点。一般情况下有村民时右键会把村民赶走，再右键就能设置重生点，但村民下来的时候可能会推动一下角色。

### 3.5 撤退条件

任一出现就 `POST /bt 'stop'` 撤回安全处：

掉血、火把用完/周围全黑、听到/看到怪物、饱食度见底、天黑了还在地表。

---

## 4. 死亡处理

死亡不是进度结束，而是**一个必须在 5 分钟内执行的回收任务**。

1. `POST /bt 'stop'` + `POST /ctl 'release'` —— Baritone 任务状态跨死亡存活。
2. 立刻 `GET /aif/info` 记死亡坐标（此时仍是死亡地点，血量 0）。
3. 重生：`POST /ctl 'TAB'` + `POST /ctl 'ENTER'`。
4. `POST /bt 'goto <x> <y> <z>'` 回去回收；死在深层可能要挖穿地形，预留时间。
5. 到了先 `GET /aif/inventory` 对账，再决定下一步；回收或放弃都明确记下结论和坐标。

```sh
curl -s -X POST --data-binary 'stop'     http://127.0.0.1:3420/bt
curl -s -X POST --data-binary 'release'  http://127.0.0.1:3420/ctl
curl -s http://127.0.0.1:3420/aif/info
curl -s -X POST --data-binary 'TAB'      http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'ENTER'    http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'goto 100 64 200' http://127.0.0.1:3420/bt
curl -s http://127.0.0.1:3420/aif/inventory
```

**掉落物按“区块加载时长”计时（约 5 分钟）**：区块卸载时计时暂停，实际窗口比 5 分钟宽松。

---

## 5. 本种子情报与通关路线

### 5.1 出生点

约 `(9, 64, 3)`：草原/草甸，零散橡树，附近一条河（水塘）。

### 5.2 村庄（前期发育点）

* 位置约 `(375, 68, 216)`；从出生点走 `(368, 224)` 约 440 格，Baritone 冲刺约 100 秒。
* 村内主街 `x≈371`，`z 180~210`；核心 `y≈67~70`，东侧低一档台地 `y≈50~55`（找东西注意上下层）。
* 已知资源：
  * 已知的一个有床房间：房间门前台阶坐标 `(374, 67, 185)`。`look yaw -90` 后右键开门，然后走到 `(376, 67, 185)` 后 `look yaw 60` 右键关门，再朝向 `yaw 180, pitch 30` 右键设置重生点（如果是晚上还可以去睡觉）。
  * `(376, 67, 185)` 的地方，朝向 `yaw -60, pitch 30` 有一个箱子，里面有食物。村庄内没有其它箱子。
  * `(375, 68, 216)` 附近有干草块 12 个（地表，`bt mine hay_block` 可取，够了立刻 `bt stop`）。
  * 有铁傀儡巡逻（会帮忙杀怪）。
  * 有村民工作点：附近能听到工具匠工作声，村民可交易。
* 村庄旁 `y≈64` 就是浅层露头石：拿木/石镐在村庄边沿浅浅挖即可得圆石，不必进天然矿洞。

床和箱子的具体操作：

```sh
curl -s -X POST --data-binary 'look yaw -90'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
# 走到 (376, 67, 185)
curl -s -X POST --data-binary 'look yaw 60'   http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
curl -s -X POST --data-binary 'look yaw 180'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'look pitch 30' http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
# 箱子：走到 (376, 67, 185)，朝向 yaw -60, pitch 30，右键打开
curl -s -X POST --data-binary 'look yaw -60'  http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'look pitch 30' http://127.0.0.1:3420/op
curl -s -X POST --data-binary 'mouse right'   http://127.0.0.1:3420/ctl
curl -s http://127.0.0.1:3420/aif/inventory
```

### 5.3 铁：优先交易，其次受控浅采

**交易路线（推荐）**：

* 制箭师：**32 木棍 = 1 绿宝石**（木棍最便宜，靠伐木量产）。
* 盔甲匠：铁头盔 5 / 铁靴 4 / 铁胸甲 9 绿宝石；工具匠：石制工具 1 绿宝石起。
* 账要提前算：全套铁甲 ≈ 25 绿宝石 ≈ 800 木棍 ≈ 200 木板 ≈ **50 个原木**。决定走交易，就先安排大规模伐木；别指望 3 个小麦换来的 1 个绿宝石。
* 铁镐 = 3 铁锭 + 2 木棍；下矿前备好至少一把。
* Baritone 的 autoTool 会挑「最快」工具，挖普通石头也会吃铁镐耐久；铁镐只有 250 耐久，浅采也备一把备份。

**浅采（备选，不算进矿洞）**：

* 村庄旁 `y≈64` 有露头石，向下不超过 5~6 格、边走边插火把，只在白天做。
* `GET /aif/info` 盯 y，见洞就撤。
* 不要用 `bt mine stone` / `bt mine iron_ore`，它会带你钻矿洞。

### 5.4 通关路线（主线）

种子 `-818810680825819701`，`(1697, -2, 1124)` 有一座已激活的末地传送门——不用下界，不用凑末影珍珠/之眼。沿路草原/林地，途中有水塘与坡地；出发前备货，饱食度 < 18 不要出发。

| 阶段 | 做法 | 完成标志 |
| --- | --- | --- |
| 0 | 走到村庄发育：木头 → 工作台 → 木/石工具 → 食物 → 火把 | 石镐 + 石剑 + 面包 ≥ 20 |
| 1 | 干草块 → 面包；砍树攒原木；床/重生点 | 重生点已在村庄 |
| 2 | 铁：村庄交易或受控浅层坑采 | 铁镐 + 盾 + 少量铁甲 |
| 3 | 备进末地物资：食物、圆石垫脚块、床（炸龙）、弓+箭（可选） | 圆石 ≥ 64、食物 ≥ 30 |
| 4 | `POST /bt 'goto 1697 -2 1124'`，约 1400 格 | 到传送门附近 |
| 5 | 传送门在 y=-2，埋在地下：自己挖下去，边走边插火把 | 站到传送门框上 |
| 6 | 跳进传送门 → 末地 → 打龙 | 龙死 |

```sh
curl -s -X POST --data-binary 'goto 1697 -2 1124' http://127.0.0.1:3420/bt
```

### 5.5 末地打法

先拆黑曜石柱顶的水晶（弓射，或搭方块上去砍），再打龙（它停中央传送门上方是输出窗口）；备用方案是在龙附近放床并右键引爆（自己别站在爆心）。

物品 id 就是原版的命名空间 ID（`minecraft:` 可省略），例如 `oak_log`、`iron_pickaxe`、`raw_iron`；`/aif/inventory` 的物品和 `/aif/info` 的 `目标方块` 输出的都是完整 ID，照着抄即可（`目标方块` 是方块 ID，去掉 `minecraft:` 后可直接用于 `bt mine <方块>`），不需要另查表。

---

## 6. 常见坑与一页速查

### 6.1 常见坑速查

| 坑 | 对策 |
| --- | --- |
| 合成返回「配方未解锁」 | 服务端没装 `unlockRecipe` 时配方书要自己攒；先凑材料解锁 |
| `POST /op 'craft …'` 报「请先放下鼠标光标上持有的物品」 | 合成产物卡在光标上：`ESC` 关界面再开一次 |
| `craft` 返回「合成格不为空」 | 关掉合成界面再打开，不必用鼠标点回背包 |
| `craft` 返回「该配方需要 3×3 的工作台」 | 先放工作台并右键打开，不要在背包 2×2 里做 |
| `craft` 返回「已有合成任务正在进行中」 | 等上一条返回结果再发 |
| Baritone 一直在跑 | 它有任务状态；换目标或死亡后先 `bt stop`，再看回显 |
| Baritone 喊 `Mining …` 但坐标不动 | 卡在屋檐/夹缝里：`GET /aif/info` 比对坐标，不动就 `bt stop` 后手动 `W/A/D` 挪出 2~3 格 |
| `POST /ctl` 发完没反应 | 命令排队执行；定时动作配 `sleep`/`delay`，或隔一步再对账 |
| 天黑被怪围 | 提前看 `/aif/world`；按 §3.4 过夜，或撤回基地 |
| `bt mine chest/red_bed` 往地下钻 | 只有确认目标在地表才用；每 10 秒 `/aif/info`，y 掉超 5 格立刻 `bt stop`；找容器/床/矿石改为走过去 + 右键 + `/aif/inventory`。详见 §2.3 |
| 无树区域用 `bt mine oak_log` | 它会向下挖去找废弃矿井 → 先在地表 `bt goto` 定向找森林/村庄 |
| 死亡后掉落物没回收 | 死亡瞬间读 `/aif/info` 记坐标 → `bt stop` → 重生 → `bt goto` 回去 |
| 死亡后角色被 Baritone 拖走 | 死亡/中断第一件事 `POST /bt 'stop'` + `POST /ctl 'release'` |
| 死亡画面点不动 | 别用鼠标，`TAB` + `ENTER` |
| 主菜单/选择世界点不动 | 用 `ENTER`；世界条目要先选中 |
| `/ctl/prtsc` 里看不到鼠标 | 正常，截图不含系统光标 → 改键盘操作或读 `/ctl/mouse` |
| `/aif/info` 返回 `no world loaded` | 客户端在主菜单/没进世界 |
| 想知道合成/冶炼/存取报了什么 | 看 `POST /op` 的响应：`200` + 结果，或 `400` + 原因 |
| 想知道指令/Baritone 报了什么 | 读 `GET /aif/msg`，聊天栏回显直接是文本 |
| 读了 `/aif/msg` 却是空的 | 正常：上次读完之后没有新消息（读取即清空） |
| `HEAD /aif/msg` 返回 405 | 正常：读取即清空，`HEAD` 会白白吃掉回显，所以只允许 `GET` |
| 想确认方块放/挖成功 | 读 `GET /aif/keysnd`：有 `minecraft:block.stone.place` / `.break` 就是成功了 |
| `/aif/keysnd` 读不到东西 | 上次读取之后确实没有重要声音（增量，读取即清空） |
| 想知道熔炉/箱子内容 | 先右键打开该容器并保持界面，再读 `/aif/inventory` |
| `chest get` 返回 400 | 它是严格的，箱里不够就会拒绝，不会拿一部分 |
| `furnace put input …` 返回 400 | 槽位名是 `raw`/`fuel`/`product` |
| 燃料不够 | 煤 8 / 木板·原木 1.5 / 木棍 0.5；先挖煤 |
| 聊天框开着，想开背包/丢东西/切格 | 直接发 `E`/`Q`/`1`–`9`，会先自动关掉聊天框再执行；`esc` 也能关 |
| 所有输入都没反应、帧也不更新 | 多半是 `esc` 开的暂停菜单或某个界面开着 → 发 `esc` 退出，再用 `W` 试一下 |
| 想合成却打不开合成格 | 2×2 用 `E 50` 开背包；3×3 先放工作台再右键它 |
| `inventory` 把工具换没了 | 它会「换出」该格原有物品回主背包，注意格号 |
| `POST /op` 返回 `409` | 客户端还没进世界（主菜单/加载界面） |
| 长途赶路饿死 | 冲刺约 40 格/点饱食度；先备食物再出发 |
| `Q` 丢了物品但数量没变 | 站着不动会立刻捡回来；丢完先走开再对账 |
| 截图整帧不变 | 等 3–5 秒再截；比对 md5，不要据此判断命令失败 |
| 方块放不下去 | 目标格被自己碰撞箱占了 → 后退，先 `look pitch 40` 再对前方 2–3 格右键 |
| 想转到确定的方向 | 用 `POST /op 'look yaw <数值>'`（相对值用 `~`）；发完读 `/aif/info` 核对 |
| `look` 返回 `400 无效的角度值` | 数值写数字，相对值带 `~`，例如 `~-20` |
| 右键方块没反应 | 先看 `/aif/info` 的 `目标方块`：`未选中` = 准星不在方块上，用 `look` 调；仍对不上再 `F3` + 截图 |
| Baritone 把你带远 | `bt stop` 后读坐标记下，必要时 `bt goto` 回来 |
| 物品分散在多格 | `inventory` 找的是第一个匹配整叠；数量对不上时先读 `/aif/inventory` 确认 |

### 6.2 一页速查

```text
生存    /aif/world 时间>12000 停户外；饱食度<18 不回血；火把=熔炉+原木+木板
光源    POST /op 'craft furnace' → 烧 oak_log（木板当燃料）→ charcoal → craft torch
放置    POST /op 'look pitch 40' → POST /ctl 'mouse right' → 再右键打开
自动攻击 POST /ka/on 开、POST /ka/off 关；GET /ka/state 看 target / lastAttack
死亡    POST /bt 'stop' + POST /ctl 'release' → GET /aif/info 记坐标 → 重生 → POST /bt 'goto …' 回收
```
