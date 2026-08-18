# 单个 SEED 生成整个场景

> OPERATION BLACKOUT 程序化内容管线总览  
> 本文汇总：从一枚整数种子出发，如何确定性生成关卡几何、材质、天气、布料风动与合并绘制；以及配套的分步演示页与 Build→Critic→Fix 质量闭环。  
> 范围覆盖本会话讨论的全部技术与美术要点，以及仓库内相关 demo 网页说明。

---

## 1. 核心命题

**同一枚 `seed`（整数）→ 同一关卡、同一时刻、同一构图下，每一次启动都得到同一帧。**

这意味着：

- 场景不是“随机好看一点”，而是**可复现的程序化作者态**。
- 视觉验收（Critic / A/B）只反映**代码改动**，不反映“今天抽到另一组随机数”。
- 关卡、道具、贴图、噪声、部分天气日程，都从这枚种子（及其派生盐）长出。

项目约束强化了这一选择：零外部资源、`file://` 可运行、禁止 `Math.random()`。一切内容必须在 boot 时由代码生成；种子是整棵生成树的根。

---

## 2. 种子如何进入系统

### 2.1 入口

捕获 / 运行可通过 URL 参数传入种子，例如 `seed=12345`。游戏上下文向各系统提供：

- `ctx.rng` — `GAME.RNG`（mulberry32），确定性顺序流  
- 各模块再 `fork(盐)` 或 `new GAME.RNG(seed ^ 盐)` 得到**独立子系统流**

### 2.2 `GAME.RNG` 与 `fork`

- `next()` / `range` / `int` / `gaussian` … 消费游标，推进状态。  
- **`fork(salt)`**：用当前状态与盐异或派生**新 RNG**，**不推进父游标**。  
- 因此：子系统可以任意多抽，只要它只碰自己的 fork，父流与其它子系统保持不动。

### 2.3 `GAME.Noise` 与空间哈希

贴图与大量几何扰动不走“队列里下一个随机数”，而走：

- **噪声场**：`fbm` / Perlin / Worley 等，由种子初始化置换表；采样点是**空间坐标**。  
- **格子哈希** `hash2i(ix, iz, salt)`：决策只依赖格子索引与盐，**与调用顺序无关**。

同一位置永远得到同一噪声值；中间多加一个无关系统，不会把“下一棵树”的骰子整体平移——前提是该决策没有误用共享顺序流。

### 2.4 隔离层级（由粗到细）

| 层级 | 做法 | 作用 |
|------|------|------|
| 全项目 | 禁止 `Math.random()` | 捕获可复现 |
| 系统 | Weather / Weapons / Level 各自 `fork` 大盐 | 改枪不影响雨 |
| 阶段 | 贴图 atlas、闪电、雪花各一条子流 | 改 splash 不影响闪电方位 |
| 物体 | `rng.fork(0x7241 + i*37)`（如地铁车厢） | 改第 0 节不影响第 1 节 |
| 空间 | `hash2i` / `Noise.fbm2(x,z)` | 与插入顺序无关 |
| 作者参数 | 计划阶段写入 `T.seed`，建造时用噪声扭曲 | 形状绑物体，不绑全局游标 |

---

## 3. 从种子到整关：生成栈

```text
seed
  ├─ Level.rng / Noise          地形、建筑、树计划、锚点
  ├─ Props.rng / Noise          散落道具、布料、绳索
  ├─ Materials / Textures       库材质配方（高度场→PBR）
  ├─ Lighting / Sky / PostFX    环境档、雾、色级（关卡声明）
  ├─ Weather（自有 fork）       雨/雪/闪电日程与网格
  ├─ Weapons / AI / Audio       各自盐，不污染场景流
  └─ Builder 材质桶 → mergeAll / InstancedMesh / mergeFlex
        └─ 少数静态 draw + 风动物体 + 实例植被
```

美术上，每一关还有一份**艺术方向契约**（roster / art direction）：同一套生成器，不同的时间、天气、主光与调色，形成十关各自的“一张照片”目标，而不是十个互不相关的随机世界。

---

## 4. 贴图管线（Height-first）

### 4.1 原则

**先写一张高度/结构场，再派生 albedo、normal、roughness、AO、ORM。**  
同源高度保证磨损、裂缝、接缝在各通道上物理相关，而不是“各画一张图碰巧叠在一起”。

### 4.2 典型步骤

