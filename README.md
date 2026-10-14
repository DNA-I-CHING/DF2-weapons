# DF2-weapons
weapons analysis of DF2

## 一、这是什么游戏

从目录结构和导入表可以确定：这是 **Novalogic《Delta Force 2》可执行文件，不是近年的 UE5《Delta Force》。

判据：
- 同目录有 `Df2.pff`（300 MB）、`Terrains.pff`（314 MB）、`DF2BLD`、`Stock Maps`、`Df2med.exe`、`DF2PlayerEditor.exe`
- PE 导入：`DirectDrawCreate`(DDRAW.dll)、`DirectSoundCreate`、`DPLAYX.dll`(DirectPlay 对战网络)、`netsock.dll`、`Netcomm.dll`、`Cmprssh0.dll`
- 完全没有 D3D9/D3D11/OPENGL 导入 —— 用户目录里的 `DDraw.dll` + `dgVoodoo.conf` 就是用来把这套 DirectDraw 7 老代码转到现代渲染器上的

段落布局值得注意：`.data` 虚拟大小 **0x9629B8（9.8 MB）** 但原始大小只有 0x31000（200 KB），说明绝大部分是运行时填充的全局状态（玩家数组、武器槽、网络缓冲等）。

---

## 二、核心结论：武器数值不在 exe 里

这是本次分析最重要的判断。

对 `Df2.exe` 全量字符串 + Ghidra 交叉引用分析的结果：**武器的伤害、射速、后坐力、弹匣容量等数值参数不在 Df2.exe 中**，exe 只负责状态机与调度；真实数值存放在 `Df2.pff` 数据包里，由内置脚本虚拟机（NovaWorld Script）在任务脚本里驱动。

证据链：

1. exe 里能找到的所有武器字符串，属于**三类**：武器音效事件名、HUD 元素名、调试/控制台命令名 —— 全是名字，不含数值。
2. exe 里的武器相关调试串暴露了字段名，却没有字段值：
   ```
   0x0cc81c  bad ammo clip size
   0x0cc830  bad weapon attribs
   0x0cc844  bad weapon scope mag
   0x0cc85c  bad weapon fire delay
   0x0cc874  bad weapon ammo type
   0x0bfaa0  ammo type = %i
   ```
   → 武器属性结构至少有 5 个字段：`clip size`、`attribs`、`scope mag`、`fire delay`、`ammo type`，但赋值发生在别处。
3. 在 `Df2.pff`（300 MB）中搜到对应的数据侧证据：
   | 标识 | 在 PFF 中出现次数 | 首次偏移 |
   |---|---|---|
   | `WeaponNames` | 2 | `0x22D9E2` |
   | `M16` | 58 | `0x22A7AB` |
   | `weapon` | 60 | `0x0B4061` |
   | `weapon_clips` | 6 | `0x449539A` |
   | `attrib` | 8 | `0x0B9E041` |

---

## 三、武器清单（39 个槽位）

从 `Df2.pff` 的 `WeaponNames2` / `WeaponNames2L` 分支提取，即玩家实际可持有的武器序列：

