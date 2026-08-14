# FortiToolkit (SPA) — 原則指引

FortiGate 設定檔的**稽核前資料整理工具**(單一 HTML 版)。定位很重要:它把散落的 config 攤開、看清楚、產出**候選清單**,真正的稽核判斷由人在後面做。所以它的首要美德是**看得清楚、用得順手、好維護**,而非苦行式的零依賴——產出是候選、會被人複查,不是被信任的結論。

純瀏覽器端,無 server、無安裝。輸入一份或多份 config 匯出檔,輸出合規檢查、清理候選、重複合併、網路檢視與拓樸圖,以及勾選後生成的 CLI 與 CSV。前身是 PowerShell WinForms 版;本版存在的理由是**核心邏輯可用 node 執行驗證**,交付前作者自己就能驗到底,不再依賴「看起來對」。

---

## 檔案結構(開發用 4 檔;交付是 1 檔;另有一個獨立的配套工具)

| 檔 | 性質 | 對外提供 |
|----|------|---------|
| `core.js` | 純邏輯,**零 DOM**。解析、引用分析、重複偵測、合規引擎、CLI 產生、網路模型 | `window.FortiCore` / `module.exports`(雙掛載,故 node 可載入測試) |
| `ui.js` | UI 層,依賴 `FortiCore`。渲染、勾選、CLI 輸出、樹、細節視窗、規則包 `PACK`、站點值 `SITE`、規則包匯入/匯出模組 | 掛在 `DOMContentLoaded` |
| `shell.html` | HTML 外殼 + CSS,含 `<!--VENDOR-->` / `<!--CORE-->` / `<!--UI-->` 三個佔位符 | 版面骨架 |
| `build.js` | 打包器:把 vendor + core + ui 內聯進 shell,輸出單一 HTML | `node build.js` |
| `vendor.dagre.js` | 第三方庫 dagre,由 esbuild 預先 bundle 成自足 IIFE(拓樸分層佈局用) | `window.dagre` |
| `test*.js` / `verify.js` | node 測試,涵蓋核心、CLI、網路、排序、規則包、多檔、拓樸、佈局、端到端 | 回歸防線 |

**`FortiRuleEditor.html` 是獨立的配套工具,不屬於上述 build 流程**:它是另一個完全獨立、無共用程式碼的單一 HTML 檔,唯一與 `FortiToolkit.html` 的連結是**同一份 JSON 檔案格式**(`fortitoolkit-rulepack`,見下方「規則包外部匯入/匯出」一節)。修改其中一邊的驗證邏輯,不會自動同步到另一邊,需要手動對齊。

**建構用 `node build.js`,不是手動字串替換**:它把三段(vendor / core / ui)內聯進 `shell.html` 的佔位符,輸出 `FortiToolkit.html`。交付的永遠是這個單一檔;開發永遠改源檔,**不要直接編輯 `FortiToolkit.html`**(它是產物,會被覆蓋)。

**第三方庫走 vendored,不走 CDN**:dagre 由 esbuild 預先 `--bundle` 成 `vendor.dagre.js`(自足、無外部 import),再由 `build.js` 內聯。要更新或新增庫:`npm install` 後用 esbuild bundle 成一個 vendor 檔,加進 build。**永遠不用 CDN**——理由是「離線會掛」(config 常在隔離網段/跳板機上看),不是信任問題。附 lockfile 確保可重現。

**依賴方向硬性單向**:`shell.html → ui.js → core.js`。`core.js` 檔頭寫明 "No DOM in this file",**不得引用 `document` / `window` 上的任何控制項**。這條讓 core 能在 node 下跑,是整個測試策略的地基;違反它等於廢掉自我驗證能力。

---

## 鐵則(違反即產出錯誤 CLI、誤判合規,或讓工具重回「不可驗證」狀態)

1. **交付前必須用 node 執行過測試** — 本版的全部意義就在這。任何改動後跑 `node test*.js`,全綠才 build。這條取代了 PowerShell 版「只能靠 grep 數數量」的困境——那正是當初視窗跳不出來、卻遲遲定位不到的根因。**靜態檢查不是執行,數量吻合不等於行為正確。**

