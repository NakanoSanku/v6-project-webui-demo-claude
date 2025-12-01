# 阴阳师游戏自动化脚本开发计划

## 项目概述

基于现有的 **AutoX v6 + Vue 3 + TypeScript WebView** 混合架构，开发阴阳师手游自动化脚本。

**核心目标：**
- Web UI 提供现代化配置界面和实时监控
- AutoX 脚本使用无障碍服务实现游戏自动化
- 通过 JS Bridge 实现双向通信和状态同步

**技术特点：**
- 前端：Vue 3 + Framework7 + TypeScript
- 脚本：AutoX v6 + TypeScript + 截图找图/找色
- 通信：基于 JSBridge 的异步消息协议

---

## 技术架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      Web UI (Vue 3)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Dashboard   │  │  Config      │  │  Log         │     │
│  │  (控制面板)   │  │  (任务配置)   │  │  (实时日志)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                           │                                 │
│                    ┌──────▼──────┐                         │
│                    │  WebBridge  │                         │
│                    └──────┬──────┘                         │
└───────────────────────────┼─────────────────────────────────┘
                            │ JS Bridge (JSON Messages)
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                   AutoX Script Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  AutoxBridge │  │ TaskManager  │  │   Logger     │     │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘     │
│         │                 │                                 │
│  ┌──────▼─────────────────▼─────────────────────┐          │
│  │          Core Services Layer                 │          │
│  │  ┌─────────────┐  ┌─────────────┐           │          │
│  │  │ScreenService│  │InputService │           │          │
│  │  │(截图/找图)   │  │(点击/滑动)   │           │          │
│  │  └─────────────┘  └─────────────┘           │          │
│  │  ┌─────────────────────────────────┐        │          │
│  │  │    GameRecognizer              │        │          │
│  │  │    (场景识别/状态判断)          │        │          │
│  │  └─────────────────────────────────┘        │          │
│  └──────────────────────────────────────────────┘          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Task Layer (状态机)                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│  │  │ YuhunTask│  │ExploreTask│ │BreakTask │  ...     │   │
│  │  └──────────┘  └──────────┘  └──────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  Android OS   │
                    │ (无障碍服务)   │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │   阴阳师游戏   │
                    └───────────────┘
