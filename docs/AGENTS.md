# Agent 規格書

## 設計原則

不將所有能力塞進單一 Agent。拆分為專責 Agent，各自聚焦、共用同一套 Visual Memory（Figma Library）與 Bridge（MCP），降低單一 Agent 的判斷負擔，也讓每個 Agent 可獨立疊代優化。

---

## Claude Orchestrator（總協調）

**職責**：接收使用者意圖，判斷任務屬於「靜態簡報／圖卡」還是「動畫／短影音」，分派給對應 Agent；必要時同時啟動兩者。

**輸入**：使用者一句話需求（如「做一套企業 AI 課程簡報」）
**輸出**：任務分派指令 + 呼叫 Visual Retrieval Agent

---

## Visual Retrieval Agent

**職責**：任務執行前，先確認 Figma Library 是否已有可用素材，避免每次重新生成造成的風格漂移。

**核心工具**：`search_design_system`

**判斷邏輯**：

| 情況 | 動作 |
|------|------|
| 完全符合需求 | Reuse — 直接引用既有素材 |
| 部分符合，需微調 | Modify — 在既有基礎上修改 |
| 完全無對應素材 | Generate — 呼叫 Figma Make 生成新素材，並歸入 Library |

---

## Claude Presentation Agent

**職責**：教材理解、故事線規劃、投影片架構、版面設計

**處理流程**：
1. 內容結構化（標題／重點／案例／數據拆解）
2. 敘事骨架判定（起承轉合／SOP／清單型／對比型）
3. 呼叫 Visual Retrieval Agent 取得素材
4. 依內容屬性選擇 Visual Skill：

   | 內容屬性 | 對應 Skill |
   |---------|-----------|
   | 學術／醫療／EMBA | `dual-brand-glassmorphism`（藍版） |
   | 自媒體／短影音 | `dual-brand-glassmorphism`（橘版） |
   | AI 概念／趨勢盤點 | `tech-style` |
   | 步驟教學／SOP | `stepclear-tutorial-glass` |
   | 多分類清單／工具盤點 | `penta-glass-listcard` |

5. 透過 `use_figma` 組版寫回 Figma

**輸出**：Figma 投影片檔案 → 匯出 PPTX／PDF／PNG

---

## Claude Motion Agent

**職責**：動畫分鏡、轉場、節奏、字幕、影片輸出

**處理流程**：
1. 接收 Presentation Agent 產出的靜態投影片（或獨立動畫需求）
2. 分鏡拆解：每頁對應的動畫節奏與敘事重點
3. 呼叫 `get_motion_context` 取得既有動畫語言（keyframe／easing）
4. 套用 Motion Skill 定義的轉場規則
5. 產生動畫序列並寫回 Figma

**輸出**：Figma 動畫畫面 → 匯出 MP4／Shorts

---

## QA Agent

**職責**：把關輸出品質，規則式審查（非自由發散）

**Checklist**：見 [`ARCHITECTURE.md`](ARCHITECTURE.md#qa-agent-規則式-checklist)

**動作**：
- 通過 → 進入輸出流程，定稿歸入 OUT 倉庫
- 不通過 → 打回對應 Agent（Presentation 或 Motion）修正，記錄修正原因供學習閉環參考

---

## Agent 間通訊摘要

```text
使用者 → Orchestrator
             │
             ├→ Visual Retrieval Agent → Figma Library
             │
             ├→ Presentation Agent ─┐
             │                      ├→ 共用 Figma Library / MCP / Skills
             └→ Motion Agent ───────┘
                      │
                      ▼
                 QA Agent
                      │
                  輸出 + 回存 Library
```
