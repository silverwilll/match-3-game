# 技术架构 — Cocos Creator 3.8 + 微信小游戏

> 配合 `GDD.md` 阅读。本文档给出从原型搬到生产的可执行方案。

---

## 1. 工程结构（推荐）

```
match-3-game/
├─ assets/
│  ├─ scenes/
│  │   ├─ Loading.scene
│  │   ├─ Home.scene
│  │   └─ Game.scene
│  ├─ prefabs/
│  │   ├─ Vase.prefab
│  │   ├─ Flower.prefab
│  │   ├─ TraySlot.prefab
│  │   ├─ ToolButton.prefab
│  │   └─ Dialog.prefab
│  ├─ scripts/
│  │   ├─ core/                      # 纯逻辑，可单测
│  │   │   ├─ GameManager.ts         # 关卡总控、计时、状态机
│  │   │   ├─ VaseModel.ts           # 花瓶数据 + 三同款检测 + 刷新
│  │   │   ├─ LevelGenerator.ts      # 关卡参数 & 批次生成
│  │   │   ├─ Inventory.ts
│  │   │   └─ EventBus.ts
│  │   ├─ entities/                  # 节点行为
│  │   │   ├─ FlowerEntity.ts
│  │   │   ├─ VaseEntity.ts          # 渲染 shadow / flowers / body
│  │   │   └─ DragController.ts      # 接管 touch/mouse 拖拽，找目标瓶
│  │   ├─ ui/                        # 视图层
│  │   │   ├─ TopHUD.ts
│  │   │   ├─ ToolBar.ts
│  │   │   ├─ VictoryDialog.ts
│  │   │   └─ DefeatDialog.ts
│  │   ├─ tools/                     # 道具实现
│  │   │   ├─ WandTool.ts
│  │   │   ├─ TimePauseTool.ts
│  │   │   └─ RefreshTool.ts
│  │   ├─ data/
│  │   │   ├─ LevelConfig.ts
│  │   │   ├─ FlowerKind.ts
│  │   │   └─ Save.ts
│  │   └─ platform/
│  │       └─ WechatBridge.ts        # wx.* 封装
│  ├─ resources/
│  │   ├─ levels/level_001.json …
│  │   ├─ atlas/flowers.plist + .png
│  │   └─ audio/{bgm.mp3, match.wav, …}
│  └─ shaders/                       # 暂无
├─ build-templates/
│   └─ wechatgame/                   # 微信发布模版
├─ settings/
└─ project.json
```

---

## 2. 关键脚本骨架（TypeScript）

### 2.1 `data/FlowerKind.ts`
```ts
export interface FlowerKind {
  id: number;            // 0..N-1
  name: string;          // 玫瑰 / 郁金香 / 向日葵 …
  color: string;         // #RRGGBB（仅占位时使用）
  spriteFrame?: string;  // atlas 内的帧名
}

export const FLOWER_KINDS: FlowerKind[] = [
  { id: 0, name: '玫瑰',     color: '#e74c3c', spriteFrame: 'rose' },
  { id: 1, name: '郁金香',   color: '#3498db', spriteFrame: 'tulip' },
  { id: 2, name: '向日葵',   color: '#f1c40f', spriteFrame: 'sunflower' },
  { id: 3, name: '樱花',     color: '#e91e63', spriteFrame: 'cherry' },
  { id: 4, name: '满天星',   color: '#27ae60', spriteFrame: 'baby_breath' },
  { id: 5, name: '紫郁金香', color: '#9b59b6', spriteFrame: 'tulip_p' },
  { id: 6, name: '万寿菊',   color: '#ff8c00', spriteFrame: 'marigold' },
  { id: 7, name: '马蹄莲',   color: '#1abc9c', spriteFrame: 'calla' },
  { id: 8, name: '木槿',     color: '#8e44ad', spriteFrame: 'hibiscus' },
];
```

