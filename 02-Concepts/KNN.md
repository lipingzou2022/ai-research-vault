---
type: concept
note_status: reviewed
maturity: seed
concept_kind: model
aliases:
  - K-Nearest Neighbors
  - K近邻算法
  - KNN 学习记录
created: 2026-09-09
updated: 2026-09-09
---

# KNN：从最近邻思想到特征向量与超参数选择

> [!summary] 一句话理解
> KNN 把每个样本表示为同一特征空间中的向量；预测新样本时，计算它与训练样本的距离，选出最近的 K 个邻居，再以投票完成分类或以邻居数值综合完成回归。

## 我的原始疑惑

- KNN 所谓“最近”究竟由机器怎样计算？
- 欧氏距离理解后，曼哈顿距离和 Minkowski 距离为什么又出现？
- 特征尺度、量纲、单位、变化幅度和特征重要性是否是一回事？
- 归一化、标准化、正则化，以及“正常化 / 正规化”等说法有什么区别？
- 为什么测试集不能使用自己的 min / max 或 mean / std？
- Cross Validation 为什么会自然引出 Pipeline？
- 图片怎样变成 KNN 能接收的 Feature？Feature、Feature Vector 与 Feature Matrix 又有什么关系？

这些困惑本质上集中在一条链上：KNN 依赖距离，而距离依赖输入表示和预处理；一旦预处理需要从数据中学习规则，就必须严格防止训练、验证与测试信息混用。

## 我的当前理解

### KNN 的基本思想

KNN（K-Nearest Neighbors，K 近邻）预测未知样本时，不先学习一个复杂函数，而是参考距离它最近的若干已知训练样本。

```text
新样本
→ 与所有训练样本计算距离
→ 按距离排序
→ 选出最近的 K 个邻居
→ 分类：多数投票
→ 回归：综合邻居的数值
```

在电子鼻分类中，若最近 3 个已标注样本为“酒精、酒精、甲醛”，最简单的多数投票会预测“酒精”。K 在 scikit-learn 中对应 `n_neighbors`。

### 分类与回归

| 任务 | 输出 | 最后一步 |
| --- | --- | --- |
| KNN Classification | 离散类别，如酒精 / 甲醛 / 氨气 | 对 K 个邻居的类别投票 |
| KNN Regression | 连续数值，如气体浓度 | 综合 K 个邻居的数值；最简单时取平均 |

```python
from sklearn.neighbors import KNeighborsClassifier, KNeighborsRegressor

classifier = KNeighborsClassifier(n_neighbors=3)
regressor = KNeighborsRegressor(n_neighbors=3)
```

### Lazy Learning（惰性学习）

KNN 的 `fit(X_train, y_train)` 不像许多参数化模型那样通过迭代学习大量权重。当前可以把它理解为保存训练样本及标签；预测时才计算新样本与训练样本的距离，因此预测开销可能更大。具体实现也可能建立搜索索引，不能把“fit 什么都不做”理解得过于绝对。

## 为什么需要它

KNN 是理解监督学习、特征表示、距离度量、数据预处理和模型选择之间关系的好入口。它没有复杂的参数公式，因此可以直接看见：一旦样本的表示或尺度改变，“谁是谁的邻居”就会改变。

在 [[电子鼻模式识别]] 中，KNN 是把传感器响应或其特征向量映射为气体类别、浓度或状态的传统方法之一。它是否适合某项电子鼻任务，仍取决于数据规模、特征、噪声、漂移和评价协议。

## 输入与输出

- 输入：训练集特征矩阵 `X_train`、对应标签或目标值 `y_train`，以及经过同一预处理规则转换的新样本。
- 输出：分类时为预测类别；回归时为预测连续值。
- 基本形状：`X.shape = (n_samples, n_features)`。

即使预测单个样本，scikit-learn 也通常需要二维输入：

```python
X_new = np.array([[1.1, 1.15]])  # shape: (1, 2)
```

而 `np.array([1.1, 1.15])` 的形状是 `(2,)`，只是一维特征向量，不满足 API 对“样本数 × 特征数”的通常约定。

## Feature、Feature Vector 与 Feature Matrix

```text
单个 Feature：一个描述数字，例如最大响应值
→ Feature Vector：一个样本的多个特征按固定顺序排列
→ Feature Matrix X：多个样本的 Feature Vector 堆叠成矩阵
```

电子鼻中的一个样本可以写成：

$$
x = [120, 15, 32, 850]
$$

分别表示最大响应值、响应时间、恢复时间和曲线面积。多个样本构成：

```text
X = [
  [120, 15, 32, 850],
  [135, 12, 29, 910],
  [82,  25, 48, 630]
]
```

单个有 4 个特征的向量形状通常是 `(4,)`；3 个样本、每个样本 4 个特征组成的矩阵形状是 `(3, 4)`。

### 图像也能成为特征向量

在 scikit-learn 的 Digits 数据集中，一张 `8 × 8` 手写数字图像可展开为 64 个像素值，即一个 64 维 Feature Vector。`digits.images.shape ≈ (1797, 8, 8)` 表示 1797 张图像；`digits.data.shape ≈ (1797, 64)` 表示 1797 个样本、每个样本 64 个特征。

