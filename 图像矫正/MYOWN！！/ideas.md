# my own paper

## 参考论文的梳理
### 已经参考过的论文
1. RAFT:  [RAFT论文解读](../RAFT/RAFT论文解读.md)
2. FOD扩散模型加持下的超分辨率 [FOD论文解读](..\FOD扩散模型超分辨\FOD论文比.md)
3. DvD [DvD](../DVD\DvD论文理解.md)
4. DocScanner [DocScanner](../DocScanner\DocScanner论文理解.md)
5. DewarpNet [DewarpNet](../DewarpNet\DewarpNet论文理解.md)

## 初步的ideas
1. 前景掩码估计外加一个深度估计，测算这张纸相对于相机各个地方的距离。根据这种深度信息，学习一种矫正场作为先验，和扩散模型相结合，来恢复出展平的文档图像

2. **flowmatching能不能做这个？**

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
