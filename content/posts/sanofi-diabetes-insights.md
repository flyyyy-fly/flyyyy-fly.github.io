---
title: "赛诺菲 (Sanofi) 小红书糖尿病人群洞察研究"
date: 2026-02-13
tags: ["数据分析", "AI打标", "人群画像分析", "医疗行业"]
categories: ["实习项目"]
draft: false
---

## 项目背景
本次项目甲方为 **赛诺菲 (Sanofi)**，核心任务是为其出具《小红书糖尿病人群洞察研究报告》。数据源涵盖聚光、灵犀平台及小红书 M300 笔记评论，2025 年圈包人数达 **12,748,995**。

本人核心负责 **AI 辅助打标、用户圈群画像分析、各阶段热点趋势洞察**，由于商业原因代码不开源，但在博文中展示核心逻辑。

---

## 核心工作流：从原始数据到决策洞察

<div style="background-color: #f8f9fa; border-radius: 10px; padding: 20px; border: 1px solid #e9ecef; margin-bottom: 25px;">
  <div style="display: flex; justify-content: space-around; align-items: center; flex-wrap: wrap; text-align: center; font-size: 0.9em;">
    <div style="flex: 1; min-width: 120px;">
      <div style="font-weight: bold; color: #005a8b;">阶段一：AI 基础打标</div>
      <p style="color: #666;">人群划分 (4类)</p>
    </div>
    <div style="font-size: 1.5em; color: #ccc;">➔</div>
    <div style="flex: 1; min-width: 120px;">
      <div style="font-weight: bold; color: #005a8b;">画像分析与优化</div>
      <p style="color: #666;">分类调整 (4类➔6类)</p>
    </div>
    <div style="font-size: 1.5em; color: #ccc;">➔</div>
    <div style="flex: 1; min-width: 120px;">
      <div style="font-weight: bold; color: #005a8b;">阶段二：AI 精细打标</div>
      <p style="color: #666;">行为细化拆解</p>
    </div>
    <div style="font-size: 1.5em; color: #ccc;">➔</div>
    <div style="flex: 1; min-width: 120px;">
      <div style="font-weight: bold; color: #005a8b;">竞品专项调研</div>
      <p style="color: #666;">诺和锐表现分析</p>
    </div>
  </div>
</div>

---

## 第一阶段：AI 大模型辅助打标（核心负责）

在本项目中，我负责构建了两阶段的 AI 自动标注流，严格遵循医疗业务语义规则，确保千万级数据的处理精度。

### 1. 人群分类颗粒度优化 (4类 ➔ 6类)
**挑战**：初始 4 类分类导致“风险觉察”与“长期维持”人群占比过大，颗粒度不足。
**优化**：完善为 **6 类（风险觉察、确诊胰岛素抵抗、确诊期、胰岛素实操、口服药+饮食管理、长期维持）**。

