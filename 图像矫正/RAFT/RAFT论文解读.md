# RAFT 论文理解以及DocScanner 论文对比学习
[DocScanner 论文](./DocScanner.pdf)
[RAFT 论文](./RAFT.pdf)



## RAFT

### 基本点理解
#### 这个网络用来干啥？
给两帧画面，RAFT算出每个像素从第一帧挪到第二帧时，向哪个方向，移动了多少。

#### 输入输出？
输入：两帧画面
输出：一张光流图，包含每个像素的移动方向和移动距离

#### 基本流程：
1. 提取特征
2. 算所有像素对的相似度
3. 反复查相似度表来修正光流，越修越准

#### 什么是光流？
简单来说就是画面里每个像素的运动速度和方向。

#### 论文的基本框架：
RAFT 网络包含三个主要组件：
1. 特征提取器：从两张输入图像中提取逐像素的特征，以及一个上下文编码器，仅从第一帧图像中提取。
2. 相关层：对所有特征向量对计算内积，构建一个$W\times H\times W\times H$的相关体。最后两位经过多尺度优化，生成一组多尺度相关体。3
3. 一个基于GRU光流更新算子：从相关体中检索取值，并对初始为零的光流场进行迭代更新。
```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'primaryColor': '#E8F0FE',
  'primaryTextColor': '#1a2a3a',
  'primaryBorderColor': '#4A90D9',
  'lineColor': '#5B7FA5',
  'fontSize': '14px'
}}}%%
flowchart TD
    I1["🖼️ 第 t 帧 I₁"]
    I2["🖼️ 第 t+1 帧 I₂"]

    I1 --> FE1["特征编码器 Feature Encoder"]
    I2 --> FE2["特征编码器 Feature Encoder"]
    I1 --> CE["上下文编码器 Context Encoder"]

    FE1 --> |"逐像素 256 维特征"| CORR["② 4D 相关金字塔 Correlation Pyramid"]
    FE2 --> |"逐像素 256 维特征"| CORR

    CORR --> |"多尺度匹配分数"| GRU

    CE --> |"上下文特征直接注入"| GRU

    subgraph ITER["③ 更新算子 ⚙️ 循环 12~100+ 次"]
        GRU["ConvGRU<br/>门控循环单元<br/>2.7M 参数"]
        FLOW["光流场 fₖ"]
        GRU -->|"修正量 Δf"| FLOW
        FLOW -->|"当前光流去查相关体"| GRU
    end

    FLOW --> |"Convex Upsampling<br/>1/8 → 原分辨率"| OUT["🎯 最终光流<br/>H×W×2"]
```
![RAFT模型框架](./RAFT模型框架.png)

#### 什么是相关体?
- 他是一张巨大的相似度对照表——记录了第一帧每个像素和第二帧图像中每个像素的长的有多像。
- 经过特征提取后，每个像素拥有一个长度为$D=256$的向量。
- 第一帧中每个像素和第二帧中的每个像素都做向量内积，得到一个标量，如此一来第一帧中每个像素位置上的就会获得一个$H\times W$形状的张量，而一张图像拥有$W\times H$个像素，所以相关体的形状就是$W\times H\times W\times H$。

#### 多尺度池化是何意为？相关金字塔是？
- 论文中提到在最后两个维度做不同隔离度的平均池化，是为了直接查4D体，上下文感受范围有限的问题。
- 池化方式：

| 尺寸 | 形状 | 意义 |
|:---:|:---:|:---:|
|原始|$H\times W\times H\times W$|精细结构，适合小位移|
|1/2|$H\times W\times \frac{H}{2}\times \frac{W}{2}$|稍微粗糙|
|1/4|$H\times W\times \frac{H}{4}\times \frac{W}{4}$|更加粗糙|
|1/8|$H\times W\times \frac{H}{8}\times \frac{W}{8}$|最粗糙，但能够覆盖256像素的大位移|
```mermaid
flowchart TD
    C0["原始 4D 相关体<br/>C₁: H × W × H × W"] --> POOL2["2×2 平均池化<br/>（后两维）"]
    POOL2 --> C2["C₂: H × W × H/2 × W/2"]
    C2 --> POOL4["2×2 平均池化<br/>（后两维）"]
    POOL4 --> C4["C₃: H × W × H/4 × W/4"]
    C4 --> POOL8["2×2 平均池化<br/>（后两维）"]
    POOL8 --> C8["C₄: H × W × H/8 × W/8"]

    subgraph LEG["图例"]
        L1["C₁: 精细，查小位移"]
        L2["C₂: 中等"]
        L3["C₃: 较粗"]
        L4["C₄: 最粗，2⁴=16倍池化<br/>半径 4 = 原图 256px 范围"]
    end
```
那进一步怎么查呢？
更新算子每次迭代时，会用当前的光流估计在每一层金字塔的对应位置上查一个 半径 r 的小窗口（r=4 即 9×9=81 个特征值），然后把四层的查询结果拼接起来喂给 GRU。
```mermaid
flowchart LR
    FLOW["当前光流 fₖ<br/>第1帧像素 (u,v)"]
    FLOW --> MAP["映射到第2帧<br/>(u',v') = (u+dx, v+dy)"]

    MAP --> L1["在 C₁ 层查<br/>±4 像素范围"]
    MAP --> L2["在 C₂ 层查<br/>±4×2=±16px 范围"]
    MAP --> L3["在 C₃ 层查<br/>±4×4=±64px 范围"]
    MAP --> L4["在 C₄ 层查<br/>±4×8=±256px 范围"]

    L1 --> CAT["拼接所有层的<br/>查询结果"]
    L2 --> CAT
    L3 --> CAT
    L4 --> CAT

    CAT --> GRU["喂给 ConvGRU"]
```

#### 算子更新怎么做？
```mermaid
flowchart TD
    INIT["初始化 f₀ = 0（全部为零）"] --> LOOP

    subgraph LOOP["🔄 迭代更新循环 (12~100+ 次)"]
        direction TB
        STEP1["📥 输入拼接<br/>x_t = [相关特征 | 光流特征 | 上下文特征]"]
        STEP2["🧠 ConvGRU 门控单元<br/>z_t 更新门 / r_t 重置门<br/>h̃_t 候选状态 / h_t 输出"]
        STEP3["🎯 两层卷积预测 Δf"]
        STEP4["➕ f_{k+1} = f_k + Δf"]
        
        STEP1 --> STEP2 --> STEP3 --> STEP4
        STEP4 -->|"用新光流重新查相关体"| STEP1
    end

    LOOP --> UP["📐 Convex Upsampling<br/>3×3 邻域加权组合<br/>1/8 分辨率 → 原分辨率"]
    UP --> OUT["✅ 最终光流 H×W×2"]
```

## DocScanner 与本论文的异同对比学习
DocScanner中的核心点是`warping flow`矫正流，与RAFT在结构上高度类似：
1. 权重绑定
2. ConvGRU更新单元
3. flow机制
4. 可学习的上采样模块

存在一下几点核心差异：

|差异维度|RAFT|DocScanner|
|:---:|:---:|:---:|
|匹配维度|4D全配对相关特征金字塔|warping操作|

- RAFT：构建了一个所有像素对相似度的大表，查询第一张图像中每个像素和第二帧图像中每个像素相似度。
- DocScanner：没有建相关体，而是直接用当前 warping flow 把扭曲图的特征 "掰直" 到矫正域，然后 GRU 看掰直后的特征哪里还不直、接着修 → 不需要全配对，因为文档矫正的映射关系没有光流那么复杂（不需要解决遮挡、快速运动等）。