2. **測試斷言從原始碼抽取,不手抄** — `test_sort.js` / `test_rules.js` 用正則從 `ui.js` / `core.js` 抽出真正的函式來測,不維護第二份副本。手抄的副本會與真檔漂移,測試就變成測一份假東西。**這條是被真的踩過才寫下的**:曾用手抄副本測 sortKey,還把斷言寫反印出假 FAIL。

3. **斷言寫下前先手算期望值** — 排序測試曾斷言「10.9 應排在 10.113 之後」,但第二段 9 < 113,正解是之前;是斷言錯不是程式錯。**測試紅了先問「我的期望對嗎」,再問「程式錯嗎」**,兩個方向都要查。

4. **CLI 註解永不接在指令後面** — FortiOS 把指令之後的一切當參數。`buildSafeCli` / `buildDecideCli` / `buildFixCli` 產出的每個 `#` 一律自成一行。有測試專門掃「行內有沒有 `#` 前面還有非空白」,別讓它變紅。

5. **引用分析不得引入欄位白名單** — `invokeCleanup` 掃每個物件的每個值(跳過 `SKIPKEYS` 這種明確非引用欄),任何 token 命中已知物件名即算引用。白名單只要漏一個欄位,就會**靜默地把在用的物件放進刪除清單**——失敗方向朝著刪除,最壞。root 是推導的(不在 `WATCH` 者即 root),不是列舉。

6. **合規規則跑物件樹,不跑文字** — 匯入的官方 NCM 規則是文字 regex,萬用字元 `(.|\n)*` 會越過物件邊界造成假通過。`PACK` 一律轉成結構化規則,引擎一次只看一個物件。轉換時**用 token 比對而非子字串**:`allowaccess` 檢查 `http` 要用 `nothas`(token),否則 `https` 會被誤判。這條有專門的 precision 測試守著。

7. **缺鍵 ≠ 未設定,但「一致性基準」是例外** — FortiOS 匯出省略原廠預設。安全下限類規則要宣告 `def`,取值時代入。**但**國維處那種「必須明確設定 admintimeout 10」的一致性基準**刻意不宣告 def**,讓原廠預設也算不合規。兩者意圖不同,新增規則時先判斷是哪一種。

8. **內建物件永不進刪除清單** — `isBuiltin`(名單 + 前綴)是硬停,獨立於引用分析。內建物件合法地零引用但刪不得,改列 `held`。

9. **刪除前的 refcnt 驗證步驟不可從產出中拿掉** — config 檔看不到 FortiManager、SDN connector、裝置記憶體裡的引用。`buildSafeCli` 的 Step 2 逐物件輸出 `diagnose sys cmdb refcnt show`。**裝置才是權威,檔案不是。**

10. **重複值的刪除一律註解掉** — `buildDecideCli` 產出的 `delete` 全部前綴 `#`。合併重複要先把每處引用改指到 keeper,不是批次操作。有測試驗「沒有任何未註解的 delete」。

11. **刪除段落順序:群組先於成員** — `sectionOrder` 把 `GROUPSET` 排 0、其餘排 1。反了會在刪成員時群組還指著它,FortiOS 回 `-23`。

12. **localStorage 只存偏好,不存 config 或分析結果** — 只存「顯示 passing」「跨 VDOM 比對」這類 UI 偏好。config 可能很大,且每次開檔重新解析才符合「檔案是輸入、不是狀態」。分析結果活在記憶體,關頁即消,這是隱私上的正確預設(config 不落地)。匯入的自訂規則(`CUSTOM_RULES`)同理**只存記憶體、不進 localStorage**——規則包跟 config 一樣是輸入,不是需要持久化的狀態,重新整理頁面即還原為內建規則。

13. **CLI 產生一律以 host 為界** — 勾選項目帶 `_dev`,`regenCli` 只在單一設備範圍內產生 CLI;勾選橫跨多台設備時直接拒絕,不靜默產出物件名稱/行號互相混雜的錯誤內容。曾是真實 bug:`Duplicates` 的 `usedBy` 一度固定抓 `curDevice()||DEVICES[0]`,跟勾選項目實際所屬設備對不上。

14. **自我產生的規則,跟外部輸入走同一套驗證** — 規則包匯入(不論是使用者手寫,或未來由任何自動化建議產生)一律過 `validateRulePack`,沒有「自己產生的所以可信」這種捷徑。裝置層級規則用固定列舉 `checkType` 還原 `check()`,不允許匯入任意程式碼。

---

