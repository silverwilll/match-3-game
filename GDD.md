# 插花三消 — 游戏设计文档 (GDD)

> 中文工作名：**花艺禅心**（暂定，可改）
> 平台：微信小游戏（兼容抖音小游戏 / H5）
> 引擎：Cocos Creator 3.8.x
> 类型：休闲、三消、解谜
> 目标用户：30+ 女性休闲玩家、上班族打发碎片时间
> 单局时长：3 ~ 6 分钟

---

## 1. 玩法核心 (One-pager)

### 1.1 核心目标
玩家通过拖动花瓶中"露出"的花朵，把它们拖拽到其他**花瓶**中，每个花瓶最多可以容纳三朵花，每凑齐 **3 朵相同的花** 即自动消除得分。

花瓶上有数字，代表该花瓶内剩余的花朵刷新次数。每次刷新会刷新出1-3朵花不等，刷新出来的花朵一定不满足三个相同的规则，而且即将刷新出来的花朵会以黑影的形式显示在目前花朵后面。每次在一个花瓶中刷新出下一轮花朵，则花瓶的数字减少1，最后花瓶的数字归零则该花瓶上的数字消失，只留下空瓶，允许后续玩家插入花朵。

在限定时间内消除完所有花朵，把所有花瓶中的次数都消耗完即过关。

### 1.2 核心循环 (Core Loop)
```
进入关卡 → 看场上花瓶布局 → 拖动花朵到其他有空位的花瓶中 → 如果满足三朵同样花色的花在一个花瓶里 → 消除 → 得分
            ↑                                              ↓
            └────── 凑不出消除时使用道具 ←─── 所有花瓶的空位都已填满 = 失败
```

### 1.3 胜负条件
| 条件 | 触发 |
|---|---|
| **胜利** | 场上所有花瓶的"刷新次数"都消耗为 0，且场上没有任何剩余花朵 |
| **失败-A** | 倒计时归零 |
| **失败-B** | 死锁：所有花瓶都装满 3 朵（无法继续拖入）且没有花瓶能凑齐 3 同款 |

> **数字含义**：花瓶上显示的数字 = 该瓶 **剩余刷新次数**。每次该瓶被清空（拖空或被消除）后，自动刷新出下一批 1–3 朵新花，数字 −1。数字归 0 后该瓶不再刷新、数字消失，**但空瓶依旧保留**，可继续接收玩家拖入的花朵（充当"暂存格"）。当全部瓶子都不再刷新且场上没有花，即胜利。

### 1.4 一句话定位
> "在 6 分钟里完成一束属于自己的花艺作品 —— 把佛系节奏的三消，包装成花店打工小日子。"

---

## 2. 玩法细则

### 2.1 场地结构
- **3 层木质花架**，每层放 3 个花瓶。
- 总共有 9 个花瓶，每个花瓶有 1-3 朵花，每个花瓶的刷新次数初始都是8.
- 花瓶上显示数字 = 该瓶 **当前剩余刷新次数**（当刷新后数字减1，新的花朵会被刷新出来）。
- 玩家在每个花瓶里只能看到初始伸出瓶口的 1-3 朵花。

### 2.2 玩家操作
| 操作 | 反馈 |
|---|---|
| 长按/按下花朵 | 花朵被"举起"，跟随手指；其余瓶子高亮为可放置目标（仅当瓶内 < 3 朵） |
| 拖拽到目标瓶松手 | 花朵插入目标瓶；若瓶内含 3 朵同款 → 0.3 秒后整组消除 + 得分；若目标瓶满 / 无效 → 弹回原瓶 |
| 拖空源瓶 | 源瓶刷新次数 −1；新一批花朵（1–3 朵，不可能 3 同款）从黑影变实色显示；下一批黑影预览刷新 |
| 点击道具 | 触发对应技能 |

### 2.3 刷新批次详则
- **批次构造**：每个花瓶在关卡开始时已预生成 `refreshCount + 1` 个批次（首批 + N 个待刷新批次）。
- **批次大小**：每个批次随机 1–3 朵花。
- **批次内规则**：单个批次不可能"3 朵全同款"（否则刷出来直接消除会破坏节奏）。允许 2 朵同款 + 1 朵异款。
- **黑影预览**：下一批的剪影渲染在当前花朵后面（半透明黑色 silhouette），让玩家提前规划。
- **刷新时机**：当前瓶中的花数变为 0（被拖空或消除）→ 0.3 秒后预览黑影"实体化"，刷新次数 −1。
- **刷新次数归 0**：黑影消失、数字消失，仅保留空瓶身。空瓶仍可作为拖拽目标。

