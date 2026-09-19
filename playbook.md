# Minecraft 通关操作手册（mcctl）

窗口固定 **854×480**，guiScale=2。接口基址 **`http://127.0.0.1:3420`**

---

## 1. 接口

```
GET  :3420/         使用说明
GET  :3420/info     玩家信息（纯文本）
GET  :3420/prtsc    当前帧 PNG（854×480）
POST :3420/         执行命令（text/plain）
```

POST 命令：
```
<按键> [时长ms]              默认 50ms
A+B+C [时长ms]               同时按住（如 W+Ctrl）
mouse left|right|mid [ms]
mouse move <dx> <dy>         视角/光标相对移动（+右 +下）
mouse scroll <数值>          滚轮（正数=向上）
delay <ms> <命令>
release                      松开全部
bt <baritone命令>
chat <文本>                  以 / 开头＝作为指令发送
```
```bash
curl -s -X POST --data-binary 'bt mine iron_ore'          http://127.0.0.1:3420
curl -s -X POST --data-binary 'chat /craft iron_pickaxe'  http://127.0.0.1:3420
curl -s http://127.0.0.1:3420/info
curl -s -o shot.png http://127.0.0.1:3420/prtsc
```

---

## 2. `/info`（主反馈手段）

```
玩家：DSH
维度：minecraft:overworld
坐标：-222.94 106.0 103.1
方块：-223 106 103
方位：west
yaw：-312.1
pitch：24.0
选中：1
背包：
minecraft:crafting_table 1
副手：
minecraft:iron_chestplate 1
盔甲：
minecraft:iron_chestplate 1
```

- `坐标/方块/方位/yaw/pitch` → 定位与转向；`选中` → 当前快捷栏 1~9；`背包/副手/盔甲` → 物品（按 id 聚合）
- 用途：读背包（不用开界面）、对账 `/craft`·`/furnace`·Baritone 的收支、确认转向
- 不显示：是否开着 GUI、光标上拿的东西、熔炉内部（用 `/furnace getinfo`）

---

## 3. 输入通道

| 类别 | 例子 | 可用 |
|---|---|---|
| isDown（按住持续生效） | `W S A D`、`mouse left/right` | ✅ |
| consumeClick（点按触发一次） | **`E` 开背包、`1`~`9` 切格、`Q`、`T`** | ❌ |
| 特判 | `esc`、`F3` | ✅ |
| 滚轮 | `mouse scroll n` | ✅ 但**一次只走一格** |

- **切快捷栏**：`mouse scroll 1` 反复发，用 `/info` 的「选中」核对
- **读背包**：用 `/info`，不要试图开背包
- `esc` 在世界里会开暂停菜单（世界不 tick，Baritone 全停），误按再按 `esc` 退出

---

## 4. 合成 / 冶炼（craftcmd，客户端指令）

```
chat /craft <物品id> [数量]
```
前提：**打开着合成界面**（工作台 3×3，或背包 2×2）；光标空；合成格空。
3×3 配方（镐/剑/斧/熔炉/铁砧…）必须在**工作台**界面里做，2×2 配方（木板/木棍/工作台）背包里也行。

```
chat /furnace getinfo
chat /furnace put <raw|fuel|product> <物品id> [数量]
chat /furnace get <raw|fuel|product> [数量]
```
前提：**打开着熔炉界面**。槽位名就是 `raw`/`fuel`/`product`。
燃料：煤 8、木板/原木 1.5、木棍 0.5（3 块木板只烧 4 个）。输出槽不能拆分。

常用 id：`oak_planks` `stick` `crafting_table` `furnace` `wooden_pickaxe` `stone_pickaxe` `iron_pickaxe` `iron_sword` `raw_iron` `iron_ingot` `coal` `diamond` `diamond_pickaxe` `obsidian` `bucket` `flint_and_steel` `shield` `ender_eye` `bed`

---

## 5. Baritone

```
bt mine <block>        自动寻路+挖掘+捡拾（oak_log / stone / coal_ore / iron_ore / diamond_ore）
bt stop                停止
bt goto <x> <y> <z>    走到坐标
```
挖矿会把你带到很深的 y 层，**离开前用 `/info` 记坐标**；挖完 `bt stop` 再对账。
本版本 Baritone **没有 `craft` 命令**。

---

## 6. 坐标（854×480）

**HUD**：准星 (427,240)；快捷栏第 i 格 x=266+40×(i-1), y=458

**工作台界面**：3×3 格 x={325,361,397} × y={123,159,195}；输出 (513,159)；
玩家区 x=281+36c (c=0…8)，y=257/293/329；界面内快捷栏 y=373

**熔炉界面**：raw (377,123)、fuel (377,195)、product (497,159)；玩家区同上

**转向**：每 1 像素 ≈ 0.15°，转 θ 度用 `dx≈θ/0.15`，转完用 `/info` 读 yaw/pitch 核对。

**放置方块**：目标格不能被自己的碰撞箱占据 —— 朝脚下放必失败，窄隧道贴脸放常失败。
正确做法：**先后退 1~2 格**，对着前方 2~3 格的地面/墙面右键。成功时动作栏显示「方块：被放置」。

> GUI 槽位点击尽量避免：光标起始位置读不到，手点不可靠。用 craftcmd 代替。

---

## 7. 操作节奏

1. 动作 → 等 3~5 秒 → `/info`（必要时 `/prtsc`）确认
2. 截图两张 md5 相同＝画面没更新，不代表操作失败
3. 备料顺序：先煤（燃料）→ 再矿 → 冶炼 → 合成
4. 长任务结束（`bt stop`）必须 `/info` 对账

---

## 8. 通关路线

| 阶段 | 做法 |
|---|---|
| 1 | `bt mine oak_log` → 工作台 → 木镐 → `bt mine stone` → 石镐 → `/craft furnace` |
| 2 | `bt mine coal_ore` + `bt mine iron_ore` → 熔炉冶炼 → **铁镐**、铁剑、铁桶、打火石、盾 |
| 3 | `bt mine diamond_ore` → 钻石镐、钻装 |
| 4 | **钻石镐挖黑曜石** → 搭 4×5 下界门 → 打火石点燃 |
| 5 | 下界要塞打烈焰人拿**烈焰棒** |
| 6 | 主世界打**末影人**拿**末影珍珠** → 合成**末影之眼** → 投掷并跟着飞行方向走 → 找到要塞 → 激活末地传送门 |
| 7 | 末地：用**床**炸 / 弓箭 / 近战打**末影龙** |

---

## 9. 坑速查

| 坑 | 对策 |
|---|---|
| `E`/`1-9`/`Q` 无效 | 滚轮切格；`/info` 读背包 |
| 截图整帧不变 | 等 3~5s 再截 |
| 点击落错格 | 移动后 settle ≥2.5s；尽量不用光标 |
| 方块放不下 | 后退，换不被碰撞箱占据的面 |
| `/craft` 报「需要 3×3 工作台」 | 放工作台并右键打开，保持界面 |
| `/craft` 报「合成格不为空」 | 清空合成格 |
| `/furnace put input` 报错 | 槽位名是 `raw`/`fuel`/`product` |
| 燃料不够 | 煤 8 / 木板·原木 1.5 / 木棍 0.5 |
| 误按 `esc` 进暂停菜单 | 再按 `esc` 退出 |
| Baritone 带远 | `bt stop` 后 `/info` 记坐标，`bt goto` 回来 |