```

---

## 核心技术方案

### 1. 游戏画面识别

**挑战：** 游戏使用 OpenGL/Unity 渲染，无 DOM 节点可用。

**解决方案：** 截图 + 图像识别（混合策略）

| 技术 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **findImage** | 按钮图标、式神头像 | 精确匹配，容错性好 | CPU 开销较大 |
| **findColor/findMultiColors** | 大块颜色区域（血条、金色文字） | 性能高，速度快 | 对颜色变化敏感 |
| **OCR** | 动态数值（体力、数量） | 通用性强 | 识别率不稳定 |

**优化策略：**
- 区域裁剪：限定 `region` 参数，避免全屏搜索
- 锚点定位：先找固定图标，再偏移找按钮
- 阈值调优：`threshold` 设为 0.8-0.9，容忍画质差异
- 状态驱动：根据游戏状态选择检测区域和频率

### 2. 无障碍操作

**API 选择：**
- `click(x, y)`: 快速点击，适合按钮操作
- `press(x, y, duration)`: 长按，适合拖动起点
- `swipe(x1, y1, x2, y2, duration)`: 滑动，适合地图移动
- `gestures([...])`: 复杂手势（曲线、多指）

**防检测随机化：**
```typescript
function humanTap(rect: Rect) {
  const cx = rect.x + rect.w * random(0.3, 0.7);  // 区域内随机
  const cy = rect.y + rect.h * random(0.3, 0.7);
  const duration = random(60, 200);               // 随机按压时长
  const preDelay = random(80, 260);               // 随机延迟
  sleep(preDelay);
  press(cx, cy, duration);
  sleep(random(200, 500));                        // 随机后延迟
}
```

### 3. 状态机设计

**状态定义：**
```typescript
enum GameState {
  UNKNOWN = 0,
  HOME_COURTYARD = 1,    // 庭院
  MAP_SELECT = 2,        // 副本选择
  PREPARING = 3,         // 备战界面
  COMBAT = 4,            // 战斗中
  RESULT_WIN = 5,        // 胜利结算
  RESULT_LOSE = 6,       // 失败结算
  REWARD = 7,            // 奖励获取
}
```

**状态转移逻辑：**
```
PREPARING → (点击挑战) → COMBAT
COMBAT → (检测到胜利) → RESULT_WIN
RESULT_WIN → (点击开箱) → REWARD
REWARD → (点击确认) → PREPARING (循环)
```

**异常处理：**
- 全局扫描：定期检测网络错误、更新提示等弹窗
- 超时保护：战斗超时（5分钟）自动退出
- 安全回退：未知状态时尝试返回主界面

---

## 开发阶段规划

**采用"垂直切片"模式**，每个阶段都交付可运行的完整功能。

### Phase 1: 骨架与联通 (The Skeleton)

**目标：** 打通 Web ↔ AutoX 通信链路

**交付物：**
1. ✅ 完善 `js_bridge.ts`，实现 Request/Response/Event 消息协议
2. ✅ Web UI 基础框架（Dashboard.vue + 基础布局）
3. ✅ AutoX 端 Bridge 实现（AutoxBridge 类）
4. ✅ 日志系统：AutoX → Web UI 实时日志显示

**验收标准：**
- Web 页面能发送 JSON 消息到 AutoX
- AutoX 能接收消息并回传日志到 Web 显示
- 日志窗口实时更新，无明显延迟

**预计时间：** 2-3 天

---

### Phase 2: 最小可行性原型 (MVP - Hello World)

**目标：** 验证 AutoX 在真实设备上的能力和环境兼容性

**交付物：**
1. ✅ "环境检测"功能：
   - 申请截图权限
   - 获取屏幕分辨率
   - 执行一次截图 + 找色操作
   - 模拟随机点击
2. ✅ 将检测结果返回 Web UI 显示

**核心代码示例：**
```typescript
// AutoX 端
bridge.onRequest('demo.checkEnv', async () => {
  const ok = requestScreenCapture(false);
  if (!ok) throw new Error('截屏权限申请失败');

  const img = captureScreen();
  const resolution = `${device.width}x${device.height}`;

  // 测试找色
  const redPoint = images.findColor(img, colors.parseColor('#ff0000'));
  img.recycle();

  return {
    captureOk: true,
    resolution,
    foundRed: redPoint !== null
  };
});
```

**验收标准：**
- 在真实 Android 设备上运行成功
- 截图权限申请正常
- 找色操作返回正确结果
- Web UI 显示设备信息

**预计时间：** 2-3 天

**⚠️ 关键里程碑：** 此阶段成功后，80% 的技术风险被排除。

---

### Phase 3: 核心封装与重构 (Core Services)

**目标：** 基于 Phase 2 的原型代码，提取可复用的核心服务类

**交付物：**

#### 3.1 ScreenService (截图/找图服务)
```typescript
export class ScreenService {
  private templates = new Map<string, Image>();
  private captureReady = false;

  ensureCaptureReady(): boolean {
    if (this.captureReady) return true;
    this.captureReady = requestScreenCapture(false);
    return this.captureReady;
  }

  capture(): Image | null {
    return captureScreen();
  }

  findTemplate(name: string, options?: FindImageOptions): Point | null {
    const img = this.capture();
    if (!img) return null;
    try {
      const template = this.getTemplate(name);
      return images.findImage(img, template, options);
    } finally {
      img.recycle();
    }
  }

  loadTemplate(name: string, path: string): void {
    const img = images.read(path);
    if (img) this.templates.set(name, img);
  }

  private getTemplate(name: string): Image {
    const img = this.templates.get(name);
    if (!img) throw new Error(`Template not found: ${name}`);
    return img;
  }

  dispose(): void {
    for (const img of this.templates.values()) {
      img.recycle();
    }
    this.templates.clear();
  }
}
```

#### 3.2 InputService (输入服务)
```typescript
export class InputService {
  constructor(private config: InputConfig) {}

  tap(rect: Rect): void {
    const cx = rect.x + rect.w * random(0.3, 0.7);
    const cy = rect.y + rect.h * random(0.3, 0.7);
    const duration = random(...this.config.clickDelay);
    sleep(random(...this.config.preDelay));
    press(cx, cy, duration);
    sleep(random(...this.config.postDelay));
  }

  swipeRandom(from: Rect, to: Rect): void {
    const x1 = from.x + from.w * random(0.4, 0.6);
    const y1 = from.y + from.h * random(0.4, 0.6);
    const x2 = to.x + to.w * random(0.4, 0.6);
    const y2 = to.y + to.h * random(0.4, 0.6);
    const duration = random(300, 600);
    swipe(x1, y1, x2, y2, duration);
  }
}
```

#### 3.3 GameRecognizer (场景识别)
```typescript
export class GameRecognizer {
  constructor(private screen: ScreenService) {}