### 2.2 `data/LevelConfig.ts`
```ts
export interface LevelConfig {
  level: number;
  flowerKinds: number;     // 本关花种类数
  vases: VaseSeed[];       // 9 个花瓶的初始构造
  timeLimit: number;       // 秒
  vaseCapacity: number;    // 每瓶可见上限（默认 3）
}
export interface VaseSeed {
  /** 长度为 refreshCount+1 的批次序列。
   *  batches[0] = 初始可见花朵；其余按顺序在瓶被清空时刷新出。 */
  batches: number[][];
}
```

### 2.3 `core/LevelGenerator.ts`
```ts
import { LevelConfig, VaseSeed } from '../data/LevelConfig';

export class LevelGenerator {
  static generate(n: number): LevelConfig {
    const tier = Math.min(Math.floor((n-1)/3), 6);
    const flowerKinds  = Math.min(3 + tier, 9);
    const vaseCount    = 9;
    const refreshCount = 2 + tier;
    const vaseCapacity = 3;

    // 估算总花数：每瓶平均每批 2 朵 × (refreshCount+1) 批
    let total = vaseCount * (refreshCount + 1) * 2;
    const unit = 3 * flowerKinds;            // 保证每种花是 3 的倍数
    total = Math.ceil(total / unit) * unit;

    const perKind = total / flowerKinds;
    const pool: number[] = [];
    for (let k = 0; k < flowerKinds; k++) {
      for (let i = 0; i < perKind; i++) pool.push(k);
    }
    LevelGenerator.shuffle(pool);

    // 每瓶分配批次
    const vases: VaseSeed[] = [];
    for (let i = 0; i < vaseCount; i++) {
      const budget = Math.floor(pool.length / (vaseCount - i));
      const myPool = pool.splice(0, budget);
      const batches: number[][] = [];
      for (let b = 0; b < refreshCount + 1; b++) {
        if (myPool.length === 0) { batches.push([]); continue; }
        const size = 1 + Math.floor(Math.random() * Math.min(3, myPool.length));
        let take = myPool.splice(0, size);
        // 禁止单批次 3 朵全同款
        if (take.length === 3 && take[0]===take[1] && take[1]===take[2]) {
          const swapIdx = myPool.findIndex(k => k !== take[0]);
          if (swapIdx >= 0) [myPool[swapIdx], take[2]] = [take[2], myPool[swapIdx]];
          else myPool.unshift(take.pop()!);
        }
        batches.push(take);
      }
      while (myPool.length) pool.unshift(myPool.shift()!);  // 退回剩余
      vases.push({ batches });
    }

    return {
      level: n,
      flowerKinds,
      vases,
      timeLimit: Math.max(240, 300 - 6*tier),
      vaseCapacity,
    };
  }

  private static shuffle<T>(a: T[]): void {
    for (let i = a.length-1; i > 0; i--) {
      const j = Math.floor(Math.random()*(i+1));
      [a[i], a[j]] = [a[j], a[i]];
    }
  }
}
```

