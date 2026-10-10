# 程式碼架構

本文說明 AXLEON AI 的分層設計、模組職責與請求生命週期。不含原始碼。

---

## 分層

資料由左向右流動。清洗與語意層負責統一口徑，規則與統計負責判斷，生成式模型只把已驗證的事實寫成文字，寫完還要通過檢核才會出現在畫面上。

```
資料來源            資料平面              分析平面            AI 生成管線         用戶端
─────────          ─────────            ─────────          ──────────         ────────
iTRAQ CAN-BOX  →   清洗與行程識別   →    即時規則引擎   ┐
車隊維護參數        營運資料庫            統計分析       ├→  意圖路由           營運主管 Web
                    語意層                期間彙整       ┘    取事實             駕駛 App（PWA）
                                                              生成（Gemini）     每月報告（Email）
                                                              Reviewer 檢核
                                                              答案快取
                                        ← ← ← ← ← 回寫：主管核決、駕駛確認與回覆 ← ← ← ← ←
```

離開自有系統的只有兩個外部服務：Google Gemini API 與 SMTP 郵件服務。送往模型的內容僅包含查詢後的統計結果與代碼，不含姓名、地址等個資。

---

## 模組職責

### 伺服器核心

| 模組 | 職責 |
|---|---|
| `server/index.js` | HTTP 路由表與處理函式。路由以 `on(method, pattern, handler)` 註冊，`:param` 轉為具名擷取群組。主管端的各頁查詢共用 `fleetGet` 包裝，統一套用登入檢查與期間換算。 |
| `server/http.js` | JSON 讀寫、狀態碼與錯誤物件轉換，集中處理回應標頭。 |
| `server/auth.js` | 密碼雜湊驗證、簽章 Cookie 的簽發與解析、角色對應（manager／driver）、登入失敗節流。主管與駕駛使用各自獨立的 Cookie，因此同一瀏覽器可並排展示兩端。 |
| `server/config.js` | 所有環境變數的唯一讀取處。Session 金鑰優先取環境變數，否則在資料卷中產生一次並保存。 |
| `server/db.js` | 以 Node 內建 `node:sqlite` 建立連線，啟用 WAL、外鍵與 busy timeout，並宣告 18 張資料表。 |

### 語意層與領域邏輯

| 模組 | 職責 |
|---|---|
| `server/services.js` | **指標的唯一定義處**。百公里油耗、時間加權怠速佔比、同車型中位數偏差、事件密度等皆在此定義一次，畫面、報表與 AI 共用。各頁資料的組裝函式亦集中於此。 |
| `server/range.js` | 期間解析與區間換算：本日／本週／本月／本年／自訂。查詢結果依期間重新成形，官方固定樣本頁則刻意跳過此步驟。 |
| `server/official-evidence.js` | 讀取官方樣本實算結果。載入時即驗證口徑（車輛數、趟次）是否相符，不符則拒絕啟動，避免資料換版後靜默錯誤。回傳副本以免被後續成形邏輯污染。 |
| `server/provenance.js` | 資料來源三級標籤的定義，供畫面、報表與 Reviewer 共同引用。 |
| `server/scenario.js` | 情境估算試算，輸出明確標示為估算而非實測。 |
| `server/report.js` | 營運彙整報告的章節組版、封存副本與寄送。 |
| `server/live.js` | 模擬時鐘與即時狀態推進，容器重啟後接續而不倒回。 |

### AI 編排