### 2.4 得分公式
```
单次消除得分 = 10 × 连击倍率
连击倍率：每次连续消除（2 秒内）+1，最高 ×5
分数仅用作排行 / 结算展示，过关判定看"刷新次数全清 + 场上无花"
```

### 2.5 道具系统

| 道具 | 中文名 | 效果 | 获取 |
|---|---|---|---|
| ⏱ 时停 | 时停 | 给倒计时增加 **2 分 30 秒** | 看广告 / 商城购买 |
| 🔀 刷新 | 刷新 | 把场上所有 **当前可见花朵 + 各瓶剩余批次** 整体重新随机分配（保留每个瓶子的位置和剩余批次数），主要用于解死锁 | 看广告 / 商城购买 |
| ✨ 魔法棒 | 魔法棒 | 直接 **消除 3 朵相同的花**（从场上数量最多的花种里挑出 3 朵，无视位置） | 关卡通关奖励 / 充值 |

**实现要点：**
- 三种道具均独立计数，UI 角标显示剩余数量。
- 道具按钮在使用后播放 0.5 秒粒子动效。
- 看广告获取的道具：通过微信 `wx.createRewardedVideoAd` 实现。

---

## 3. 关卡系统 + 难度曲线

### 3.1 关卡参数化
每关由以下参数描述：

| 参数 | 含义 |
|---|---|
| `level` | 关卡编号 |
| `flowerKinds` | 本关花的种类数 |
| `vaseCount` | 花瓶数（固定 9：3 层 × 3 列） |
| `refreshCount` | 每瓶的"刷新次数" |
| `vaseCapacity` | 每瓶可见花朵上限（固定 3） |
| `timeLimit` | 倒计时（秒） |

### 3.2 难度曲线（前 30 关）

| 关卡区间 | 花种类 | 花瓶数 | 每瓶刷新次数 | 时间 | 备注 |
|---|---|---|---|---|---|
| 1–3   | 3 | 9 | 2–3 | 5:00 | 新手教学，必通关 |
| 4–6   | 4 | 9 | 3–4 | 5:00 | 引入第 4 种花 |
| 7–10  | 5 | 9 | 4–5 | 5:00 | 颜色变多，得规划 |
| 11–15 | 6 | 9 | 5–6 | 5:00 | 中期难度 |
| 16–20 | 7 | 9 | 6–7 | 4:30 | 时间压力 ↑ |
| 21–25 | 8 | 9 | 7–8 | 4:30 | 引入"卡死"风险 |
| 26–30 | 9 | 9 | 8–9 | 4:00 | 高阶，几乎一定要用刷新道具 |

**核心难度变量：**
1. **花种类数 ↑** = 同类相遇概率 ↓ = 难
2. **刷新次数 ↑** = 总花数 ↑ = 长 + 死锁概率 ↑ = 难
3. **时间 ↓** = 压力 ↑

**通关率目标（运营曲线）：**
- 第 1 关 95%，第 5 关 85%，第 10 关 70%，第 20 关 50%，第 30 关 30%
- 利用通关率倒推参数

### 3.3 关卡配置示例 (JSON)
```json
{
  "level": 4,
  "flowerKinds": 4,
  "vaseCount": 9,
  "vaseCapacity": 3,
  "refreshCount": 3,
  "timeLimit": 300,
  "specialFlowers": [],
  "rewardCoins": 20,
  "rewardFertilizer": 1
}
```

### 3.4 自动难度生成器（伪代码）
```js
function generateLevel(n) {
  const tier = Math.min(Math.floor((n-1)/3), 6);
  const flowerKinds  = Math.min(3 + tier, 9);
  const vaseCount    = 9;
  const refreshCount = 2 + tier;                 // 每瓶刷新次数
  // 每瓶最多 (refreshCount + 1) 个批次，每批 1–3 朵 → 平均 2 朵
  // 总花数 ≈ 9 * (refreshCount + 1) * 2
  return {
    flowerKinds, vaseCount, refreshCount,
    vaseCapacity: 3,
    timeLimit: Math.max(240, 300 - 6*tier),
  };
}
```