KNN 此时把每张图片视为 64 维特征空间中的一个点，并与训练集中其他点计算距离。

## 距离度量

### 欧氏距离

欧氏距离是两点之间的直线距离：

$$
d(A,B) = \sqrt{\sum_i (x_i-y_i)^2}
$$

例如 $A=(1,2)$、$B=(4,6)$，距离为 $\sqrt{3^2+4^2}=5$。

### 曼哈顿距离

曼哈顿距离把各维差异的绝对值相加，直觉上像只能横向和纵向穿行的城市街区：

$$
d(A,B) = \sum_i |x_i-y_i|
$$

同一例子中距离为 $|4-1|+|6-2|=7$。

### Minkowski 距离

Minkowski 距离是统一框架：

$$
d(A,B) = \left(\sum_i |x_i-y_i|^p\right)^{1/p}
$$

```text
p = 1 → Manhattan Distance
p = 2 → Euclidean Distance
```

在 scikit-learn 中，`metric="minkowski", p=2` 等价于使用欧氏距离；`p=1` 等价于使用曼哈顿距离。

## 特征尺度与预处理

### Scale 不是重要性

特征的**尺度**可暂时理解为：数值通常多大、分布在哪个范围、变化有多大。它不同于量纲、单位、变化幅度和特征的重要性。

```text
量纲：物理量属于什么类型，例如时间、长度、浓度
单位：怎样测量，例如秒、分钟
变化幅度：数值最大值与最小值相差多大
尺度：数值典型范围和大小
重要性：对任务预测是否有贡献
```

尺度大不等于特征更重要。但若一个样本是 `[1000, 5]`，另一个是 `[2000, 6]`，欧氏距离中的第一个特征差异为 1000，第二个为 1，前者会几乎完全主导距离。因此 KNN 对尺度特别敏感。

### 归一化、标准化、正则化

| 概念 | 作用对象 | 核心理解 |
| --- | --- | --- |
| Normalization / Min-Max Scaling（归一化） | 数据 / 特征 | 常按比例缩放到固定范围，如 0 到 1 |
| Standardization / Z-score（标准化） | 数据 / 特征 | 转换到均值约为 0、标准差约为 1 的尺度 |
| Regularization（正则化） | 模型 / 参数 | 约束模型复杂度，降低过拟合风险 |

Min-Max 形式：

$$
x' = \frac{x-x_{min}}{x_{max}-x_{min}}
$$

Z-score 形式：

$$
z = \frac{x-\mu}{\sigma}
$$

标准化与统计学中的 z-score 使用同一基本思想，结果不必落在 0 到 1，可能是负数。为避免中文翻译混乱，当前固定使用“归一化、标准化、正则化”三个术语，而不单独使用含义不清的“正常化 / 正规化”。

## fit、transform 与数据泄漏

以缩放器为例：

```text
fit()：从训练数据学习规则
transform()：使用已经学到的规则转换数据
fit_transform()：先学习规则，再用其转换同一训练数据
```

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

对于 `StandardScaler`，`fit(X_train)` 会计算并保存训练集每一列的均值和标准差；`MinMaxScaler` 则学习每列最小值和最大值。

测试集模拟未来未知数据，不能参与制定预处理规则。因此下面的做法会形成数据泄漏：

```python
X_test_scaled = scaler.fit_transform(X_test)  # 错误：测试集参与学习 min/max 或 mean/std
```

关键原则是：训练数据负责“学规则”，验证 / 测试数据只负责“用规则”。这与 [[机器学习基础总地图]] 中的数据划分和预处理原则一致。

## 超参数、交叉验证与 Pipeline

### 超参数与 K 的选择

`n_neighbors=5` 中的 5 不是模型自动学到的参数，而是人预先设置的 Hyperparameter（超参数）。KNN 常见超参数还有 `weights`、`metric` 和 Minkowski 距离的 `p`。

```text
K 太小：局部、对噪声或异常点敏感，可能过拟合
K 太大：把较远样本也混入，局部规律被冲淡，可能欠拟合
```

`weights="uniform"` 代表邻居同权投票；`weights="distance"` 代表距离更近的邻居影响更大。具体权重函数仍未深入。

### Cross Validation 与 GridSearchCV

K-Fold Cross Validation 把训练数据分成 K 份，轮流用其中一份验证、其余部分训练，最后汇总多轮分数。它比单次固定验证集更不易受偶然划分影响。

> 5-Fold 中的 5 与 KNN 的 K 是两个不同概念：前者是验证折数，后者是最近邻数量。

Grid Search 负责系统尝试给定参数组合；Cross Validation 负责评价每组参数。`GridSearchCV` 把二者组合起来。

### Pipeline 为什么自然出现

交叉验证的每一折都会产生“本折训练数据”和“本折验证数据”。若在交叉验证前对完整 `X_train` 执行 `scaler.fit_transform(X_train)`，每一折的验证部分已经提前参与均值、标准差或最小 / 最大值计算，形成验证数据泄漏。