  detectState(): GameState {
    // 按优先级检测
    if (this.isCombat()) return GameState.COMBAT;
    if (this.isResultWin()) return GameState.RESULT_WIN;
    if (this.isPreparing()) return GameState.PREPARING;
    return GameState.UNKNOWN;
  }

  private isCombat(): boolean {
    // 检测左上角行动条或自动按钮
    return this.screen.findTemplate('combat_indicator') !== null;
  }

  private isResultWin(): boolean {
    // 检测"胜利"金色文字区域
    const img = this.screen.capture();
    if (!img) return false;
    try {
      const region = [w * 0.3, h * 0.1, w * 0.4, h * 0.2];
      return images.findColor(img, GOLD_COLOR, { region }) !== null;
    } finally {
      img.recycle();
    }
  }
}
```

#### 3.4 依赖注入容器
```typescript
// src-autox/core/context.ts
export interface AppServices {
  screen: ScreenService;
  input: InputService;
  recognizer: GameRecognizer;
  bridge: AutoxBridge;
  taskManager: TaskManager;
  logger: Logger;
}

export function createAppServices(): AppServices {
  const logger = new Logger();
  const screen = new ScreenService({ logger });
  const input = new InputService({
    clickDelay: [60, 200],
    preDelay: [80, 260],
    postDelay: [200, 500]
  });
  const bridge = new AutoxBridge(/* ... */, logger);
  const recognizer = new GameRecognizer({ screen, logger });
  const taskManager = new TaskManager({
    screen, input, recognizer, bridge, logger
  });

  return { screen, input, recognizer, bridge, taskManager, logger };
}
```

**验收标准：**
- 所有服务类单元可测试
- 通过 DI 容器统一管理生命周期
- 内存无泄漏（Image 正确 recycle）

**预计时间：** 3-4 天

---

### Phase 4: 任务系统与状态机 (Task Framework)

**目标：** 实现通用任务框架和状态机引擎

#### 4.1 任务基类
```typescript
export abstract class BaseTask {
  protected state: GameState = GameState.UNKNOWN;
  protected running = false;

  constructor(
    protected name: string,
    protected services: AppServices
  ) {}

  async start(): Promise<void> {
    this.running = true;
    this.services.logger.info(`Task ${this.name} started`);

    while (this.running) {
      try {
        this.state = this.services.recognizer.detectState();
        await this.tick();
        sleep(500); // 状态检测周期
      } catch (e) {
        this.handleError(e);
      }
    }
  }

  stop(): void {
    this.running = false;
    this.services.logger.info(`Task ${this.name} stopped`);
  }

  protected abstract tick(): Promise<void>;

  protected abstract handleError(e: Error): void;
}
```

#### 4.2 御魂任务示例
```typescript
export class YuhunTask extends BaseTask {
  private count = 0;
  private targetCount: number;

  constructor(services: AppServices, config: YuhunConfig) {
    super('Yuhun', services);
    this.targetCount = config.targetCount;
  }

  protected async tick(): Promise<void> {
    const { state } = this;
    const { screen, input, logger, bridge } = this.services;

    switch (state) {
      case GameState.PREPARING:
        const btn = screen.findTemplate('challenge_btn');
        if (btn) {
          input.tap(btn);
          logger.info('点击挑战按钮');
        }
        break;

      case GameState.COMBAT:
        // 战斗中等待
        sleep(1000);
        break;

      case GameState.RESULT_WIN:
        input.tap(SCREEN_CENTER); // 点击开箱
        logger.info('战斗胜利，点击结算');
        break;

      case GameState.REWARD:
        input.tap(SCREEN_CENTER); // 点击确认
        this.count++;
        bridge.emitEvent('task.progress', {
          round: this.count,
          total: this.targetCount
        });
        if (this.count >= this.targetCount) {
          this.stop();
        }
        break;
    }
  }

  protected handleError(e: Error): void {
    this.services.logger.error('Task error: ' + e.message);
    // 尝试恢复或停止任务
  }
}
```

#### 4.3 TaskManager
```typescript
export class TaskManager {
  private currentTask: BaseTask | null = null;

  constructor(private services: AppServices) {
    this.setupBridgeHandlers();
  }

