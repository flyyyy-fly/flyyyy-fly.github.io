---
title: "项目数据集部分展示"
date: 2026-02-05
draft: false
---

### 📖 数据集说明
- **研究主题**：数据用于上市公司融资融券与资本结构调整速度研究。
- **数据来源**：CSMAR 数据库、Wind 数据库。
- **样本范围**：2006-2021 年 A 股上市公司。
- **核心逻辑**：通过构建 DID 模型，探究融资融券政策对企业资本结构调整速度的影响。

### 📊 数据集预览 (核心字段)

<details>
<summary><b>🔍 点击展开查看数据样本表格 (Top 5 Rows)</b></summary>

| 证券代码 | 年份 | 实际负债率 | 企业规模 | 盈利能力 | HHI(A) | 行业竞争分组 | Treat | Post | DID |
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
| 000002 | 2021 | 0.7974 | 28.2930 | 0.0292 | 0.1022 | 低竞争组 | 1 | 1 | 1 |
| 000002 | 2020 | 0.8128 | 28.2565 | 0.0454 | 0.1167 | 低竞争组 | 1 | 1 | 1 |
| 000002 | 2019 | 0.8436 | 28.1791 | 0.0476 | 0.1035 | 低竞争组 | 1 | 1 | 1 |
| 000002 | 2018 | 0.8459 | 28.0554 | 0.0481 | 0.0957 | 高竞争组 | 1 | 1 | 1 |
| 000002 | 2017 | 0.8398 | 27.7840 | 0.0457 | 0.1018 | 高竞争组 | 1 | 1 | 1 |

</details>

> **说明**：原始数据集包含 10,000+ 条记录，此处仅展示经过预处理后的核心实证变量。

---
### 🐍 处理此数据的 Python 代码