### 2.4 `core/VaseModel.ts`（纯逻辑，**Cocos 无关，可 Node 单测**）
```ts
export interface Flower { id: string; kind: number; }

/**
 * 单个花瓶的数据模型：
 *   - flowers: 当前可见 (≤ 3)
 *   - nextBatch: 下一批黑影预览
 *   - queuedBatches: 之后未上场的批次
 *   - refreshCount: 剩余刷新次数
 *
 * 它不直接操作 UI，全部经事件回调返回，方便单元测试。
 */
export class VaseModel {
  flowers: Flower[] = [];
  nextBatch: Flower[] = [];
  queuedBatches: Flower[][] = [];
  refreshCount = 0;
  readonly id: string;
  readonly capacity: number;

  onMatch?: (kind: number, flowers: Flower[]) => void;
  onRefresh?: (newFlowers: Flower[]) => void;
  onChange?: () => void;

  constructor(id: string, capacity = 3) { this.id = id; this.capacity = capacity; }

  /** 装载关卡数据 */
  load(batches: Flower[][]) {
    this.flowers       = batches[0] || [];
    this.nextBatch     = batches[1] || [];
    this.queuedBatches = batches.slice(2);
    this.refreshCount  = (this.nextBatch.length ? 1 : 0) + this.queuedBatches.length;
    this.onChange?.();
  }

  /** 玩家把一朵花从本瓶取走 */
  removeFlower(idx: number): Flower | null {
    if (idx < 0 || idx >= this.flowers.length) return null;
    const f = this.flowers.splice(idx, 1)[0];
    this.onChange?.();
    if (this.flowers.length === 0 && this.refreshCount > 0) this.refresh();
    return f;
  }

  /** 玩家把一朵花插入本瓶（前置：未满） */
  addFlower(f: Flower): 'ok' | 'full' {
    if (this.flowers.length >= this.capacity) return 'full';
    this.flowers.push(f);
    this.onChange?.();
    this.checkMatch();
    return 'ok';
  }

  /** 检测 3 同款，若是则消除 */
  private checkMatch() {
    if (this.flowers.length === this.capacity
        && this.flowers.every(f => f.kind === this.flowers[0].kind)) {
      const kind = this.flowers[0].kind;
      const removed = this.flowers.splice(0, this.capacity);
      this.onMatch?.(kind, removed);
      this.onChange?.();
      if (this.refreshCount > 0) this.refresh();
    }
  }

  private refresh() {
    this.flowers = this.nextBatch;
    this.refreshCount--;
    this.nextBatch = this.queuedBatches.length > 0
      ? this.queuedBatches.shift()!
      : [];
    this.onRefresh?.(this.flowers.slice());
    this.onChange?.();
    // 防御：理论上刷新出的批次不会 3 同款（生成时已过滤）
    this.checkMatch();
  }

  isDone(): boolean { return this.flowers.length === 0 && this.refreshCount === 0; }
  isFull(): boolean { return this.flowers.length >= this.capacity; }
}
```

### 2.5 `core/GameManager.ts`（场景级单例）
```ts
import { _decorator, Component, Node } from 'cc';
import { LevelConfig } from '../data/LevelConfig';
import { LevelGenerator } from './LevelGenerator';
import { VaseModel, Flower } from './VaseModel';
import { EventBus } from './EventBus';
const { ccclass, property } = _decorator;

@ccclass('GameManager')
export class GameManager extends Component {
  static I: GameManager;

  @property(Node) shelfContainer!: Node;

  level = 1;
  config!: LevelConfig;
  vases: VaseModel[] = [];
  combo = 0;
  score = 0;
  timeRemaining = 300;
  status: 'playing'|'won'|'lost'|'paused' = 'playing';

  onLoad() { GameManager.I = this; }
  start() { this.loadLevel(1); }

  loadLevel(n: number) {
    this.level = n;
    this.config = LevelGenerator.generate(n);
    this.timeRemaining = this.config.timeLimit;
    this.combo = 0; this.score = 0;
    this.status = 'playing';

    this.vases = this.config.vases.map((seed, i) => {
      const v = new VaseModel('v'+i, this.config.vaseCapacity);
      const batches = seed.batches.map(b =>
        b.map(k => ({ id: 'f'+Math.random().toString(36).slice(2,9), kind: k }))
      );
      v.load(batches);
      v.onMatch = (kind, fs) => this.onMatchTriggered(v, kind, fs);
      v.onChange = () => EventBus.emit('vase:update', v);
      return v;
    });

    EventBus.emit('level:start', this.config);
    this.schedule(this.tick, 1);
  }

  /** DragController 在合法 drop 时调用此函数 */
  moveFlower(fromId: string, flowerIdx: number, toId: string): boolean {
    if (this.status !== 'playing') return false;
    if (fromId === toId) return false;
    const from = this.vases.find(v => v.id === fromId);
    const to   = this.vases.find(v => v.id === toId);
    if (!from || !to || to.isFull()) return false;
    const f = from.removeFlower(flowerIdx);
    if (!f) return false;
    to.addFlower(f);   // 内部会自动 checkMatch / refresh
    this.checkLock();
    this.checkWin();
    return true;
  }

  private onMatchTriggered(_v: VaseModel, kind: number, flowers: Flower[]) {
    this.combo++;
    const gain = 10 * this.combo;
    this.score += gain;
    EventBus.emit('match', { kind, flowers, combo: this.combo, gain });
    this.scheduleOnce(()=> this.combo = 0, 2.2);
  }

  private checkWin() {
    if (this.vases.every(v => v.isDone())) this.endLevel(true);
  }
  private checkLock() {
    const allFull   = this.vases.every(v => v.isFull());
    if (!allFull) return;
    const anyMatch  = this.vases.some(v => v.flowers.length === v.capacity
                          && v.flowers.every(f => f.kind === v.flowers[0].kind));
    if (anyMatch) return; // 即将自动消除
    this.endLevel(false, 'locked');
  }

  tick() {
    if (this.status !== 'playing') return;
    this.timeRemaining--;
    EventBus.emit('time:update', this.timeRemaining);
    if (this.timeRemaining <= 0) this.endLevel(false, 'timeout');
  }

  endLevel(won: boolean, reason='') {
    if (this.status !== 'playing') return;
    this.status = won ? 'won' : 'lost';
    this.unschedule(this.tick);
    EventBus.emit('level:end', { won, reason, score: this.score, time: this.timeRemaining });
  }
}
```