  private setupBridgeHandlers(): void {
    this.services.bridge.onRequest('task.start', async (payload) => {
      const task = this.createTask(payload);
      this.currentTask = task;
      await task.start();
    });

    this.services.bridge.onRequest('task.stop', async () => {
      this.currentTask?.stop();
      this.currentTask = null;
    });
  }

  private createTask(config: TaskConfig): BaseTask {
    switch (config.type) {
      case TaskType.SOUL:
        return new YuhunTask(this.services, config);
      // ... 其他任务类型
      default:
        throw new Error('Unknown task type');
    }
  }
}
```

**验收标准：**
- 任务可启动、暂停、停止
- 状态机正确转移
- 进度实时上报到 Web UI

**预计时间：** 4-5 天

---

### Phase 5: 具体游戏逻辑实现

**目标：** 实现阴阳师各类日常任务

**任务列表：**
1. ✅ 御魂副本（魂十/魂土）
2. ✅ 觉醒副本
3. ✅ 探索（狗粮刷经验）
4. ✅ 结界突破
5. ✅ 活动副本

**每个任务需要：**
- 准备游戏截图素材（按钮、图标）
- 编写状态识别逻辑
- 实现状态转移处理
- 异常恢复流程

**预计时间：** 每个任务 2-3 天，总计 10-15 天

---

### Phase 6: 优化与健壮性

**目标：** 提升脚本稳定性和用户体验

**优化项：**

#### 6.1 异常处理
- 网络断线自动重连
- 系统弹窗自动关闭
- 游戏更新提示处理
- 体力不足自动购买（可选）

#### 6.2 性能优化
- 图片资源懒加载
- 截图降采样（2K → 1080p）
- 定时内存清理
- 日志批量发送（降低 Bridge 频率）

#### 6.3 UI 完善
- 任务配置表单验证
- 实时统计图表（掉落、效率）
- 日志导出功能
- 主题切换（深色模式）

#### 6.4 防封控增强
- 点击轨迹曲线化
- 操作间隔正态分布随机
- 偶发"失误点击"
- 人工暂停模拟

**预计时间：** 5-7 天

---

## 代码架构设计

### 目录结构

```
v6-project-webui-demo/
├── src/                          # Web UI 源码
│   ├── components/
│   │   ├── Dashboard.vue         # 控制面板
│   │   ├── ConfigPanel.vue       # 任务配置
│   │   ├── LogTerminal.vue       # 日志终端
│   │   └── StatsCard.vue         # 统计卡片
│   ├── composables/
│   │   ├── useBridge.ts          # Bridge 通信 hook
│   │   ├── useTaskManager.ts     # 任务管理 hook
│   │   └── useLogger.ts          # 日志管理 hook
│   ├── stores/
│   │   ├── configStore.ts        # 配置状态（Pinia）
│   │   └── taskStore.ts          # 任务状态
│   ├── types/
│   │   ├── config.ts             # 配置类型定义
│   │   └── bridge.ts             # Bridge 消息类型
│   └── core/
│       └── WebBridge.ts          # Web 端 Bridge 实现
│
├── src-autox/                    # AutoX 脚本源码
│   ├── main.tsx                  # 脚本入口
│   ├── core/
│   │   ├── context.ts            # DI 容器
│   │   ├── ScreenService.ts      # 截图/找图服务
│   │   ├── InputService.ts       # 输入服务
│   │   ├── GameRecognizer.ts     # 场景识别
│   │   ├── Logger.ts             # 日志服务
│   │   └── AutoxBridge.ts        # AutoX Bridge
│   ├── tasks/
│   │   ├── BaseTask.ts           # 任务基类
│   │   ├── TaskManager.ts        # 任务管理器
│   │   ├── YuhunTask.ts          # 御魂任务
│   │   ├── ExploreTask.ts        # 探索任务
│   │   └── BreakTask.ts          # 结界突破任务
│   ├── types/
│   │   ├── autox.d.ts            # AutoX 全局类型
│   │   ├── config.ts             # 配置类型（与 Web 共享）
│   │   └── bridge.ts             # Bridge 类型（与 Web 共享）
│   └── assets/
│       └── templates/            # 游戏截图素材
│           ├── common/           # 通用按钮
│           ├── yuhun/            # 御魂相关
│           ├── explore/          # 探索相关
│           └── break/            # 结界突破相关
│
└── scrpits/
    └── build-autox.mjs           # 需修改：复制 assets 到 dist-autox
