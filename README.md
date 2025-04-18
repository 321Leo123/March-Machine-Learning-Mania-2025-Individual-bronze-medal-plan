# March-Machine-Learning-Mania-2025-Individual-bronze-medal-plan

NCAA Kaggle 竞赛方案总结：基于多层次特征工程与集成学习的胜率预测模型

### 1. 项目概述与目标

本项目旨在解决 NCAA（美国大学体育协会）篮球锦标赛（“疯狂三月”）的胜负预测问题。目标是利用历史常规赛和锦标赛的详细比赛数据（Box Score）、球队种子排名等信息，构建一个机器学习模型，以尽可能准确地预测未来锦标赛中任意两支球队之间的胜率。最终产出为 Kaggle 竞赛要求的提交文件，包含每场可能对决的 ID 和预测的 Team1 获胜概率。

### 2. 数据处理与准备

数据整合: 代码首先将男子（M）和女子（W）篮球的常规赛、锦标赛结果及种子数据分别合并 (pd.concat)，便于后续统一处理和特征工程，并通过 men_women 标志位区分，体现了数据处理的全面性。

赛季筛选: 限定了分析的起始赛季 (season = 2003)，确保使用相对现代且数据模式可能更一致的数据。

核心函数 prepare_data:

标准化统计数据 (Overtime Adjustment): 认识到加时赛 (NumOT) 会导致比赛时间延长，进而使得得分、篮板等原始统计数据虚高。代码通过计算调整因子 adjot = (40 + 5 * df["NumOT"]) / 40，并将所有 Box Score 相关统计量除以该因子，将数据标准化到常规40分钟比赛的水平。这是一个非常精细且重要的处理，确保了不同比赛（有无加时）数据间的可比性，提升了特征质量。

数据对称性增强 (Data Augmentation): 对于每一场比赛 WTeamID vs LTeamID，代码不仅保留原始记录，还创建了一个镜像记录，将 LTeam 作为 T1，WTeam 作为 T2 (dfswap)，并将两部分数据合并 (pd.concat)。这使得数据集加倍，并且让模型能够从两个视角学习同一场比赛。这强制模型学习相对实力差异 (PointDiff)，而不是依赖于队伍出现在“胜者”列还是“败者”列的偶然性，极大地增强了模型的鲁棒性和泛化能力。

目标变量定义: 计算 PointDiff (T1得分 - T2得分) 作为核心预测目标之一，并由此衍生出二元分类标签 win (PointDiff > 0)。同时保留 men_women 标志。

### 3. 特征工程

代码采用了分层级的特征工程策略，从易到难，逐步构建更复杂的特征，体现了系统性的建模思路：

简单难度特征 (Easy Features):

种子信息: 从 Seed 字段（如 W01, X16）中提取纯数字种子排名 (seed)。

种子合并与差异: 将 T1 和 T2 的种子信息合并到比赛数据中，并计算核心特征 Seed_diff (T2_seed - T1_seed)。这是预测比赛结果最直观、最常用的特征之一，代码通过可视化验证了其与 PointDiff 的强相关性。

中等难度特征 (Medium Features - 基于常规赛平均表现):

核心思想: 认为球队在锦标赛中的表现与其整个常规赛的平均水平相关。

计算方法: 对 regular_data 按赛季 (Season) 和球队 (T1_TeamID) 分组，计算选定 Box Score 指标 (boxcols) 的赛季平均值。

特征构建: 不仅计算了球队自身的平均表现（如 T1_avg_Score, T1_avg_FGA 等，前缀 T1_avg_），还计算了该球队对手的平均表现（如 T1_avg_opponent_Score, T1_avg_opponent_FGA 等，前缀 T1_avg_opponent_）。这组特征刻画了球队的“场均失分/对手命中率”等防守相关指标，提供了更全面的球队画像。

特征合并: 将计算出的 T1 和 T2 各自的赛季平均统计数据合并到 tourney_data 中，为模型提供了关于两队常规赛攻防能力的丰富信息。

