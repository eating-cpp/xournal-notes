# my own paper

## 参考论文的梳理
### 已经参考过的论文
1. RAFT:  [RAFT论文解读](../RAFT/RAFT论文解读.md)
2. FOD扩散模型加持下的超分辨率 [FOD论文解读](..\FOD扩散模型超分辨\FOD论文比.md)
3. DvD [DvD](../DVD\DvD论文理解.md)
4. DocScanner [DocScanner](../DocScanner\DocScanner论文理解.md)
5. DewarpNet [DewarpNet](../DewarpNet\DewarpNet论文理解.md)

## 初步的ideas
<!-- 1. 前景掩码估计外加一个深度估计，测算这张纸相对于相机各个地方的距离。根据这种深度信息，学习一种矫正场作为先验，和扩散模型相结合，来恢复出展平的文档图像 -->

1. **flowmatching能不能做这个？** → 结论：能，见下文「flow matching应用方向」

2. **通过某种手段获取一种先验知识**

2. **采用FOD那种类似于双memory的机制，保存这种先验**  

3. **计划使用 DvD 作为baseline**

4. FOD中的uncertainty loss aware也许也可以直接应用到DvD中【后续】

5. MFE改成 dinoV3samll 版本的预训练模型（下采样 projector投影成多尺度）



## 双memory机制

### HQ Memory —— 存「平面矫正文档的结构先验」，而不是纹理。

文档去畸变最核心的先验其实是一个几何断言：矫正后文本行是水平的、布局是规则的、页面边界是矩形。现在 DvD 的 line_msk（文本行检测）只告诉你「哪里有文本行」，不告诉你「文本行应该水平」。HQ memory 正好可以补这块——存「规则文档布局」的全局结构先验，用 cross-attention 检索出来，告诉 DiT「矫正后的目标形态长这样」。

这其实和 FOD 的动机同构：FOD 是「给生成过程一个更可靠的目标参照」，DvD 是「给坐标场回归一个更可靠的目标形态参照」。

### Degradation Memory —— 语义要反过来。

这是最有意思的一点。FOD 的退化记忆是「减掉退化成分」（残差连接）。但 DvD 的坐标场本身就是畸变的逆映射——模型要预测的恰恰是「退化」这件事。所以 DvD 的「退化记忆」不该用来减，而该用来建模畸变模式的先验：圆柱弯曲 / 褶皱 / 透视 / 折痕，各是一类可检索的形变模式。让 DiT 更快判断「当前这页是哪种弯」，从而给出更准的坐标场（或更好的 init_flow 初始化）。

一句话：FOD 的退化记忆是「把坏东西赶走」，DvD 的应该是「把坏东西认出来」。

## flow matching应用方向

> 完整精读与论证见 [DvD-FM改造方向](../DvD-FM改造方向.md)，这里记结论和与本项目 ideas 的结合点。

### 三个有利观察（DvD 精读结论）

1. DvD 已经是 x0-prediction（预测去噪后的映射而非噪声）→ 换 FM 只动 noising 公式 + 采样循环，改动面小
2. DvD 3 步 DDIM 最优、**50 步反而变差**（误差累积）→ 任务不需要长轨迹，FM 直线路径 + 大步长正好对症
3. DvD 的 TVCR 训练时要多步采样模拟（作者自认 limitation 1）→ **FM 下可以"免费"**

### 方向清单

1. **直线路径 + 速度回归（保底）**：$x_t=(1-t)x_0+tx_1$，回归 $v=x_1-x_0$，欧拉采样；预期 1~2 步 vs DvD 3 步
2. **TVCR 免费化（novelty 亮点 ⭐）**：$x_t$ 本身就是映射——用 $x_t$ 直接 warp 特征做条件（零成本）或单次前向反解 $\hat{x}_1=x_t+(1-t)v_\theta$ 再 warp（无多步模拟）；同时解决 DvD limitation 1
3. **源分布设计（残差流）↔ 双 memory 的天然接口 ⭐**：Degradation Memory 的职责从"提供 init_flow"升级为"提供 FM 源分布"——先认出"这页是哪种弯"，用检索到的形变先验构造粗矫正映射 + 扰动，作为 FM 的起点，网络只学残差。HQ Memory 可作为 TVCR 时变条件的补充通道（告诉模型"目标形态长这样"）
4. **1 步极速 + reflow**：坐标场低维平滑，一步欧拉可能够用；不够就 reflow 拉直（SD3 图像空间 reflow 无收益 ≠ 坐标空间，值得单独验证）
5. **SD3 技巧移植**：logit-normal 采 t + 损失加权；与 idea「MFE 换 DINOv3-small」可并行做
6. **打 DvD limitation 2**：未见文档类型（发票等）泛化差 → FM 确定性 ODE 泛化实验 + AnyPhotoDoc6300 域标注细粒度评估
7. 【后续】FOD 的 uncertainty-aware loss → 套在 FM 速度回归上

### 定位与风险

- DvD 已经很猛，质量未必能超 → 论文定位建议：**同等质量 + 2~3 倍推理加速 + TVCR 训练免费化**，速度是主线，双 memory 是第二个贡献点
- 算力：64×64×2 坐标场 + 151M 参数，Doc3D 单卡可行
- 起步：clone DvD 代码先落地方向 1
