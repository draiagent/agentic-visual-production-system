# Figma MCP 工具規格

> 本表依實際連線的 Figma MCP Server 工具定義核對（2026-08-23）。標註「官方文件頁未列」者，表示工具存在於實際 server 但尚未出現在官方 tools-and-prompts 頁面上。

## 連線方式

| 類型 | 端點 | 適用情境 |
|------|------|---------|
| Remote MCP Server（建議） | `https://mcp.figma.com/mcp` | 一般使用，功能範圍最廣 |
| Desktop MCP Server | 本機 Figma Desktop App 內啟用 | 企業／組織特定場景 |

多數本系統會用到的工具（`search_design_system`、`use_figma`、`create_new_file`、`get_libraries`、`download_assets`、`upload_assets`、`generate_diagram`、`whoami`）為 **remote only**，Desktop server 不提供。這是建議走 remote 的主因。

---

## 檔案類型支援矩陣

Figma 有四種檔案類型，URL 路徑不同，工具支援範圍差異很大。**這是本系統最容易踩到的邊界**：

| 工具 | `/design/` | `/slides/` | `/board/`(FigJam) | `/make/` |
|------|:---:|:---:|:---:|:---:|
| `get_design_context` | ✅ | ✅ | ✅ | ✅（nodeId 固定 `0:1`） |
| `get_variable_defs` | ✅ | ❌ | ❌ | ❌ |
| `get_metadata` | ✅ | ❌ | ❌ | ❌ |
| `get_screenshot` | ✅ | ✅ | ✅ | ❌ |
| `download_assets` | ✅ | ✅ | ✅ | ❌ |
| `upload_assets` | ✅ | ✅ | ✅ | ❌ |
| `use_figma` | ✅ | ✅ | ✅ | ❌ |
| `generate_diagram` | ❌ | ❌ | ✅ | ❌ |

**兩個實務後果**：

1. **Figma Make 幾乎是唯讀的死路** —— 只有 `get_design_context` 能讀它，其餘工具全部不支援。所以流程第 2 步（Figma Make 生成素材）產出的東西，**必須先人工搬進 Figma Design 檔案**，Agent 才有辦法檢索、改寫、匯出。這不是選擇，是硬限制。
2. **Slides 讀不到 Design Token** —— `get_variable_defs` 只吃 `/design/` URL。若簡報直接做在 Figma Slides，色票／字體／間距要從 Design 檔案的 Library 讀，不能指望從 Slides 檔案本身讀出來。

---

## 讀取工具

| 工具 | 說明 |
|------|------|
| `get_design_context` | 讀取 Layout／Component／結構，回傳參考程式碼、截圖與素材下載連結。design-to-code 的主要工具 |
| `get_metadata` | 節點/頁面概觀（僅 node id、layer type、name、position、size）。**官方建議優先用 `get_design_context`**，此工具只在需要先摸清結構時使用。省略 nodeId 時回傳頂層頁面清單 |
| `get_variable_defs` | 讀取 Color／Typography／Spacing 等 Design Token |
| `search_design_system` | 搜尋既有 Components／Styles／Variables，是 Visual Retrieval Agent 的核心工具。**必填 `fileKey`**；可用 `includeLibraryKeys` 限定搜尋範圍 |
| `get_libraries` | 列出檔案「已訂閱」與「可加入」的設計 Library，回傳 library key 供 `search_design_system` 縮小範圍。組織 Library 部分有分頁 |
| `get_screenshot` | 產生節點截圖（PNG）。預設回傳短效 URL + curl 指令而非內嵌圖片，較省 token |
| `download_assets` | 下載單一節點的匯出圖、原始上傳圖片與向量層 SVG。格式可指定 png／jpg／svg／pdf。**PNG 交付物就是走這個工具** |
| `get_motion_context` | 取得動畫 selection 的 keyframe、easing 與可用的 CSS/@keyframes、motion.dev 程式碼片段，以及 timeline 協調提示 |
| `get_context_for_code_connect` | 取得元件結構化屬性/變體資料，用於 Code Connect |
| `whoami` | 取得使用者的 plan 清單（team/organization key）。`create_new_file` 與 `generate_diagram` 需要 `planKey`，通常得先呼叫這個 |

### `search_design_system` 的查詢限制

官方定義明確要求：**每次查詢只能表達一個搜尋意圖，不可把替代方案、同義詞或不相關的搜尋合併在同一句**（此工具不做 OR 語意）。