## 設計原則(改 code 前自我檢查「有沒有違反 X？」)

- **可執行驗證優先於審視** — 有辦法讓 node 跑到的邏輯,就寫進 core、寫測試;不要留在只能靠肉眼看 UI 的地方。GUI 掛不掛都不該影響「分析是否正確」的可驗證性。
- **Fail-safe direction** — 判斷任何取捨先問:這個失敗朝哪個方向倒?引用分析寧可少刪、內建寧可留、文字規則寧可不執行、規則包驗證失敗寧可整批拒絕也不要部分套用。「誤報一條要人工看」永遠優於「漏報一條讓人刪掉在用的物件,或讓一條寫壞的規則悄悄生效」。
- **Core 無 DOM,UI 無領域邏輯** — 解析、比對、遮罩、規則判定屬 core;渲染、事件、勾選狀態屬 ui。想在 `renderXxx` 裡算 IP 網段,是放錯層——那該在 core 且該有測試。
- **SSOT** — `WATCH` / `GROUPSET` / `BUILTIN` / `CMDB` / `DUPSEC` 在 core 各定義一次。嚴重度色票 `SEV`、分頁清單 `TABS` 在 ui 各一次。候選物件的視覺呈現(kind 標籤 + host.vdom 徽章)只在 `pickerEntryLabelHtml` 定義一次,不同進入點(搜尋選單、CIDR 貼上解析)共用。
- **站點特定值集中在 `SITE`** — 韌體核可清單、syslog IP、hostname 格式這些**不在 config 檔裡**的值(原為 NCM 變數),集中在 `ui.js` 頂部的 `SITE` 物件,標明「EDIT THESE」。不要散落進各規則。可透過規則包 JSON 的 `site` 欄位一併匯入,匯入時使用者可選擇是否套用。
- **依賴政策:能讓它更清楚、更好維護就用,只要 vendored 能離線** — 這是資料整理工具,不是稽核工具,不需要供應鏈潔癖。判準是「它明顯比自製好就上」(如 dagre 的分層佈局勝過手寫力導向),不是「能不加就不加」。唯一硬約束:**產物須離線可用(vendored,不靠 CDN)**,並附 lockfile。開發期工具(esbuild、測試 runner、formatter)自由選用,它們在 build 時消失、不進產物。
- **Determinism** — 同一份 config 兩次分析結果必須相同。排序用預先算好的鍵(`routeSortKey`),不靠物件列舉順序。
- **Lazy render / 記憶體快取** — 重算的東西(如 `usedBy`)算一次存 `A._usedBy`。新增重量級檢視沿用此模式。
- **可追溯到原始碼行** — 每個物件帶 `L`(行號),`getContext` 據此印原文前後文,雙擊即見。**新增任何發現類型都要帶行號**,否則使用者無從查證。這是從 config 讀出、可指向具體一行;推導出來的(如 connected route)行號指向其來源物件,並在 UI 標明是 derived。
- **多檔/多設備預留** — 樹已是「設備節點 → vdom」兩層,`deviceInfo` 獨立可重用。新增分析時假設未來會有多個 device,不要把「單一 config」寫死進資料結構。任何新增的「可勾選 → 產生 CLI」功能,預設就要假設輸入橫跨多台設備,並在設計階段就決定好 host 邊界怎麼處理(參見鐵則 13)。
- **群組沒有自己的值,只能透過成員間接命中** — 群組本身不帶 IP/CIDR 值,數值比對(CIDR 貼上解析)要命中群組必須展開成員,遇到無法解析的成員(FQDN、懸空參照)跳過、回傳部分結果,不可整組判定失敗。群組作為候選時永遠不自動勾選,因為牽連組內所有成員,範圍比單一物件大,必須由人決定。

---

## JavaScript / 瀏覽器慣例