困难难度特征 (Hard Features - Elo 等级分系统):

核心思想: 引入动态反映球队相对实力的 Elo 等级分系统。Elo 分数根据每场比赛的结果动态更新，能捕捉赛季中的状态起伏和实力变化，通常比静态的赛季平均值更具预测力。

实现细节:

定义了标准的 Elo 更新 (update_elo) 和期望胜率计算 (expected_result) 函数。

为每个赛季独立计算 Elo 分数。赛季初所有队伍赋予基础分 (base_elo)，然后按常规赛比赛日 (DayNum) 顺序遍历比赛，根据胜负结果更新胜者和负者的 Elo 分。k_factor 控制了单场比赛对 Elo 分数的影响程度。

将赛季末的 Elo 分数合并到 tourney_data 中，作为 T1_elo 和 T2_elo，并计算 elo_diff (T1_elo - T2_elo)。

优势: Elo 系统考虑了对手的强弱，战胜强队得分多，输给弱队扣分多，能更准确地反映球队的真实竞争力水平。可视化也展示了 Elo 分与种子排名的关系以及 elo_diff 对胜负的区分度。

最困难难度特征 (Hardest Features - 基于 GLM 的球队质量评分):

核心思想 (高度创新): 使用统计模型（广义线性模型 GLM）本身来生成特征。目标是估计每个球队在一个赛季内的内在“质量”或“实力”评分，该评分能最好地解释常规赛中的分差。

实现细节:

数据筛选: 仅选取参加锦标赛的球队 (st) 以及在常规赛中击败过锦标赛球队的非锦标赛队伍，聚焦于实力较强的队伍群体，使 GLM 估计更稳定和相关。

模型设定: 对每个赛季和性别（男/女）分别拟合 GLM。公式为 PointDiff ~ -1 + T1_TeamID + T2_TeamID，使用高斯分布族 (sm.families.Gaussian())。这里的 -1 表示模型没有全局截距，每个 TeamID 的系数直接代表该球队相对于某个基准（或平均水平）的进攻/防守净效应。将 TeamID 作为类别变量（通过 astype(str) 实现），模型会为每个（重要的）球队估计一个系数。

特征提取: GLM 拟合后，提取每个 TeamID 对应的系数 (glm.params) 作为该球队在该赛季的 quality 评分。

特征合并: 将 T1 和 T2 的 quality 评分合并到 tourney_data，并计算 diff_quality (T1_quality - T2_quality)。

优势: 这种方法超越了简单的平均数或 Elo，它试图在控制了所有比赛对手强弱的情况下，估计出每个球队的“真实”实力值。代码通过计算 AUC 并与种子差异的 AUC 进行比较，直接展示了 diff_quality 特征相较于专家给出的种子排名的优越预测能力，尤其在某些赛季表现突出。

### 4. 机器学习模型与训练

模型选择: 选用 XGBoost (xgb)，这是一个梯度提升决策树算法，以其高效、灵活和强大的性能在 Kaggle 竞赛中广受欢迎。目标设为回归 (reg:squarederror)，直接预测 PointDiff。

特征选择: 代码明确列出了最终用于训练的特征列表 (features)，包含了前面构建的各个层次的特征。注释掉的部分表明可能经过了特征筛选或实验。

超参数调优: 提供了一组精心调整的 XGBoost 超参数 (param)，如学习率 (eta)、子采样比例 (subsample, colsample_bynode)、树的复杂度 (max_depth, min_child_weight) 等，以及针对大数据的优化 (tree_method='hist', grow_policy='lossguide')。

交叉验证策略: 采用 Leave-One-Season-Out (LOSO) 交叉验证。对于每个赛季 S，模型使用除 S 之外的所有赛季数据进行训练，然后在赛季 S 上进行验证。这种方法非常适合时间序列相关的数据（如体育比赛），因为它模拟了真实预测场景（用过去预测未来），能更准确地评估模型在未见赛季上的泛化能力，避免了随机 K 折可能导致的数据泄露（同一赛季的比赛信息可能出现在训练集和验证集）。