1. 种子 → 噪声层（macro / med / fine、Worley、ridged、drip、shear…）  
2. 配方按材质家族叠层（混凝土骨料、砖缝、金仓瓦楞、釉面砖…）  
3. 可选去色阶（deband）  
4. `_emit`：由高度估 AO、Sobel 法线、打包 ORM  
5. 上传为 `DataTexture`；库内按名缓存

### 4.3 算法家族（概念分类）

- **FBM + Worley**：浇筑混凝土、一般矿物  
- **格子 / 模板**：砖、瓷砖、板缝  
- **板缝裂纹 / 流挂**：墙面、污渍  
- **编织 / 帆布**：布、雨棚  
- **切割 alpha**：叶片、网  

关卡通过 `SURF` / 材质桶名把几何指到库配方；可用 `base` 重定向（例如湿地面桶指向 `concrete` 图，再叠镜面）。

### 4.4 顶点磨损契约（几何侧）

白 = 完好。顶点色通道约定：

- **R** 污垢（grime）  
- **G** 潮湿（wet）  
- **B** 边棱磨损（edge wear）  

着色器在 `materials.js` 中解读。磨损分辨率 ≈ 网格细分密度。

---

## 5. 网格管线（原子 → 桶 → 合并）

### 5.1 几何原子

常见积木：`bevelBox`、圆柱、`tube` / `limb`、高度场、浮雕盒、扫掠、车削、实例化卡片等。

### 5.2 Builder 与合并

- 关卡建造时 `B.add('bark'|… , geo, matrix)` 按**材质桶**归类。  
- Finalize 时 `Geo.mergeAll`：同桶合成一张大网格 → **一次 draw**。  
- **合并主轴是材质/表面桶**，不是“会不会动”。  
- 需要逐实例变换/着色的（树冠、草）走 `InstancedMesh`。  
- 带 `aFlex` 的布料不能走普通 `mergeAll`（会丢属性），走 **`mergeFlex`**。

### 5.3 为何能压到约十几～二十次静态 draw

市场关等：十余个静态材质桶合并后，静态场景约 **18–21 draws**（另加武器、特效、后处理、实例植被等，整帧远高于此）。  
透明、自发光、贴花等会额外拆批。

### 5.4 树干（雨林）——从种子到一棵树

计划阶段为每棵树写入位置、半径、高度、倾斜、`seed` 等。建造时：

1. 沿高度采样 **stations** `[x,y,z,r]`（尖削 + 倾斜 + `fbm` 扭摆）  
2. 每站局部标架 **N / B / T**  
3. 圆周采样成环 → **`tube()`** 扫掠成管  
4. **`barkTube`**：角向 flute 调制半径 + 度量 UV + 光滑法线  
5. **`buttressFin`** 绕根旋转多片板根  
6. 全部 `B.add('bark', …)`，与其它树干合并进少数 bark draw  
7. 叶冠另路实例，不进树干网格  

形状扰动优先绑 **`T.seed` + Noise**；板根角等若误用共享顺序 `rng` 多抽，会波及后续树——正确改法见第 8 节。

### 5.5 布料与风动（雪原破布）

静态合并网格**不会**自己摆动。布料路径：

1. **`K.sheet(w,d,sag,nu,nv,pin,noise)`**  
   - UV 网格 → `pin(u,v)` 钉住权重 → 静态下垂 → **`aFlex = slack²`** → 三角化  
2. **`_clothPart`** 登记进 `clothParts`（带世界矩阵），不进普通静态桶  
3. **`mergeFlex`** 合并并保留 `aFlex`  
4. **`applyWind`**：在材质 `onBeforeCompile` 链上注入顶点位移（顺风倾倒 + 多层正弦），读 `aFlex`、风速、风向  
5. 每帧只更新 uniform（时间 / 风），**不重建几何**

钉住边几乎不动，自由角满幅扬起。洗晒硬衣是另一套 `frozenGarment`（偏板状），动画幅度更小。

市场雨棚等多为**作者态下垂**，不一定走运行时风。

---

## 6. 改中间某块 Mesh，如何不拖垮后面

**不是**“只能改生成顺序最后一项”，而是：

> **几何可以改在该物体的建造函数里；随机只许消耗该物体的私有 fork / 已有 seed；父流抽数次数与顺序保持不变。**

### 6.1 推荐策略（由稳到险）

1. **零随机修改**：改剖面、flute、板根曲线、常量、已有 `T.seed` 驱动的噪声。  
2. **物体私有流**：如地铁  
   `buildCar(..., rng.fork(0x7241 + i*37), car)`  
   在第 0 节车内多抽任意次，只影响第 0 节；父游标不动。  
