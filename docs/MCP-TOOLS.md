# Figma MCP 工具規格

## 連線方式

| 類型 | 端點 | 適用情境 |
|------|------|---------|
| Remote MCP Server（建議） | `https://mcp.figma.com/mcp` | 一般使用，功能範圍最廣 |
| Desktop MCP Server | 本機 Figma Desktop App 內啟用 | 企業／組織特定場景 |

---

## 讀取工具

| 工具 | 支援範圍 | 說明 |
|------|---------|------|
| `get_design_context` | Figma Design + Figma Make | 讀取 Layout／Component／結構，回傳參考程式碼、截圖與素材下載連結 |
| `get_metadata` | Figma Design（Desktop） | 取得節點/頁面概觀（僅 node id、layer type、name、position、size） |
| `get_variable_defs` | Figma Design | 讀取 Color／Typography／Spacing 等 Design Token |
| `search_design_system` | Figma Design | 搜尋既有 Components／Styles／Variables，是 Visual Retrieval Agent 的核心工具 |
| `get_context_for_code_connect` | Figma Design | 取得元件結構化屬性/變體資料，用於 Code Connect |
| `get_motion_context` | Figma Design | 取得動畫 selection 的 keyframe、easing、motion code context |

## 寫入工具

| 工具 | 支援範圍 | 說明 |
|------|---------|------|
| `use_figma` | Figma Design / FigJam / Slides | 通用寫入工具，透過 JavaScript（Figma Plugin API）建立或修改原生物件、變數、樣式 |
| `generate_diagram` | FigJam | 用 Mermaid 語法生成流程圖／時序圖／甘特圖等 |

## Prompt（非讀取/寫入工具）

| 名稱 | 說明 |
|------|------|
| `create_design_system_rules` | Figma MCP 提供的 Prompt，引導 Agent 綜合 `get_design_context`／`get_variable_defs` 的實際輸出，產出一份 Agent 可持續遵循的 Design Rules 文件（非直接讀取 Figma 資料） |

---

## 典型調用順序（Claude Design Agent）

```text
1. search_design_system      → 確認 Library 是否已有可用素材
2. get_design_context         → 理解目標節點的版面結構
3. get_variable_defs          → 抓取色票/字體/間距 Token
4. create_design_system_rules → 綜合以上輸出，產生規則文件（→ 歸入 TOOL 倉庫）
5. use_figma                  → 依規則與所選 Skill 寫回設計
```

---

## 官方文件

- Figma Developers – Tools and Prompts: https://developers.figma.com/docs/figma-mcp-server/tools-and-prompts/
- Figma Help Center – Guide to the Figma MCP server: https://help.figma.com/hc/en-us/articles/32132100833559-Guide-to-the-Figma-MCP-server