#### 阶段一：人群分类逻辑实现
```python
import os
import json
import pandas as pd
import re
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from openai import OpenAI
from tqdm import tqdm

# ================= 🟢 配置区域 (请在此修改) =================
# 输入和输出文件路径
INPUT_EXCEL_PATH = r""  # 🟢 请修改为你的输入文件路径
OUTPUT_FILE = r"" # 🟢 请修改为你的输出文件路径（建议与输入文件不同）

# API配置
API_KEY = ""  # 🟢 注意发给AI前先删掉这个
BASE_URL = "https://ark.cn-beijing.volces.com/api/v3"  # 🟢 请根据需要修改
MODEL_ID = "ep-20250530200846-hdjdk" # 🟢 请根据需要修改

# Excel中存储关键词的列名
KEYWORD_COL = "关键词"      # 🟢 请确保这个列名在你的Excel文件中存在

# 性能配置
BATCH_SIZE = 10           # 批处理大小
MAX_WORKERS = 10          # 并发线程数（根据API限制调整）
# =========================================================

# 初始化OpenAI客户端
client = OpenAI(api_key=API_KEY, base_url=BASE_URL, timeout=600.0)

# 定义系统提示 (System Prompt)
SYSTEM_PROMPT = """
Role
你是一位资深的糖尿病健康管理专家，擅长根据患者在不同病程阶段的关注点和行为，对关键词进行精准分类。
Context
任务目标：根据给定的"关键词"，判断其与糖尿病疾病病程的关联阶段。
核心逻辑：严格按照8个分类标准进行判断，特别注意各阶段的核心定义和特殊规则。
Action
请分析输入的每一条关键词，根据以下判断标准进行分类，并以JSON List格式输出结果。
判断标准 (Judgment Criteria)
风险觉察期
核心定义：用户尚未确诊（或不确定是否确诊），处于对身体异常的怀疑、症状排查、询问前兆的阶段。
关键词特征：症状（痒、泡沫尿、口干、黑棘皮）、前兆、是不是、怀疑、自我自测。
特殊规则：
凡是询问"症状"、"前兆"、"怎么检查"、"怎么自测"、"会不会得"、"是不是"等不确定是否患病的词条
特别注意：询问"胰岛素抵抗的症状"属于此类（因为还在对照症状，尚未确诊）
胰岛素抵抗相关的检查方法、科室查询（如"胰岛素抵抗怎么查"、"胰岛素抵抗挂什么科"）也属于此类
例子：
"脖子黑是糖尿病的前兆吗" - 症状怀疑
"糖尿病早期症状" - 前兆了解
"胰岛素抵抗有什么症状" - 症状对照
"胰岛素抵抗怎么检查" - 检查方法询问
确诊胰岛素抵抗期
核心定义：用户已明确关注"胰岛素抵抗"这一具体诊断，咨询后续的报告解读、病情发展、调理方法或药物治疗。
关键词特征：胰岛素抵抗（已确诊语境）、报告解读、数值分析、怎么调理、逆转、多久会得糖尿病、吃什么药、对怀孕影响。
特殊规则：
必须基于"已明确关注胰岛素抵抗"的语境
涉及胰岛素抵抗的"治疗"、"用药"、"预后"、"对特定情况的影响"归此类
胰岛素抵抗的"饮食食谱"归口服药+饮食管理类
例子：
"胰岛素抵抗怎么调理" - 已确诊后的调理
"胰岛素抵抗报告怎么看" - 报告解读
"胰岛素抵抗能逆转吗" - 已知IR的预后
"胰岛素抵抗吃什么药" - 确诊后的治疗
"胰岛素抵抗对怀孕的影响" - 确诊后的影响
确诊期
核心定义：用户关注糖尿病的诊断标准、疾病定性、分型区别、病因原理等基本认知。
关键词特征：诊断标准、正常范围、能治愈吗、一型二型区别、糖化血红蛋白、遗传、病因、为什么会得。
特殊规则：
侧重于对"糖尿病"这一疾病本身的定性和认知，而非具体治疗操作
所有关于"是否可以逆转/治愈"的问题归此类
例子：
"糖尿病诊断标准" - 诊断依据
"二型糖尿病能治好吗" - 疾病预后认知
"糖尿病遗传吗" - 病因了解
"糖尿病前期可以逆转吗" - 预后询问
口服药+饮食管理
核心定义：糖尿病常规治疗方案，涵盖口服药物、日常饮食咨询及相关的自我管理工具。
关键词特征：
药物：二甲双胍、吃什么药、降糖药、达格列净等口服药
饮食：能吃吗、食谱、水果、饮料、主食、可以吃...吗
工具：血糖仪、测糖等自我监测工具
生活：送礼推荐、生活建议等
特殊规则：
所有关于"吃什么/能不能吃"的饮食问题，全部归入此类
即使有胰岛素抵抗前缀，若重点在"吃什么"，归此类
自我监测工具（血糖仪、测糖）归此类
糖尿病患者的生活建议、送礼等归此类
例子：
"糖尿病可以吃苹果吗" - 饮食咨询
"二甲双胍" - 口服药物
"糖尿病人早餐吃什么" - 饮食管理
"血糖仪" - 自我管理工具
"糖尿病人送礼推荐" - 生活建议
胰岛素实操
核心定义：涉及胰岛素注射的具体操作、工具、剂量换算及使用问题。
关键词特征：胰岛素（单独出现）、怎么打、针头、单位换算、泵、冷藏、笔芯、怎么安装、多少钱一针。
特殊规则：
关注药物本身的保存、注射动作、剂量调整或工具使用
具体胰岛素品种名称（如门冬胰岛素、甘精胰岛素）归此类
例子：
"胰岛素怎么打" - 注射操作
"胰岛素针头多久换一次" - 工具使用
"胰岛素单位换算" - 剂量计算
"门冬胰岛素" - 具体胰岛素品种
长期维持期
核心定义：疾病进入深水区，涉及并发症、长期管理及生活质量影响。
关键词特征：
并发症：肾病、糖尿病足、眼底病变、神经病变、酮症酸中毒等
预后：能活多久、寿命影响
严重症状表现：早期症状图片、真实图片等
特殊规则：
所有并发症相关词汇归入此类
疾病严重程度、预后问题归此类
具体的症状表现（如真实图片）归此类
例子：
"糖尿病肾病" - 并发症
"2型糖尿病能活多久" - 预后问题
"糖尿病足早期症状真实图片" - 症状表现
"糖尿病酮症酸中毒表现" - 急性并发症
无关
核心定义：非糖尿病内容、纯减肥、医美、兽医（猫/狗）或无意义词汇。
特殊规则：
明确提到"猫"、"狗"、"减肥"（非糖尿病相关）等关键词
与糖尿病完全无关的内容
例子：
"狗狗糖尿病" - 兽医内容
"减肥针" - 非糖尿病治疗
"甲状腺危象" - 其他疾病
暂时无法判断
核心定义：仅出现 GLP-1 类药物全名（如司美格鲁肽、替尔泊汰），无法确定是减肥还是治病。
特殊规则：
只有明确的GLP-1药物全名才归此类
其他情况尽量归入已有类别
例子：
"司美格鲁肽" - GLP-1药物
"替尔泊肽" - GLP-1药物
Format
输出必须严格遵守以下 JSON List 格式，不要包含Markdown代码块标记（```）以外的任何解释性文字。
[
{
"keyword": "对应输入的关键词原文",
"tag": "风险觉察期 或 确诊胰岛素抵抗期 或 确诊期 或 口服药+饮食管理 或 胰岛素实操 或 长期维持期 或 无关 或 暂时无法判断"
},
...
]
关键调整说明：
将胰岛素抵抗的"检查方法"、"科室查询"明确归入风险觉察期
将胰岛素抵抗的"治疗用药"明确归入确诊胰岛素抵抗期
将胰岛素抵抗的"饮食食谱"明确归入口服药+饮食管理
将"是否可以逆转/治愈"的问题归入确诊期
将自我管理工具（血糖仪等）归入口服药+饮食管理
将并发症症状表现归入长期维持期
将生活建议类归入口服药+饮食管理
"""

def clean_json_string(s):
    """一个健壮的函数，用于从API返回的字符串中解析出JSON。"""
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        # 尝试从Markdown代码块中提取
        match = re.search(r"```json(.*?)```", s, re.DOTALL)
        if match:
            s = match.group(1)
        else:
            # 尝试直接匹配列表结构
            match_list = re.search(r"\[.*\]", s, re.DOTALL)
            if match_list:
                s = match_list.group(0)
        try:
            return json.loads(s.strip())
        except json.JSONDecodeError:
            # 作为最后的手段，返回一个空列表
            print(f"⚠️ JSON 解析失败，原始字符串: {s[:200]}...")
            return []

def process_batch(batch_df):
    """处理单个批次的数据，发送给API并返回结果。"""
    data_payload = []
    input_map = {}

    for original_idx, row in batch_df.iterrows():
        uid = str(original_idx)
        # 获取关键词
        keyword = str(row.get(KEYWORD_COL, "")).strip()
        if keyword:  # 仅处理非空关键词
            data_payload.append({"keyword": keyword})
            input_map[uid] = original_idx

    if not data_payload:
        return []

    try:
        response = client.chat.completions.create(
            model=MODEL_ID,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": json.dumps(data_payload, ensure_ascii=False)}
            ],
            temperature=0.0,
        )
        
        parsed_data = clean_json_string(response.choices[0].message.content)
        results = []
        if isinstance(parsed_data, list):
            for item in parsed_data:
                item_keyword = item.get("keyword", "")
                # 通过关键词匹配找到原始索引
                for uid, original_idx in input_map.items():
                    if batch_df.loc[original_idx, KEYWORD_COL] == item_keyword:
                        results.append({
                            'tag': item.get('tag', '暂时无法判断'),
                            'original_index': original_idx
                        })
                        break
        return results
    except Exception as e:
        print(f"⚠️ 批处理时发生错误: {e}")
        return []

def main():
    """主函数，负责读取、处理和保存文件。"""
    print(f"🚀 开始读取文件: {INPUT_EXCEL_PATH}")
    if not os.path.exists(INPUT_EXCEL_PATH):
        print(f"❌ 错误：输入文件不存在于路径 {INPUT_EXCEL_PATH}")
        return

    try:
        full_df = pd.read_excel(INPUT_EXCEL_PATH).fillna("")
    except Exception as e:
        print(f"❌ 错误：读取Excel文件失败，请检查文件是否损坏或格式是否正确。错误详情: {e}")
        return
    
    if KEYWORD_COL not in full_df.columns:
        print(f"❌ 错误：在Excel文件中找不到指定的列名【{KEYWORD_COL}】")
        print(f"👉 文件中包含的列有: {list(full_df.columns)}")
        return

    # 创建批次
    batches = [full_df[i:i + BATCH_SIZE] for i in range(0, len(full_df), BATCH_SIZE)]
    final_results = []
    
    print(f"⚡️ 开始分析... 共 {len(full_df)} 条关键词，分为 {len(batches)} 个批次。")
    
    # 使用线程池并发处理
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        # 提交所有任务
        futures = {executor.submit(process_batch, batch): batch for batch in batches}
        # 使用tqdm显示进度条
        for future in tqdm(as_completed(futures), total=len(batches), desc="打标进度"):
            if (result := future.result()):
                final_results.extend(result)

    if final_results:
        # 创建结果DataFrame
        result_df = pd.DataFrame(final_results)
        result_df.set_index('original_index', inplace=True)
        
        # 确保索引类型一致以便合并
        full_df.index = full_df.index.astype(int)
        result_df.index = result_df.index.astype(int)

        # 定义新列的名称 - 修正这里
        result_df.rename(columns={'tag': '糖尿病病程阶段'}, inplace=True)
        
        # 删除可能已存在的旧结果列，以避免冲突
        if '糖尿病病程阶段' in full_df.columns:
            full_df.drop(columns=['糖尿病病程阶段'], inplace=True, errors='ignore')
        
        # 将新生成的结果合并回原始DataFrame
        final_df = full_df.join(result_df)

        try:
            final_df.to_excel(OUTPUT_FILE, index=False)
            print(f"\n🎉 成功！结果已保存至: {OUTPUT_FILE}")
            print("\n🔍 结果预览:")
            # 打印包含原始关键词和新标签的预览
            preview_cols = [KEYWORD_COL, '糖尿病病程阶段']
            if '糖尿病病程阶段' in final_df.columns:
                print(final_df[preview_cols].head(10))
            else:
                print("警告：结果列未成功添加到DataFrame中")
        except Exception as e:
            print(f"❌ 错误：保存结果到Excel文件失败。请检查文件是否被其他程序占用。错误详情: {e}")
    else:
        print("⚠️ 处理完成，但没有生成任何结果。请检查API连接或输入数据是否为空。")

if __name__ == "__main__":
    main()



```

### 2. 人群行为精细化拆解
在确定 6 类人群后，我利用 AI 对每一类人群的行为总结出 **4 个核心方向**（如风险觉察期的：症状识别、风险自绘、就医关注等），支撑后续精细化报告撰写。

#### 阶段二：精细化行为标注实现
```python
import os
import json
import pandas as pd
import re
import time
from concurrent.futures import ThreadPoolExecutor, as_completed
from openai import OpenAI
from tqdm import tqdm

# ================= 🟢 配置区域 (请在此修改) =================
# 输入和输出文件路径
INPUT_EXCEL_PATH = r""  # 🟢 请修改为你的输入文件路径
OUTPUT_FILE = r""  # 🟢 输出文件路径，避免覆盖原文件

# API配置
API_KEY = ""  # 🟢 注意发给AI前先删掉这个
BASE_URL = "https://ark.cn-beijing.volces.com/api/v3"  # 🟢 请根据需要修改
MODEL_ID = "ep-20250530200846-hdjdk"  # 🟢 请根据需要修改

# Excel中存储关键词的列名
KEYWORD_COL = "搜索关键词"  # 🟢 请确保这个列名在你的Excel文件中存在
STAGE_COL = "糖尿病病程阶段"  # 🟢 已存在阶段的列名

# 性能配置
BATCH_SIZE = 10  # 批处理大小
MAX_WORKERS = 5  # 降低并发数，提高稳定性
# =========================================================

# 初始化OpenAI客户端
client = OpenAI(api_key=API_KEY, base_url=BASE_URL, timeout=600.0)

# 各阶段四字标签定义
STAGE_CONTENT_TAGS = {
    "风险觉察期": ["抵抗自查", "检查诊断", "症状识别", "其他"],
    "确诊期": ["诊断标准", "分型认知", "指标解读", "其他"],
    "确诊胰岛素抵抗期": ["抵抗概念", "抵抗确诊", "生活影响", "其他"],
    "口服药+饮食管理": ["饮食规划", "监测控糖", "用药指导", "其他"],
    "胰岛素实操": ["注射技术", "药物认知", "成本管理", "其他"],
    "长期维持期": ["并发症状", "治疗应对", "生活质量", "其他"]
    # 注意：已排除"无关"和"暂时无法判断"两个阶段
}

# 标签说明映射（用于prompt中的详细说明）
TAG_DESCRIPTIONS = {
    # 风险觉察期
    "抵抗自查": "高度检索胰岛素抵抗相关症状及风险自查，特指热词中明确包含'胰岛素抵抗'",
    "检查诊断": "关注医院检查项目、科室挂什么科、如何检查等",
    "症状识别": "检索糖尿病早期症状，对照自身情况进行自查（若热词中仅包含'胰岛素'而不含'胰岛素抵抗'，归为此类）",
    # 确诊期
    "诊断标准": "检索血糖正常值范围及糖尿病诊断标准",
    "分型认知": "了解糖尿病类型（一型、二型）的区别及病因",
    "指标解读": "关注糖化血红蛋白、空腹血糖等关键检测指标的意义",
    # 确诊胰岛素抵抗期
    "抵抗概念": "深入了解胰岛素抵抗的概念、改善方法及意义",
    "抵抗确诊": "关注胰岛素抵抗相关的检查项目、确诊标准及报告解读",
    "生活影响": "检索胰岛素抵抗对生活（如怀孕、减重）的影响及管理",
    # 口服药+饮食管理
    "饮食规划": "关注糖尿病食谱、饮食管理方案及食物选择",
    "监测控糖": "检索血糖监测工具（血糖仪）及日常控糖方法",
    "用药指导": "了解常用口服药物（如二甲双胍）及低血糖预防",
    # 胰岛素实操
    "注射技术": "学习胰岛素注射方法、操作技巧及部位选择",
    "药物认知": "了解不同类型胰岛素（门冬、甘精等）及使用设备",
    "成本管理": "关注胰岛素使用成本、价格及费用相关问题",
    # 长期维持期
    "并发症状": "关注糖尿病并发症（如糖尿病足、肾病等）的早期症状",
    "治疗应对": "检索透析、截肢等严重并发症的治疗手段及防治方法",
    "生活质量": "关注并发症对日常生活、寿命、生活质量的影响"
}

# 定义系统提示 (System Prompt) - 严格版本
SYSTEM_PROMPT = """你是一位专业的医疗健康数据分类专家，专门对糖尿病相关搜索关键词进行精准内容分类。