3. **本地种子**：`new GAME.RNG(T.seed ^ 算法盐)` 或 `hash2i`。  
4. **禁止**：在共享顺序 `rng` 上为中间物体临时多插若干 `next()`——后续所有 mesh 错位。

### 6.2 新系统 vs 改旧物体

| 场景 | 做法 |
|------|------|
| 新增天气阶段（如雪） | 新 `fork` + 构建顺序**追加在雨块之后**，不重排旧调用 |
| 改已有雨滴/水花 | 保序；闪电/雪必须已在独立流上 |
| 改第 k 棵树 / 第 i 节车厢 | 改该建造函数 + 私有随机；勿碰父流 |

经典反例（已写入天气模块注释）：splash 与闪电曾共用一条顺序流 → 只调 splash 速率导致闪电方位改变 → Critic A/B 完全失效。修复：`lrng` / `srng` 提前 fork。

---

## 7. Build → Critic → Fix（与种子的关系）

种子可复现，是对抗式审查能成立的前提。

### 7.1 角色

| 角色 | 职责 |
|------|------|
| **Build / Fix Agent** | 一人一文件；按契约写生成器与关卡内容 |
| **Critic Agent** | 独立相位；不看 Fixer 的截图；自拍、读图、读源码、打分 |
| **Verify** | 全库健康、回归、诚实汇报 |

### 7.2 Critic 行为要点

- 身份：极严苛的 AAA 美术总监；对标**已发售作品**，不对标“比上轮好一点”。  
- 读全尺寸图；拟批评处再看 2×；客观指标作辅证。  
- 批评前读 owning 源码；最高价值发现常在图外（无真灯、roughness 被钳死、错误关卡里的 emitter…）。  
- 输出 schema：`score`、`verdict`、`presentable`、`findings[]`（每条一个 owning file + 可执行 fix）。

### 7.3 Fix 如何接到下一轮

下一轮 task **嵌入 Critic 原文 verdict**，并附测量过的诊断与改法（灯 Rig 数字、缺身份特征、共享系统是否仍卡住等）。  
偏好**结构修复**，禁止无意义过冲；验证必须看图，不能只看指标。

### 7.4 闭环能做什么 / 不能做什么

- 擅长找**缺陷与缺席**（例如名单要求 Bayon 脸，帧里一张都没有）。  
- 不擅长凭空产出“美感瞬间”；去掉缺陷 ≠ 自动到达卓越。  
- 分数对标真 AAA 时，多轮后常见平台约五十上下——诚实上限来自程序化与镜头预算，不是调参能抹平。

---

## 8. 美术信息摘要

- **零资产**：无贴图/模型/音效文件；程序化即美术生产格式。  
- **关卡身份**：roster 规定主光、天气、调色与标志物（火与冷泛光、Bayon 脸、暴风雪洗衣绳等）；生成器必须交付身份，不能只交付“有几何”。  
- **光照**：室外依赖太阳/天空/practical；室内忌无衰减 fill 淹没一切——“有灯具设计”优于“全局提亮”。  
- **材质**：height-first；磨损通道语义统一；远处法线/颗粒需有 LOD 意识。  
- **植被 / 布料**：实例与风动分路；布料靠 pin + aFlex，不是整网同相位抖动。  
- **合并美学**：同表面合并换 draw；不同运动语义（静 / 风 / 实例）分路，避免“为了省 draw 杀死动画”。

---

## 9. Demo 网页导览

下列页面均在仓库 `docs/` 下，用本地 HTTP 打开（需能加载 `vendor/three.global.js` 的页面已注明）。它们是本文技术点的**交互说明书**，不是运行时游戏本身。

### 9.1 网格生成 · `survey/mesh_pipeline/`

| 路径 | 内容 |
|------|------|
| [`tree_trunk/`](mesh_pipeline/tree_trunk/) | **雨林树干**九步 wireframe：轴线 → stations → 圆盘 → N/B/T → 环点 → `tube` → flute → 板根 → 装配进 bark 桶。拖拽旋转、逐步/自动播放。 |
| [`cloth/`](mesh_pipeline/cloth/) | **雪原破布**九步：UV 网格 → pin → sag → aFlex → 三角化 → clothParts → mergeFlex → 风着色器拆解 → **实时风动**（风速滑条）。复现 `K.sheet` + `WIND_BODY`。 |
| [`oil_tank/`](mesh_pipeline/oil_tank/) | **储油罐**分步 wireframe：壳体、焊缝、开孔、梁柱、顶盖、阶梯、围堰等工业件如何由原子拼出。 |
| [`bayon_face/`](mesh_pipeline/bayon_face/) | **Bayon 人脸高度场**分步：体块 → 五官调制 → 空腔/阴影可读性 → 与“安详浅浮雕”美术约束。 |
| [`batches/`](mesh_pipeline/batches/) | **Al-Bakr 市场静态合并批次目录**：约 18–21 个材质桶各自对应库配方、UV、投射/接收阴影与几何用途——回答“为何这么少 draw”。 |

