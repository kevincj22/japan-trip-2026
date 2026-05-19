# API Interface 規格文件

**專案：** 公文AI儀表板  
**版本：** v0.2（WO5.2 修訂）  
**日期：** 2026-05-18  
**負責人：** Jasper（後端）/ Amy（前端）

---

## 共用規範

### Base URL
```
http://localhost:8001/api
```

### 角色識別
所有 API 均透過 request body 或 header 傳入角色，後端據此決定回傳內容。

| role 值 | 說明 |
|---|---|
| `supervisor` | 長官版 |
| `handler` | 承辦人版 |

### 共用 Request Header
```
Content-Type: application/json
X-User-Role: supervisor | handler        # 角色（備援用，主要以 body 傳入）
```

### 共用錯誤格式

| HTTP 狀態碼 | error code | 觸發條件 |
|---|---|---|
| 400 | `INVALID_ROLE` | `role` 不是 `supervisor` / `handler` |
| 400 | `MISSING_FIELD` | 必填欄位缺漏 |
| 403 | `PERMISSION_DENIED` | `user_id` 存在但無權限存取該資源（例如跨角色存取） |
| 404 | `USER_NOT_FOUND` | `user_id` 在系統中不存在 |

```json
{
  "error": "INVALID_ROLE",
  "message": "role 必須為 supervisor 或 handler"
}
```
```json
{
  "error": "USER_NOT_FOUND",
  "message": "找不到 user_id: U999"
}
```
```json
{
  "error": "PERMISSION_DENIED",
  "message": "user_id U001 無權限以 supervisor 角色存取此資源"
}
```

---

## 6.1 待辦即時摘要

### 說明
使用者登入時或點擊待辦卡片時觸發，由 LangGraph 產生 AI 摘要與統計數字。  
長官版回傳「送審單位 Top3」；承辦人版回傳「辦理階段 Top3」。

> **`near_due_count` 計算定義：** `0 <= days_remaining <= 3`（含當天、含第 3 天）。  
> 後端計算：`days_remaining = (deadline - today).days`，`today` 取伺服器當地日期（UTC+8）。  
> 已逾期（`days_remaining < 0`）不納入 `near_due_count`，計入 `overdue_count`。

### Endpoint
```
POST /api/summary
```

### Request Body
```json
{
  "role": "supervisor",
  "user_id": "U001",
  "trigger": "login"
}
```

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `role` | string | ✅ | `supervisor` / `handler` |
| `user_id` | string | ✅ | 使用者 ID |
| `trigger` | string | ✅ | `login`（登入觸發）/ `card_click`（卡片點擊觸發） |

### Response — 長官版（role: supervisor）
```json
{
  "summary_text": "目前共有 12 件待辦公文，其中 3 件已逾期，2 件將於 3 日內到期。送審單位以研考會件數最多，建議優先處理。",
  "overdue_count": 3,
  "near_due_count": 2,
  "role_section": {
    "type": "top3_by_unit",
    "items": [
      { "unit": "研考會", "count": 5 },
      { "unit": "人事室", "count": 4 },
      { "unit": "主計室", "count": 3 }
    ]
  }
}
```

### Response — 承辦人版（role: handler）
```json
{
  "summary_text": "您有 8 件待辦公文，其中 1 件已逾期，3 件即將到期。目前多集中在「會辦中」階段，請儘速處理。",
  "overdue_count": 1,
  "near_due_count": 3,
  "role_section": {
    "type": "top3_by_stage",
    "items": [
      { "stage": "會辦中", "count": 4 },
      { "stage": "待批示", "count": 2 },
      { "stage": "退件修正", "count": 2 }
    ]
  }
}
```

### Response Schema
| 欄位 | 型別 | 角色 | 說明 |
|---|---|---|---|
| `summary_text` | string | 共用 | AI 產生的摘要文字 |
| `overdue_count` | integer | 共用 | 已逾期件數 |
| `near_due_count` | integer | 共用 | 即將逾期件數（`days_remaining` 滿足 `0 <= days_remaining <= 3`，即今天到第3天含當天；已逾期不計入） |
| `role_section.type` | string | 各版不同 | `top3_by_unit`（長官）/ `top3_by_stage`（承辦人） |
| `role_section.items[].unit` | string | 長官版 | 送審單位名稱 |
| `role_section.items[].stage` | string | 承辦人版 | 辦理階段名稱 |
| `role_section.items[].count` | integer | 共用 | 件數 |

