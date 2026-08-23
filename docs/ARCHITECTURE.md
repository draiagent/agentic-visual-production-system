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
│ Figma MCP：讀取（libraries/search/context/    │
│                  variables/motion）           │
│            寫入（use_figma/upload/diagram）    │
│            匯出（download_assets/export_video）│
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
      │  get_libraries → search_design_system(query, fileKey)
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
[輸出 PPTX/PDF/PNG（download_assets）｜MP4（export_video）]
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

0. get_libraries(fileKey) → 取得可用 Library 的 library key
1. search_design_system(query, fileKey) → 搜尋既有 Components/Styles/Variables
   ⚠ fileKey 為必填；每次查詢只能表達「一個」搜尋意圖，
     不可把 Logo/人物/版型合併成一句，需拆成多次呼叫
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

### 檔案類型支援（最容易踩到的邊界）

| 工具 | `/design/` | `/slides/` | `/board/` | `/make/` |
|------|:---:|:---:|:---:|:---:|
| `get_design_context` | ✅ | ✅ | ✅ | ✅（nodeId 固定 `0:1`） |
| `get_variable_defs` | ✅ | ❌ | ❌ | ❌ |
| `get_metadata` | ✅ | ❌ | ❌ | ❌ |
| `download_assets` / `get_screenshot` | ✅ | ✅ | ✅ | ❌ |
| `use_figma` / `upload_assets` | ✅ | ✅ | ✅ | ❌ |
| `generate_diagram` | ❌ | ❌ | ✅ | ❌ |

**Figma Make 幾乎是唯讀的死路**：只有 `get_design_context` 讀得到它。流程第 2 步用 Figma Make 生成的素材，**必須先搬進 Figma Design 檔案**，Agent 才有辦法檢索、改寫、匯出。這是硬限制，不是流程選擇。

**Slides 讀不到 Design Token**：`get_variable_defs` 只吃 `/design/` URL。簡報若直接做在 Figma Slides，色票／字體／間距得從 Design 檔案的 Library 讀。

### 其他限制

| 項目 | 說明 |
|------|------|
| `search_design_system` | `fileKey` 必填；**每次查詢只能表達一個搜尋意圖**，不做 OR 語意，替代方案需拆成多次呼叫 |
| `create_design_system_rules` | 本質是 MCP 提供的 Prompt，用來引導 Agent 產出規則文件，並非直接讀取 Figma 資料的工具，須與 `get_design_context`／`get_variable_defs` 的實際輸出搭配使用 |
| `export_video` | 只出 MP4，不支援 GIF／動畫 SVG；nodeId 必須是擁有 timeline 的**頂層 frame**（Slides 裡是投影片本身）；非同步作業，未完成會回 `jobId` 需輪詢；產出檔案有保存期限（預設 1 小時） |
| `generate_diagram` | 僅支援 graph／flowchart／sequenceDiagram／stateDiagram／gantt／erDiagram，**不支援** class diagram／timeline／venn；不能改字體或搬移個別形狀 |
| `use_figma` | `Inter` 字重寫法是 `Semi Bold`／`Extra Bold`（有空格）；換頁須用 `await figma.setCurrentPageAsync(page)`；`loadAllPagesAsync`／`setPluginData`／`createImageAsync` 不支援 |
| Remote vs Desktop MCP | 本系統主要工具（`search_design_system`／`use_figma`／`create_new_file`／`get_libraries`／`download_assets`／`upload_assets`）皆為 **remote only**，Desktop server 不提供。Remote（`https://mcp.figma.com/mcp`）為必選 |