- **core.js 雙掛載** — 檔尾同時 `module.exports`(node)與 `window.FortiCore`(瀏覽器)。新增公開函式記得加進那個匯出物件,否則 UI 或測試拿不到。
- **`==` 只用於 `== null`** — 用 `x == null` 同時判 null/undefined 是刻意的;其餘一律 `===`。
- **HTML 一律經 `esc()`** — 任何進 `innerHTML` 的使用者資料(物件名、值、註解)必須先 `esc()` 逃逸 `& < >`。config 內容不可信,直接內插會壞版面甚至注入。
- **事件用委派,不逐列綁定** — 勾選、排序表頭、雙擊細節都掛在 `document.body` 上,靠 `data-*` 屬性辨識目標。數萬列時逐列綁 listener 是災難;新增互動沿用委派。
- **狀態放 `sel` / `A`,不放 DOM** — 勾選狀態存 `sel[tab]`(Set of id),分析結果存 `A`。重繪時從狀態還原 UI,不從 DOM 讀回狀態。
- **IP 要數值排序,不是字串** — `sortKey` 把 IPv4/CIDR 轉 `address*100+prefix`;純數字轉 number;空白排最後。字串排序會把 `10.113` 排在 `10.9` 前,錯。這條有獨立測試。
- **CSV 帶 BOM** — `toCsv` 前置 `\ufeff`,Excel 開才不亂碼;換行用 `\r\n`。
- **JSON 匯出走獨立的 `downloadJson`,不共用 `download`** — 規則包匯出使用正確的 `application/json` MIME type,不借用 CSV 匯出既有的 `download()`(那個函式寫死 `text/csv`)。兩個下載輔助函式各司其職,不要為了少寫幾行而混用。
- **剪貼簿走 `navigator.clipboard`,並保留 `execCommand` 降級** — 且包 try/catch,失敗時退到狀態列訊息,不讓例外浮出。

---

## 規則包慣例(改內建 `PACK` / `SITE` 前必讀)

- **每條規則欄位**:`id` / `sev` / `cat` / `section` / `zone`(global|vdom|any)/ `mode` / `conds` / `why` / `fix`。
- **三種 `mode`**:`each`(section 內每個物件各判一次)、`section`(對 section 設定塊判一次)、`device`(對整份 config 判一次,用 `check(parsed)` 回 `{pass,detail}`,給 VDOM/HA/韌體這種設備層級)。
- **`op` 種類**:`exists`/`missing`/`eq`/`ne`/`in`/`notin`/`has`/`nothas`/`gte`/`lte`/`match`/`notmatch`。新增 op 要**同步改 `testCond` 與 `failDetail` 兩處**——漏改後者會讓失敗訊息退化成沒有上下文的 `key = value`。
- **每條規則必須有 `why`**,寫「不做會怎樣」而非重述規則本身,雙擊細節會原樣顯示。
- **`fix` 裡需要站點特定值的行以 `#` 開頭** — `buildFixCli` 原樣輸出,使用者直接貼進裝置。
- **新增規則必須加兩向測試** — 一份合規 config 驗它 pass、一份違規 config 驗它 fail。`test_rules.js` 就是這個模式;只驗一向會漏掉「永遠 pass」或「永遠 fail」的壞規則。
- **內建 `PACK` 目前涵蓋的類別**(不釘死條數,結構會持續擴充):Administrative access(帳號 trusthost、閒置逾時)、Password policy(密碼複雜度基準)、System hardening(登入標語/備份、時區、hostname 格式、介面管理存取、ICMP redirect、UTM、VDOM/HA/韌體等裝置層級項目)、SNMP、VPN、Logging。裝置層級規則(`mode:'device'`)額外帶 `_checkType` / `_params` 欄位,對應下方「規則包外部匯入/匯出」的 `checkType` 列舉,讓內建規則也能被 `exportRulePack` 原樣序列化回 JSON。

---

## 規則包外部匯入/匯出(JSON,`fortitoolkit-rulepack` schema)

內建 `PACK` 之外,`FortiToolkit.html` 支援透過 JSON 檔案擴充或覆蓋 Compliance 規則,並提供獨立的 `FortiRuleEditor.html` 產生/編輯這份 JSON。**兩者只透過檔案交換資料,不共用程式碼**——修改一邊的 schema 或驗證邏輯,必須手動同步到另一邊。