---

## 6.2 建議辦理順序

### 說明
回傳依優先權排序的前 5 筆待辦公文，每筆含文號、主旨、期限。  
前端點擊文號後跳轉至簽核頁面（跳轉 URL 由後端提供）。

### 排序邏輯
採三層加權排序，**不是純依 deadline 排序**：

| 優先層 | 條件 | 說明 |
|---|---|---|
| 第 1 層（最高） | `days_remaining < 0` | 已逾期，越逾期（絕對值越大）越前面 |
| 第 2 層 | `0 <= days_remaining <= 3` | 即將到期，依 deadline 升冪排列 |
| 第 3 層 | `days_remaining > 3` | 一般，依 deadline 升冪排列 |

同一層內若 deadline 相同，依收文日期升冪（越早收越前）。

### Endpoint
```
POST /api/todo-priority
```

### Request Body
```json
{
  "role": "handler",
  "user_id": "U001",
  "top_n": 5
}
```

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `role` | string | ✅ | `supervisor` / `handler` |
| `user_id` | string | ✅ | 使用者 ID |
| `top_n` | integer | ❌ | 回傳筆數，預設 5，最大 10 |

### Response
```json
{
  "items": [
    {
      "doc_no": "院本部教字第1140012345號",
      "subject": "函請提供110年度預算執行情形",
      "deadline": "2026-05-20",
      "days_remaining": 2,
      "is_overdue": false,
      "priority_rank": 1,
      "sign_url": "/sign/1140012345"
    },
    {
      "doc_no": "院本部人字第1140009876號",
      "subject": "有關員工在職訓練計畫案",
      "deadline": "2026-05-17",
      "days_remaining": -1,
      "is_overdue": true,
      "priority_rank": 2,
      "sign_url": "/sign/1140009876"
    }
  ],
  "total_count": 12
}
```

### Response Schema
| 欄位 | 型別 | 說明 |
|---|---|---|
| `items[].doc_no` | string | 文號（點擊可跳轉） |
| `items[].subject` | string | 主旨 |
| `items[].deadline` | string | 期限，ISO 8601 格式（YYYY-MM-DD） |
| `items[].days_remaining` | integer | 剩餘天數；負數表示已逾期 |
| `items[].is_overdue` | boolean | 是否逾期 |
| `items[].priority_rank` | integer | 優先順序（1 最高） |
| `items[].sign_url` | string | 簽核頁面相對路徑，供前端跳轉 |
| `total_count` | integer | 使用者全部待辦總件數 |

---

## 6.3 建議操作按鈕

### 說明
依使用者角色與當前待辦狀態，動態回傳 1–4 顆建議操作按鈕。  
`is_urgent` 為 `true` 時前端以紅色或警示樣式呈現。

### 按鈕決定邏輯
按鈕組合由**角色固定配置 + 當前待辦狀態**共同決定，規則如下：

| 按鈕 | 出現條件 | 適用角色 |
|---|---|---|
| `view_overdue` | `overdue_count > 0`；`is_urgent=true` | 共用 |
| `send_extension` | `near_due_count > 0`（即將到期有件）；`is_urgent=false` | 共用 |
| `export_report` | 長官版固定顯示；承辦人版不出現 | 長官版 |
| `add_to_bank` | 固定顯示（墊底，最低優先） | 共用 |

最多取前 4 顆，按上表順序排列；若 `overdue_count = 0` 且 `near_due_count = 0`，最少回傳 `add_to_bank` 1 顆。

### Endpoint
```
POST /api/action-buttons
```

### Request Body
```json
{
  "role": "supervisor",
  "user_id": "U001"
}
```

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `role` | string | ✅ | `supervisor` / `handler` |
| `user_id` | string | ✅ | 使用者 ID |

### Response
```json
{
  "buttons": [
    {
      "label": "查看逾期公文",
      "action_type": "view_overdue",
      "is_urgent": true
    },
    {
      "label": "申請展期",
      "action_type": "send_extension",
      "is_urgent": false
    },
    {
      "label": "匯出報表",
      "action_type": "export_report",
      "is_urgent": false
    },
    {
      "label": "加入常用",
      "action_type": "add_to_bank",
      "is_urgent": false
    }
  ]
}
```