```

### 类型共享策略

**问题：** Web 和 AutoX 都是 TypeScript，如何共享类型定义？

**方案：** 使用 TypeScript `paths` 映射

```json
// tsconfig.app.json (Web)
{
  "compilerOptions": {
    "paths": {
      "@shared/*": ["./src-autox/types/*"]
    }
  }
}

// tsconfig.autox.json (AutoX)
{
  "compilerOptions": {
    "paths": {
      "@shared/*": ["./src-autox/types/*"]
    }
  }
}
```

**共享类型示例：**
```typescript
// src-autox/types/config.ts
export enum TaskType {
  SOUL = 'SOUL',
  EXPLORE = 'EXPLORE',
  BREAK = 'BREAK'
}

export interface GlobalConfig {
  clickDelayRandomness: [number, number];
  confidenceThreshold: number;
  maxRetries: number;
}

// Web 和 AutoX 都可以导入
import { TaskType, GlobalConfig } from '@shared/config';
```

---

## 关键模块详细设计

### 1. JS Bridge 消息协议

#### 消息类型定义
```typescript
// src-autox/types/bridge.ts
export interface BridgeMessageBase<T = any> {
  id?: string;
  type: string;
  payload?: T;
  kind: 'request' | 'response' | 'event';
}

export type BridgeRequest<T = any> = BridgeMessageBase<T> & {
  kind: 'request';
  id: string;
};

export type BridgeResponse<T = any> = BridgeMessageBase<T> & {
  kind: 'response';
  id: string;
  ok: boolean;
  error?: string;
};

export type BridgeEvent<T = any> = BridgeMessageBase<T> & {
  kind: 'event';
};
```

#### 标准消息流程

**Web → AutoX（请求）：**
```json
{
  "kind": "request",
  "id": "web-1234567890-0",
  "type": "task.start",
  "payload": {
    "type": "SOUL",
    "targetCount": 50,
    "floor": 10
  }
}
```

**AutoX → Web（响应）：**
```json
{
  "kind": "response",
  "id": "web-1234567890-0",
  "type": "task.start",
  "ok": true,
  "payload": {
    "taskId": "task-001",
    "status": "running"
  }
}
```

**AutoX → Web（事件）：**
```json
{
  "kind": "event",
  "type": "task.progress",
  "payload": {
    "round": 12,
    "total": 50,
    "drops": 3
  }
}
```

### 2. 配置数据结构

```typescript
// src-autox/types/config.ts
export interface AppConfig {
  global: GlobalConfig;
  activeTask: TaskConfig;
}

export interface GlobalConfig {
  clickDelayRandomness: [number, number];    // [100, 300]
  confidenceThreshold: number;               // 0.8
  maxRetries: number;                        // 3
  enableNotification: boolean;               // false
}

export enum TaskType {
  SOUL = 'SOUL',
  EXPLORATION = 'EXPLORE',
  BREAK = 'BREAK',
  ACTIVITY = 'ACTIVITY'
}

export interface SoulConfig {
  type: TaskType.SOUL;
  floor: 10 | 11;
  teamLocked: boolean;
  targetCount: number;
  enableBuff: boolean;
}

export interface BreakConfig {
  type: TaskType.BREAK;
  priority: 'STAR' | 'MEDAL';
  refreshOnFail: boolean;
}

export type TaskConfig = SoulConfig | BreakConfig /* | ... */;
```

### 3. 图片资源管理

#### 资源组织
```
src-autox/assets/templates/
├── common/
│   ├── confirm.png           # 确认按钮
│   ├── cancel.png            # 取消按钮
│   └── back.png              # 返回按钮
├── yuhun/
│   ├── challenge.png         # 挑战按钮
│   ├── victory.png           # 胜利标志
│   └── auto_on.png           # 自动战斗已开启
└── coords.ts                 # 坐标定义（锚点位置）
```

#### 坐标缩放方案
```typescript
// src-autox/core/CoordMapper.ts
const DESIGN_WIDTH = 720;
const DESIGN_HEIGHT = 1280;

export class CoordMapper {
  private scaleX: number;
  private scaleY: number;

  constructor() {
    this.scaleX = device.width / DESIGN_WIDTH;
    this.scaleY = device.height / DESIGN_HEIGHT;
  }

  map(x: number, y: number): [number, number] {
    return [
      Math.round(x * this.scaleX),
      Math.round(y * this.scaleY)
    ];
  }