### 3.5 关卡内容生成算法
```
input: vaseCount V, refreshCount K, flowerKinds N
1. 估算总花数 T ≈ V * (K+1) * 2，向上取到 3*N 的倍数（每种花数都 ≥ 3 且都是 3 的倍数）
2. 把 T 朵花按 N 种均分（每种 T/N），各种数量都是 3 的倍数
3. 摇匀总池子 pool
4. 为每个 vase 生成 K+1 个批次：
   for each batch:
     size = random(1..3)
     从 pool 取 size 朵
     检查："是否 3 朵全同款"？若是，把第 3 朵和池子里下一朵不同色的交换
   保存 batches = [batch0, batch1, ..., batchK]
5. 每个 vase 初始：
   flowers = batches[0]
   nextBatchShadow = batches[1] || []
   queuedBatches = batches[2..K]
   refreshCount = K
```

---

## 4. 数据结构

```ts
// 花朵
interface Flower {
  id: string;            // 唯一 ID
  kind: number;          // 花种 0..N-1
}

// 花瓶
interface Vase {
  id: string;
  shelfRow: number;       // 0..2
  shelfCol: number;       // 0..2
  flowers: Flower[];      // 当前可见花朵，长度 0..3
  nextBatch: Flower[];    // 黑影预览（下一批），1..3 朵
  queuedBatches: Flower[][]; // 之后还未上场的批次
  refreshCount: number;   // 剩余刷新次数（= nextBatch?1:0 + queuedBatches.length）
}

// 关卡状态
interface LevelState {
  config: LevelConfig;
  vases: Vase[];          // 默认 9 个
  score: number;
  combo: number;
  timeRemaining: number;
  status: 'playing' | 'won' | 'lost';
  inventory: { wand: number; pause: number; refresh: number };
  dragging?: {            // 当前正在拖拽的花朵（如果有）
    fromVaseId: string;
    flowerIdx: number;
    flower: Flower;
  };
}
```

---

## 5. UI / 界面

### 5.1 主要界面
1. **主菜单 Home**：背景樱花、用户头像、金币/爱心/化肥三种资源、入口按钮（开始 / 我的花店 / 商城 / 设置 / 邀请 / 存钱罐）
2. **关卡内 Game**：顶部（暂停 / 关卡进度 / 计时 / 资源），中部（3 层花架 × 3 瓶，每瓶可容纳 3 朵 + 黑影预览），底部（道具栏 3 个）
3. **胜利结算 Victory**：胜利横幅、连胜数、奖励发放、下一关按钮
4. **失败结算 Defeat**：再来一次、看广告复活
5. **图鉴 Collection**：「插花作品 / 魅力称号 / 我的花房」三个 Tab（对应截图 2）

### 5.2 界面层级（Cocos Creator）
```
Canvas
├─ BG (背景层，永久)
├─ Game (游戏内场景)
│   ├─ TopHUD
│   ├─ ShelfContainer (Layout 节点)
│   │    └─ Vase x9 (内含 shadow-layer / flowers-layer / count-label)
│   ├─ DragLayer (浮动花朵跟随手指)
│   └─ ToolBar
├─ UI (弹窗层)
│   ├─ PauseDialog
│   ├─ VictoryDialog
│   ├─ DefeatDialog
│   └─ ShopDialog
└─ Effects (粒子、消除特效)
```

---

## 6. 技术架构 (Cocos Creator + 微信小游戏)

### 6.1 推荐目录结构
```
match-3-game/
├─ assets/
│  ├─ scenes/
│  │   ├─ Home.scene
│  │   ├─ Game.scene
│  │   └─ Loading.scene
│  ├─ prefabs/
│  │   ├─ Vase.prefab
│  │   ├─ Flower.prefab
│  │   ├─ TraySlot.prefab
│  │   └─ ToolButton.prefab
│  ├─ scripts/
│  │   ├─ core/
│  │   │   ├─ GameManager.ts         # 总控
│  │   │   ├─ LevelLoader.ts         # 加载/生成关卡
│  │   │   ├─ MatchEngine.ts         # 三消逻辑
│  │   │   └─ ScoreSystem.ts
│  │   ├─ entities/
│  │   │   ├─ FlowerEntity.ts
│  │   │   └─ VaseEntity.ts
│  │   ├─ ui/
│  │   │   ├─ TopHUD.ts
│  │   │   ├─ Tray.ts
│  │   │   ├─ ToolBar.ts
│  │   │   └─ VictoryDialog.ts
│  │   ├─ tools/
│  │   │   ├─ ToolWand.ts
│  │   │   ├─ ToolPause.ts
│  │   │   └─ ToolRefresh.ts
│  │   ├─ data/
│  │   │   ├─ LevelConfig.ts
│  │   │   └─ FlowerKinds.ts
│  │   └─ platform/
│  │       └─ WechatBridge.ts        # 广告/分享/存档
│  ├─ resources/
│  │   ├─ levels/level_001.json ...
│  │   ├─ atlas/flowers.png
│  │   └─ audio/
│  └─ shaders/
├─ build-templates/
│   └─ wechatgame/                   # 微信发布模版
├─ settings/
└─ project.json
```

