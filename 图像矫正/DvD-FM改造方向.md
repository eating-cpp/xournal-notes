# DvD × Flow Matching 改造方向

基于 DvD 论文（arXiv:2505.21975, SIGGRAPH Asia 2025）精读。代码：<https://github.com/hanquansanren/DvD>

## 一、DvD 方法要点（精读笔记）

**任务形式**：输入畸变图 $I_w$ → 预测小尺寸 backward mapping $m$（64×64）→ 上采样到全分辨率 $M_0\in\mathbb{R}^{H\times W\times 2}$（每个位置存对应原图坐标 $(i,j)$）→ `grid_sample` 重采样原图得矫正图。**内容保真由构造保证**——模型只能生成坐标，不能改字。

**扩散对象**：坐标 mapping 而不是像素。DDPM 前向：$m_t=\sqrt{\bar\alpha_t}m_0+\sqrt{1-\bar\alpha_t}z$；网络直接预测**去噪后的 mapping** $\hat{m}_{0,t}$（x0-prediction，不是噪声）：$\mathcal{L}=\mathbb{E}\left[\left\|m_0-\epsilon_\theta(m_t,t,c_t)\right\|^2\right]$

**复合条件** $c_t=\{f_d,f_m,f_l,r_t\}$：

- $f_d$：原图特征（VGG16 前 3 块，联合训练）
- $f_m$：前景特征（U2Net，冻结）
- $f_l$：文本线特征（UNet 解码器，冻结）
- $r_t$：**时变条件** = $\{m_{0|t}, f_{0|t}\}$——当前去噪出的映射 + 用它矫正后的文档特征

**TVCR 的代价**：训练时为了拿到 $m_{0|t}$，要从 $T$ 到 $t$ **模拟多步采样**（Alg.1）→ 训练慢。作者自认 limitation 1。

**架构**：DiT_S_2 风格——12 个 CEB（条件嵌入块，交叉注意力：条件当 K/V、噪声潜变量当 Q，四条并行对应四种条件）+ 6 个 FGB（自注意力 + FFN）+ 3 层线性投影，**151M 参数**。

**推理**：DDIM，**3 步最优**（1 步差；50 步反而更差——误差累积），dual-hypothesis 双采样取平均。3 步约 0.59s/张。

**数据**：只用 Doc3D。**关键消融**：同架构下"映射生成"完胜"映射回归"（MS-SSIM 0.549 vs 0.487）；条件分量互补（$f_d$ 0.409 → +$f_m,f_l$ 0.519 → +$r_t$ 0.549）；64×64 潜空间最优。

**Limitations**：(1) 训练慢（TVCR 多步模拟）；(2) 未见文档类型（发票等）泛化差。

## 二、对 FM 有利的三个观察

1. **DvD 已经是 x0-prediction**——换成 FM 只需要改 noising 公式和采样循环，代码改动面小
2. **DvD 的 DDIM 3 步最优、50 步恶化**——说明这个任务根本不需要长轨迹，直线路径 + 大步长正好对症
3. **TVCR 的昂贵部分（训练时多步模拟）在 FM 下可以"免费"**——见方向 2

## 三、六个改造方向

> **术语澄清（先看这里）**：单次训练的"rectified flow"和"直线路径 flow matching"**是同一个东西**——都是 $x_t=(1-t)x_0+tx_1$ + 回归 $v=x_1-x_0$。区别只在 **reflow**：用模型自己生成的配对迭代重训（见方向 3）。SD3 自称 rectified flow 但没做 reflow，用的是直线 FM 本身。
> **结论：起步直接用直线 FM，不做 reflow**。reflow 需要"先生成整个数据集的配对再重训"，复杂度翻倍；坐标场任务接近确定性，预期 1~3 步欧拉就够。reflow 留作后续加分实验。

### 方向 1：直线路径 + 速度回归（核心改动，保底结果）

把 DDPM 换成 FM（沿笔记记号，$x_0$=噪声、$x_1$=GT mapping）：

$$
x_t=(1-t)x_0+t\,x_1,\qquad \mathcal{L}=\mathbb{E}\left[\left\|v_\theta(x_t,t,c_t)-(x_1-x_0)\right\|^2\right]
$$

采样：欧拉积分。改动清单：noising 公式、损失目标（$x_1-x_0$ 或直接 x1-prediction）、采样循环、$t$ 定义域。架构和条件管线原样保留。

**预期卖点**：1~2 步欧拉 vs DvD 3 步 DDIM；DvD 里"步数多反而差"的误差累积问题在直线路径下应消失。

### 方向 2：TVCR "免费化"（novelty 亮点 ⭐）

DvD 的 limitation 1 是 TVCR 训练时要多步模拟。FM 下 $x_t$ **本身就是一个映射**（噪声和 GT 的线性混合），时变条件可以直接构造：

- 方案 A（零额外成本）：用 $x_t$ 直接 warp 文档特征做条件
- 方案 B（一次额外前向）：用网络输出反解 $\hat{x}_1=x_t+(1-t)v_\theta$，用 $\hat{x}_1$ warp 特征做条件——**没有多步模拟，只有一次前向**

推理时逐步 refinement 天然存在（每步用当前预测更新条件）。这个方向同时解决 DvD 的 limitation 1，论文可以写"FM 让 TVCR 免去训练时采样循环"。

### 方向 3：1 步极速推理 + reflow