| 模組 | 職責 |
|---|---|
| `server/ai/index.js` | 編排層：意圖路由 → 取事實 → 生成 → 檢核 → 快取。模型不可用時改走規則式答案。 |
| `server/ai/intents.js` | 15 條規則式意圖，含一條專門攔截範圍外問題（情緒、姓名、身分、即時路況）。主管端與駕駛端套用不同子集。 |
| `server/ai/answers.js` | 規則式答案，以及供 Reviewer 比對用的接地資料集（可引用的數值與代碼集合）。 |
| `server/ai/llm.js` | **全系統唯一呼叫模型的模組**。封裝 Gemini provider、系統指令、明確快取、responseSchema 與逾時取消。 |
| `server/ai/reviewer.js` | 六道確定性檢核。刻意不使用模型，以免與產生者共享盲點。 |
| `server/ai/glossary.js` | 語意層字典，隨系統指令一併送出並快取。 |
| `server/ai/report-analysis-v2.js` | 月報章節名稱的唯一來源，規則式輸出與模型指示都從此取，避免兩邊各寫一套。 |

詳見 [ai-pipeline.md](ai-pipeline.md)。

### 前端

`public/index.html` 是營運主管 Web 與駕駛 App 的**單一來源**：同一份檔案依路徑與登入角色切換成兩個應用。沒有框架、沒有建置步驟，因此可離線快取、載入快，部署面也縮到最小。

駕駛 App 以 PWA 形式運作：沒有訊號時操作先存在手機，回線後自動補傳。

---

## 請求生命週期

以主管端開啟「營運成效總覽」為例：

1. `index.js` 比對路由表，命中 `GET /api/v1/fleet/overview`
2. `fleetGet` 包裝呼叫 `requireUser(req, "fleet")`，解析並驗證簽章 Cookie
3. `range.js` 解析 `range` 查詢參數，算出起訖時間
4. `services.js` 以語意層定義的口徑查詢資料庫並組裝該頁資料
5. 依期間重新成形後，以 JSON 回應；每個數字都帶著來源標籤

AI 小助理的提問路徑另見 [ai-pipeline.md](ai-pipeline.md)。

---

## 錯誤處理與降級

| 情況 | 系統行為 |
|---|---|
| 未設定模型金鑰 | AI 小助理以規則式答案運作，功能不中斷 |
| 模型逾時、拒答或服務中斷 | 自動降級為規則式答案 |
| Reviewer 未通過 | 帶著錯誤處要求模型重寫一次；第二次仍不通過則改用規則式答案 |
| 資料不足以回答 | 明確回覆資料不足，不推測、不補數字 |
| 官方樣本口徑不符 | 啟動時即拋錯，不以錯誤資料提供服務 |

---

## 設定

所有環境變數集中在 `server/config.js` 讀取，無需改程式即可調整：

| 變數 | 用途 |
|---|---|
| `GEMINI_API_KEY` | 模型金鑰。未設定時走規則式回答 |
| `AI_FLEET_MODEL` / `AI_DRIVER_MODEL` | 模型路由，預設 `gemini-2.5-pro` / `gemini-2.5-flash` |
| `AI_MODE` | `auto`（預設）或 `off` |
| `AI_TIMEOUT_MS` | 模型逾時，預設 45000 |
| `AI_CACHE_CONTEXT` | 明確快取開關，設 `0` 關閉 |
| `COOKIE_SECURE` | 部署於 HTTPS 後方時設 `true` |
| `SESSION_SECRET` | Session 簽章金鑰。未設定會自動產生並存於資料卷 |
| `SESSION_HOURS` | 登入效期，預設 12 小時 |
| `DEMO_TOOLS` | 示範工具開關。**公開部署請設 `0`** |
| `SMTP_*` / `REPORT_TO` | 月報寄送設定 |

---

## 測試與工具

| 指令 | 內容 |
|---|---|
| `server/tools/smoke.js` | 19 項端到端檢查，實際走過登入與各營運流程 |
| `tools/ui-regression.py` | 63 項介面迴歸測試，涵蓋主管端與駕駛端完整操作流程 |
| `server/tools/ai-eval.js` | 50 題評估題庫，六類：官方數字、因果與排序、資料不足、隱私、示範與實證混用、估算邊界 |
| `server/tools/import-hino.js` | 由官方資料集重算官方樣本摘要 |

對唯讀的公開網址執行檢查時，會自動略過所有會改動資料的項目。
