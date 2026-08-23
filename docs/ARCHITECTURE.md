# 系統架構（Architecture）

## 分層模型

系統共分五層，由下而上：

```
┌─────────────────────────────────────────────┐
│ Layer 5：輸出層 Output Layer                  │
│ PPTX / PDF / PNG / MP4 / Shorts              │
├─────────────────────────────────────────────┤
│ Layer 4：驗收層 QA Layer                      │
│ QA Agent：語言／品牌／版面／色彩／字體／動畫    │
├─────────────────────────────────────────────┤
│ Layer 3：執行層 Execution Layer               │
│ Claude Presentation Agent / Motion Agent      │
│ ← Visual Skill Layer（設計治理）               │
├─────────────────────────────────────────────┤
│ Layer 2：橋接層 Bridge Layer                  │
│ Figma MCP：讀取（context/variables/search）    │
│            寫入（use_figma/generate_diagram） │
├─────────────────────────────────────────────┤
│ Layer 1：記憶層 Memory Layer                  │
│ Figma Library：素材／版型／元件／品牌資產       │
│ （由 Figma Make 持續補充生成）                 │
└─────────────────────────────────────────────┘
```

每一層對應的職責與工具：

| 層級 | 職責 | 主要工具/機制 |
|------|------|--------------|
| Layer 1 記憶層 | 儲存與結構化素材 | Figma Design / Figma Make / Figma Library |
| Layer 2 橋接層 | 讓 Agent 讀寫 Figma 資料 | Figma MCP（remote server） |
| Layer 3 執行層 | 理解內容、選版型、組版 | Claude Design Agent + Visual Skills |
| Layer 4 驗收層 | 把關輸出品質 | QA Agent（規則式 checklist） |
| Layer 5 輸出層 | 產出最終交付物 | PPTX/PDF/PNG/MP4 匯出引擎 |

---

## 資料流（End-to-End Data Flow）

```text
[使用者意圖]
      │
      ▼
[Claude Orchestrator] ── 判斷任務類型（簡報／圖卡／動畫）
      │
      ▼
[Visual Retrieval Agent]
      │  search_design_system
      ▼
[Figma Library 是否已有可用素材？]
      │
   ┌──┴──┐
  是     否
   │     │
   ▼     ▼
 Reuse  Figma Make 生成缺口素材 → 歸入 Library
   │     │
   └──┬──┘
      ▼
[Claude Design Agent 組版]
      │  get_design_context / get_variable_defs
      ▼
[選擇 Visual Skill]（依內容屬性自動判定）
      │
      ▼
[use_figma 寫回 Figma 畫布]
      │
      ▼
[QA Agent 審核]
      │
   ┌──┴──┐
  通過   不通過 → 打回 Claude Design Agent 修正
   │
   ▼
[輸出 PPTX/PDF/PNG/MP4]
      │
      ▼
[優秀版本回存 Figma Library]（學習閉環）
```

---

## 雙 Agent 架構細節

### Claude Presentation Agent

**輸入**：教材大綱、PDF、課程重點、既有素材連結
**處理**：
1. 內容結構化（拆解為標題／重點／案例／數據）
2. 敘事骨架判定（起承轉合／SOP／清單型／對比型）
3. 呼叫 Visual Retrieval Agent 取得可用素材
4. 呼叫對應 Visual Skill 取得版面骨架
5. 透過 `use_figma` 組版

**輸出**：Figma 投影片檔案 → 匯出 PPTX/PDF/PNG

### Claude Motion Agent

**輸入**：Presentation Agent 產出的靜態投影片、或獨立的動畫需求
**處理**：
1. 分鏡拆解（每頁對應的動畫節奏）
2. 呼叫 `get_motion_context` 取得既有動畫語言（keyframe/easing）
3. 套用 Motion Skill 定義轉場規則
4. 產生動畫序列

**輸出**：Figma 動畫畫面 → 匯出 MP4/Shorts

兩者共用 **同一套 Figma Visual Library**，確保靜態版與動態版的視覺語言一致，不會出現「投影片是藍版、動畫卻變橘版」的品牌斷裂問題。

---

## Visual Retrieval Agent 判斷邏輯

```text
輸入：任務描述 + 內容屬性

1. search_design_system(query) → 搜尋既有 Components/Styles/Variables
2. 逐項判斷：
   - 完全符合需求 → Reuse（直接引用）
   - 部分符合，需微調 → Modify（在既有基礎上修改）
   - 完全沒有對應素材 → Generate（呼叫 Figma Make 生成新素材）
3. 生成的新素材 → 歸入 Figma Library + 對應倉庫編號（MAT/BRD）
```

---

## QA Agent 規則式 Checklist

QA Agent 不是自由發散式審查，而是依固定 checklist 逐項核對：

```yaml
qa_checklist:
  language:
    - 繁體中文用字正確
    - 無簡體字殘留
    - 標點符號全形/半形一致
  brand_consistency:
    - 色票落在所選 Skill 定義範圍內
    - 字體搭配符合規範（中文思源黑體/Noto Sans TC，英文 Montserrat/Inter）
  layout:
    - 對齊網格
    - 留白比例符合 Skill 骨架
    - 資訊階層清楚（一頁一重點原則）
  color:
    - 無超出品牌色票的色彩使用
  typography:
    - 標題/副標/內文層級正確
  image_consistency:
    - 素材風格統一，無拼貼感
  motion_rhythm:
    - 轉場時長符合 Motion Skill 規範
    - easing 曲線一致
```

---

## 技術邊界與已知限制

| 項目 | 說明 |
|------|------|
| `get_variable_defs` | 目前僅支援 Figma Design 檔案，不支援 Figma Make |
| `get_design_context` | 支援 Figma Design 與 Figma Make 兩者 |
| `create_design_system_rules` | 本質是 MCP 提供的 Prompt，用來引導 Agent 產出規則文件，並非直接讀取 Figma 資料的工具，須與 `get_design_context`/`get_variable_defs` 的實際輸出搭配使用 |
| Remote vs Desktop MCP | Remote MCP Server（`https://mcp.figma.com/mcp`）功能範圍最廣，為建議首選；Desktop MCP Server 主要用於企業/組織特定場景 |