模型训练与评估: 循环遍历每个赛季作为 OOF (Out-of-Fold) 验证集，训练一个对应的 XGBoost 模型，并记录每个赛季的验证集 MAE (平均绝对误差)。最终报告了平均 MAE，展示了模型预测分差的精度。

### 5. 预测后处理与概率校准

回归到概率的转换: XGBoost 模型输出的是预测的 PointDiff (分差)，而 Kaggle 竞赛要求提交的是获胜概率 Pred。

校准方法: 使用 scipy.interpolate.UnivariateSpline 进行概率校准。

首先，收集所有 OOF 预测的分差 (oof_preds) 和真实的比赛结果 (oof_targets > 0)。

然后，基于这些 OOF 数据，拟合一个单变量样条函数 (spline_model)。该函数学习从预测的分差映射到实际观测到的胜率。输入分差被限制在 [-t, t] 范围内 (t=25)，以提高样条拟合的稳定性。

优势: 相比简单的 sigmoid 函数转换 (1 / (1 + exp(-pred_diff)))，样条拟合是一种非参数方法，能更灵活地捕捉预测分差与实际胜率之间的复杂（可能非线性）关系。这有助于产生更准确、校准得更好的概率预测，这对于 Brier Score (竞赛的评估指标) 至关重要。输出概率也被裁剪到 [0.01, 0.99] 以避免极端预测。

评估校准效果: 计算了整体和分赛季的 Brier Score Loss，这是衡量概率预测准确性的标准指标，得分越低越好。可视化展示了校准前后的预测值与实际胜率的关系。

### 6. 提交文件生成

特征复现: 加载 SampleSubmissionStage2.csv 文件，并完全重复训练阶段的特征工程流程，为测试集中的每一场对决生成所有必需的特征（合并种子、赛季平均、Elo、GLM Quality 等）。

集成预测: 对测试集进行预测时，使用了所有 LOSO 训练出的模型。对每个模型，使用 model.predict() 得到预测分差，然后应用之前拟合的 spline_model 将分差转换为概率。最后，将所有模型产生的概率进行平均 (np.array(preds).mean(axis=0))，得到最终的集成预测概率 Pred。

优势: 模型集成（特别是基于不同数据子集训练的模型，如 LOSO）通常比单一模型更稳定、更鲁棒，能够降低过拟合风险，提高整体预测性能。

最终产出: 生成符合 Kaggle 格式的 predictions.csv 文件，包含 ID 和预测的 Pred。包含一个基于种子对阵的预测概率透视表作为最终检查。

### 总结

我的方案是一个结构清晰、技术深入的 NCAA 预测解决方案。其主要亮点在于：

精细的数据预处理: 特别是加时赛调整和数据对称性增强，体现了对业务场景的深刻理解。

多层次、高质量的特征工程: 从简单的种子差异，到复杂的赛季平均统计（含对手数据），再到动态的 Elo 评分，最后引入创新的基于 GLM 的球队质量评分，逐步提升了特征的预测能力。

健壮的建模与验证策略: 采用强大的 XGBoost 模型，并结合 Leave-One-Season-Out 交叉验证，确保了模型评估的可靠性和泛化能力。

先进的概率校准技术: 使用 Univariate Spline 对回归输出进行非参数校准，以获得更准确的概率预测，直接优化了竞赛的 Brier Score 指标。

有效的集成学习: 通过平均多个 LOSO 模型的预测结果，进一步提升了预测的稳定性和准确性。

该方案融合了领域知识（篮球统计）、数据处理技巧、多种特征工程方法（统计、动态评分、模型生成特征）以及先进的机器学习技术（XGBoost、LOSO CV、样条校准、集成），构成了一个强大且具有竞争力的 Kaggle 竞赛解决方案。
