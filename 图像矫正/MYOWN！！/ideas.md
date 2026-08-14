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

2. **通过某种手段获取一种先验知识**

2. **采用FOD那种类似于memory的机制，保存这种先验**

3. **计划使用 DvD 作为baseline**