### 2.5b `entities/DragController.ts`
```ts
import { _decorator, Component, Node, EventTouch, Vec3, UITransform, find, instantiate } from 'cc';
import { GameManager } from '../core/GameManager';
const { ccclass, property } = _decorator;

@ccclass('DragController')
export class DragController extends Component {
  @property(Node) dragLayer!: Node;       // 浮动花朵的父节点（Canvas 顶层）

  private dragging?: {
    fromVaseId: string;
    flowerIdx: number;
    ghost: Node;
  };

  /** 由 FlowerEntity 在 touch start 时调用 */
  beginDrag(fromVaseId: string, flowerIdx: number, ghost: Node) {
    this.dragging = { fromVaseId, flowerIdx, ghost };
    this.dragLayer.addChild(ghost);
    // 注册触摸事件到 Canvas 根节点
    const root = find('Canvas')!;
    root.on(Node.EventType.TOUCH_MOVE, this.onMove, this);
    root.on(Node.EventType.TOUCH_END,  this.onEnd,  this);
    root.on(Node.EventType.TOUCH_CANCEL, this.onEnd,  this);
  }

  private onMove(e: EventTouch) {
    if (!this.dragging) return;
    const ui = this.dragLayer.getComponent(UITransform)!;
    const p = ui.convertToNodeSpaceAR(new Vec3(e.getUILocation().x, e.getUILocation().y, 0));
    this.dragging.ghost.setPosition(p);
  }

  private onEnd(e: EventTouch) {
    const root = find('Canvas')!;
    root.off(Node.EventType.TOUCH_MOVE, this.onMove, this);
    root.off(Node.EventType.TOUCH_END,  this.onEnd,  this);
    root.off(Node.EventType.TOUCH_CANCEL, this.onEnd, this);

    if (!this.dragging) return;
    const d = this.dragging;
    this.dragging = undefined;

    const targetVaseId = this.hitTestVase(e.getUILocation().x, e.getUILocation().y);
    if (targetVaseId && targetVaseId !== d.fromVaseId) {
      const ok = GameManager.I.moveFlower(d.fromVaseId, d.flowerIdx, targetVaseId);
      if (ok) { d.ghost.destroy(); return; }
    }
    // 不合法 → 弹回（在 ghost 上播 tween 后 destroy）
    d.ghost.destroy(); // 简化：原型阶段直接销毁，正式版加弹回动画
  }

  /** UI 空间坐标 → 命中的 vase id（遍历 shelfContainer 的子节点） */
  private hitTestVase(x: number, y: number): string | null {
    const shelves = GameManager.I.shelfContainer;
    for (const shelf of shelves.children) {
      for (const vaseNode of shelf.children) {
        if (vaseNode.name.startsWith('v') === false) continue;
        const ui = vaseNode.getComponent(UITransform)!;
        const localP = ui.convertToNodeSpaceAR(new Vec3(x, y, 0));
        if (ui.getBoundingBox().contains(new Vec3(localP.x, localP.y, 0) as any)) {
          return vaseNode.name;  // 用 vase id 命名 node
        }
      }
    }
    return null;
  }
}
```