  mapRect(rect: Rect): Rect {
    return {
      x: Math.round(rect.x * this.scaleX),
      y: Math.round(rect.y * this.scaleY),
      w: Math.round(rect.w * this.scaleX),
      h: Math.round(rect.h * this.scaleY)
    };
  }
}
```

#### 模板加载
```typescript
// src-autox/core/TemplateLoader.ts
export class TemplateLoader {
  private basePath: string;
  private cache = new Map<string, Image>();

  constructor() {
    this.basePath = files.join(files.cwd(), 'assets/templates');
  }

  load(name: string): Image | null {
    if (this.cache.has(name)) {
      return this.cache.get(name)!;
    }
    const path = files.join(this.basePath, name);
    const img = images.read(path);
    if (img) this.cache.set(name, img);
    return img;
  }

  dispose(): void {
    for (const img of this.cache.values()) {
      img.recycle();
    }
    this.cache.clear();
  }
}
```

---

## 风险和挑战

### 技术风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **截图权限被拒** | 脚本无法运行 | Phase 2 提前验证，提供详细引导 |
| **分辨率适配失败** | 点击偏移，识别失败 | 坐标缩放 + 多设备测试 |
| **游戏更新UI变化** | 模板失效 | 版本兼容性检测 + 快速更新机制 |
| **内存泄漏** | 长时间运行崩溃 | 严格执行 `img.recycle()`，定期GC |
| **被游戏检测封号** | 用户账号风险 | 随机化 + 保守策略 + 免责声明 |

### 开发风险

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **素材收集耗时** | 延期 | 优先核心流程，边开发边补充 |
| **状态识别不稳定** | 脚本卡死 | 多轮测试 + 兜底恢复逻辑 |
| **Bridge 通信异常** | UI 与脚本失联 | 超时重试 + 心跳检测 |

---

## 开发约定

### 代码规范

1. **TypeScript 严格模式**
   - 所有函数必须声明返回类型
   - 禁止使用 `any`（除非有充分理由）
   - 优先使用接口而非类型别名

2. **命名约定**
   - 类名：PascalCase（`ScreenService`）
   - 函数/变量：camelCase（`captureScreen`）
   - 常量：UPPER_SNAKE_CASE（`MAX_RETRIES`）
   - 私有成员：下划线前缀（`_cache`）

3. **错误处理**
   - 底层服务抛出具体错误类型
   - 上层捕获并转换为用户友好消息
   - 关键操作必须 try-finally（如 Image.recycle）

4. **日志规范**
   ```typescript
   logger.debug('详细调试信息');      // 仅开发时
   logger.info('任务进度更新');       // 常规流程
   logger.warn('找图失败，重试中');   // 可恢复异常
   logger.error('致命错误，停止任务'); // 不可恢复
   ```

### 测试策略

**Phase 2-3（基础阶段）：**
- 在真实设备上手动测试每个服务类
- 用 Hello World 验证核心功能

**Phase 4-5（任务开发）：**
- 每个任务至少完整运行 10 轮
- 测试异常场景（网络中断、弹窗等）

**Phase 6（优化阶段）：**
- 连续运行 2 小时以上
- 多设备（不同分辨率）验证

### Git 工作流

```
main (稳定分支)
  ├── feature/phase1-bridge      # Phase 1 开发
  ├── feature/phase2-mvp          # Phase 2 开发
  ├── feature/phase3-core         # Phase 3 开发
  └── feature/task-yuhun          # 御魂任务开发
```

**提交规范：**
```
feat: 添加 ScreenService 核心功能
fix: 修复图片资源未释放导致的内存泄漏
refactor: 重构 GameRecognizer 状态检测逻辑
docs: 更新 Plan.md 任务进度
```

---

## 下一步行动

**立即开始 Phase 1：**

1. ✅ 完善 `src/core/WebBridge.ts`
2. ✅ 实现 `src-autox/core/AutoxBridge.ts`
3. ✅ 创建 `src/components/Dashboard.vue`
4. ✅ 创建 `src/components/LogTerminal.vue`
5. ✅ 打通 Web → AutoX → Web 消息循环

**预期产出：**
- 能在 Web UI 点击按钮，AutoX 接收到消息并打印日志
- 日志实时显示在 Web UI 的终端窗口

**开发优先级：** Phase 1 → Phase 2（关键验证）→ Phase 3 → Phase 4 → Phase 5 → Phase 6

**时间估算：** 总计 6-8 周（兼职开发）

---

*本计划基于 Gemini (UI/UX 专家) 和 Codex (AutoX 技术专家) 的联合建议制定。*
