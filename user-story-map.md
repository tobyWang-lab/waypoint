以下是整理後的 **WAYPOINT User Story Map**，已標註優先順序：

- **P0 (Critical):** 沒做出來就無法 Demo 的核心功能。
- **P1 (Important):** 黑客松的加分亮點（Differentiator）。
- **P2 (Nice-to-have):** 行有餘力再做，或用 Hard-code 呈現。

---

### Epic 1: Signal Acquisition (訊號攝取與降噪)

**目的：** 建立數據管道，並在第一線過濾掉絕對不需要處理的垃圾。

- **US-1.1 [P0] 授權與定時攝取**
    - **As a** 使用者,
    - **I want** 系統能透過 OAuth 安全連接我的 Gmail，並每隔固定時間（或手動）抓取未讀信件,
    - **So that** 專注於最新資訊，無需手動複製貼上。
    - *(整合原 Story 1.1, US-01, US-02)*
- **US-1.2 [P0] 智慧雜訊過濾 (The Noise Filter)**
    - **As a** 追求效率的使用者,
    - **I want** 系統在進行深度分析前，先將廣告、系統通知 (Notification)、電子報標記為「雜訊 (Noise)」並自動忽略,
    - **So that** 後續的 AI 運算資源只花在真正有價值的信件上，且我的待辦清單不會被垃圾塞滿。
    - *(整合原 User Story 1, US-03)*

---

### Epic 2: Intelligence & Navigation (核心分析與導航)

**目的：** 這是 WAYPOINT 的大腦。利用 LLM 將文字轉化為結構化數據。

- **US-2.1 [P0] 意圖識別與分類 (Intent Classification)**
    - **As a** 使用者,
    - **I want** AI 判斷信件性質是「需行動 (Actionable)」、「僅需知悉 (FYI)」還是「需轉發 (Forward)」,
    - **So that** 我能對症下藥，只將「需行動」的項目轉化為待辦任務。
    - *(整合原 Story 1.2, US-03)*
- **US-2.2 [P0] 關鍵航點提取 (Waypoint Extraction)**
    - **As a** 專案人員,
    - **I want** 系統自動從信件內文提取出「截止日期」、「關鍵對象」與「具體指令」,
    - **So that** 任務內容是具體可執行的，而不是模糊的信件主旨。
    - *(整合原 Story 2.1, US-04, User Story 2)*
- **US-2.3 [P1] 任務工時與優先級預估 (AI Estimation)**
    - **As a** 使用者,
    - **I want** AI 根據任務內容的複雜度與緊急程度，自動標記「優先級 (High/Med/Low)」並預估「建議工時 (e.g., 30 mins)」,
    - **So that** 我能更快速地安排行程，不用自己憑空猜測。
    - *(整合原 US-003, US-05, Story 1.2)*
    - *註：這是展示 LLM 推理能力的絕佳機會。*
- **US-2.4 [P2] 技能匹配與指派建議 (Skill Matching)**
    - **As a** 團隊管理者或多工者,
    - **I want** 系統判斷這項任務是否符合我的技能標籤（如 Backend, Doc, Design）,
    - **So that** 我能快速決定是自己做，還是轉發給別人。
    - *(整合原 US-002, User Story 3)*
    - *建議：Hackathon 中若沒時間做 User Profile，這項可簡化為「判斷是否需轉發」。*

---

### Epic 3: Structure & Synchronization (結構化與同步)

**目的：** 將分析結果輸出到使用者習慣的工作平台。

- **US-3.1 [P0] 生成標準化任務卡片 (Task Generation)**
    - **As a** 使用者,
    - **I want** 每一封 Actionable Mail 都被轉換成一張標準的任務卡片，包含標題（一句話摘要）、截止日、預估工時,
    - **So that** 我可以直接閱讀重點，而不需要回頭看長篇大論的信。
    - *(整合原 Story 2.2, US-001, US-005)*
- **US-3.2 [P0] 溯源連結 (Deep Linking)**
    - **As a** 使用者,
    - **I want** 每個生成的任務都附帶原始郵件的 Deep Link,
    - **So that** 當我真正開始執行任務時，可以一鍵跳轉回 Gmail 查看附件或細節。
    - *(整合原 US-07, Story 2.2)*

---

### Epic 4: Command Center (人機協作與決策)

**目的：** 使用者介面 (UI)，強調 Human-in-the-loop。

- **US-4.1 [P0] 任務審核儀表板 (The Dashboard)**
    - **As a** 使用者,
    - **I want** 在一個統一介面看到 AI 建議的所有任務 (Drafts)，並能執行「批准 (Approve)」、「修改 (Edit)」或「刪除 (Reject)」,
    - **So that** 我擁有最終決定權，確保同步到 Google Tasks 的內容是乾淨且正確的。
    - *(整合原 Story 3.1, Epic 3, US-06)*
- **US-4.2 [P1] 快速調整參數 (Quick Tuning)**
    - **As a** 使用者,
    - **I want** 能手動調整 AI 預估的「工時」或「截止日」,
    - **So that** 任務更貼近我的實際狀況。
    - *(整合原 US-004)*