### 2.6 `core/EventBus.ts`
```ts
type Cb = (...args: any[]) => void;
class _EventBus {
  private map = new Map<string, Cb[]>();
  on(ev: string, cb: Cb) { (this.map.get(ev) || this.map.set(ev, []).get(ev)!).push(cb); }
  off(ev: string, cb: Cb) {
    const arr = this.map.get(ev); if (!arr) return;
    const i = arr.indexOf(cb); if (i>=0) arr.splice(i,1);
  }
  emit(ev: string, ...args: any[]) { this.map.get(ev)?.forEach(cb => cb(...args)); }
}
export const EventBus = new _EventBus();
```

### 2.7 `entities/VaseEntity.ts`
```ts
import { _decorator, Component, Node, Label, instantiate, Prefab, tween, Vec3 } from 'cc';
import { VaseModel } from '../core/VaseModel';
import { FlowerEntity } from './FlowerEntity';
import { EventBus } from '../core/EventBus';
const { ccclass, property } = _decorator;

@ccclass('VaseEntity')
export class VaseEntity extends Component {
  @property(Label) countLabel!: Label;
  @property(Prefab) flowerPrefab!: Prefab;
  @property(Node) flowersLayer!: Node;
  @property(Node) shadowLayer!: Node;

  model!: VaseModel;

  bind(model: VaseModel) {
    this.model = model;
    this.node.name = model.id;
    EventBus.on('vase:update', this.onUpdate.bind(this));
    this.render();
  }

  private onUpdate(v: VaseModel) {
    if (v.id !== this.model.id) return;
    this.render();
  }

  private render() {
    this.countLabel.string = this.model.refreshCount > 0 ? String(this.model.refreshCount) : '';
    this.flowersLayer.removeAllChildren();
    this.shadowLayer.removeAllChildren();

    this.model.flowers.forEach((f, i) => {
      const node = instantiate(this.flowerPrefab);
      const fe = node.getComponent(FlowerEntity)!;
      fe.bindAsFlower(f, this.model.id, i);
      this.flowersLayer.addChild(node);
    });
    this.model.nextBatch.forEach((f, i) => {
      const node = instantiate(this.flowerPrefab);
      const fe = node.getComponent(FlowerEntity)!;
      fe.bindAsShadow(f);
      this.shadowLayer.addChild(node);
    });
  }
}
```

### 2.8 `platform/WechatBridge.ts`
```ts
declare const wx: any;
export class WechatBridge {
  /** 看广告获得奖励 */
  static showRewardedAd(adUnitId: string): Promise<boolean> {
    return new Promise(resolve => {
      if (typeof wx === 'undefined') { resolve(true); return; } // 调试环境
      const ad = wx.createRewardedVideoAd({ adUnitId });
      ad.onClose((res: any) => resolve(!!res.isEnded));
      ad.load().then(()=> ad.show());
    });
  }
  static save(key: string, val: any) {
    if (typeof wx !== 'undefined') wx.setStorageSync(key, val);
    else localStorage.setItem(key, JSON.stringify(val));
  }
  static load<T=any>(key: string): T | null {
    if (typeof wx !== 'undefined') return wx.getStorageSync(key) || null;
    const v = localStorage.getItem(key); return v ? JSON.parse(v) : null;
  }
  static share(title: string, imageUrl: string) {
    if (typeof wx === 'undefined') return;
    wx.shareAppMessage({ title, imageUrl });
  }
}
```

---

## 3. 预制体 (Prefab) 设计

### 3.1 `Flower.prefab`
- Root (Node, 32×48)
  - `stem` (Sprite, 3×28，绿色) — 居中底部
  - `head` (Sprite, 24×24，圆形) — 居中偏上
  - `headLabel` (Label, optional，调试期显示 kind) — 居中
- 组件：`FlowerEntity` (kind, vaseId, stackIndex)

### 3.2 `Vase.prefab`
- Root (Node, 90×140) + UITransform（**name 在运行时被赋成 vaseId 用于命中测试**）
  - `vase_body` (Sprite, 60×72) — 底部，z 最低
  - `count_label` (Label, 22pt 白色，描边) — vase_body 中央
  - `shadow_layer` (Node + Layout HORIZONTAL gap=0) — vase_body 上方，z=1，**半透明黑色**
  - `flowers_layer` (Node + Layout HORIZONTAL gap=0) — 与 shadow_layer 同位置，z=2
