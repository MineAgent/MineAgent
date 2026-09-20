# Minecraft 通关操作手册

给模型看的操作手册。窗口固定 **854×480**（guiScale=2）。

---

## 0. 开工自检

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3420/    # 200 = mcctl 在（必需）
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3421/    # 200 = AdvancedInfoFetcher 在
curl -s http://127.0.0.1:3420/mods                                 # 看装了哪些模组
```

装了哪些模组，决定了能用哪些手段：

| 模组 | 没有它时的后果 |
| --- | --- |
| **mcctl**（必需） | 什么都做不了 |
| AdvancedInfoFetcher | 只能靠截图判断背包和血量 |
| cmdCraft | 合成/冶炼只能去点 GUI 格子（很容易点错） |
| Baritone | 只能一步步按键走路、手动挖矿 |

`bt` 命令依赖 Baritone；没装时它只会往聊天栏发 `#...`，不会执行。

---

## 1. 接口总览

| 端口 | 模组 | 用途 |
| --- | --- | --- |
| **3420** | mcctl | 控制（POST）+ 截图 + 模组列表 |
| **3421** | AdvancedInfoFetcher | 只读状态（GET） |

### 3420 — 控制

```
GET  :3420/          使用说明
GET  :3420/prtsc     当前帧 PNG（别名 /screenshot、/prtsc.png）
GET  :3420/mods      已加载模组列表（别名 /modlist）
POST :3420/          执行命令（text/plain）
```

POST 命令：

| 命令 | 说明 |
| --- | --- |
| `<按键> [时长ms]` | 按住按键，到时松开（默认 50ms） |
| `A+B+C [时长ms]` | 同时按住多个键，例如 `W+Ctrl 100` |
| `mouse left\|right\|mid [ms]` | 鼠标键（默认 50ms） |
| `mouse move <dx> <dy>` | 视角/光标相对移动（像素；+右 +下） |
| `mouse scroll <数值>` | 滚轮（正数向上） |
| `delay <ms> <命令>` | 延迟后执行 |
| `release` | 立刻松开所有按键/鼠标 |
| `bt <命令>` | 执行 Baritone 命令 |
| `#<命令>` | 同 `bt` |
| `chat <文本>` | 发聊天消息；`/` 开头则作为指令发送 |

* 一个 POST 可以写多行，**严格按顺序执行**；`//` 开头是注释；时长上限 600000ms。
* 返回 `{"ok":true,"queued":N,"inWorld":true,"actions":[...]}`；409 = 客户端没启动。

### 3421 — 状态

```
GET :3421/info        玩家状态（别名 /player、/info.txt）
GET :3421/inventory   背包与容器（别名 /inv、/inventory.txt）
```

---

## 2. 状态读取

### `GET :3421/info`

```
玩家：DSH
维度：minecraft:overworld
坐标：-222.94 106.0 103.1
方块：-223 106 103
方位：west
yaw：-266.7
pitch：30.1
选中：3
生命值：20.0
饱食度：18
饱和度：5.0
效果：
minecraft:haste 2 95
```

| 字段 | 用途 |
| --- | --- |
| `坐标` / `方块` | 定位；**记下基地、熔炉、矿洞口的坐标** |
| `方位` / `yaw` / `pitch` | 判断朝向；pitch=90 是正下方 |
| `选中` | 当前快捷栏 1~9 |
| `生命值` / `饱食度` / `饱和度` | 危险判断：血少就撤，饿了自己找吃的 |
| `效果：` | 只有有效果时才出现，`<效果ID> <等级> <剩余秒数>` |

### `GET :3421/inventory`

```
背包：
minecraft:crafting_table 1
minecraft:stone 15
副手：
minecraft:torch 7
盔甲：
minecraft:diamond_helmet 1
熔炉：
类型：minecraft:furnace
原料：
minecraft:raw_iron 8
燃料：
minecraft:coal 4
产物：
空
燃烧：0.85
烧炼：0.45
箱子：
类型：minecraft:generic_9x6
容量：54
1：
minecraft:stone 64
```