| # | 名称 | 类别 |
|---|---|---|
| 0 | No Weapon | 空手 |
| 1 | Knife | 近战 |
| 2 | Knife - Thrown | 投掷 |
| 3 | Garrote | 绞索 |
| 4 | High Standard .22 suppressed pistol | 手枪 |
| 5 | SpecOps .45 caliber pistol | 手枪 |
| 6 | M4 Assault Rifle, Burst | 突击步枪 |
| 7 | M4 Assault Rifle, Single Shot | 突击步枪 |
| 8 | M16, Burst | 突击步枪 |
| 9 | M16, Single Shot | 突击步枪 |
| 10 | M203 40mm grenade launcher | 榴弹发射器 |
| 11 | Shotgun | 霰弹枪 |
| 12 | SAW Auto. Weap., Full Auto | 轻机枪 |
| 13 | SAW, Semi-auto | 轻机枪 |
| 14 | SAW, Clip | 轻机枪 |
| 15 | Barret .50 Caliber Sniper Rifle | 狙击枪 |
| 16-18 | H&K MP5, Full Auto / Burst / Single Shot | 冲锋枪 |
| 19-21 | H&K MP5 SD, Full Auto / Burst / Single Shot | 冲锋枪 |
| 22-23 | Ingram M11, Full Auto / Semi Auto | 冲锋枪 |
| 24 | Stinger | 肩扛发射器 |
| 25 | Laser Designator | 激光标定器 |
| 26 | LAW Light Anti-tank Weapon | 反坦克 |
| 27 | Satchel Charge | 炸药包 |
| 28 | Radio Detonator | 遥控器 |
| 29 | Camera | 特殊 |
| 30 | M40 Sniper Rifle | 狙击枪 |
| 31 | Frag Grenade | 手雷 |
| 32 | Smoke Grenade | 烟雾弹 |
| 33 | AK 47, Burst | 突击步枪 |
| 34 | AK 47（第二条变体） | 突击步枪 |
| 35 | Claymore mine, Motion Sensor | 地雷 |
| 36 | Claymore mine, Radio Det | 地雷 |

配套的本地化键 `STR_PWNAME00` ~ `STR_PWNAME38`（39 条）与武器槽位**一一对应**，`STR_SPWNAME*` 为副武器/短名版本。

---

## 四、武器音效事件表 = 武器 ID 索引

`Df2.exe` 中 `.data` 段 `0x0CE2C6` 起是一张排好序的音效事件名表，每种武器固定四个事件（开火/空仓/换弹/拉栓），**文件偏移顺序即事件 ID 顺序**：

```
GS_M203_SHOT      GF_M16_RELOAD      GS_MP5_BURST      GF_SAW_DRYFIRE     GS_AK47_BURST
GS_M16_BURST      GF_M16_CHARGE      GF_MP5_DRYFIRE    GF_SAW_RELOAD      GS_AK74_BURST
GS_M16_SHOT       GF_M16_DRYFIRE     GF_MP5_RELOAD     GF_SAW_CHARGE      GS_HS22_SHOT
GF_M19_SHOT       GF_M19_RELOAD      GF_MP5_CHARGE     GS_M40_SHOT        GF_HS22_DRYFIRE
GF_M19_DRYFIRE    GF_M19_CHARGE      GS_MP5_SHOT       GF_M40_DRYFIRE     GF_HS22_RELOAD
GS_BARRETT_SHOT   GF_BARRETT_DRYFIRE GS_SAW_SHOT       GF_M40_RELOAD      GF_HS22_CHARGE
GF_BARRETT_RELOAD GF_BARRETT_CHARGE                     GF_M40_CHARGE      GS_M203_RELOAD
                                                                   GS_M203_DRYFIRE / GS_LAW_SHOT / GF_LAW_DRYFIRE / GF_LAW_RELOAD
通用弹道：GS_762(5.56) / GS_50CAL / GS_20MM / GS_SHOTGUN_SHOT / GS_HELO / GS_AUTO_GRENADE
自定义：  GS_UDGUN / GS_UDPISTOL（UD = user-defined，玩家自定义枪械）
```

前缀约定：`GF_` = 通用枪械（Grenade Fire 系列）、`GS_` = 专用枪械（Gun Specific）。

`GS_UDGUN` / `GS_UDPISTOL` 的存在说明游戏支持**用户自定义武器模型**（编辑器 `DF2PlayerEditor.exe` 配套）。

---

## 五、武器相关的输入命令表

`Df2.exe` 中 `0x0C8400` 起是一张网络/脚本命令名表（武器相关部分）：

```
FIRE_WEAPON          SECONDARY_WEAPON     KNIFE                SIDEARM
CYCLE_WEAPONS        SCOPE                GRENADE              CHANGE_MAGAZINE
AUXILIARY_GEAR       DETONATOR            LASER_DESIGNATOR     BINOCULARS
PRIMARY_WPN_FN_1/2/3                       ZOOM_IN / ZOOM_OUT
```

同表还包含移动与姿态：`STAND` / `CROUCH` / `GO_PRONE` / `JUMP` / `WALK` / `MOVE_FORWARD` / `MOVE_BACK` / `STRAFE_LEFT` / `STRAFE_RIGHT`。

