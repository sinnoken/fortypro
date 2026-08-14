# FortiToolkit — Compliance 規則包（PACK）資料結構規格書

> 依據 `FortiToolkit.html` 內嵌 `ui.js` 的 `var PACK = [...]` 與 `core.js` 的
> `invokeCompliance` / `testCond` / `failDetail` 實作整理。
> 本文件只描述**資料結構與引擎行為**，不重複列出每條規則的業務理由（詳見附錄或
> 各規則自身的 `why` 欄位，雙擊 Compliance tab 的列即可在原始 config 看到上下文）。

---

## 1. 總覽

一條規則（rule）是 `PACK` 陣列裡的一個物件，由 `Core.invokeCompliance(parsed, PACK)`
逐條套用到 `Core.parseConfig()` 解析出的物件樹（`parsed.objects`），輸出一筆一筆的
`compliance record`（下稱 **result row**），供 Compliance tab 渲染、勾選、產生
`buildFixCli` 用的修復 CLI。

規則分兩大類，差在**評估單位**：

| 類別 | 判定對象 | mode 值 |
|---|---|---|
| **一般規則** | `config <section>` 底下的物件（單一設定塊或每個物件） | `each` / `section` |
| **設備層級規則** | 整份 parsed config（`parsed` 本身），不綁定某個 section | `device` |

---

## 2. Rule 物件 Schema（一般規則：`mode: 'each' / 'section'`）

```ts
{
  id:      string,          // 規則唯一代號，如 'ACC-01'
  sev:     'critical'|'high'|'medium'|'low'|'info',
  cat:     string,           // 分類標籤，如 'Administrative access'
  section: string,           // 要比對的 FortiOS config section，如 'system admin'
  zone:    'global'|'vdom'|'any',  // 這條規則要在哪些 VDOM 範圍內評估
  mode:    'each'|'section',
  title:   string,           // 規則標題（顯示於 Requirement 欄）
  conds:   Cond[],           // 一組條件，全部通過才算 Pass（AND 邏輯）
  why:     string,           // 「不做會怎樣」，非重述規則本身；雙擊細節原樣顯示
  fix:     string[]          // buildFixCli 直接輸出的 CLI 行；站點特定值前綴 '#'
}
```

### 2.1 `zone` 的實際涵義

| zone 值 | VDOM 模式（`vdomMode=true`）下的目標 | 單一 VDOM（`vdomMode=false`）下的目標 |
|---|---|---|
| `global` | 只評估 `['global']` | `['root']` |
| `vdom` | 評估除了 `global` 以外的每個 vdom | `['root']` |
| `any`（或其他值） | 評估**每一個**偵測到的 vdom（含 global） | `['root']` |

若目標清單為空，強制退回 `['root']`。

### 2.2 `mode` 的行為差異

- **`each`**：對 `idx[vdom|section]` 底下**每一個物件**各自跑一次 `conds`，
  每個物件各產生一筆 result row（`Obj` = 物件名稱）。
  若該 vdom 底下完全沒有該 section 的物件 → 產生一筆 `Skip` row
  （`Pass:true, Skip:true, Detail:'Not applicable (section absent)'`），
  不計入失敗也不計入通過。
- **`section`**：把整個 `config <section>` 當成**一個設定塊**，只取
  `idx[vdom|section][0]`（第一個物件）跑一次 `conds`，`Obj` 固定顯示
  `'(settings)'`。同樣，該 section 不存在時產生 Skip row。

---

## 3. Rule 物件 Schema（設備層級規則：`mode: 'device'`）

```ts
{
  id:    string,
  sev:   'critical'|'high'|'medium'|'low'|'info',
  cat:   string,
  mode:  'device',
  title: string,
  check: (parsed: ParsedConfig) => { pass: boolean, detail?: string, line?: number },
  why:   string,
  fix:   string[]
}
```

- **沒有** `section` / `zone` / `conds` 欄位——因為判定邏輯是任意 JS 函式
  `check(parsed)`，直接讀 `parsed` 整體（例如 `parsed.vdomMode`、
  `Core.deviceInfo(parsed).ha`），不走 `testCond` 引擎。
- 只跑**一次**，`V` 固定顯示 `parsed.vdomMode ? 'global' : 'root'`，
  `Obj` 固定顯示 `'(device)'`。
