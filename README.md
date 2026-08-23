# Agentic Visual Production System

### Figma Visual Memory × Claude Design Agent × MCP × Visual Skills × VAD

一套將 **Figma 素材庫**、**Claude 設計代理**、**Model Context Protocol（MCP）**、視覺技能層（Visual Skills）與 **Visual Agent Design（VAD）方法論**整合為單一生產線的系統架構——用來將教材、大綱、品牌素材自動轉化為投影片、圖卡與動畫影片。

> 本系統是 [VAD（Visual Agent Design）](https://github.com/draiagent/VAD-Promptless) 方法論在「視覺內容生產」場景下的具體執行層。VAD 定義藍圖，本系統負責落地執行。

---

## 目錄

- [核心定位](#核心定位)
- [系統架構總覽](#系統架構總覽)
- [四大支柱](#四大支柱)
- [完整生產流程](#完整生產流程)
- [雙 Agent 分工](#雙-agent-分工)
- [Visual Retrieval：先檢索，再生成](#visual-retrieval先檢索再生成)
- [學習閉環（Learning Loop）](#學習閉環learning-loop)
- [與倉庫編號系統整合](#與倉庫編號系統整合)
- [Figma MCP 工具對照表](#figma-mcp-工具對照表)
- [視覺技能層（Skills）](#視覺技能層skills)
- [QA 驗收標準](#qa-驗收標準)
- [輸出格式](#輸出格式)
- [延伸文件](#延伸文件)
- [版本紀錄](#版本紀錄)

---

## 核心定位

傳統的 AI 生圖／生簡報流程是：

```
Prompt → AI 直接生成
```

這種模式**沒有記憶、沒有品牌一致性、無法重用**。

本系統改為：

```
Intent（意圖）→ Retrieve（檢索既有素材）→ Compose（組版）→ Generate Missing Parts（補生成缺口）→ QA（驗收）
```

四個角色分工明確：

| 角色 | 負責 |
|------|------|
| **Claude** | Creative Intelligence（內容理解、故事線、設計決策） |
| **Figma** | Visual Memory（素材庫、版型庫、品牌資產的長期記憶） |
| **MCP** | Context Bridge（讓 Claude 與 Figma 的資料互通） |
| **Skill** | Design Governance（設計規範的治理與一致性保證） |

---

## 系統架構總覽

```text
素材蒐集
照片／Logo／品牌色／字體／圖示／插圖／版型
        ↓
Figma Make
AI 生成視覺素材與原型
        ↓
整理進 Figma Design / Library
建立正式素材倉庫（Visual Memory）
        ↓
Claude Design / Presentation Agent
        │
        ├─ Figma MCP
        │    ├─ get_design_context        → 理解 Layout／Component／結構（支援 Figma Design + Figma Make）
        │    ├─ get_variable_defs         → 讀取 Color／Typography／Spacing（支援 Figma Design）
        │    ├─ search_design_system      → 搜尋既有 Components／Styles／Variables
        │    └─ create_design_system_rules → 引導 Agent 建立可持續遵循的 Design Rules（Prompt，非讀取工具）
        │
        ↓
Visual Skill Layer
        │
        ├─ tech-style（暗黑科技資訊圖表）
        ├─ stepclear-tutorial-glass（企業教學步驟卡）
        ├─ penta-glass-listcard（五色玻璃多分類清單卡）
        ├─ dual-brand-glassmorphism（雙品牌玻璃質感：藍版學術 / 橘版自媒體）
        ├─ Presentation Skill（簡報骨架與敘事結構）
        └─ Motion Skill（動畫節奏與轉場語言）
        ↓
Claude Design Agent
理解內容 ＋ 選 Skill ＋ 找素材 ＋ 組版
        ↓
Figma
建立／修改：投影片、圖卡、流程圖、動畫畫面
        ↓
QA Agent
繁體中文｜品牌一致性｜版面｜色彩｜字體｜圖像一致性｜動畫節奏
        ↓
輸出
   ┌──────┼────────┬────────┐
   ↓      ↓        ↓        ↓
 PPTX    PDF      PNG      動畫影片 → MP4 / Shorts
```

> 完整技術架構與資料流細節見 [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)。

---

## 四大支柱

| 支柱 | 說明 |
|------|------|
| **Visual Memory（視覺記憶）** | Figma 素材庫不是「靜態資料夾」，而是可被 Agent 搜尋、理解、重用的知識庫 |
| **Context Bridge（脈絡橋接）** | MCP 讓 Claude 即時讀取 Figma 的結構化資料，不需人工匯出匯入 |
| **Design Governance（設計治理）** | Skill 層確保不同素材、不同 Agent 產出的視覺語言始終一致 |
| **Vault Governance（倉庫治理）** | 每個生產階段的產物自動對應到既有的十大倉庫編號系統，不留孤兒檔案 |

---

## 完整生產流程

1. **素材蒐集**：照片、Logo、品牌色、字體、圖示、插圖、既有版型
2. **Figma Make 生成**：針對缺口素材，用 AI 生成視覺素材與原型
3. **歸入 Figma Library**：整理進正式的 Figma Design 檔案，建立結構化素材倉庫
4. **Visual Retrieval Agent 檢索**：接到任務後，先搜尋 Library 而非直接生成（見下節）
5. **Claude Design Agent 組版**：理解內容屬性 → 選擇對應 Skill → 找素材 → 組版
6. **寫回 Figma**：透過 MCP 建立／修改投影片、圖卡、流程圖、動畫畫面
7. **QA Agent 審核**：語言、品牌一致性、版面、色彩、字體、圖像、動畫節奏
8. **輸出**：PPTX／PDF／PNG／動畫影片（MP4/Shorts）
9. **回存 Library**：優秀版本回存 Figma Library，供下次直接檢索重用（學習閉環）

---

## 雙 Agent 分工

不將所有能力塞進單一 Agent，而是拆成兩個專責 Agent，共用同一套 Visual Memory：

```text
                    Figma Visual Library
                           │
                        Figma MCP
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
 Claude Presentation Agent       Claude Motion Agent
             ↓                           ↓
         Slides                     Storyboard
             ↓                           ↓
         PPT / PDF                   Animation
                                         ↓
                                       MP4
```

| Agent | 角色 | 核心產出 |
|------|------|---------|
| **Claude Presentation Agent** | 教材理解、故事線、投影片架構、版面設計 | PPTX / PDF / PNG |
| **Claude Motion Agent** | 動畫分鏡、轉場、節奏、字幕、影片輸出 | Storyboard / MP4 |

兩個 Agent 透過 `get_motion_context`（取得動畫 selection 的 keyframe、easing、motion code context）共享同一套動畫設計語言，確保投影片與動畫版本視覺一致。

---

## Visual Retrieval：先檢索，再生成

這是本系統與傳統「Prompt → 生圖」模式最大的差異。Claude 不應該每次都直接「設計」，而是：

```text
使用者：「做一套企業 AI 課程簡報」
        ↓
Claude Orchestrator
        ↓
Visual Retrieval Agent
        ↓
搜尋 Figma Library（search_design_system）
        ↓
找到：✓ 公司 Logo　✓ 人物素材　✓ 品牌色　✓ AI 科技背景　✓ 16:9 Template　✓ Header Component　✓ Diagram Component
        ↓
判斷：Reuse／Modify／Generate
        ↓
Claude Design Agent → Figma
```

這稱為 **Visual RAG**（Visual Retrieval-Augmented Generation）：

> 不是 `Prompt → AI 生圖`，而是 `Intent → Retrieve → Compose → Generate Missing Parts → QA`

---

## 學習閉環（Learning Loop）

```text
Figma Make 生成
   ↓
歸入 Figma Library
   ↓
Agent 調用
   ↓
產出作品
   ↓
QA 審核
   ↓
優秀版本
   ↓
回存 Library
   ↓
下一次直接檢索重用
```

素材庫的定位會隨時間演進：

**素材資料夾** → **AI 可理解、可搜尋、可重用的 Visual Knowledge Base**

---

## 與倉庫編號系統整合

本系統的每個階段產物，對應到既有的十大倉庫編號系統（詳見 [`docs/VAULT-INTEGRATION.md`](docs/VAULT-INTEGRATION.md)）：

| 生產階段 | 對應倉庫 | 編號格式 |
|---------|---------|---------|
| Figma Make 生成的原始素材／參考素材 | **MAT**（Material Vault） | `MAT-[領域代碼]-[來源簡稱]-[序號]` |
| 品牌資產本體（Logo、色票檔、字體檔） | **BRD**（Brand Asset Vault） | `BRD-[品牌線]-[資產類型]-[序號]` |
| `create_design_system_rules` 產生的規則檔 | **TOOL**（Tool Vault） | `TOOL-[類型]-[簡稱]-[序號]` |
| 套用的視覺技能 | **SKL**（Skill Vault，已編號） | `SKL-[簡稱]-[序號]` |
| 定稿產出（PPTX／PNG／MP4） | **OUT**（Output Vault） | `OUT-[領域代碼]-[標題簡稱]-[序號]` |

---

## Figma MCP 工具對照表

| 工具 | 類型 | 支援範圍 | 用途 |
|------|------|---------|------|
| `get_design_context` | 讀取 | Figma Design + Figma Make | 理解 Layout／Component／結構 |
| `get_variable_defs` | 讀取 | Figma Design | 讀取 Color／Typography／Spacing |
| `search_design_system` | 讀取 | Figma Design | 搜尋既有 Components／Styles／Variables |
| `get_motion_context` | 讀取 | Figma Design | 取得動畫 selection 的 keyframe／easing／motion code |
| `create_design_system_rules` | Prompt（非讀取工具） | — | 引導 Agent 建立可持續遵循的 Design Rules |
| `use_figma` | 寫入 | Figma Design / FigJam / Slides | 建立或修改原生 Figma 物件 |
| `generate_diagram` | 寫入 | FigJam | 用 Mermaid 語法生成流程圖 |

> 完整工具規格與官方文件連結見 [`docs/MCP-TOOLS.md`](docs/MCP-TOOLS.md)。

---

## 視覺技能層（Skills）

| Skill | 適用場景 |
|------|---------|
| `dual-brand-glassmorphism` | EMBA／學術醫療（藍版）、自媒體短影音（橘版）雙品牌玻璃質感系統 |
| `tech-style` | AI 趨勢盤點、暗黑科技風格資訊圖表 |
| `stepclear-tutorial-glass` | 軟體/工具教學、SOP 步驟拆解 |
| `penta-glass-listcard` | 3-5 個平行分類的多分類清單圖卡 |
| `graph-infographic-brand` | 知識點懶人包、吉祥物角色教學圖卡 |
| `premium-commerce-ui-brand` | 電商產品介面、商品頁 |

每個 Skill 都是獨立的設計規範文件，定義色票、字體、版面骨架與輸出格式對應表，供 Claude Design Agent 依內容屬性自動選用。

---

## QA 驗收標準

| 檢核項目 | 說明 |
|---------|------|
| 語言 | 繁體中文用字、標點正確性 |
| 品牌一致性 | 是否符合所選 Skill 的色票／字體／版面規則 |
| 版面 | 對齊、留白、階層是否清楚 |
| 色彩 | 是否落在品牌色票範圍內 |
| 字體 | 中英文字體搭配是否正確 |
| 圖像一致性 | 素材風格是否統一（避免拼貼感） |
| 動畫節奏 | 轉場時長、easing 是否符合 Motion Skill 規範 |

---

## 輸出格式

| 格式 | 用途 |
|------|------|
| **PPTX** | 課程簡報、公開教學 |
| **PDF** | 衛教教材、論文附件 |
| **PNG** | IG／TikTok 圖卡、單張海報 |
| **MP4 / Shorts** | 短影音、動畫版教材 |

---

## 延伸文件

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — 完整技術架構與資料流
- [`docs/AGENTS.md`](docs/AGENTS.md) — 雙 Agent 詳細規格
- [`docs/VAULT-INTEGRATION.md`](docs/VAULT-INTEGRATION.md) — 倉庫編號系統整合細節
- [`docs/MCP-TOOLS.md`](docs/MCP-TOOLS.md) — Figma MCP 工具規格與官方文件連結

---

## 版本紀錄

| 版本 | 日期 | 內容 |
|------|------|------|
| v0.1.0 | 2026-08 | 初版架構定案：Figma Visual Memory × Claude Design Agent × MCP × Visual Skills × VAD |

---

## 授權

[MIT](LICENSE) © 2026 draiagent
