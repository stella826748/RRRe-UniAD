# RRRe-UniAD
基于 UniAD 2.0 的预训练端到端自动驾驶框架
你的情况我会很明确地建议：**现在不要租多卡去完整复现 UniAD**。你是以学习和小论文探索为目标，最划算的路线是：**本地做理解和代码改动，租单卡做验证和小实验，只有当你要对标官方 full nuScenes 训练指标时才租多卡**。

**先给结论**

| 目标 | 是否需要租 GPU | 推荐 GPU |
|---|---:|---|
| 读论文、读代码、改配置、理解 pipeline | 不需要 | 本地 RTX 5060 8GB 足够 |
| 跑 mini nuScenes dataloader / 小规模 debug | 可本地尝试，但大概率吃紧 | 本地或单卡 16-24GB |
| 用官方 checkpoint 做 full nuScenes eval | 单卡可以，但慢；多卡更快 | 3090/4090/A5000/A6000/A100 |
| 微调 Stage2 的部分模块，做小论文 ablation | 单卡优先 | 24GB 起步，48GB 更舒服 |
| 训练 UniAD Stage2 full setting | 理论单卡可做但很慢，推荐多卡 | 4-8 卡 24GB+ / A100 |
| 训练 Stage1 track+map full setting | 基本需要高显存多卡 | 8×A100 80GB 最接近官方 |
| 完整复现官方训练结果 | 需要多卡 | 8×A100 级别 |

官方资源需求大概是：Stage1 约 **50GB/GPU**，8×A100 训练约 2 天；把 `queue_length=5` 降到 3 后约 **30GB/GPU**；Stage2 约 **17GB/GPU**，8×A100 训练约 4 天。来源是官方 UniAD 训练说明和仓库 README：[OpenDriveLab/UniAD](https://github.com/OpenDriveLab/UniAD)。

所以你的 RTX 5060 8GB 本地主要适合做这些事：

1. 读代码和论文。
2. 准备数据目录、检查 pkl、看 config。
3. 改模型模块。
4. 写小规模脚本。
5. 用极小 batch / 极少帧做 smoke test，确认代码没语法错误。

但别指望它认真跑 UniAD。8GB 对 BEVFormer/UniAD 这种多相机、多帧 BEV 模型太小了。

**什么时候租单卡就够**

如果你的目标是“学习 + 写小论文初稿”，单卡已经够了，尤其是你不从零训练 UniAD，而是站在官方 checkpoint 上做改动。

比较合理的单卡目标：

1. **跑通官方 checkpoint 评估**
   这一步证明你的环境、数据、代码链路是通的。可以租 3090/4090 24GB 或 A5000/A6000/A100。单卡会慢，但能接受。

2. **用 mini nuScenes 或 full nuScenes 子集做 debug**
   这一步是为了快速试错，不追求官方指标。可以验证 loss 是否下降、输出是否合理、可视化是否正常。

3. **冻结大部分 backbone，只改 planning / motion / occupancy head**
   这是我最推荐你的小论文路线。UniAD 的重资产部分是 BEV encoder 和 tracking/map 表征，你不一定要重训它。你可以把官方 Stage1 或 Stage2 checkpoint 当作基础模型，然后研究：
   - planning head 改进；
   - motion prediction 和 planning 的交互；
   - occupancy 对 planning 的影响；
   - query 交互机制；
   - cost function / collision loss / trajectory loss 改进；
   - 轻量化 planning module。

4. **做小规模 ablation**
   单卡足够你说明“这个想法在受控设置下有效”。小论文不一定必须完整复现官方 8 卡训练，只要实验设计诚实、对照清楚、结论不过度夸大。

推荐单卡优先级：

```text
A100 80GB > A100 40GB / L40S 48GB / A6000 48GB > 4090 24GB / 3090 24GB > 16GB 卡
```

如果预算有限，**4090 24GB 是性价比选择**；如果你想少折腾，**A6000 48GB / L40S 48GB / A100 40GB 更稳**。

**什么时候才需要租多卡**

只有下面这些目标，才值得租多卡：

1. **你要完整复现官方 UniAD 训练结果**
   也就是从 BEVFormer / Stage1 / Stage2 按官方流程训练，并希望指标接近官方表格。这是多卡任务，不适合现在作为第一目标。

2. **你的论文需要 full nuScenes 上严肃对标**
   如果你要投稿，且 claim 是“我比 UniAD/VAD/FusionAD 在 nuScenes planning 指标上更好”，那后期需要多卡做完整实验。

3. **你要改动 Stage1 感知部分**
   比如 tracking、map、BEV encoder、temporal BEV aggregation。这类改动会牵动 Stage1，显存和训练时间都上来了。

4. **你要做多组大规模 ablation**
   比如 5 个模块 × 3 个 setting × full trainval，这种单卡会拖到怀疑人生。

换句话说：**多卡是论文后期“补强证据”的工具，不是前期“开始学习”的门票。**

**我建议你的实际路线**

第一阶段，不租卡：

读 UniAD 论文和代码，重点看这几块：

```text
uniad_e2e.py              # 总 pipeline
track_head                # 目标跟踪
map_head                  # 在线地图
motion_head               # 轨迹预测
occ_head                  # occupancy
planning_head             # 规划输出
stage1_track_map config
stage2_e2e config
```

第二阶段，租单卡 24GB 或 48GB：

目标不是训练，而是：

```text
官方 checkpoint eval
mini/full subset 可视化
确认 planning 输出
跑一个你自己的小改动
```

第三阶段，继续单卡：

选一个小论文切口。我的建议优先级是：

1. **planning head 改进**
   最贴近端到端自动驾驶，实验成本相对低。

2. **motion-planning interaction**
   比较有研究味道，也容易写成故事。

3. **occupancy-aware planning**
   直观、可视化好看，适合小论文展示。

4. **轻量化 / 蒸馏 UniAD planning**
   适合你本地显存小的现实条件，也有应用价值。

暂时不建议你一上来改 BEV encoder、tracking 或 map。那些模块太重，实验成本高，而且容易陷入“环境和训练技巧大于研究问题”的泥潭。

第四阶段，如果前面结果有苗头，再租多卡：

这时候才考虑：

```text
4×4090 / 4×A100：中等规模补实验
8×A100：官方级 full reproduction
```

**对你的小论文最现实的定位**

你现在可以把目标定成：

> 基于 UniAD 2.0 的预训练端到端自动驾驶框架，研究 planning 模块中某个交互机制 / 损失函数 / occupancy 约束 / motion 信息利用方式，并在 nuScenes mini、full validation subset、以及必要的 full val evaluation 上验证。

这个定位非常适合研一探索期。它既能让你真正理解 UniAD，又不会被完整复现的算力成本吞掉。

我的建议很直接：**先租单卡，不租多卡；先做 Stage2 planning 相关小改动，不碰 Stage1 全量训练。**等你有一个初步有效的 idea，再考虑多卡把实验做漂亮。