這對 Visual Retrieval Agent 是設計約束，不是建議：要找「Logo、人物素材、16:9 版型」就得**拆成三次呼叫**，不能一次問完。

---

## 寫入工具

| 工具 | 說明 |
|------|------|
| `use_figma` | 通用寫入工具，透過 JavaScript（Figma Plugin API）建立或修改原生物件、變數、樣式。本系統的組版主力 |
| `generate_figma_design` | 擷取網頁 app 畫面為 Figma 設計。**僅適用於「第一次擷取網頁 app 畫面」**；非網頁（iOS/Android/一般 UI）與從零設計一律用 `use_figma` |
| `create_new_file` | 建立空白 Figma 檔案，`editorType` 可選 `design` / `figjam` / `slides`。需要 `planKey`（先呼叫 `whoami` 取得） |
| `upload_assets` | 上傳圖片與 SVG 進 Figma 檔案。支援 PNG/JPG/GIF/WebP/SVG，單檔上限 10MB。SVG 會被匯入為可編輯的向量節點樹 |
| `generate_diagram` | 用 Mermaid 語法在 FigJam 生成圖表。**自行建立檔案，不要先呼叫 `create_new_file`** |
| `export_video` | 將 Figma timeline 節點算繪為 MP4（官方文件頁未列，實際 server 提供） |

### `use_figma` 的已知地雷

- 字體 `Inter` 的 style 是 `Semi Bold`（有空格）不是 `SemiBold`，`Extra Bold` 不是 `ExtraBold`
- 換頁必須用 `await figma.setCurrentPageAsync(page)`，直接設 `figma.currentPage` 不支援
- `loadAllPagesAsync`、`setPluginData`、`createImageAsync` **不支援**，不要使用

### `generate_diagram` 支援的圖表類型

僅支援 `graph`、`flowchart`、`sequenceDiagram`、`stateDiagram`、`stateDiagram-v2`、`gantt`、`erDiagram`。

**不支援** class diagram、timeline、venn diagram，也不支援改字體或搬移個別形狀 —— 那些要請使用者自己開 Figma 調整。

### `export_video` 的三個硬限制

Motion Agent 的最終輸出靠這個工具，但它有三個必須先知道的限制：

1. **只出 MP4** —— 不支援 GIF、不支援動畫 SVG。
2. **nodeId 必須是擁有 timeline 的「頂層 frame」** —— 在 design 檔案裡是直接放在 page 或 section 上的 frame；在 Slides 裡是**投影片本身**，不是投影片裡的圖層。若動畫掛在子節點上，要往上找到它所屬的頂層 frame（`get_motion_context` 回傳的 `timelineCohorts.rootNodeId` 就是）。
3. **非同步作業，需要輪詢** —— 算繪可能要好幾分鐘。若未在時限內完成，回應會帶 `jobId` 與 `status: "processing"`，需在 10–15 秒後帶 `{ fileKey, jobId }` 再呼叫一次。產出檔案有保存期限（`ttlSeconds`，預設 1 小時），逾期刪除。

---

## Prompt（非工具）

| 名稱 | 說明 |
|------|------|
| `create_design_system_rules` | Figma MCP 提供的 **Prompt**，引導 Agent 綜合 `get_design_context`／`get_variable_defs` 的實際輸出，產出一份可持續遵循的 Design Rules 文件。它不直接讀取 Figma 資料 |

---

## 典型調用順序（Claude Design Agent）

```text
0. whoami                     → 取得 planKey（僅在需要新建檔案時）
1. get_libraries              → 確認可用的 Library，取得 library key
2. search_design_system       → 逐項搜尋既有素材（一次一個意圖，不可合併）
3. get_design_context         → 理解目標節點的版面結構
4. get_variable_defs          → 抓取色票/字體/間距 Token（限 /design/ 檔案）
5. create_design_system_rules → 綜合以上輸出，產生規則文件（→ 歸入 TOOL 倉庫）
6. use_figma                  → 依規則與所選 Skill 寫回設計
7. download_assets            → 匯出 PNG／PDF
   export_video               → 匯出 MP4（Motion Agent）
```

---

## 官方文件

- Figma Developers – Tools and Prompts: https://developers.figma.com/docs/figma-mcp-server/tools-and-prompts/
- Figma Help Center – Guide to the Figma MCP server: https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server