### Response Schema
| 欄位 | 型別 | 說明 |
|---|---|---|
| `buttons` | array | 長度 1–4 |
| `buttons[].label` | string | 按鈕顯示文字 |
| `buttons[].action_type` | string | 動作類型（見下表） |
| `buttons[].is_urgent` | boolean | 是否以警示樣式呈現 |

### action_type 列舉
| action_type | 說明 |
|---|---|
| `view_overdue` | 跳轉至逾期公文清單 |
| `send_extension` | 開啟展期申請流程 |
| `export_report` | 匯出目前報表（PDF / Excel） |
| `add_to_bank` | 將本件加入常用公文庫 |

---

## 6.4 制式快捷功能

### 說明
依角色回傳快捷功能清單，前端以 Grid 呈現。  
長官版 6 項、承辦人版 4 項，項目固定（非 AI 動態產生）。

### Endpoint
```
POST /api/shortcuts
```

### Request Body
```json
{
  "role": "supervisor",
  "user_id": "U001"
}
```

| 欄位 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `role` | string | ✅ | `supervisor` / `handler` |
| `user_id` | string | ✅ | 使用者 ID |

### Response — 長官版（role: supervisor）
```json
{
  "role": "supervisor",
  "shortcuts": [
    { "id": "daily_close",     "label": "今日結案統計",       "icon": "chart-bar",   "route": "/stats/daily-close" },
    { "id": "monthly_eff",     "label": "本月辦理效率比較",   "icon": "chart-line",  "route": "/stats/monthly-efficiency" },
    { "id": "pending_ext",     "label": "待展期公文",         "icon": "clock",       "route": "/docs/pending-extension" },
    { "id": "doc_flow",        "label": "公文流程",           "icon": "flow",        "route": "/docs/flow" },
    { "id": "legislator",      "label": "立委質詢分析",       "icon": "analysis",    "route": "/analysis/legislator" },
    { "id": "overdue",         "label": "逾期公文",           "icon": "warning",     "route": "/docs/overdue" }
  ]
}
```

### Response — 承辦人版（role: handler）
```json
{
  "role": "handler",
  "shortcuts": [
    { "id": "pending_ext",  "label": "待展期公文",   "icon": "clock",     "route": "/docs/pending-extension" },
    { "id": "doc_flow",     "label": "公文流程",     "icon": "flow",      "route": "/docs/flow" },
    { "id": "legislator",   "label": "立委質詢分析", "icon": "analysis",  "route": "/analysis/legislator" },
    { "id": "overdue",      "label": "逾期公文",     "icon": "warning",   "route": "/docs/overdue" }
  ]
}
```

### Response Schema
| 欄位 | 型別 | 角色 | 說明 |
|---|---|---|---|
| `role` | string | 共用 | 回傳角色確認 |
| `shortcuts` | array | 各版不同 | 長官版 6 項，承辦人版 4 項 |
| `shortcuts[].id` | string | 共用 | 功能唯一識別碼 |
| `shortcuts[].label` | string | 共用 | 顯示文字 |
| `shortcuts[].icon` | string | 共用 | 前端圖示 key（由前端 icon map 對應） |
| `shortcuts[].route` | string | 共用 | 前端路由路徑 |

### 各角色快捷項目對照

| id | 長官版 | 承辦人版 |
|---|---|---|
| `daily_close` | ✅ 今日結案統計 | — |
| `monthly_eff` | ✅ 本月辦理效率比較 | — |
| `pending_ext` | ✅ 待展期公文 | ✅ 待展期公文 |
| `doc_flow` | ✅ 公文流程 | ✅ 公文流程 |
| `legislator` | ✅ 立委質詢分析 | ✅ 立委質詢分析 |
| `overdue` | ✅ 逾期公文 | ✅ 逾期公文 |

---

## API 一覽表

| 功能 | Method | Path | 角色差異 |
|---|---|---|---|
| 6.1 待辦即時摘要 | POST | `/api/summary` | `role_section` 欄位不同 |
| 6.2 建議辦理順序 | POST | `/api/todo-priority` | 無（資料依 user_id 過濾） |
| 6.3 建議操作按鈕 | POST | `/api/action-buttons` | 按鈕組合依角色不同 |
| 6.4 制式快捷功能 | POST | `/api/shortcuts` | 項目數量不同（6 / 4） |
