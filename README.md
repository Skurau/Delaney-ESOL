# AI for Chemistry：ESOL 水溶解度预测

本项目学习如何从分子结构预测水溶解度：使用 RDKit 构建分子描述符和 Morgan 指纹，训练定量构效关系（QSPR）随机森林；再使用图神经网络（GNN）直接学习原子和化学键组成的分子图。
## 数据

使用 `data/delaney-processed.csv`，共 1128 行、10 列。目标列为 `measured log solubility in mols per litre`，即 **logS = log10(S / (mol L⁻¹))**；分子结构由 `smiles` 列提供。数据自带的 ESOL predicted 列仅用于探索，没有输入训练模型。

数据对应 Delaney 的 ESOL 数据，公开处理版本见 [DeepChem 数据文件](https://github.com/deepchem/deepchem/blob/master/datasets/delaney-processed.csv)，原始方法见 [Delaney, 2004, ESOL](https://doi.org/10.1021/ci034243x)。

原始文件 SHA256：`3fa1b501b2f940d4b016c3798f5fcd4c897b8455a0217efbb3715ac35011f290`。运行时检查必要列、缺失目标、无效 SMILES、空分子及重复结构；发现无效数据时停止，重复结构只报告，未删除或修改原始记录。

## 文件与运行

```text
Delaney ESOL/
├── data/delaney-processed.csv
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_qspr_random_forest.ipynb
│   └── 03_gnn_training.ipynb
├── results/
│   ├── model_comparison.csv
│   └── cross_validation.csv
├── archive/
│   ├── original_notebook
│   └── validation.json
├── requirements.txt
└── README.md
```

`archive/original_notebook` 保存整理前的完整 `ESOL.ipynb`（含当时未提交的 GNN 代码与输出），供历史追溯。

已验证使用 Windows 下的 **Python 3.12.15 / conda `chem-ai`** 环境。已有该环境时，在项目根目录运行：

```powershell
conda activate chem-ai
python -m jupyterlab
```

在其他机器上可新建独立环境，避免修改已有环境：

```powershell
conda create -n esol-final python=3.12
conda activate esol-final
python -m pip install -r requirements.txt
python -m ipykernel install --user --name esol-final --display-name "Python (esol-final)"
python -m jupyterlab
```

打开三份笔记，选择安装了依赖的内核，按 `01 → 02 → 03` 执行 **Restart Kernel and Run All Cells**。每份笔记均自行读取数据，不共享内核变量；从项目根目录或 `notebooks/` 启动都可定位数据。`02` 与 `03` 分别更新最终比较表中自己的结果；首次复现需运行两份训练笔记才能生成完整表。笔记保存了本次执行输出，重跑会更新输出和结果 CSV。

`requirements.txt` 固定本次验证环境的直接依赖版本；不同系统、底层计算库或 PyTorch 构建仍可能产生数值差异。无需安装 PyTorch Geometric 或使用 GPU。

## 方法

- **探索**：描述统计、目标分布、分子量与极性表面积关系、Pearson 相关性，以及数据自带 ESOL 预测的误差检查。
- **RDKit / QSPR**：5 个描述符为分子量、logP、极性表面积（TPSA）、氢键供体数、氢键受体数；Morgan 指纹半径 2、2048 位；拼接为 2053 维。随机森林使用 300 棵树、`random_state=42`，并保留描述符线性回归基线。
- **随机与骨架划分**：随机训练/测试 902/226；Bemis–Murcko 骨架按组大小降序贪心分配，训练/测试同为 902/226，且骨架组不交叉。无环分子统一在空骨架组内。另保留既有打乱 5 折描述符验证及组合特征的 GroupKFold，后者将无环分子逐行单独分组；交叉验证独立保存，不与留出测试混合。
- **GNN**：基础原子特征是 10 维元素 one-hot；增强特征加入 6 维杂化和 5 项原子属性，合计 21 维。每个键建立双向边，使用单键、双键、三键、芳香键的 4 维特征。两层含键特征的消息传递，隐藏维度 32，分子内求和池化后回归 logS，基于原生 PyTorch 实现。
- **GNN 训练**：固定划分 721/181/226（训练/验证/测试），两步划分种子均为 42。仅用训练标签标准化，Adam 学习率 0.001、权重衰减 1e-4、batch 32、最多 100 epochs，验证损失连续 15 epochs 未改善即早停。恢复验证损失最低的模型，测试预测反标准化。保留原有种子 42、123、2026，均使用同一数据划分；CPU 单线程和确定性算法用于收尾复现。

## 最终结果

下表来自整理后三份 Notebook 的实际重跑；未增加模型、参数搜索或随机种子。

| 模型 | 评估方式 | MAE | RMSE | R² |
|---|---|---:|---:|---:|
| 随机森林：描述符 + Morgan | 骨架 80/20 | 0.7061 | 0.9512 | 0.7818 |
| 随机森林：描述符 | 骨架 80/20 | 0.7773 | 1.0338 | 0.7422 |
| 随机森林：Morgan | 骨架 80/20 | 1.2909 | 1.6805 | 0.3190 |
| 线性回归：描述符 | 随机 80/20 | 0.8377 | 1.1052 | 0.7416 |
| 随机森林：描述符 + Morgan | 随机 80/20 | 0.5490 | 0.7959 | 0.8660 |
| 随机森林：描述符 | 随机 80/20 | 0.5863 | 0.8428 | 0.8497 |
| 随机森林：Morgan | 随机 80/20 | 0.8755 | 1.1610 | 0.7148 |
| GNN：Bond-aware | 固定随机划分，3 种子 | 0.6858 ± 0.0176 | 0.8950 ± 0.0159 | 0.8305 ± 0.0060 |
| GNN：Enhanced | 固定随机划分，3 种子 | 0.6398 ± 0.0222 | 0.8398 ± 0.0286 | 0.8507 ± 0.0102 |

收尾复现明确固定了 CPU 单线程与确定性算法，GNN 数值可能与原 Notebook 的历史输出不同。此处只报告本次完整重跑结果。

`model_comparison.csv` 记录每个模型、划分、种子、样本数、最佳 epoch 与 MAE/RMSE/R²；MAE、RMSE 在原始 logS 标度计算，越低越好，R² 越高越好。GNN 汇总为三个训练种子的均值 ± 样本标准差（ddof=1）。

随机划分与骨架划分的测试样本不同，不能直接当作同一测试集上的模型排名。随机森林与 GNN 的随机测试索引相同，但随机森林使用 902 个训练样本，GNN 使用 721 个训练样本和 181 个验证样本，训练设置也不同。

## 局限性

数据量较小，包含 11 条重复的规范化结构记录；随机训练/测试集共有 4 种规范化结构重叠。笔记报告这些情况，沿用既有划分并保留全部数据。模型开发期间曾查看测试集，因此它也不是完全未触碰的外部验证集。骨架划分没有单独评估无环分子外推；修改后的 GroupKFold 也不能保证无环相似结构隔离。三个 GNN 种子的差异只反映固定划分上的训练随机性，不是不同数据划分的不确定性。

模型仅使用二维分子结构，没有显式建模温度、pH、晶型或其他实验条件，也未做外部数据验证或适用域分析。预测值是计算结果，不代表实验测量；不能据此宣称 GNN 普遍优于传统 QSPR 模型。

## 收尾验证

2026-10-10，在现有 `chem-ai` 环境中为每份笔记启动全新 Jupyter 内核，全部代码单元按顺序执行完成。

| Notebook | 已执行代码单元 | 结果 |
|---|---:|---|
| `01_data_exploration.ipynb` | 10 | 通过 |
| `02_qspr_random_forest.ipynb` | 10 | 通过 |
| `03_gnn_training.ipynb` | 12 | 通过 |