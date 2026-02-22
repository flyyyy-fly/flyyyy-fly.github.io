---
title: "实训项目：Bayestone AI 图文一体化生成平台"
date: 2026-02-22
tags: ["Vue3", "FastAPI", "AIGC", "实习项目"]
categories: ["项目展示"]
draft: false
---

## 🌟 项目背景
在内容营销领域，高效产出符合小红书、抖音风格的“文案+配图”是核心痛点。我在实习期间参与开发的 **Bayestone Article Generator** 实现了从 Excel 素材到最终营销博文的全自动化生产流。

---

## 🛠️ 技术架构

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

## 🚀 核心模块解析

### 1. 批量文案生成引擎 (Vue 3 + SheetJS)
- **数据驱动**：支持从 Excel 导入数千条产品卖点。
- **本地持久化**：集成 **IndexedDB** 存储生成记录，确保浏览器刷新不丢失数据，极大提升了用户编辑大量文案时的安全性。
- **模糊检索**：利用 `Fuse.js` 实现素材库的秒级匹配。

### 2. 生图后端服务 (FastAPI + Python)
- **多步 Prompt 流程**：不直接生图，而是先由 Claude 生成“拍摄场景脚本”（包含光影、构图描述），再传递给生图引擎，显著提升了图片的可控性。
- **图像处理**：支持在生成的 AI 图片上自动叠加文字贴纸，生成符合社交媒体审美的配图。

---

## 📁 项目结构
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