* `背包：` 是**主背包 + 快捷栏**合并后按物品 ID 聚合（同一物品跨格相加），按 ID 排序。
* `副手：`/`盔甲：`/`效果：` 空的时候整段消失。
* `熔炉：` **只有熔炉界面开着时才出现**，附带 `燃烧`（燃料剩余）和 `烧炼`（当前进度），
  都是 `0.00`-`1.00`。
* `箱子：` 同理，`<槽位号>：` 从 1 开始，空槽位直接跳过。
* 潜影盒不支持。

> **重点**：客户端只有在界面打开时才知道容器内容。想知道熔炉烧好没有，必须**右键打开熔炉并保持界面**，
> 然后读 `/inventory` 的 `烧炼`/`产物`。

---

## 3. 输入通道实测结论

| 类别 | 例子 | 通过 HTTP | 绕开办法 |
| --- | --- | --- | --- |
| isDown（按住持续生效） | `W` `S` `A` `D`、`mouse left/right` | ✅ | — |
| **consumeClick（点按触发一次）** | `E` 开背包、`1`~`9` 切格、`Q` 丢弃、`T` 聊天 | ❌ **无效** | 用 `/inventory` 指令切格；读背包用 `:3421/inventory`；开界面靠右键方块 |
| 特判 | `esc`、`F3` | ✅ | — |
| 滚轮 | `mouse scroll n` | ✅ 但**一次只走一格** | 用 `/inventory` 更省事 |

* **切快捷栏优先用 `chat /inventory <物品id> <1-9>`**，不要靠滚轮。
* `esc` 在世界里会开**暂停菜单**（世界不 tick，Baritone 全停），误按再按一次退出。
* 右键（使用/放置/开界面）和左键（攻击/挖掘）都正常。

---

## 4. 高层动作：cmdCraft

四条客户端指令，**服务端不用装任何东西**。它们和玩家亲手点格子完全等价（发的是原版点击包）。

### `/craft <物品id> [数量]`

```bash
curl -s -X POST --data-binary 'chat /craft stone_pickaxe'      http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /craft oak_planks 8'       http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /craft crafting_table'     http://127.0.0.1:3420
```

* `数量` 指**产物个数**，会自动向上取整到整数次合成，并把实际结果报在聊天里。
* 前置条件：**打开着合成界面**（工作台 3×3，或背包 2×2）；合成格空；光标空。
* 3×3 配方（镐/剑/斧/熔炉/铁砧…）**必须开着工作台**，否则报「请先打开工作台」。
* 配方必须已在客户端配方书里（原版规则：拿到材料就解锁）。
* 任一检查不过就**什么都不做**，不会半途消耗材料。

### `/inventory <物品id> [1-9]`

```bash
curl -s -X POST --data-binary 'chat /inventory torch 2'    http://127.0.0.1:3420   # 火把换到第 2 格
curl -s -X POST --data-binary 'chat /inventory iron_pickaxe 1' http://127.0.0.1:3420
```

把背包里的整叠物品换到指定快捷栏格（原版数字键交换），默认第 1 格。找的是**第一个匹配的整叠**，
先在主背包 27 格里找，再找其他快捷栏格。**用来切工具/方块，比滚轮可靠。**

### `/furnace put|get <raw|fuel|product> <物品id> [数量]`

```bash
curl -s -X POST --data-binary 'chat /furnace put raw raw_iron 8'  http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /furnace put fuel coal 4'     http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /furnace get product'         http://127.0.0.1:3420
```

* 槽位名就是 `raw` / `fuel` / `product`。
* **必须开着熔炉界面**（熔炉 / 高炉 / 烟熏炉共用）。
* `get raw`/`get fuel` 可以按数量拆；`get product` 不能拆，要多少都会整堆取出并提示。
* 燃料：煤 8、木板/原木 1.5、木棍 0.5 个物品。