### 6.2 关键模块设计

**GameManager**（单例，挂在 Game 场景根节点）
- 状态机：`Loading → Playing → Paused → Won/Lost`
- 暴露事件 bus：`onMatch`、`onFlowerPicked`、`onLevelEnd`

**MatchEngine**（纯逻辑、可单元测试）
- `addToTray(flower)` → 返回 `MatchResult`
- 内部维护 `tray: Flower[]`
- 每次添加后调用 `detectMatch()`：扫描相同 kind 计数 ≥ 3 → 取 3 个返回

**LevelLoader**
- 两种模式：
  1. 读 `resources/levels/level_xxx.json`（精心设计的前 30 关）
  2. 调 `generateLevel(n)`（30 关之后程序生成）

**WechatBridge**
- 封装：`wx.createRewardedVideoAd`、`wx.createInterstitialAd`
- 封装：`wx.setStorageSync` 存档
- 封装：`wx.shareAppMessage` 分享

### 6.3 渲染策略
- 花朵使用 **图集 atlas**（TexturePacker 打包），单张 ≤ 2048×2048
- 花瓶 9 宫格切图
- 背景使用 **大图 + 局部遮罩** 而非整图，节省内存

### 6.4 性能预算（微信小游戏）
| 项 | 上限 |
|---|---|
| 主包大小 | ≤ 4 MB |
| 总分包大小 | ≤ 20 MB |
| 单帧 Draw Call | ≤ 30 |
| 内存峰值 | ≤ 200 MB |
| FPS | ≥ 60（iPhone 8 基线） |

策略：
- 花朵图集合一张
- UI 用 Sliced 9-grid 减少图
- 关闭物理引擎（本游戏不需要）
- 粒子上限 ≤ 80

### 6.5 存档
本地：`wx.setStorageSync('save', state)`
云端（后期）：腾讯云开发 (CloudBase) + OpenID 关联

---

## 7. 关卡内事件 / 反馈细节
- 点击花朵：缩放 1.2 → 0.6 后飞入托盘（贝塞尔曲线 0.35s）
- 消除：3 朵花闪两下白光 → 同时缩放消失 → 粒子（同色花瓣 8 个）
- 连击 ≥ 3：屏幕中央显示 `Combo x3!` 黄色字体
- 时间 ≤ 30s：倒计时变红 + 心跳音效
- 倒计时归零：屏幕轻微变灰 → 弹失败窗
- 全消除：花瓶绽放金光，弹胜利窗

---

## 8. 商业化设计（仅蓝图，本期不实现）
- **看广告获道具**：每日 5 次
- **商城**：金币 → 道具
- **首充**：6 元解锁 30 个道具 + 永久去广告
- **存钱罐**：48 元到上限，存钱罐攒满后弹支付
- **关卡宝箱**：每 5 关给一个宝箱（金币 + 道具）

---

## 9. 开发路线图

| 里程碑 | 周期 | 交付 |
|---|---|---|
| **M0 — 原型** | 1 周 | HTML5 单文件，验证核心循环（**本次交付**） |
| **M1 — Cocos Demo** | 2 周 | 单关卡走通：花瓶、托盘、道具 |
| **M2 — 关卡系统** | 2 周 | 30 关数据、自动生成、保存进度 |
| **M3 — UI / 美术** | 3 周 | 接入正式美术 + 动效 + 音效 |
| **M4 — 微信发布** | 1 周 | 接入广告 SDK、分享、提审 |
| **M5 — 运营** | 持续 | 关卡更新、活动、节日皮肤 |

---

## 10. 风险与备选方案

| 风险 | 应对 |
|---|---|
| 玩法被误解（玩家不知道点哪里） | 第 1 关全程手把手新手教学 |
| 关卡难度跳跃过大 | 上线后做漏斗分析，按通关率热修复 |
| 微信小游戏审核驳回（赌博类素材） | 避免转盘 / 抽卡形式，宝箱内容直出 |
| 性能在低端机不达标 | 准备低画质模式（关粒子、降图集分辨率） |

---

附录：完整代码骨架请见 `ARCHITECTURE.md`，可玩原型请打开 `prototype.html`。