```python

import pandas as pd
import numpy as np

# ====================== 全局配置 ======================
INPUT_PATH = "./"
OUTPUT_PATH = "./"
ENCODING = "utf-8-sig"

# ====================== 工具函数 ======================
def linear_regression(X, y):
    """手动实现线性回归，避免依赖第三方统计库"""
    X = np.column_stack([np.ones(len(X)), X])
    beta = np.linalg.inv(X.T @ X) @ X.T @ y
    return beta

# ====================== 步骤1：合并原始财务与交易数据 ======================
print("="*60)
print("步骤1：合并原始财务数据与交易数据")
print("="*60)

# 读取FS和BF合并文件
df_fs_bf = pd.read_excel(f"{INPUT_PATH}合并_FS_BF.xlsx", engine="openpyxl")
print(f"FS_BF数据量: {df_fs_bf.shape}")

# 读取交易数据
df_trd = pd.read_excel(f"{INPUT_PATH}TRD_Co.xlsx", engine="openpyxl")
print(f"TRD数据量: {df_trd.shape}")

# 按证券代码左连接
merged_data_1 = pd.merge(df_fs_bf, df_trd, on="证券代码", how="left")
print(f"合并后merged_data_1数据量: {merged_data_1.shape}")

# ====================== 步骤2：合并公司治理数据 ======================
print("\n" + "="*60)
print("步骤2：合并公司治理数据")
print("="*60)

df_chn = pd.read_excel(f"{INPUT_PATH}CHN_Stkmt_ydetails(Merge Query).xlsx", engine="openpyxl")
print(f"CHN数据量: {df_chn.shape}")

# 按证券代码和统计截止日期左连接
merged_data_2 = pd.merge(merged_data_1, df_chn, on=["证券代码", "统计截止日期"], how="left")
print(f"合并后merged_data_2数据量: {merged_data_2.shape}")

# ====================== 步骤3：处理HHI行业集中度数据 ======================
print("\n" + "="*60)
print("步骤3：处理HHI行业集中度数据")
print("="*60)

df_hhi = pd.read_excel(f"{INPUT_PATH}INDFI_HHI.xlsx", engine="openpyxl")
print(f"原始HHI数据量: {df_hhi.shape}")

# 筛选条件
df_hhi = df_hhi[
    (df_hhi["市场类型"] == 21) &
    (df_hhi["是否剔除 ST 或 * ST 股"] == 1) &
    (df_hhi["是否剔除当年新上市，已经退市或被暂停上市的公司"] == 1)
]

# 提取年份
df_hhi["统计截止日期"] = pd.to_datetime(df_hhi["统计截止日期"])
df_hhi["年份"] = df_hhi["统计截止日期"].dt.year

# 行业代码格式化：制造业保留两位，其他保留首字母
def format_industry_code(code):
    code = str(code).strip()
    if code.startswith("C"):
        return code[:2]
    return code[0]

df_hhi["行业代码"] = df_hhi["行业代码"].apply(format_industry_code)

# 保留需要的列
hhi_cleaned = df_hhi[["年份", "行业代码", "HHI (A)"]].rename(columns={"HHI (A)": "HHI_A"})
hhi_cleaned.to_csv(f"{OUTPUT_PATH}hhi_cleaned.csv", encoding=ENCODING, index=False)
print(f"处理后HHI数据量: {hhi_cleaned.shape}")

# ====================== 步骤4：清洗公司财务数据 ======================
print("\n" + "="*60)
print("步骤4：清洗公司财务数据")
print("="*60)

# 剔除金融业
df_clean = merged_data_2[merged_data_2["Indcd"] != "0001"].copy()

# 只保留合并报表
df_clean = df_clean[df_clean["报表类型"] == "A"]

# 提取年份
df_clean["统计截止日期"] = pd.to_datetime(df_clean["统计截止日期"])
df_clean["年份"] = df_clean["统计截止日期"].dt.year

# 构建财务指标
df_clean["实际负债率"] = df_clean["负债合计"] / df_clean["资产总计"]
df_clean["企业规模"] = np.log(df_clean["资产总计"])
df_clean["盈利能力"] = df_clean["息税前利润EBIT"] / df_clean["资产总计"]
df_clean["成长性"] = df_clean["年个股流通市值"] / df_clean["资产总计"]
df_clean["资产担保价值"] = df_clean["固定资产净额"] / df_clean["资产总计"]

# 保存清洗后的公司数据
df_clean.to_excel(f"{OUTPUT_PATH}company_data_cleaned.xlsx", index=False, engine="openpyxl")
print(f"清洗后公司数据量: {df_clean.shape}")

# ====================== 步骤5：合并数据并构建DID模型 ======================
print("\n" + "="*60)
print("步骤5：构建DID模型与行业竞争分组")
print("="*60)

# 合并行业对照表
df_industry_map = pd.read_csv(f"{INPUT_PATH}证券行业代码对照表.csv", encoding=ENCODING)
df_merged = pd.merge(df_clean, df_industry_map, on=["证券代码", "年份"], how="left")

# 合并HHI数据
df_merged = pd.merge(df_merged, hhi_cleaned, on=["年份", "行业代码"], how="left")

# 剔除HHI为空的数据
df_merged = df_merged.dropna(subset=["HHI_A"])

# 构建实验组标签Treat
df_merged["融资融券余额"] = df_merged["融资余额(元)"] + df_merged["融券余额(元)"]
treat_list = df_merged[df_merged["融资融券余额"] > 0]["证券代码"].unique()
df_merged["Treat"] = df_merged["证券代码"].isin(treat_list).astype(int)

# 构建政策执行标签Post
first_year = df_merged[df_merged["融资融券余额"] > 0].groupby("证券代码")["年份"].min().to_dict()
df_merged["Post"] = df_merged.apply(
    lambda x: 1 if (x["Treat"] == 1 and x["年份"] >= first_year.get(x["证券代码"], 9999)) else 0,
    axis=1
)

# 核心交互项
df_merged["DID"] = df_merged["Treat"] * df_merged["Post"]

# 行业竞争分组
year_median = df_merged.groupby("年份")["HHI_A"].median().to_dict()
df_merged["行业竞争分组"] = df_merged.apply(
    lambda x: "高竞争组" if x["HHI_A"] < year_median[x["年份"]] else "低竞争组",
    axis=1
)

# 保存实证分析基础库
df_merged.to_csv(f"{OUTPUT_PATH}实证分析基础库2.csv", encoding=ENCODING, index=False)
print(f"实证分析基础库数据量: {df_merged.shape}")

# ====================== 步骤6：测算资本结构调整速度 ======================
print("\n" + "="*60)
print("步骤6：测算资本结构调整速度")
print("="*60)

# 准备回归数据
required_cols = ["证券代码", "年份", "实际负债率", "企业规模", "盈利能力", "成长性",
                "第一大股东持股比率(%)", "资产担保价值", "独立董事占比"]
df_reg = df_merged[required_cols].dropna()

# 转换数值类型
for col in required_cols[2:]:
    df_reg[col] = pd.to_numeric(df_reg[col], errors="coerce")
df_reg = df_reg.dropna()

# 计算目标负债率（按公司个体回归）
results = []
for code in df_reg["证券代码"].unique():
    company_data = df_reg[df_reg["证券代码"] == code].copy()
    if len(company_data) < 3:
        continue
    
    X = company_data[["企业规模", "盈利能力", "成长性", "第一大股东持股比率(%)",
                    "资产担保价值", "独立董事占比"]].values
    y = company_data["实际负债率"].values
    
    try:
        beta = linear_regression(X, y)
        X_with_const = np.column_stack([np.ones(len(X)), X])
        company_data["目标负债率"] = X_with_const @ beta
        results.append(company_data)
    except:
        continue

df_with_target = pd.concat(results)

# 计算调整状态指标
df_with_target = df_with_target.sort_values(["证券代码", "年份"])
df_with_target["上期实际负债率"] = df_with_target.groupby("证券代码")["实际负债率"].shift(1)
df_with_target["上期目标负债率"] = df_with_target.groupby("证券代码")["目标负债率"].shift(1)
df_with_target["负债率变动"] = df_with_target["实际负债率"] - df_with_target["上期实际负债率"]
df_with_target["偏离程度"] = df_with_target["上期目标负债率"] - df_with_target["上期实际负债率"]

# 剔除空值
df_reg_final = df_with_target.dropna(subset=["负债率变动", "偏离程度"])

# 回归测算调整速度
X = df_reg_final["偏离程度"].values.reshape(-1,1)
y = df_reg_final["负债率变动"].values
beta = linear_regression(X, y)

print(f"\n资本结构调整速度测算结果:")
print(f"调整速度系数: {beta[1]:.4f}")
print(f"截距项: {beta[0]:.4f}")

# 保存最终回归数据
df_reg_final.to_csv(f"{OUTPUT_PATH}最终回归数据.csv", encoding=ENCODING, index=False)
print(f"\n最终回归数据量: {df_reg_final.shape}")
print("="*60)
print("所有流程执行完成！")