### `/chest put|get <物品id> [数量]`

```bash
curl -s -X POST --data-binary 'chat /chest put cobblestone 64'  http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /chest get iron_ingot 16'   http://127.0.0.1:3420
```

* **必须开着箱子界面**（箱子 / 陷阱箱 / 大箱子 / 木桶）。
* `get` 是严格的：箱里不够要的数量就报错，不会只拿一部分。

---

## 5. Baritone

```bash
curl -s -X POST --data-binary 'bt mine oak_log'    http://127.0.0.1:3420
curl -s -X POST --data-binary 'bt mine iron_ore'   http://127.0.0.1:3420
curl -s -X POST --data-binary 'bt stop'            http://127.0.0.1:3420
curl -s -X POST --data-binary 'bt goto 100 64 200' http://127.0.0.1:3420
```

* `bt mine <方块>`：自动寻路 + 挖掘 + 捡拾，是最省事的采矿方式。
* 挖矿会把你带到很深的 y 层，**离开前用 `:3421/info` 记住坐标**，必要时 `bt goto` 回来。
* 任务跑着的时候可以随时 `:3421/info` 看进度；`bt stop` 之后必须对账。
* 本版本 Baritone **没有 `craft` 命令**。

---

## 6. 决策循环

```
① 观察  GET :3421/info  +  GET :3421/inventory
② 决策  结合目标，选**一个**动作
③ 执行  POST :3420（bt / chat / 按键 / 鼠标）
④ 校验  再读一次状态对账，必要时 GET :3420/prtsc
   ↺ 回到 ①
```

### ① 观察

* 常规：`/info`（在哪、面朝哪、血够不够）+ `/inventory`（有什么、熔炉烧到哪了）。
* 想知道某个容器里有什么，**先右键打开它**，再读 `/inventory`。
* 画面只在需要确认界面状态、聊天回显、图形信息时才截图，不要每次都截。

### ② 决策

* **一次只做一个动作**，做完立刻校验。别把"挖矿+合成+冶炼"塞进一个请求里。
* 需要材料先看 `/inventory` 够不够，别凭记忆。
* 目标优先级：活下来（血/饿/怪）> 当前里程碑 > 囤资源。

### ③ 执行

* 移动/挖掘/放置用 `bt` 或按键；合成/冶炼/存取用 `chat /craft`·`/furnace`·`/chest`·`/inventory`。
* 想开工作台/熔炉/箱子：**走过去右键**（准星判定），不要试图用光标点 GUI。

### ④ 校验

* **截图有延迟**：发完命令等 3~5 秒再截；两张截图 md5 相同只说明画面没更新，不代表命令失败。
* **对账**：动作前后各读一次 `/inventory`，看材料少了多少、产物多了多少。
* **长任务**：Baritone 在跑时定期 `/info` 看坐标变化；`bt stop` 后必须 `/inventory` 对账。
* **记坐标**：基地、熔炉、矿洞入口、传送门，都用 `/info` 读出来记下。

### 纪律

* 材料不要一次全投进熔炉；留 1 个原木做工作台；别把唯一的木板当燃料烧了。
* 血少/天黑/有怪时优先处理安全问题，别硬推进度。
* 拿不准的操作先发一次看回显（比如不确认物品 id），再决定下一步。

---

## 7. 坐标兜底表（854×480）

> 正常流程不需要手点 GUI —— 这些坐标只在万不得已要手动点格子时用。
> 注意：光标起始位置读不到，`mouse move` 只是相对位移，**手点不可靠**；点之前先 `/prtsc` 确认真实位置。

**HUD**：准星 `(427,240)`；快捷栏第 i 格 `x = 266 + 40×(i-1)`，`y = 458`

**工作台界面**：