- `check` 回傳 `{pass, detail}`；`pass:true` 時 `detail` 會被忽略
  （result row 的 `Detail` 強制清空）。

---

## 4. `Cond`（條件）物件 Schema

```ts
{
  key: string,             // 要讀的欄位名（FortiOS config 的 set 鍵名）
  op:  OpType,             // 比較運算子，見 4.1
  val?: string | string[], // 期望值（exists/missing 不需要）
  def?: string             // 該鍵缺省時代入的預設值（見 4.2）
}
```

一條規則的 `conds` 是**陣列**，彼此以 **AND** 邏輯合併
（`rule.conds.every(c => testCond(o, c))`），只要有一項不通過，整條規則對該物件
就是 Fail。

### 4.1 支援的 `op`（運算子）

| op | 語意 | 需要 `val` |
|---|---|---|
| `exists` | 該鍵有值且非空字串 | 否 |
| `missing` | 該鍵不存在或為空字串 | 否 |
| `eq` | 字串完全相等 | 是（string） |
| `ne` | 字串不相等 | 是（string） |
| `in` | 值必須是 `val` 陣列中的一個 | 是（string[]） |
| `notin` | 值不得是 `val` 陣列中任何一個 | 是（string[]） |
| `has` | 值以空白/逗號等分詞後（`splitTokens`），需包含 `val` 這個 token | 是（string） |
| `nothas` | 分詞後不得包含 `val` 這個 token | 是（string） |
| `gte` | 數值 ≥ `val`（`parseFloat` 比較） | 是（string，可轉數字） |
| `lte` | 數值 ≤ `val` | 是（string，可轉數字） |
| `match` | 值需符合正則 `val`（`new RegExp(val).test(value)`） | 是（string，正則字串） |
| `notmatch` | 值不得符合正則 `val` | 是（string，正則字串） |

> ⚠️ **`has`/`nothas` 必須用 token 比對而非子字串**——例如檢查 `allowaccess`
> 是否含 `http`，若用子字串比對，`https` 會被誤判為包含 `http`。這是
> CLAUDE.md 鐵則明文要求的精確度守則。

### 4.2 取值邏輯 `getVal(obj, cond)`

```
if obj.T[cond.key] 存在  → 回傳該值（字串化）
else if cond 有 'def'    → 回傳 cond.def（視為 FortiOS 原廠預設值）
else                     → 回傳 null（視為「未設定」）
```

- **缺鍵 ≠ 未設定**：FortiOS 匯出檔常省略等於原廠預設值的欄位。有 `def` 的規則
  在鍵不存在時會**代入預設值繼續判斷**，而非直接跳過。
- 例外：一致性基準類規則（要求「必須明確設定」的規則，如 `hostname` 格式）
  **刻意不宣告 `def`**，讓「沒寫」也視為不合規。
- `exists` / `missing` 這兩個 op 不受 `def` 影響，只看鍵本身是否存在
  （已在 `testCond` 裡於呼叫 `getVal` 之前用 `raw != null && raw !== ''`
  單獨處理，其結果會受 `def` 影響——若鍵不存在但有 `def`，`getVal` 回傳
  `def` 字串，`exists` 仍會判定為存在；設計規則時需留意這個交互）。

---

## 5. Result Row（`invokeCompliance` 輸出）Schema

這是 `A.comp`（Compliance tab 資料來源）裡每一筆記錄的結構：

```ts
{
  Sev:   string,          // 同 rule.sev
  V:     string,          // 實際評估的 vdom 名稱（或 'global'/'root'）
  Id:    string,          // 同 rule.id
  Cat:   string,          // 同 rule.cat
  Title: string,          // 同 rule.title
  Obj:   string,          // 物件名稱 / '(settings)' / '(device)' / '-'（skip）
  Scope: string,          // section 名稱，或 'device'（device-level 規則）
  Pass:  boolean,
  Skip:  boolean,         // true = 該 vdom 沒有對應 section，不計入 pass/fail 統計
  Detail: string,         // Pass 時為 ''；Fail 時為 failDetail() 產生的說明文字
  Line:  number,          // 該物件在原始 config 的行號（0 = 無，如某些 device 規則）
  Why:   string,          // 同 rule.why
  Fix?:  string[],        // 同 rule.fix（device 規則才會帶上；each/section 規則的 Fix
                           //  實際由 Compliance tab 勾選後透過 Core.buildFixCli 另外組裝）
  SectLevel: boolean       // true = section/device 層級（一筆代表整個設定塊）；
                           //  false = each 層級（一筆代表一個物件）
}
```