坐标场是低维（64×64×2）平滑场，直线路径下一步欧拉可能就够用；目标 1-step dewarping。若不够直，做一次 reflow 拉直（坐标空间计算代价低，和 SD3 图像空间"reflow 无收益"的结论不冲突，值得单独验证）。卖点：实时文档矫正（手机端）。

### 方向 4：源分布设计（残差流）

源不一定是纯高斯：用**恒等映射 + 扰动**或**粗回归器输出**当起点，FM 只学残差修正。文档矫正天然有好的粗先验（恒等映射≈小幅矫正、回归器≈粗矫正）。呼应 DvD 作者自己的 Coarse-to-Fine Registration（Zhang et al. ICDAR 2024）。

### 方向 5：SD3 技巧移植（低成本小改进）

- $t$ 从 $U(0,1)$ 换成 logit-normal + 损失加权
- 对 mapping 场测最优 $t$ 分布（坐标场和自然图像的分布差异大，可能结论不同）

### 方向 6：打 DvD 的 limitation 2

DvD 对未见文档类型（发票等）泛化差。可尝试：FM 确定性 ODE 的泛化稳定性分析 + AnyPhotoDoc6300 的域标注做细粒度评估 + 数据增强。若 FM 版本在域泛化上更好，是第二个 solid 卖点。

## 四、实验设计建议

- **数据**：Doc3D 训练（与 DvD 公平对比）；DocUNet + DIR300 + AnyPhotoDoc6300 评测
- **指标**：MS-SSIM / LD / AD + CER / ED + MMCER / MMED
- **基线**：DvD 原版、DvD-回归变体、DewarpNet、DocGeoNet、FTA、DocScanner
- **核心消融**：DDPM vs FM 路径（同架构）；TVCR 方案 A/B vs 无；**NFE-质量曲线**（1/2/3/5 步）；t 分布；源分布
- **算力**：64×64×2 坐标场 + 151M 参数，Doc3D 单卡 4090 级别可行，比图像级 FM 便宜一个量级
- **起步**：直接 clone DvD 代码，先把方向 1 落地跑通（改动最小），再加方向 2

## 五、风险提示

- DvD 的 x0-prediction + 3 步 DDIM 已经很强，FM 版**质量未必能超越**——论文定位建议是"同等质量下的速度/训练成本优势 + TVCR 免费化"，而不是只拼指标
- 坐标 mapping 的 GT 来自 Doc3D 渲染（3D 坐标图），数据管线要先复现对
- 若方向 1+2 组合后指标持平但快 2-3 倍、训练快数倍，就是完整故事；方向 3（1 步）若成功是加分项

## 六、方向 1 落地代码地图（DvD repo，improved-diffusion 框架）

代码已 clone 到 `d:\personal\study_station\DvD-FM\DvD`。改动集中在三个函数，**不需要外部 FM 库**：

| 函数 | 位置 | 现状 | 改法 |
|---|---|---|---|
| `q_sample` | `train_settings/dvd/improved_diffusion/gaussian_diffusion.py:250` | DDPM：$\sqrt{\bar\alpha_t}x+\sqrt{1-\bar\alpha_t}\epsilon$ | 直线插值：$x_t=(1-t)\epsilon+t\,x_1$ |
| `training_losses_new` / `training_losses_time_variant` | 同文件 `:833` / `:890` | 目标 = START_X（即 GT mapping，x0-prediction） | **目标不用改**——FM 的 $x_1$ 就是 GT mapping，网络继续输出 mapping 估计即可（x1-prediction） |
| `ddim_sample` | 同文件 `:445` | DDIM 更新 | 欧拉步：$x_{t+\Delta t}=x_t+(\hat{x}_1-x_t)\frac{\Delta t}{1-t}$ |

**训练循环**：`train_util.py:211 run_loop_dewarping`（调 `training_losses_*`）；**推理**：`evaluation.py:121`（调 `ddim_sample_loop`）。

**注意事项**：

1. **t 域映射**：DDPM 的 $t$ 是离散索引（0..T-1），FM 的 $t\in[0,1)$。模型时间编码走 `_scale_timesteps`（`:440`，$t\times 1000/T$），FM 的 $t$ 乘回 $T$ 传入即可保持编码一致
2. **采样循环**：从 $t=0$（纯噪声）走到 $t=1$，$N$ 步；$t=1$ 处 $\frac{\Delta t}{1-t}$ 除零——循环到 $t=1$ 时直接取 $\hat{x}_1$ 不更新。训练时 $t\sim U(0,1)$ 即可（或 `randint(0,T)/T`）
3. **TVCR 训练版自动跟随**：`ddim_sample_for_training`（`:694`）内部复用 `ddim_sample`——把后者改成欧拉后，TVCR 的多步模拟也自动变成 FM 一致，无需单独改
4. **最小版建议**：先不动 `reg_model_bilin`（粗到精注册）和 mask 组件，只改上述三处跑通；双采样平均（dual-hypothesis）对确定性 ODE 可先去掉对比
5. **参考实现（需要对照公式时）**：[torchcfm](https://github.com/atong01/conditional-flow-matching)（pip 可装，条件 FM / OT-CFM 损失）、[facebookresearch/flow_matching](https://github.com/facebookresearch/flow_matching)（积分器/条件器齐全）、[minRF](https://github.com/cloneofsimo/minRF)（最小可读 rectified flow 训练循环）
