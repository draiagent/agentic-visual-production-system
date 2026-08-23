# 倉庫編號系統整合（Vault Integration）

本系統的每個生產階段，都對應到既有的**十大倉庫編號系統**（MAT/SKL/CRS/SCR/PRJ/TOOL/OUT/PMT/CSE/BRD），確保生成的每一份素材、規則、產出都有明確歸屬，不產生孤兒檔案。

## 對應總表

| Agentic Visual Production System 階段 | 對應倉庫 | 編號格式 | 範例 |
|--------------------------------------|---------|---------|------|
| Figma Make 生成的原始／參考素材 | **MAT**（Material Vault） | `MAT-[領域代碼]-[來源簡稱]-[序號]` | `MAT-AIEDU-SAMS-001` |
| 品牌資產本體（Logo、色票檔、字體檔案） | **BRD**（Brand Asset Vault） | `BRD-[品牌線]-[資產類型]-[序號]` | `BRD-CGMCoach-色票-001` |
| `create_design_system_rules` 產生的規則檔 | **TOOL**（Tool Vault） | `TOOL-[類型]-[簡稱]-[序號]` | `TOOL-Figma規則-EMBA-001` |
| 套用的視覺技能規範 | **SKL**（Skill Vault，已編號） | `SKL-[簡稱]-[序號]` | `SKL-DUAL-001` |
| 進行中的多輪設計修改過程檔 | **PRJ**（Project Vault） | `PRJ-[專案代碼]-[序號]` | `PRJ-AVPS-001` |
| 定稿產出（PPTX/PNG/MP4） | **OUT**（Output Vault） | `OUT-[領域代碼]-[標題簡稱]-[序號]` | `OUT-AIEDU-企業AI課程簡報-001` |

> CRS／SCR／PMT／CSE 四個倉庫視具體內容決定是否涉入本系統（例如課程簡報產出同時歸入 CRS，短影音腳本歸入 SCR）。

---

## 判斷準則

延續既有倉庫系統的判斷邏輯：

- **MAT vs BRD**：外部參考、尚未內化的素材 → MAT；已確認為己方品牌資產本體（可重複使用的 Logo/色票/字體檔） → BRD
- **TOOL vs SKL**：Figma MCP 產生的「這次任務專屬」規則檔 → TOOL；已提煉為跨任務通用、有正式名稱與版本的設計系統 → 升級為 SKL
- **PRJ vs OUT**：仍在多輪修改的過程檔 → PRJ；審核通過的定稿 → OUT

---

## 儲存層對應

- 素材／規則／產出檔案本體 → 依既有規則存放於 **Google Drive Meta-Vault**（`/Meta-Vault/Material-Vault/`、`/Meta-Vault/Tool-Vault/`、`/Meta-Vault/Output-Vault/` 等）
- Figma 內的版型／元件庫 → 存放於 **Figma Library**（Layer 1 記憶層），視為 Meta-Vault 的「即時可調用鏡像」，非取代關係
- 每次由本系統產生新素材/規則/產出時，比照既有流程在倉庫說明書版本紀錄中新增一筆異動