`Detail`（失敗說明）由 `failDetail(obj, conds)` 產生，規則是：
1. 依序檢查 `conds`，回傳**第一個**未通過的條件的說明文字（不會列出全部失敗原因）。
2. 文字格式依 `op` 而異，例如 `eq` → `"key = 現值, expected 期望值"`；
   `exists` → `"key is not configured"`。
3. 若現值其實是靠 `def` 代入的（原始 config 沒寫），文字會附加 `" (default)"`
   提示這是預設值，不是使用者手動設定的。

### Skip Row（section 不存在時）

```ts
{
  Sev, V, Id, Cat, Title,
  Obj: '-',
  Scope: rule.section,
  Pass: true,
  Skip: true,
  Detail: 'Not applicable (section absent)',
  Line: 0,
  Why: rule.why
  // 無 Fix / SectLevel 欄位
}
```

Compliance tab 預設**不顯示** Skip 列（需勾選「顯示 passing」`chkPass` 才會連同
Skip 一起顯示，見 `renderComp()`）。

---

## 6. 新增規則的檢查清單（CLAUDE.md 規則包慣例）

新增或修改一條規則前，需確認：

1. **欄位齊全**：`id / sev / cat / section / zone / mode / conds / why / fix`
   （device 規則則是 `id / sev / cat / mode / check / why / fix`）。
2. **新增 `op` 種類**時必須同步修改 `testCond` **與** `failDetail` 兩處，
   漏改後者會讓失敗訊息退化成無上下文的 `key = value`。
3. **`why` 寫「不做會怎樣」**，不要重述規則本身。
4. **`fix` 中站點特定值的行以 `#` 開頭**（`buildFixCli` 原樣輸出，使用者需自行填值）。
5. **必須新增兩份測試**：一份合規 config 驗證 pass、一份違規 config 驗證 fail
   （對應 `test_rules.js`），只驗一個方向會漏掉「永遠 pass」或「永遠 fail」的壞規則。
6. **`has`/`nothas` 一律用 token 比對**（`splitTokens`），不可用子字串判斷。
7. **判斷缺鍵語意**：先確認這條規則屬於「安全下限」（該宣告 `def`）還是
   「一致性基準」（刻意不宣告 `def`），兩者意圖不同。

---

## 附錄：目前 `PACK` 內已實作的 16 條規則一覽

| ID | Sev | Cat | mode | section / check | 摘要 |
|---|---|---|---|---|---|
| ACC-01 | high | Administrative access | each | `system admin` | 每個管理者帳號需設定 `trusthost1` |
| ACC-02 | medium | Administrative access | section | `system global` | `admintimeout` ≤ 10 |
| PWD-01 | high | Password policy | section | `system password-policy` | 密碼長度/複雜度/到期/禁止重用基準 |
| GLB-01 | medium | System hardening | section | `system global` | 登入標語 + 自動備份啟用 |
| GLB-02 | low | System hardening | section | `system global` | 時區為台北 |
| SYS-01 | low | System hardening | section | `system global` | hostname 符合 `SITE.hostnamePattern` |
| IF-01 | high | System hardening | each | `system interface` | 介面不得開放 HTTP 管理 |
| IF-02 | medium | System hardening | each | `system interface` | 介面不得開放 Telnet 管理 |
| IF-03 | low | System hardening | each | `system interface` | 停用 ICMP redirect（收/送） |
| SNMP-01 | high | SNMP | each | `system snmp community/hosts` | SNMP host ACL 限制為 /32 |
| VPN-01 | high | VPN | section | `vpn ssl settings` | SSL VPN 應停用 |
| UTM-01 | medium | System hardening | each | `firewall policy` | 政策不應啟用 UTM/Deep inspection |
| LOG-01 | medium | Logging | section | `log syslogd setting` | 必須設定 syslog（可比對 `SITE.syslogServers`） |
| DEV-01 | low | System hardening | device | `check(parsed)` | 應啟用 VDOM |
| DEV-02 | medium | System hardening | device | `check(parsed)` | 應啟用 HA，不得 standalone |
| DEV-03 | low | System hardening | device | `check(parsed)` | 韌體需在 `SITE.approvedFirmware` 核可清單內 |