```text
本折训练部分 → scaler.fit() → scaler.transform() → KNN.fit()
本折验证部分 → 只 scaler.transform() → KNN.predict()
```

Pipeline 将“预处理 → 模型”绑定在一起，使交叉验证能在每一折按正确顺序执行：

```python
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("knn", KNeighborsClassifier())
])
```

Pipeline 不是 KNN 算法本身，而是避免工程流程中预处理泄漏的工具。Pipeline 内部参数搜索写法和完整 `GridSearchCV` 语法暂未深入。

## 本次纠正的错误理解

1. KNN 的 `fit()` 必然像神经网络一样训练权重 → KNN 主要保存训练数据，核心距离计算主要发生在预测时。
2. 尺度大意味着特征重要 → 尺度只反映数值范围，却会在距离计算中产生不成比例的影响。
3. 归一化、标准化、正则化都是压到 0 到 1 → 三者作用对象和目的不同。
4. 测试集应使用自己的缩放规则 → 测试数据不应参与任何 `fit()`。
5. Pipeline 是突然出现的新模型 → 它是处理“交叉验证中的预处理泄漏”问题的工程组织工具。
6. Feature Vector 是抽象新名词 → 它就是一个样本的多个特征按固定顺序排成的向量。

## 与其他概念的关系

- [[机器学习基础总地图]]：KNN 是总地图中第一条具体传统机器学习算法学习线。
- [[电子鼻模式识别]]：KNN 可使用传感器响应的特征向量进行气体分类或回归。
- [[电子鼻信号处理]]：原始响应需先被转换为可比较的特征或时序表示；距离度量对这种表示和尺度敏感。
- [[传感器漂移]]：若时间变化导致特征空间整体移动，KNN 的“邻居”关系也可能失效；如何处理仍需结合真实数据与评价协议验证。

## 来源与证据

### 已验证来源

- [[ZOTERO-QB28SJ6R - 电子鼻技术及其应用研究进展]]、[[ZOTERO-MICUHCB4 - 气体传感器阵列与模式识别]]：两篇综述都将 KNN 列为电子鼻模式识别中的方法范围，但当前不把它们当作 KNN 原理、参数或性能结论的直接证据。

### AI辅助解释／待验证

- 本页关于 KNN、距离、缩放器、交叉验证、GridSearchCV、Pipeline 和 Digits 数据集的说明，来自 2026-09-09 的 ChatGPT 学习总结，经 Codex 结构化整理；尚未逐项由教材或 scikit-learn 官方文档核验。
- 代码用于理解 API 和数据流，不构成真实电子鼻任务的完整实验协议。

## 尚未理解

- 邻居搜索效率：brute force、KD Tree、Ball Tree 与 `algorithm` 参数。
- 高维空间中距离可能不可靠的原因，即维度灾难。
- `weights="distance"` 的具体数学权重计算。
- Manhattan 与 Euclidean 在不同数据条件下如何选择。
- Pipeline 与 GridSearchCV 的完整参数搜索语法，例如 `knn__n_neighbors`。
- 手写数字案例的混淆矩阵、不同 K 的性能和预测效率分析。

## 自我检查

1. KNN 中的 K 表示什么？为什么 K=1 容易受噪声影响？
2. KNN 分类与 KNN 回归最后一步有什么不同？
3. 为什么 KNN 被称为 Lazy Learning？
4. 新样本进入 KNN 后依次经历哪些步骤？
5. 欧氏距离、曼哈顿距离和 Minkowski 距离的关系是什么？
6. Minkowski 的 p=1、p=2 分别对应什么？
7. 什么叫 Feature Scale？尺度大是否代表特征重要？
8. 量纲、单位、变化幅度、尺度分别是什么？
9. 归一化、标准化、正则化各自作用于什么？
10. 为什么标准化结果可以为负数？
11. `fit()`、`transform()`、`fit_transform()` 分别做什么？
12. 为什么测试集不能执行 `fit_transform()`？
13. KNN 的 K 与 K-Fold 的 K 有什么区别？
14. Grid Search 和 Cross Validation 分别解决什么问题？
15. 为什么交叉验证与缩放器结合时容易出现新型数据泄漏？
16. Pipeline 为什么能降低这类错误的风险？
17. 单个 Feature、Feature Vector、Feature Matrix X 三者是什么关系？
18. 为什么一张 8 × 8 图片能成为 64 维特征向量？
19. 对电子鼻数据而言，为什么响应表示和尺度会改变 KNN 中“邻居”的含义？

## 学习记录

### 2026-09-09

- 讨论主题：KNN、距离、特征尺度、预处理、模型选择与 Pipeline。
- 理解变化：从“找邻居”的口号式理解，发展到能串起特征向量、距离、缩放、数据泄漏、交叉验证和 Pipeline 的因果链。
- 后续问题：先完成 KNN 的最小代码练习和 Digits 数据集实验，再学习 KNN 搜索效率、维度灾难和参数选择细节。
