# 臺東縣消防局災害應變中心 (EOC) 自動化系統架構

## 系統背景

災害應變中心成立時，需將相關資料登打至 **EMIC 系統**。目前痛點：
- 會議記錄仍需人工聽錄音打成逐字稿
- 各局處資料需人工彙整
- 一級開設時需每 **3 小時** 出一報
- 報告需經陳核後再發佈

---

## 系統架構總覽

```mermaid
flowchart TB
    subgraph 資料來源層["📥 資料來源層"]
        A1[🎙️ 會議錄音檔]
        A2[📋 各局處進駐人員資料]
        A3[📱 LINE 報案/回報]
        A4[📞 語音報案系統]
        A5[📄 既有 EMIC 資料]
    end

    subgraph 資料擷取層["🔄 資料擷取與轉換層"]
        B1[語音轉文字 ASR]
        B2[OCR 文件辨識]
        B3[API 資料串接]
        B4[表單資料收集]
    end

    subgraph 判斷邏輯層["🧠 AI 判斷與整合層"]
        C1{資料完整性檢查}
        C2[AI 摘要生成]
        C3[資料分類標籤]
        C4[缺漏資料警示]
    end

    subgraph 備案機制["⚠️ 備案機制"]
        D1[人工補件通知]
        D2[預設範本填充]
        D3[歷史資料調用]
    end

    subgraph 報告生成層["📊 報告生成層"]
        E1[處置報告]
        E2[重點災情報告]
        E3[工作會報]
        E4[會議紀錄]
    end

    subgraph 發送與同步層["📤 發送與同步層"]
        F1[陳核系統]
        F2[EMIC 同步]
        F3[LINE 推播通知]
        F4[Email 發送]
    end

    A1 --> B1
    A2 --> B2
    A2 --> B4
    A3 --> B3
    A4 --> B1
    A5 --> B3

    B1 --> C1
    B2 --> C1
    B3 --> C1
    B4 --> C1

    C1 -->|資料完整| C2
    C1 -->|資料缺漏| C4
    C4 --> D1
    C4 --> D2
    C4 --> D3
    D1 --> C1
    D2 --> C2
    D3 --> C2

    C2 --> C3
    C3 --> E1
    C3 --> E2
    C3 --> E3
    C3 --> E4

    E1 --> F1
    E2 --> F1
    E3 --> F1
    E4 --> F1

    F1 -->|核定通過| F2
    F1 -->|核定通過| F3
    F1 -->|核定通過| F4
```

---

## 詳細流程說明

### 1️⃣ 資料擷取流程

```mermaid
flowchart LR
    subgraph 音訊處理["🎵 音訊/會議處理"]
        R1A[Microsoft Teams 會議] --> R2A[內建逐字稿功能]
        R1B[錄音檔上傳] --> R2B[Whisper/ASR 轉文字]
        R2A & R2B --> R3[AI 逐字稿校正]
        R3 --> R4[發言人辨識]
    end

    subgraph 文件處理["📄 文件處理"]
        D1[PDF/圖片上傳] --> D2[PaddleOCR 辨識]
        D2 --> D3[結構化資料抽取]
    end

    subgraph EMIC整合["🔗 EMIC 整合 (RPA)"]
        RPA1[RPA 自動登入 EMIC]
        RPA1 --> RPA2[自動填寫表單]
        RPA2 --> RPA3[截圖確認/錯誤重試]
    end
```

### 2️⃣ 判斷邏輯（含備案機制）

```mermaid
flowchart TB
    START[資料進入] --> CHECK{必填欄位檢查}
    
    CHECK -->|✅ 完整| VALIDATE{資料格式驗證}
    CHECK -->|❌ 缺漏| FALLBACK1

    subgraph 備案處理["⚠️ 備案處理"]
        FALLBACK1[記錄缺漏欄位]
        FALLBACK1 --> NOTIFY[發送補件通知]
        FALLBACK1 --> TEMPLATE[套用預設範本]
        FALLBACK1 --> HISTORY[調用歷史資料]
        
        NOTIFY --> WAIT{等待回覆}
        WAIT -->|已補件| CHECK
        WAIT -->|超時 30 分鐘| TEMPLATE
        
        TEMPLATE --> MANUAL_FLAG[標記為待審核]
        HISTORY --> MANUAL_FLAG
    end

    VALIDATE -->|✅ 有效| AI_PROCESS
    VALIDATE -->|❌ 格式錯誤| FALLBACK1

    subgraph AI處理["🤖 AI 處理"]
        AI_PROCESS[AI 資料整合]
        AI_PROCESS --> SUMMARY[生成摘要]
        AI_PROCESS --> CLASSIFY[分類標籤]
    end

    MANUAL_FLAG --> AI_PROCESS
    SUMMARY --> OUTPUT[輸出至報告生成]
    CLASSIFY --> OUTPUT
```