站點特定值集中於 `SITE` 物件（`approvedFirmware` / `syslogServers` /
`hostnamePattern`），三者皆已被上表規則實際引用（DEV-03 / LOG-01 / SYS-01）。

---

## 7. JSON 規則包匯入/匯出（FortiToolkit.html ⇄ FortiRuleEditor.html）

`FortiToolkit.html` 現在支援透過 JSON 檔案匯入/匯出自訂規則，並提供一個獨立的
`FortiRuleEditor.html` 編輯器來產生/編輯這份 JSON。兩者**只透過檔案交換資料**，
不共用程式碼（各自獨立、離線可用的單一 HTML 檔）。

### 7.1 檔案 schema（`fortitoolkit-rulepack` v1）

```json
{
  "schema": "fortitoolkit-rulepack",
  "version": 1,
  "meta": { "name": "...", "note": "...", "generatedAt": "ISO time" },
  "site": {
    "approvedFirmware": ["7.2.5", "7.4.4"],
    "syslogServers": ["10.1.1.116"],
    "hostnamePattern": ".+_.+_.+"
  },
  "rules": [ /* 結構化規則或設備規則，見下 */ ]
}
```

- **結構化規則**（`mode: 'each'|'section'`）：與本文件第 2 節 schema 完全相同
  （`id/sev/cat/mode/section/zone/title/conds/why/fix`）。
- **設備規則**（`mode: 'device'`）：因 JSON 無法承載函式，改用
  `checkType` 列舉 + 可選 `params`：

  | checkType | 對應行為 | params |
  |---|---|---|
  | `vdomEnabled` | 應啟用 VDOM（同 DEV-01） | 無 |
  | `haEnabled` | 應啟用 HA，不得 standalone（同 DEV-02） | 無 |
  | `firmwareApproved` | 韌體需在核可清單內（同 DEV-03） | `{ approvedFirmware: [...] }`（缺省則退回檔案的 `site.approvedFirmware`） |

### 7.2 FortiToolkit.html 端行為

- 工具列新增「匯入規則 (JSON)」「匯出規則 (JSON)」「還原內建規則」按鈕與規則統計
  badge；拖放 `.json` 檔到主畫面也會自動導向規則匯入（`.conf/.txt/.cfg` 仍走原本
  的 config 載入流程）。
- 匯入流程：讀檔 → `validateRulePack()` 驗證（欄位齊全、`op` 合法、正則可編譯、
  id 不重複…）→ 顯示確認彈窗（列出新增/覆蓋規則、可選是否套用 `site` 值）→ 按
  「套用」後寫入 `CUSTOM_RULES`（**僅存於記憶體，重新整理頁面即還原內建**，與
  config/分析結果同樣不落地 localStorage）。
- `getEffectivePack()`：同 id 的自訂規則會**覆蓋**內建規則，新 id 則**附加**；
  Compliance tab 的 `invokeCompliance` 一律吃這個合併後的清單。
- 匯出的是**目前生效**的規則（內建 + 已套用的自訂），而非永遠固定的 16 條，
  方便「匯出目前規則 → 用編輯器改一條 → 再匯入」的迭代流程。

### 7.3 FortiRuleEditor.html（獨立編輯器）

- 表單式 CRUD 介面：規則列表（可編輯/複製/刪除）、新增規則彈窗（含 conds
  動態編輯器、mode 切換顯示對應欄位）、即時 JSON 預覽（唯讀，可複製）。
- 「載入內建 16 條規則」按鈕內嵌了與 `FortiToolkit.html` 目前 PACK 完全一致的
  16 條規則快照，可作為編輯起點。
- 匯入/匯出皆使用與 FortiToolkit 相同邏輯的**平行複製版** `validateRulePack`
  （兩檔案不共用程式碼，只共用 JSON schema，修改 schema 時需手動同步兩處）。
- 同樣無任何外部依賴、無 CDN，單一檔案離線可用。