任务说明：
根据给定的搜索关键词及其所属的病程阶段标签，为该关键词打上更具体的内容标签。每个阶段有且只有4个标签选项：3个主要内容标签 + 1个"其他"标签。

重要规则：
1. 严格遵循阶段限制：每个关键词只能使用其所属阶段对应的标签
2. 强制其他选项：如果不能明确匹配三个主要内容之一，必须选择"其他"
3. 严禁跨阶段使用标签：如确诊期的关键词不能使用确诊胰岛素抵抗期的标签
4. 严禁使用图片中出现的错误标签：如确诊期不能使用"概念理解"、"生活影响"、"症状自查"等

各阶段标签定义（必须从以下选项中选择）：

一、风险觉察期（4选1）：
- 抵抗自查：高度检索胰岛素抵抗相关症状及风险自查，特指热词中明确包含"胰岛素抵抗"
- 检查诊断：关注医院检查项目、科室挂什么科、如何检查等
- 症状识别：检索糖尿病早期症状，对照自身情况进行自查（若热词中仅包含"胰岛素"而不含"胰岛素抵抗"，归为此类）
- 其他：不属于以上三类的风险觉察期内容

二、确诊期（4选1）：
- 诊断标准：检索血糖正常值范围及糖尿病诊断标准
- 分型认知：了解糖尿病类型（一型、二型）的区别及病因
- 指标解读：关注糖化血红蛋白、空腹血糖等关键检测指标的意义
- 其他：不属于以上三类的确诊期内容