### 3️⃣ 報告生成流程

```mermaid
flowchart TB
    subgraph 報告類型["📋 報告類型"]
        T1["處置報告<br/>（災情處置進度）"]
        T2["重點災情報告<br/>（重大災情彙整）"]
        T3["工作會報<br/>（各局處工作報告）"]
        T4["會議紀錄<br/>（會議決議事項）"]
    end

    subgraph 生成引擎["⚙️ 生成引擎"]
        G1[選擇報告範本]
        G2[AI 內容填充]
        G3[格式轉換]
        G4[版本控制]
    end

    subgraph 輸出格式["📤 輸出格式"]
        O1[Word 文件]
        O2[PDF 文件]
        O3[系統表單]
    end

    T1 & T2 & T3 & T4 --> G1
    G1 --> G2 --> G3 --> G4
    G4 --> O1 & O2 & O3
```

### 4️⃣ 發送與同步流程

```mermaid
flowchart TB
    REPORT[報告完成] --> SUBMIT[提交陳核]
    
    SUBMIT --> REVIEW{陳核審查}
    REVIEW -->|退回修改| EDIT[編輯修正]
    EDIT --> SUBMIT
    
    REVIEW -->|核定通過| APPROVE[核定完成]
    
    APPROVE --> SYNC_EMIC[同步至 EMIC]
    APPROVE --> PUSH_LINE[LINE 推播]
    APPROVE --> SEND_EMAIL[Email 發送]
    APPROVE --> ARCHIVE[歸檔保存]

    subgraph 通知對象["👥 通知對象"]
        N1[局長室]
        N2[相關局處]
        N3[消防署]
    end

    PUSH_LINE --> N1 & N2
    SEND_EMAIL --> N2 & N3
    SYNC_EMIC --> N3
```

---

## 系統模組規劃（依優先級排序）

| 優先級 | 模組 | 功能 | 技術建議 |
|--------|------|------|----------|
| 🔴 1 | **報告自動彙整生成** | 各局處資料彙整並自動生成報告 | Python + AI (GPT-4/Claude) |
| 🟡 2 | **會議記錄自動化** | 會議轉逐字稿 + 摘要 | MS Teams 內建 / Whisper API |
| 🟢 3 | **EMIC 同步 (RPA)** | 自動登打 EMIC 系統 | Selenium / Playwright RPA |
| ⚪ 4 | 通知系統 | LINE / Email 推播 | LINE Bot / SMTP |

---

## 預期效益

1. **會議記錄自動化**：錄音轉文字 + AI 摘要，減少 80% 人工時間
2. **報告自動生成**：一級開設時每 3 小時自動出一報
3. **資料一致性**：自動同步至 EMIC，減少重複登打
4. **即時通知**：核定後自動推播至相關人員

---

## 下一步行動

### Phase 1：報告自動彙整生成 (最高優先)
1. 蒐集「處置報告」「重點災情報告」「工作會報」現有範本
2. 定義各局處需提供的資料欄位與格式
3. 建立 AI 報告生成 Prompt 與範本
4. 開發報告自動組裝系統 (Python)

### Phase 2：會議記錄自動化
1. 評估 Microsoft Teams 內建逐字稿功能
2. 若需離線方案，測試 Whisper API
3. 開發 AI 會議摘要模組

### Phase 3：EMIC 同步 (RPA)
1. 錄製 EMIC 操作流程
2. 使用 Selenium/Playwright 開發 RPA 腳本
3. 建立錯誤處理與重試機制