- 组件：`VaseEntity`
- 注：拖拽事件挂在 **flower** 上，不在 vase 上。Vase 仅作为放置目标接收命中测试。

### 3.4 `ToolButton.prefab`
- Root (Node, 90×72)
  - `bg` (Sprite, 蓝色按钮)
  - `icon` (Sprite, 28×28)
  - `name` (Label, 12pt)
  - `badge` (Sprite + Label, 右上角红圈)

---

## 4. 关卡数据格式（手工配置示例）

`assets/resources/levels/level_001.json`:
```json
{
  "level": 1,
  "flowerKinds": 3,
  "vaseCapacity": 3,
  "timeLimit": 300,
  "vases": [
    { "batches": [[0,1],   [2,0,1], [1,2]] },
    { "batches": [[2,1,0], [0,2],   [1,0,2]] },
    { "batches": [[1,2],   [0,1,2], [2,0]] },
    { "batches": [[0,2,1], [1,0],   [2,1,0]] },
    { "batches": [[2,0],   [1,2,0], [0,1]] },
    { "batches": [[0,1,2], [2,1],   [0,2,1]] },
    { "batches": [[1,0,2], [0,2,1], [2,0]] },
    { "batches": [[2,1,0], [1,0,2], [0,2]] },
    { "batches": [[0,2],   [1,2,0], [2,1,0]] }
  ]
}
```

> 每个瓶的 `batches[0]` 是初始可见，`batches[1]` 是黑影预览，后面按顺序刷新。
> 前 30 关建议手工调（保证可解 & 难度曲线），30 关后用 `LevelGenerator` 程序生成。

---

## 5. 数据流总览

```
┌──────────┐  drag start  ┌────────────────┐
│ Player   │ ───────────► │ FlowerEntity   │
└──────────┘              │ (touch begin)  │
                          └────────┬───────┘
                                   │ beginDrag(fromId, idx, ghostNode)
                                   ▼
                          ┌────────────────┐
                          │ DragController │ ◄─── touch move (跟随手指)
                          └────────┬───────┘
                                   │ touch end → hit test → targetId
                                   ▼
                          ┌────────────────┐ moveFlower(from, idx, to)
                          │  GameManager   │
                          └────────┬───────┘
                                   │ from.removeFlower → maybe refresh
                                   │ to.addFlower → checkMatch → maybe refresh
                                   ▼
                          ┌────────────────┐
                          │ VaseModel × 9  │ —— onMatch / onChange events
                          └────────┬───────┘
                                   ▼
                          ┌────────────────┐
                          │ EventBus.emit  │
                          └────┬───────────┘
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
        ┌────────────┐  ┌──────────┐   ┌──────────┐
        │ VaseEntity │  │ TopHUD   │   │ Effects  │
        │ (重渲)     │  │ (分数/计时)│   │ (粒子)    │
        └────────────┘  └──────────┘   └──────────┘
```

---

## 6. 性能 / 资源策略

| 项 | 做法 |
|---|---|
| 图集 | 9 种花朵打 1 张 `flowers.png`（512×512），TexturePacker |
| UI | 9-grid sliced sprite |
| 字体 | 系统字体（中文用 `system`），避免内嵌字库 |
| 粒子 | 不用 cc.ParticleSystem2D，自己用 Graphics + tween 实现（≤ 80 个/秒）|
| 物理 | **关闭**物理引擎（设置 → 项目 → 关闭 box2d） |
| 音频 | 短音效用 `wx.createInnerAudioContext()`，BGM 用 cc.AudioSource |
| 内存 | 切场景时调 `assetManager.releaseAsset()` |

---

## 7. 微信小游戏发布步骤

1. **创建 AppID**：在 `mp.weixin.qq.com` 注册小游戏。
2. **Cocos 项目设置**：
   - 项目设置 → 项目数据 → 平台勾选 `微信小游戏`
   - 构建发布 → 构建平台选择 `微信小游戏`
   - AppID 填入