三、确诊胰岛素抵抗期（4选1）：
- 抵抗概念：深入了解胰岛素抵抗的概念、改善方法及意义
- 抵抗确诊：关注胰岛素抵抗相关的检查项目、确诊标准及报告解读
- 生活影响：检索胰岛素抵抗对生活（如怀孕、减重）的影响及管理
- 其他：不属于以上三类的确诊胰岛素抵抗期内容

四、口服药+饮食管理（4选1）：
- 饮食规划：关注糖尿病食谱、饮食管理方案及食物选择
- 监测控糖：检索血糖监测工具（血糖仪）及日常控糖方法
- 用药指导：了解常用口服药物（如二甲双胍）及低血糖预防
- 其他：不属于以上三类的口服药+饮食管理内容

五、胰岛素实操（4选1）：
- 注射技术：学习胰岛素注射方法、操作技巧及部位选择
- 药物认知：了解不同类型胰岛素（门冬、甘精等）及使用设备
- 成本管理：关注胰岛素使用成本、价格及费用相关问题
- 其他：不属于以上三类的胰岛素实操内容

六、长期维持期（4选1）：
- 并发症状：关注糖尿病并发症（如糖尿病足、肾病等）的早期症状
- 治疗应对：检索透析、截肢等严重并发症的治疗手段及防治方法
- 生活质量：关注并发症对日常生活、寿命、生活质量的影响
- 其他：不属于以上三类的长期维持期内容

