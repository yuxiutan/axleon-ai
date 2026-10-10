# API 一覽

約 45 個端點，全部置於 `/api/v1` 之下。本文僅列出介面與用途，不含實作。

共通規則：

- 主管端與駕駛端使用**各自獨立的簽章 Cookie**，因此同一瀏覽器可並排展示兩端
- 主管端的查詢類端點接受 `range` 參數：`today` / `week` / `month` / `year` / 自訂區間
- 官方固定樣本端點刻意**不套用**期間換算，與示範資料完全隔離
- 回應中的每個數字都帶有來源標籤（官方實算／情境估算／Demo Mock）

---

## 系統

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/healthz` | 健康檢查，並回報目前 AI 模式（`model` 或 `rules`） |
| GET | `/api/v1/meta` | 車隊基本資訊與介面設定 |

---

## 認證

| 方法 | 路徑 | 用途 |
|---|---|---|
| POST | `/api/v1/auth/login` | 登入。含失敗節流 |
| POST | `/api/v1/auth/logout` | 登出 |
| GET | `/api/v1/auth/me` | 取得目前登入者 |

---

## 主管端 — 營運查詢

| 方法 | 路徑 | 對應畫面 |
|---|---|---|
| GET | `/api/v1/fleet/overview` | 營運成效總覽 |
| GET | `/api/v1/fleet/official-evidence` | 官方數據實證（唯讀、不受示範資料影響） |
| GET | `/api/v1/fleet/performance` | 車隊營運效能 |
| GET | `/api/v1/fleet/attribution` | 影響因子分析 |
| GET | `/api/v1/fleet/tracking` | 車輛動態與軌跡 |
| GET | `/api/v1/fleet/events` | 異常駕駛與事件分析 |
| GET | `/api/v1/fleet/drivers` | 駕駛支援 |
| GET | `/api/v1/fleet/maintenance` | 維修預警 |
| POST | `/api/v1/fleet/scenario/roi` | 情境估算試算 |

---

## 主管端 — 處置與核決

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/fleet/work-orders` | 工單清單 |
| POST | `/api/v1/fleet/work-orders` | 建立工單。若該車有綁定駕駛，同步推送通知 |
| POST | `/api/v1/fleet/work-orders/:id/status` | 工單進度：`open` → `doing` → `done` |
| POST | `/api/v1/fleet/events/:id/status` | 事件處置狀態 |
| POST | `/api/v1/fleet/tasks/reassign` | 任務改派，雙方駕駛同步收到通知 |
| POST | `/api/v1/fleet/drivers/:code/coach` | 指派教練主題 |

核決結果回寫資料庫，成為下一次判斷的依據。

---

## 主管端 — 訊息

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/fleet/messages` | 車隊訊息 |
| POST | `/api/v1/fleet/messages` | 發送車隊訊息 |
| GET | `/api/v1/fleet/messages/replies` | 駕駛回覆 |

---

## 主管端 — 報表

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/fleet/report/preview` | 營運彙整報告預覽 |
| GET | `/api/v1/fleet/report/analysis` | 報告的 AI 分析敘述 |
| GET | `/api/v1/fleet/report/archive` | 封存副本 |
| GET | `/api/v1/fleet/report/notes` | 報告註記 |
| POST | `/api/v1/fleet/report/notes` | 新增註記 |
| POST | `/api/v1/fleet/report/notes/delete` | 刪除註記 |
| GET | `/api/v1/fleet/report-settings` | 寄送設定 |
| POST | `/api/v1/fleet/report-settings` | 更新寄送設定 |
| POST | `/api/v1/fleet/report-settings/test` | 寄送測試信 |
| POST | `/api/v1/fleet/report-settings/send-now` | 立即寄送 |

---

## 主管端 — 設定

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/fleet/settings` | 車隊維護參數：連續駕駛上限、單次休息長度、延誤判定分鐘數、油耗偏差百分比、SLA 與違約條件 |
| POST | `/api/v1/fleet/settings` | 更新參數 |

---

## 駕駛端

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/driver/me` | 本人今日任務、車況與行駛狀態 |
| POST | `/api/v1/driver/trip/start` | 出車 |
| POST | `/api/v1/driver/trip/state` | 回報行駛狀態 |
| POST | `/api/v1/driver/tasks/:id/complete` | 完成交貨 |
| POST | `/api/v1/driver/rest` | 休息回報 |
| POST | `/api/v1/driver/shift/end` | 收工 |
| POST | `/api/v1/driver/notify` | 延誤通知 / SOS |
| POST | `/api/v1/driver/sos/cancel` | 取消 SOS |
| POST | `/api/v1/driver/messages/read` | 訊息已讀 |
| POST | `/api/v1/driver/messages/reply` | 一鍵快速回覆 |
| POST | `/api/v1/driver/work-orders/:id/ack` | 確認維修工單 |

駕駛僅能存取自己的紀錄，由角色權限在伺服器端強制。沒有訊號時操作先存在手機，回線後自動補傳。

---

## AI 小助理

| 方法 | 路徑 | 用途 |
|---|---|---|
| GET | `/api/v1/assistant/home` | 建議問題與今日重點 |
| POST | `/api/v1/assistant/ask` | 提問。走完整管線：意圖路由 → 取事實 → 生成 → Reviewer 檢核 |
| POST | `/api/v1/assistant/feedback` | 答案回饋 |

回應包含答案文字、引用的依據明細，以及該則答案的檢核結果。主管端與駕駛端共用同一組端點，由登入角色決定意圖子集、模型與輸出長度。

詳見 [ai-pipeline.md](ai-pipeline.md)。