3. **构建模板**：将 `build-templates/wechatgame/` 中的 `project.config.json`、`game.json`、`game.js` 调整为本游戏配置。
4. **打开微信开发者工具**：导入 `build/wechatgame` 目录。
5. **测试**：扫码真机预览，关注 FPS / 内存。
6. **上传 → 提审**：注意：
   - 不能含转盘、抽卡形式（赌博风险）
   - 主包必须 < 4 MB
   - 必须接入「实名认证」「健康提示」（微信会注入）
   - 广告位需在小游戏后台先创建广告单元 ID

### 7.1 `game.json` 关键字段
```json
{
  "deviceOrientation": "portrait",
  "showStatusBar": false,
  "networkTimeout": { "request": 8000 },
  "subpackages": [
    { "name": "levels", "root": "assets/resources/levels" }
  ]
}
```

---

## 8. 单元测试建议

`VaseModel` 和 `LevelGenerator` 都是纯函数，**强烈建议**写 Jest 测试：

```ts
// __tests__/VaseModel.test.ts
import { VaseModel } from '../assets/scripts/core/VaseModel';

test('three of a kind in vase triggers match', () => {
  const v = new VaseModel('v0', 3);
  const matches: any[] = [];
  v.onMatch = (k,fs) => matches.push({ k, n: fs.length });
  v.load([[{id:'a',kind:0},{id:'b',kind:0}]]);  // 仅 1 批，无刷新
  v.addFlower({ id:'c', kind: 0 });
  expect(matches).toEqual([{ k: 0, n: 3 }]);
  expect(v.flowers.length).toBe(0);
  expect(v.refreshCount).toBe(0);
});

test('empty vase auto-refreshes next batch', () => {
  const v = new VaseModel('v0', 3);
  v.load([
    [{id:'a',kind:0}],
    [{id:'b',kind:1},{id:'c',kind:2}],
  ]);
  expect(v.refreshCount).toBe(1);
  v.removeFlower(0);
  expect(v.flowers.map(f=>f.kind)).toEqual([1,2]);
  expect(v.refreshCount).toBe(0);
});
```

---

## 9. 从原型迁移到 Cocos 的步骤

| Step | 工作 | 工时估计 |
|---|---|---|
| 1 | 新建 Cocos 工程，新建 `Game.scene` 和 4 个 Prefab | 0.5 天 |
| 2 | 把 `prototype.html` 的逻辑搬到 `MatchEngine.ts` / `GameManager.ts` / `LevelGenerator.ts` | 1 天 |
| 3 | 写 `VaseEntity` `FlowerEntity` `TrayUI` `TopHUD` `ToolBar` | 1.5 天 |
| 4 | 接入图集（先用方块色块占位） | 0.5 天 |
| 5 | 接入 `EventBus`，跑通点击 → 三消 → 数字归 0 | 1 天 |
| 6 | 道具实现（魔法棒/时停/刷新） | 1 天 |
| 7 | 胜负弹窗、关卡推进、保存进度 | 0.5 天 |
| 8 | 微信平台接入（广告位、分享、存档） | 1 天 |
| 9 | 美术资源替换、调动效、音效 | 视美术速度 |
| **合计** | **MVP** | **7 ~ 9 天** |

---

## 10. 常见坑提示

1. **iOS 微信 touch 事件**：Cocos 默认会处理多点触控，但要在 `Game.scene` 根节点关闭 `multi-touch`，否则双指会触发奇怪行为。
2. **Label 描边失效**：微信小游戏对 Label 描边支持不完整，建议用 `BMFont` 位图字。
3. **`scheduleOnce` 在场景切换时不清**：手动 `unscheduleAllCallbacks()`。
4. **`window` 不存在**：纯 wx 环境没 `window`，封装 polyfill 或用 `globalThis`。
5. **图片加载白屏**：`resources/atlas/flowers.png` 不能用 `loadRemote`，要走 `cc.resources.load`。
6. **首包过大被拒**：把 `levels/*.json` 放进子包；把背景大图改成 ASTC 压缩。

---

附：可玩原型 `prototype.html`、设计文档 `GDD.md`。