- **Schema(version 1)**:`{ schema:"fortitoolkit-rulepack", version:1, meta:{name,note,generatedAt}, site:{approvedFirmware,syslogServers,hostnamePattern}, rules:[...] }`。
- **結構化規則**(`mode:'each'|'section'`)與內建 `PACK` 的欄位 schema 完全相同。
- **裝置規則**(`mode:'device'`)因 JSON 無法承載函式,改用 `checkType` 列舉(`vdomEnabled` / `haEnabled` / `firmwareApproved`)+ 可選 `params`,由 `CHECK_BUILDERS` 還原成真正的 `check()` 函式——**這是固定的已知集合,不允許匯入任意程式碼**,同一套 fail-safe 精神(參見鐵則 14)。
- **匯入流程**:讀檔 → `validateRulePack()` 驗證 → 顯示確認彈窗(列出新增/覆蓋規則、可選是否套用 `site` 值)→ 使用者按下「套用」才寫入 `CUSTOM_RULES`(**session-only,見鐵則 12**)。
- **`getEffectivePack()`**:同 id 的自訂規則**覆蓋**內建規則,新 id **附加**;Compliance 的 `invokeCompliance` 一律吃這個合併後的清單,不直接用 `PACK`。
- **匯出的是目前生效的規則**(內建 + 已套用的自訂),不是永遠固定的內建清單,支援「匯出目前規則 → 用 `FortiRuleEditor.html` 改一條 → 再匯入」的迭代流程。
- **拖放 `.json` 檔到主畫面會自動導向規則匯入**(`readFiles` 依副檔名分流),`.conf`/`.txt`/`.cfg` 仍走原本的 config 載入流程,兩者可同時拖放、互不干擾。

---

## 測試慣例(本版最重要的資產)

- **改 core 必跑 `node test*.js`,改規則必跑 `test_rules.js`,改排序必跑 `test_sort.js`** — build 前全綠是硬門檻。
- **端到端在 `verify.js`** — 用真實 FortiOS 格式(全雙引號)跑「解析→挑選→產 CLI」整條流;合成測試資料**用雙引號**,曾因用單引號 `edit 'admin'` 產出 `edit "'admin'"` 的假象浪費時間。
- **build 後驗內嵌 script 語法** — 用 node 的 `vm.Script` 對抽出的三段 `<script>`(vendor / core / ui)做語法檢查,並確認 core、ui 內嵌後與源檔逐字元相同、dagre 有內聯、無 CDN 外鏈。這是「產物 = 源碼」的最後一道保險,而且它真的擋下過損毀(見下)。
- **`build.js` 的 `replace` 一定用函式,不用字串** — `String.prototype.replace(marker, str)` 會把替換字串裡的 `$'`/`$&`/`` $` ``/`$$` 當特殊語法。我們的 JS 含正則字面(如 SNMP 規則的 `\s*$` 後面跟引號,湊成 `$'`),用字串替換會把該處靜默替換成「匹配之後的全部內容」,產物損毀且**測試全過**(因為 node 測試只從源檔正則抽函式、不整檔載入)。**這正是「build 後驗語法」存在的理由**——它是唯一會整檔驗證、抓到這類損毀的關卡。改用 `replace(marker, function(){ return code; })` 即免疫。
- **規則包驗證邏輯有平行複本,兩邊都要測** — `validateRulePack`/`validateOneRule` 在 `FortiToolkit.html` 與 `FortiRuleEditor.html` 各有一份(刻意不共用程式碼,見檔案結構一節)。修改其中一份的驗證規則(例如新增一種 `op`),另一份不會自動同步,必須手動比對兩邊的測試是否都涵蓋了新情況。

---

## CI/CD 與自我改進原則

協作沒有記憶可以跨 session 傳遞,能傳遞的只有結果。所以每次的驗證,要 materialize 成一個會被機械式重新執行的關卡(測試、自我稽核函式、round-trip 驗證),不要指望文件裡的敘述能讓下一次讀懂推理過程。關卡不會隨記憶消失,文字會。

現有關卡(`test*.js`、build 後的語法與內容一致性驗證、`verify.js`)目前都靠人手動跑,尚未接上真正的 CI 系統——這是現狀,不是已完成的自動化,不要把「應該做」寫成「做到了」。

任何新增的自我檢查機制(規則庫矛盾偵測、parser round-trip、規則候選建議等),只能產出候選或驗證結果,不能自己寫入 `PACK`/`CUSTOM_RULES`;新機制本身也要有測試證明抓得到問題,不能只憑看起來合理。

---

- **不寫死可變數字** — 規則數、測試數會變,用結構或量級表述,別在文件裡釘死某個數。
- **記錄「為什麼」與「踩過的坑」** — 這份指引的價值不在描述現狀(讀 code 就知道),而在保留取捨理由與失敗教訓,讓接手者不重蹈。
