---
title: "实训项目：Bayestone AI 图文一体化生成平台"
date: 2026-02-22
tags: ["Vue3", "FastAPI", "AIGC", "实习项目"]
categories: ["项目展示"]
draft: false
---

## 项目背景
在内容营销领域，高效产出符合小红书、抖音风格的“文案+配图”是核心痛点。我在实习期间参与完善的 **Bayestone Article Generator** 实现了从 Excel 素材到最终营销图文的全自动化生产流。
> 🔗 **项目开源地址**：[Bayestone Article Generator (GitHub)](https://github.com/ACS1-0/Article-Writing)
> 
> *注：该项目为实习期间参与的商业级 AIGC 落地实践，已在 GitHub 维护。*
---

## 技术架构

<div style="background-color: #f8f9fa; border-radius: 10px; padding: 20px; border: 1px solid #e9ecef;">
  <div style="display: flex; justify-content: space-around; align-items: center; flex-wrap: wrap; text-align: center;">
    <div style="flex: 1; min-width: 150px; margin: 10px;">
      <div style="font-weight: bold; color: #007bff; font-size: 1.2em;">前端展示 (Vue 3)</div>
      <p style="font-size: 0.9em;">Element Plus / Pinia<br>Batch Processing UI</p>
    </div>
    <div style="font-size: 2em; color: #6c757d;">➔</div>
    <div style="flex: 1; min-width: 150px; margin: 10px;">
      <div style="font-weight: bold; color: #28a745; font-size: 1.2em;">逻辑后端 (FastAPI)</div>
      <p style="font-size: 0.9em;">Prompt Engineering<br>Claude / OpenRouter</p>
    </div>
    <div style="font-size: 2em; color: #6c757d;">➔</div>
    <div style="flex: 1; min-width: 150px; margin: 10px;">
      <div style="font-weight: bold; color: #dc3545; font-size: 1.2em;">AI 渲染层 (Coze/SD)</div>
      <p style="font-size: 0.9em;">Image Generation<br>Text Sticker Overlay</p>
    </div>
  </div>
</div>

---

## 深度复盘：技术攻坚与工作流优化

在实习期间，我深入参与了 AI 生成链路的“调优（Tuning）”过程，以下是两个核心优化案例：

### 1. Prompt 提示词工程：解决“情感过载”问题
*   **现象观察**：在文案生成阶段，如果输入的原始产品卖点自带情绪化词汇（如“大吃一惊”、“震撼人心”），LLM 在扩写时会产生**情感反馈循环**，导致生成的最终文案过于夸张、虚假，充满“AI 味”。
*   **调优对策**：我提出了**“原子化去情绪输入”**策略。将初始输入规范化为“一句话的客观场景描述”（例如：将“绝美气质风”改为“清晨阳光洒在木质书桌旁的安静瞬间”），严格限制初期输入中的修饰词。
*   **最终效果**：生成的文案显得含蓄且真实，更符合小红书等社交平台的“素人感”叙事风格。

### 2. 生图工作流优化：从“并行错乱”到“逻辑串行”
*   **技术瓶颈**：原工作流在为文案配图时，采用模特、服装、饰品并行的抽取逻辑。这导致了严重的**视觉元素冲突**（例如：AI 同时抽取了“厚围巾”和“夸张大耳环”，导致颈部画面挤压、风格违和）。
*   **视觉逻辑重构**：我从计算机视觉与审美逻辑出发，提出了两点改进：
    *   **库位重新划分**：重新定义了服装库与饰品库的标签权重，避免风格互斥。
    *   **逻辑链路串行化**：将并行选择改为“模特 -> 服装 -> 匹配饰品”的串行工作流。先确定大面积的服装风格，再由下游节点根据服装领口、材质异步匹配饰品。
*   **成果**：显著降低了生图的“违和感”，成图的专业度和视觉协调性提升了约 30%。

---

## 模型选型与协同：寻找视觉的“去油感”

通过在项目中对主流模型的深度调用，我对不同引擎的边界有了清晰的认知，并在视觉生成上经历了一次核心的技术选型更迭：

### 1. 视觉模型的取舍：从 Seedream 4.0 到 NanoBanana
在项目初期，我们尝试使用 **Seedream 4.0** 作为主要的生图引擎。但在实测中发现两个难以通过 Prompt 解决的顽疾：
- **画面“油腻感”**：AI 渲染痕迹过重，光影过于夸张，不符合社交平台的审美。
- **眼神失真**：无论如何调整提示词，模特在微距视角下的眼神往往散漫且缺乏灵性。

针对这一瓶颈，我主动参与了模型选型调研。通过 **OpenRouter** 接入测试了多款模型，最终通过 **NanoBanana** 实现了突破：
- **效果对比**：NanoBanana 生成的质感更偏向于真实的胶片摄影，有效降低了渲染感。
- **神态捕捉**：其对人物眼神和面部微表情的刻画更加自然，显著提升了图片的“可信度”。

### 2. 文案与逻辑模型的错位协同
除了视觉模型，我在后端链路中也实施了差异化部署：
- **Claude 3.5 Sonnet**：它在处理“去情绪化”指令时表现得极其精准，能敏锐捕捉我设定的“含蓄场景”，生成的文案具有极佳的节奏感和留白，是目前处理社交媒体博文的最佳选择。
- **GPT-4o**：我利用其强大的结构化能力，将其放在工作流的最前端。主要负责将用户凌乱的 Excel 输入拆解为标准的 JSON 节点参数，用于驱动后端的并行任务。

---

## 实习收获与贡献
- **核心贡献**：主导了文案生成的 Prompt 标准化流程，并成功优化了生图工作流的组合逻辑，解决了视觉冲突这一长期痛点。
- **结语**：AIGC 时代的开发，不再仅仅是写代码，更多是**对模型边界的探索**和**对复杂业务流的解构**。通过这个项目，我积累了宝贵的“Prompt 调优”与“AI 架构逻辑”实战经验。

---

## 项目结构
<details>
<summary><b>📂 点击查看详细目录树</b></summary>

```bash
Article Generator/
├── src/                   # 前端源码：Batch Creator 模块
│   ├── configs/           # 核心业务 Prompt 配置
│   ├── stores/            # Pinia 状态流
│   └── views/             # 响应式工作台视图
├── backend/               # 图片生成后端
│   ├── main.py            # FastAPI 入口
│   ├── services/          # 场景脚本生成逻辑
│   └── reference_images/  # AI 训练/参考图库