| 元素 | 坐标 |
| --- | --- |
| 3×3 合成格 | `(325,123) (361,123) (397,123)` / `(325,159) (361,159) (397,159)` / `(325,195) (361,195) (397,195)` |
| 输出槽 | `(513,159)` |
| 玩家 3×9 | `x = 281 + 36c`（c=0…8），`y = 257 / 293 / 329` |
| 界面内快捷栏 | `y = 373` |

**熔炉界面**：`raw (377,123)`、`fuel (377,195)`、`product (497,159)`

**转向换算**：`mouseSensitivity=0.5` 时每 1 像素 ≈ **0.15°**，转 θ 度用 `dx ≈ θ/0.15`；转完用 `/info` 读 yaw 核对。

**放置方块**：目标格不能被自己的碰撞箱占据——朝脚下放必失败，窄隧道贴脸放也常失败。
先后退 1~2 格，对着前方 2~3 格的地面/墙面右键；成功后动作栏显示「方块：被放置」。

---

## 8. 通关路线

| 阶段 | 做法 |
| --- | --- |
| 1 | `bt mine oak_log` → 工作台 → 木镐 → `bt mine stone` → 石镐 → `/craft furnace` |
| 2 | `bt mine coal_ore`（燃料）+ `bt mine iron_ore` → 熔炉冶炼 → **铁镐**、铁剑、铁桶、打火石、盾 |
| 3 | `bt mine diamond_ore` → 钻石镐、钻装 |
| 4 | **钻石镐挖黑曜石** → 搭 4×5 下界门 → 打火石点燃 |
| 5 | 下界：找下界要塞打烈焰人拿**烈焰棒** |
| 6 | 主世界打**末影人**拿**末影珍珠** → 合成**末影之眼** → 投掷并跟着飞行方向走 → 找到要塞 → 激活末地传送门 |
| 7 | 末地：用**床**炸 / 弓箭 / 近战打**末影龙** |

常用物品 id：`oak_log` `oak_planks` `stick` `crafting_table` `furnace` `torch` `chest`
`wooden_pickaxe` `stone_pickaxe` `iron_pickaxe` `diamond_pickaxe` `iron_sword` `bow` `arrow`
`raw_iron` `iron_ingot` `coal` `diamond` `obsidian` `bucket` `water_bucket` `flint_and_steel` `shield`
`ender_pearl` `blaze_rod` `blaze_powder` `ender_eye` `bed` `white_wool`

---

## 9. 坑速查

| 坑 | 对策 |
| --- | --- |
| `E`/`1-9`/`Q`/`T` 通过 HTTP 无效 | 切格用 `/inventory`；读背包用 `:3421/inventory`；开界面靠右键 |
| 想知道熔炉/箱子内容却没有数据 | 先右键打开该容器并保持界面，再读 `:3421/inventory` |
| 截图整帧不变 | 等 3~5 秒再截；比对 md5，不要据此判断命令失败 |
| 方块放不下去 | 目标格被自己碰撞箱占了 → 后退，换个面 |
| `/craft` 报「请先打开工作台」 | 3×3 配方必须先放好工作台并右键打开 |
| `/craft` 报「合成格不为空」 | 把合成格里的东西取走 |
| `/furnace put input` 报错 | 槽位名是 `raw`/`fuel`/`product` |
| 燃料不够 | 煤 8 / 木板·原木 1.5 / 木棍 0.5；先挖煤 |
| `/chest get` 报错 | 它是严格的，箱里不够就会拒绝，不会拿一部分 |
| 误按 `esc` 进暂停菜单 | 再按一次 `esc`；暂停时世界不 tick，Baritone 会停 |
| Baritone 把你带远 | `bt stop` 后读坐标记下，必要时 `bt goto` 回来 |
| 物品分散在多格 | `/inventory` 找的是第一个匹配整叠；数量对不上时先读 `/inventory` 确认 |