判断流程：
1. 确认关键词已有的"糖尿病病程阶段"标签
2. 查看该阶段对应的4个内容标签选项
3. 严格从这4个选项中选一个最匹配的
4. 如果匹配不明确或不符，选择"其他"
5. 严禁使用其他阶段的任何标签

输出格式：
严格按以下JSON格式输出：
[
  {
    "keyword": "关键词原文",
    "stage": "已有阶段标签",
    "content_tag": "内容标签（必须是该阶段4个选项之一）"
  }
]

特别强调：确诊期的关键词不能使用"概念理解"、"生活影响"等标签，这些属于其他阶段！
"""

def validate_content_tag(stage, content_tag):
    """严格验证内容标签是否属于指定的阶段"""
    if stage not in STAGE_CONTENT_TAGS:
        return "其他"
    
    valid_tags = STAGE_CONTENT_TAGS[stage]
    if content_tag in valid_tags:
        return content_tag
    else:
        # 尝试修正可能的标签错误
        for tag in valid_tags:
            if tag in content_tag or content_tag in tag:
                return tag
        return "其他"

def clean_json_string(s):
    """从API返回的字符串中解析出JSON"""
    try:
        return json.loads(s)
    except json.JSONDecodeError:
        # 尝试从Markdown代码块中提取
        match = re.search(r"```json(.*?)```", s, re.DOTALL)
        if match:
            s = match.group(1)
        else:
            # 尝试直接匹配列表结构
            match_list = re.search(r"\[.*\]", s, re.DOTALL)
            if match_list:
                s = match_list.group(0)
        try:
            return json.loads(s.strip())
        except json.JSONDecodeError:
            print(f"⚠️ JSON 解析失败，原始字符串: {s[:200]}...")
            return []

def process_batch(batch_df):
    """处理单个批次的数据"""
    data_payload = []
    input_map = {}
    
    for original_idx, row in batch_df.iterrows():
        uid = str(original_idx)
        keyword = str(row.get(KEYWORD_COL, "")).strip()
        stage = str(row.get(STAGE_COL, "")).strip()
        
        # 只处理有阶段标签的关键词，排除"无关"和"暂时无法判断"
        if keyword and stage and stage != "" and stage in STAGE_CONTENT_TAGS:
            data_payload.append({
                "keyword": keyword,
                "stage": stage
            })
            input_map[uid] = original_idx
    
    if not data_payload:
        return []
    
    try:
        response = client.chat.completions.create(
            model=MODEL_ID,
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": json.dumps(data_payload, ensure_ascii=False)}
            ],
            temperature=0.0,
        )
        
        parsed_data = clean_json_string(response.choices[0].message.content)
        results = []
        if isinstance(parsed_data, list):
            for item in parsed_data:
                item_keyword = item.get("keyword", "")
                item_stage = item.get("stage", "")
                content_tag = item.get("content_tag", "")
                
                # 严格验证和修正标签
                validated_tag = validate_content_tag(item_stage, content_tag)
                
                # 查找匹配的原始索引
                for uid, original_idx in input_map.items():
                    if (batch_df.loc[original_idx, KEYWORD_COL] == item_keyword and 
                        batch_df.loc[original_idx, STAGE_COL] == item_stage):
                        results.append({
                            '内容分类标签': validated_tag,  # 使用中文列名
                            'original_index': original_idx
                        })
                        break
        return results
    except Exception as e:
        print(f"⚠️ 批处理时发生API错误: {e}")
        # 规则回退方法
        return rule_based_fallback(batch_df, input_map)

def rule_based_fallback(batch_df, input_map):
    """规则回退方法"""
    print("使用规则方法进行回退处理...")
    results = []
    
    for uid, original_idx in input_map.items():
        row = batch_df.loc[original_idx]
        keyword = str(row.get(KEYWORD_COL, "")).strip()
        stage = str(row.get(STAGE_COL, "")).strip()
        
        if not keyword or not stage or stage not in STAGE_CONTENT_TAGS:
            continue
        
        validated_tag = "其他"  # 默认值
        
        # 风险觉察期规则
        if stage == "风险觉察期":
            if "胰岛素抵抗" in keyword:
                validated_tag = "抵抗自查"
            elif any(word in keyword for word in ["检查", "挂什么科", "怎么查", "项目", "ogtt", "糖耐", "胰岛素释放", "血清胰岛素"]):
                validated_tag = "检查诊断"
            elif any(word in keyword for word in ["症状", "前兆", "征兆", "表现", "早期", "初期", "口干", "口渴", "尿多", "容易饿"]):
                validated_tag = "症状识别"
        
        # 确诊期规则
        elif stage == "确诊期":
            if any(word in keyword for word in ["诊断标准", "正常值", "标准", "范围"]):
                validated_tag = "诊断标准"
            elif any(word in keyword for word in ["一型", "二型", "型糖尿病", "分型", "1型", "2型"]):
                validated_tag = "分型认知"
            elif any(word in keyword for word in ["糖化血红蛋白", "糖化", "指标", "空腹血糖", "餐后血糖"]):
                validated_tag = "指标解读"
        
        # 确诊胰岛素抵抗期规则
        elif stage == "确诊胰岛素抵抗期":
            if any(word in keyword for word in ["是什么", "什么意思", "概念", "改善", "怎么调理", "能逆转吗"]):
                validated_tag = "抵抗概念"
            elif any(word in keyword for word in ["检查", "项目", "确诊", "标准", "报告怎么看", "数值"]):
                validated_tag = "抵抗确诊"
            elif any(word in keyword for word in ["怀孕", "影响", "生活", "减重", "减肥", "体重"]):
                validated_tag = "生活影响"
        
        # 口服药+饮食管理规则
        elif stage == "口服药+饮食管理":
            if any(word in keyword for word in ["食谱", "吃", "饮食", "食物", "水果", "可以吃", "早餐", "晚餐"]):
                validated_tag = "饮食规划"
            elif any(word in keyword for word in ["血糖仪", "监测", "测糖", "控糖", "血糖"]):
                validated_tag = "监测控糖"
            elif any(word in keyword for word in ["二甲双胍", "药", "口服药", "降糖药", "低血糖", "副作用"]):
                validated_tag = "用药指导"
        
        # 胰岛素实操规则
        elif stage == "胰岛素实操":
            if any(word in keyword for word in ["怎么打", "注射", "针头", "部位", "注射部位", "怎么用"]):
                validated_tag = "注射技术"
            elif any(word in keyword for word in ["门冬", "甘精", "胰岛素泵", "笔芯", "类型", "品种"]):
                validated_tag = "药物认知"
            elif any(word in keyword for word in ["多少钱", "价格", "成本", "费用", "报销", "医保"]):
                validated_tag = "成本管理"
        
        # 长期维持期规则
        elif stage == "长期维持期":
            if any(word in keyword for word in ["症状", "早期症状", "前兆", "表现", "脚麻", "视力模糊", "肾病", "糖尿病足"]):
                validated_tag = "并发症状"
            elif any(word in keyword for word in ["透析", "治疗", "怎么治", "截肢", "手术", "防治"]):
                validated_tag = "治疗应对"
            elif any(word in keyword for word in ["能活多久", "寿命", "生活质量", "影响生活", "日常", "工作"]):
                validated_tag = "生活质量"
        
        results.append({
            '内容分类标签': validated_tag,
            'original_index': original_idx
        })
    
    return results

def main():
    """主函数"""
    print(f"🚀 开始读取文件: {INPUT_EXCEL_PATH}")
    
    if not os.path.exists(INPUT_EXCEL_PATH):
        print(f"❌ 错误：输入文件不存在")
        return
    
    try:
        full_df = pd.read_excel(INPUT_EXCEL_PATH).fillna("")
        print(f"✅ 成功读取文件，共 {len(full_df)} 行数据")
    except Exception as e:
        print(f"❌ 读取Excel文件失败: {e}")
        return
    
    # 检查必要的列是否存在
    required_cols = [KEYWORD_COL, STAGE_COL]
    missing_cols = [col for col in required_cols if col not in full_df.columns]
    if missing_cols:
        print(f"❌ 错误：找不到列名【{', '.join(missing_cols)}】")
        print(f"📋 文件中包含的列: {list(full_df.columns)}")
        return
    
    # 过滤掉"无关"和"暂时无法判断"的数据
    original_count = len(full_df)
    full_df = full_df[full_df[STAGE_COL].isin(STAGE_CONTENT_TAGS.keys())]
    filtered_count = len(full_df)
    
    if filtered_count == 0:
        print(f"⚠️ 警告：文件中没有需要处理的6个阶段数据")
        print(f"   仅处理以下阶段: {list(STAGE_CONTENT_TAGS.keys())}")
        return
    
    print(f"📊 过滤后数据: {filtered_count} 行（原始 {original_count} 行）")
    
    # 创建批次
    batches = [full_df[i:i + BATCH_SIZE] for i in range(0, len(full_df), BATCH_SIZE)]
    print(f"⚡️ 分为 {len(batches)} 个批次处理")
    
    print("\n🔍 各阶段标签对应关系:")
    for stage, tags in STAGE_CONTENT_TAGS.items():
        print(f"  {stage}: {', '.join(tags)}")
    
    final_results = []
    
    # 使用线程池并发处理
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futures = {executor.submit(process_batch, batch): batch for batch in batches}
        
        for future in tqdm(as_completed(futures), total=len(batches), desc="打标进度"):
            result = future.result()
            if result:
                final_results.extend(result)
    
    if final_results:
        print(f"\n✅ 成功处理 {len(final_results)} 条结果")
        
        # 创建结果DataFrame
        result_df = pd.DataFrame(final_results)
        
        if not result_df.empty:
            # 设置索引以便合并
            result_df.set_index('original_index', inplace=True)
            
            # 确保索引类型一致
            full_df.index = full_df.index.astype(int)
            result_df.index = result_df.index.astype(int)
            
            try:
                # 先删除可能存在的重复列
                if '内容分类标签' in full_df.columns:
                    full_df.drop(columns=['内容分类标签'], inplace=True, errors='ignore')
                
                # 使用merge合并，避免列名冲突
                final_df = pd.merge(full_df, result_df[['内容分类标签']], 
                                   left_index=True, right_index=True, 
                                   how='left')
                
                # 填充可能的空值
                final_df['内容分类标签'] = final_df['内容分类标签'].fillna('其他')
                
                # 保存到新文件
                final_df.to_excel(OUTPUT_FILE, index=False)
                print(f"🎉 结果已保存至: {OUTPUT_FILE}")
                
                # 统计结果
                print("\n📊 详细统计（按阶段分组）:")
                for stage in STAGE_CONTENT_TAGS.keys():
                    stage_data = final_df[final_df[STAGE_COL] == stage]
                    if not stage_data.empty:
                        print(f"\n【{stage}】 ({len(stage_data)} 条)")
                        tag_counts = stage_data['内容分类标签'].value_counts()
                        
                        # 按预设顺序显示标签
                        expected_tags = STAGE_CONTENT_TAGS[stage]
                        for tag in expected_tags:
                            if tag in tag_counts:
                                print(f"  ✓ {tag}: {tag_counts[tag]} 条")
                            else:
                                print(f"  - {tag}: 0 条")
                
                print(f"\n🔍 结果预览（前10条）:")
                print(final_df[[KEYWORD_COL, STAGE_COL, '内容分类标签']].head(10))
                    
            except Exception as e:
                print(f"❌ 保存结果失败: {e}")
        else:
            print("⚠️ 结果DataFrame为空")
    else:
        print("⚠️ 处理完成，但没有生成任何结果")

if __name__ == "__main__":
    main()


```

---

## 第二阶段：千万级用户圈群画像分析

基于灵犀平台 **12,748,995** 名圈包用户的海量数据，我主导了画像分类体系的搭建与动态优化：

- **精准捕捉**：通过对每一阶段搜索行为（TOP20 关键词）的聚类分析，洞察不同期用户的行为特征”。。
- **热点洞察**：精准捕捉不同阶段的趋势亮点，为赛诺菲提供了更具针对性的策略建议。

---

## 核心成果与价值产出

<div style="display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0;">
  <div style="border: 1px solid #ddd; padding: 15px; border-radius: 8px;">
    <h4 style="margin-top: 0; color: #005a8b;"> ➔流程闭环</h4>
    <p style="font-size: 0.85em; color: #666;">构建了“数据标注—画像优化—热点洞察”的完整工作流。</p>
  </div>
  <div style="border: 1px solid #ddd; padding: 15px; border-radius: 8px;">
    <h4 style="margin-top: 0; color: #005a8b;"> ➔精准画像</h4>
    <p style="font-size: 0.85em; color: #666;">优化后的 6 类精细化圈群以及二阶段聚类分析，使小红书糖尿病人群特征呈现的准确度大幅提升，显著提高此部分洞察的准确性和精确度。</p>
  </div>
</div>

---

> **项目总结**：
> 本项目不仅是技术层面的 AI 打标应用，更是对医疗垂直行业业务逻辑的深刻拆解。通过不断迭代分类颗粒度，我成功实现了海量碎片化数据到商业洞察报告的价值转化。