---

## 六、已定位的关键函数（Ghidra 反编译）

共识别 **1771 个函数**，其中与武器字符串直接关联的有 20 个。按关联度排序的关键几个：

| 地址 | 大小 | 作用 |
|---|---|---|
| `0x00441140` | 2061 B | **输入绑定注册表**：把 weapon0~weapon9、scope、scopeinc/dec、magazine、attack_1/2 等动作名绑定到 `DAT_004f37bc` 结构体偏移（`+0x2E0` 起，每项 0xC 字节） |
| `0x0044A810` | 1452 B | **HUD 元素名解析器**：从 HUD 布局读 `weapon_clips`/`weapon_rounds`/`weapon_chamber`/`weapon_name`/`gun_icon`/`waypoint_*`/`carrying_icon`，存入全局 `DAT_00A162xx` |
| `0x004758F0` | 1316 B | 解析 `WeaponNames2` / `WeaponNames2L` 分组字符串 |
| `0x00434FD0` | 1747 B | 引用 `MAGAZINE`（弹匣）状态名 |
| `0x00439B40` | 275 B | `FRAG_GRENADE` / `DELAYED_FRAG_GRENADE` 分支 |
| `0x0043B110` | 2511 B | `showgun` / `attributes` 控制台变量注册（2511 字节，是最大的一个） |
| `0x0043BD10` | 1098 B | `showgun = %i` / `attributes = %i` 赋值处理 |

注意：这批命中**没有一个是开火判定/弹道计算本体** —— 因为判定逻辑不引用任何武器名字符串，而是直接读写武器状态结构体字段。这印证了第二节的结论。

### 已存在的调试/控制台开关（游戏自带，非我添加）

exe 内置了一个调试控制台，含以下武器相关命令：

```
!Weapons          weapon0 ~ weapon9     !Cycle Weapons    !Fire Weapon
!Full Ammo        !Infinite Ammo        !Change Magazine  !Knife   !Grenade
!Scope            !Scope Scale          !Scope Zero Increase / Decrease  !Scope Mag
showgun / attributes / scopemag / magazine / scope
```

另有任务脚本层的作弊项：`!Mega HP`、`!Invisible`、`!Artilary Strikes`。
`!Full Ammo` 与 `!Infinite Ammo` 是分离的两条，说明存在**有限弹匣**与**无限备弹**两个独立层。

---

## 七、弹道穿透系统

exe 中存在一套按材质分类的弹道穿透事件名，这是老引擎少见的设计：

```
BULLET_WOOD    BULLET_METAL   BULLET_BUSH   BULLET_FLESH
BULLET_SAND    BULLET_WATER   BULLET_DIRT   BULLET_STONE
BULLET_ARMOR   BULLET_CANVAS  BULLET_GRASS
```

即子弹命中不同材质时触发不同事件，掩体穿透规则是**材质驱动**的。

配套的命中调试串：
```
hit_owner = %i     hit_subowner = %i    hit_ssubowner = %i
hit_distance = %i  hit = %i             Last Hit Structure    Last Object Hit
player being dead too long              player killing
```

---
---

## 附：产物清单

全部位于 `D:\360MoveData\Users\Lenovo\Documents\Desktop\已转换\df2-weapon-analysis\`：

| 文件 | 内容 |
|---|---|
| `weapon_xrefs.txt` | 武器字符串 → 函数交叉引用（20 个函数） |
| `weapon_pseudocode.txt` | 上述 20 个函数的完整反编译伪代码（172 KB） |
| `weapon_func_map.txt` | 多级 xref 评分排序 |
| `weapon_tables.txt` | 武器字符串指针表定位 |
| `all_functions.txt` | 全部 1771 个函数清单 |
| `scripts/WeaponScan.java` | Ghidra 脚本：字符串 xref 定位 + 反编译 |
| `scripts/WeaponTable.java` | Ghidra 脚本：数据表识别 + 多级 xref 回溯 |
| `ghidra_proj/` | Ghidra 工程（可继续复用分析） |