### 9.2 贴图生成 · `survey/texture_pipeline/`

| 路径 | 内容 |
|------|------|
| [`concrete/`](texture_pipeline/concrete/) | 混凝土配方中间帧：噪声层、高度项、albedo/粗糙度步骤与最终图，对照 height-first。 |
| [`tile/`](texture_pipeline/tile/) | 釉面砖：缝、釉、磨损、破损掩码等分层可视化。 |
| [`reefer_panel/`](texture_pipeline/reefer_panel/) | 冷藏柜/金仓类面板：瓦楞、接缝、霜污、螺栓等结构层到最终 albedo/roughness。 |

### 9.3 Critic 案例 · `survey/critic/`

| 路径 | 内容 |
|------|------|
| [`harbor_splash_lightning_fork/`](critic/harbor_splash_lightning_fork/) | **RNG 串流污染**完整故事：Build 共用顺序流 → 调 splash 使闪电捕获漂移 → Critic A/B 失效 → `fork` 隔离闪电/雪；雨块**保序**、雪**末尾追加**；含交互示意“多抽几次如何错位”。 |
| [`ruins_bayon_faces/`](critic/ruins_bayon_faces/) | **身份缺失**案例：塔在、脸不在 → Critic 判 identity failure → Fix 上浮雕网格 + 凹处 wear；对比“只改指标”与“交付名单承诺的标志物”；链到 mesh 演示页。 |

---

## 10. 一张总图：SEED 如何“长”成可玩场景

```text
                    ┌─────────────────────────────────────┐
   seed (int)  ───► │  RNG / Noise / 各系统 fork(盐)      │
                    └─────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
   贴图库配方                    关卡计划 + 建造               天气 / 武器 / AI
   height → PBR                 stations / sheet / 箱体         独立子流
          │                           │
          └────────────► SURF / 材质桶 ◄────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
              mergeAll 静态批                     Instanced / mergeFlex
              （混凝土、树皮…）                    （叶、草、布料风动）
                    │                                   │
                    └────────────► 一帧画面 ◄───────────┘
                         ▲
                         │ 同一 seed ⇒ 同一像素（确定性模拟）
                         │ Critic 只评价代码差分
```

---

## 11. 实践信条（给生成与修复）

1. **种子是契约，不是灵感旋钮**——好看靠作者参数与结构，不靠重抽。  
2. **新随机先问：挂在哪条流？** 默认私有 fork 或空间哈希。  
3. **改中间物体：改函数，不改父游标。**  
4. **新阶段：追加 + 新流；旧阶段：保序。**  
5. **合并按表面；运动按语义分路。**  
6. **Critic 要可执行 finding；Fix 要可测量验收。**  
7. **身份特征缺席是最高严重度缺陷之一**（有名字的东西必须在签名镜头里可读）。

---

## 12. 相关源码索引（阅读用）

| 主题 | 主要位置 |
|------|----------|
| RNG / Noise / fork | `src/core/util.js` |
| 贴图配方 | `src/render/textures.js`、`materials.js` |
| 建造与合并 | `src/world/level.js`（Builder）、各 `level_*.js` / `props_*.js` |
| 树干 | `src/world/level_jungle.js`（`tube` / `barkTube` / `buttressFin` / `buildTree`） |
| 布料与风 | `src/world/props_snowbound.js`（`K.sheet`、`mergeFlex`、`WIND_BODY`） |
| 天气流隔离 | `src/fx/weather.js`（`lrng` / `srng`、雨块保序） |
| 车厢级 fork | `src/world/level_metro.js`（`buildTrain` / `buildCar`） |
| Critic 工作流模板 | `docs/round4-workflow.js` |
| 方法论文 | `docs/technical-writeup.md` §5 |
| 架构契约 | `ARCHITECTURE.md`、`DEVELOPMENT.md`、`LEVELS_ROSTER.md` |

---

*文档性质：设计与教学总览。交互细节以各 demo 页为准；运行时行为以源码为准。*
