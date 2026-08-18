# 算法即场景：学习 Claude-of-Duty

## 在线阅读：[网页版讲义](https://kilia.github.io/learn-claude-of-duty/)（含演讲实录与交互演示）

![算法即场景 · 从零资产到可玩的世界](images/slide01-1.webp)

本仓库是引擎 PCG 团队学习 [Claude-of-Duty](https://github.com/mshumer/Claude-of-Duty) 的技术分享。整个分享基于 TA 同学 kelvin 对 Claude-of-Duty 的完整复刻经验（[operation-blackout](https://github.com/kelvincai522/operation-blackout)），以及对[生成过程](docs/OPERATION%20BLACKOUT%20项目解析.md)、生成算法（[几何](docs/几何生成算法.md)、[贴图](docs/贴图生成算法.md)）、[Build-Critic-Fix 审查流程](docs/创造-审查-修复循环.md)、[单一随机数种子](docs/单个SEED生成整个场景.md)等各项内容的调研和思考。另有与《无人深空》的 [PCG 对比分析](docs/OPUS%205%20PCG与《无人深空》PCG详细对比分析报告.md)。

## 交互演示（GitHub Pages）

索引页：[https://kilia.github.io/learn-claude-of-duty/interactive-demo/](https://kilia.github.io/learn-claude-of-duty/interactive-demo/)

各 Demo 也可直接打开：

**几何（MESH）**
- [Bayon 人脸 · 高度场](https://kilia.github.io/learn-claude-of-duty/interactive-demo/mesh_pipeline/bayon_face/)
- [布料几何 · Wireframe](https://kilia.github.io/learn-claude-of-duty/interactive-demo/mesh_pipeline/cloth/)
- [储油罐 · 分步 Wireframe](https://kilia.github.io/learn-claude-of-duty/interactive-demo/mesh_pipeline/oil_tank/)
- [树干几何 · 分步生成](https://kilia.github.io/learn-claude-of-duty/interactive-demo/mesh_pipeline/tree_trunk/)
- [市场关卡 · 合并批次](https://kilia.github.io/learn-claude-of-duty/interactive-demo/mesh_pipeline/batches/)

**贴图（TEXTURE）**
- [Tile · 生成流程图](https://kilia.github.io/learn-claude-of-duty/interactive-demo/texture_pipeline/tile/graph.html)
- [Tile · 釉砖格与釉变](https://kilia.github.io/learn-claude-of-duty/interactive-demo/texture_pipeline/tile/)
- [Concrete · 全中间态](https://kilia.github.io/learn-claude-of-duty/interactive-demo/texture_pipeline/concrete/)
- [Reefer Panel · 百叶几何](https://kilia.github.io/learn-claude-of-duty/interactive-demo/texture_pipeline/reefer_panel/)

**审查循环（CRITIC）**
- [Harbor · RNG Fork](https://kilia.github.io/learn-claude-of-duty/interactive-demo/critic/harbor_splash_lightning_fork/)
- [Bayon Ruins · Faces](https://kilia.github.io/learn-claude-of-duty/interactive-demo/critic/ruins_bayon_faces/)

## 调研文档

- [OPERATION BLACKOUT 项目解析.md](docs/OPERATION%20BLACKOUT%20项目解析.md)：解析 Operation Blackout 如何用多轮 Agent 从零生成整款零外部资源的 FPS，以及各阶段的实测成本。
- [创造-审查-修复循环.md](docs/创造-审查-修复循环.md)：说明 Build → Critic → Fix → Verify 如何并行创造、对抗式审查并定点修复程序化内容。
- [单个SEED生成整个场景.md](docs/单个SEED生成整个场景.md)：说明同一枚整数种子如何确定性生成关卡几何、材质、天气与运行时效果。
- [几何生成算法.md](docs/几何生成算法.md)：汇总树干、布料、储油罐、Bayon 人脸等代表性几何管线与合并批次。
- [贴图生成算法.md](docs/贴图生成算法.md)：整理过程式 PBR 贴图的高度优先流水线，以及 concrete / reefer_panel / tile 三种材质。
- [OPUS 5 PCG与《无人深空》PCG详细对比分析报告.md](docs/OPUS%205%20PCG与《无人深空》PCG详细对比分析报告.md)：对比 OPUS 5 与《无人深空》在 Seed、资产生产、质量闭环等维度上的异同。

