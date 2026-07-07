# 通用開發工作流（可攜版）

> 但拿掉了專案專屬的路徑、工具名稱，讓你之後接手任何新專案都能直接套用，
> 只要把中括號 `[ ]` 裡的項目換成新專案的實際名稱即可。

---

## 核心精神

1. **先對齊、再動手**：規格、範圍沒確認清楚之前不寫程式。
2. **文件是唯一真相來源**：程式行為跟文件不一致時，先處理落差，不要憑印象開發。
3. **改東西前先搞懂影響範圍**：不管有沒有像 GitNexus 這種工具，改既有邏輯前都要先找出呼叫者/依賴者。
4. **測試是完成的一部分，不是額外加分項**：沒有測試、測試沒過，都不算做完。
5. **不可跳步**：下面每個步驟都建立對應的 todo 項目，逐一勾選完再進下一步。

---

## Step 0 — 確認任務範圍（開發前必做）

跟需求方或自己對齊以下問題，任一項不清楚就先問清楚再開始：

- [ ] 這個功能 / 修改屬於哪個 domain（模組）？
- [ ] 對應的需求文件 / PRD / issue 是哪一份？
- [ ] 是否涉及資料庫 schema 變更？需不需要新的 migration？
- [ ] 是否涉及外部系統整合（第三方 API、webhook、callback、async job）？
- [ ] 影響範圍是單一 repo，還是跨前後端 / 跨服務？

> 純文件修改、設定檔調整（如 `.env.example`）、CI 設定變更可以跳過完整流程。

---

## Step 1 — 找到並讀完對應規格文件

在動手寫程式之前，先找出並讀完：

| 文件類型 | 用途 |
|---|---|
| PRD / 需求文件 | 確認「為什麼要做」與驗收標準 |
| API 規格 / Contract | 確認 endpoint、request/response、錯誤格式 |
| Workflow / 流程文件 | 確認 Main Flow 與 Alternate Flow（異常情境） |
| 架構文件 | 確認這個功能該落在哪一層、哪個目錄 |
| DB schema | 確認資料表、欄位、關聯是否已定義 |

找不到對應文件 → 回報並停下來，**不要憑空實作**；如果真的沒文件，先把規格用文件寫下來再繼續。

---

## Step 2 — 讀取並遵守專案的 Coding / Testing Standards

進入實作前，重新確認這個專案目前的：

- 套件管理工具是唯一指定的哪一個（例如 `uv` / `yarn` / `pnpm`），不要混用其他工具。
- 專案目錄結構慣例（route / service / repository / model 各自該放哪）。
- 命名慣例（檔案、函式、測試）。
- Commit message 規範（例如 Conventional Commits：`feat:` / `fix:` / `refactor:` / `docs:` / `test:` / `chore:`）。

同時搜尋既有程式庫，避免重複實作已經存在的 utility / service：

- 用關鍵字搜尋現有相似功能（grep、專案內建的 code intelligence 工具、或直接看目錄結構）。
- 找到相似實作就重用或延伸，而不是從零寫一份。

---

## Step 3 — 影響分析（修改既有程式碼前必做）

- [ ] 這個 function / class / method 有沒有其他地方在呼叫？
- [ ] 改了之後，哪些上游呼叫者、哪些流程會被影響？
- [ ] 如果有程式碼影響分析工具（如 GitNexus、IDE 的 Find Usages、`grep -r`），先跑過再動手。
- [ ] 風險評估為高／關鍵（會動到核心流程、多處呼叫、公開 API）→ 先跟人確認再繼續。
- [ ] 禁止用「全域搜尋取代」做 rename，要用理解呼叫關係的方式重新命名。

---

## Step 4 — 實作

依專案既有的分層結構落點程式碼，常見對照：

| 種類 | 常見位置 |
|---|---|
| Route / Controller | `routes/` 或 `controllers/` |
| 業務邏輯 / Service | `services/` |
| 資料存取 / Repository | `repositories/` |
| ORM Model / Entity | `models/` |
| 外部整合 Client | `clients/` |
| Middleware / Validation | `middleware/`、`validation/` |
| 資料庫異動 | migration 檔案 |

實作中若發現規格文件跟實際需求有落差，**先回頭修文件**，再繼續寫程式，避免程式碼與文件永久性分裂。

---

## Step 5 — 撰寫測試

- 測試目錄結構鏡像程式碼結構（同樣的資料夾層級）。
- 測試命名清楚描述「情境 + 預期結果」，例如：`test_should_[expected]_when_[condition]`。
- 涵蓋：
  - [ ] 正常路徑（happy path）
  - [ ] 邊界條件
  - [ ] 錯誤 / 例外情境（含外部依賴失敗、逾時、非預期輸入）
- 不清楚怎麼寫的地方，先參考專案裡既有的測試檔案當範例。

---

## Step 6 — 執行測試並確認覆蓋率

```bash
# 針對本次修改範圍執行
[test-runner] tests/<path> -v

# 附覆蓋率報告
[test-runner] tests/<path> --cov=<module> --cov-report=term-missing

# 提交前跑全套
[test-runner] --cov --cov-report=term-missing
```

- 紅燈就修到綠燈，**不可用註解、skip、mock 掉核心邏輯來矇混過關**。
- 核心業務邏輯覆蓋率需達到專案要求門檻（若無明訂，建議 ≥ 80%，關鍵邏輯 ≥ 95%）。

---

## Step 7 — 跨端整合驗證（若涉及前後端 / 多服務）

1. **啟動所有相依服務**：資料庫、快取、後端 API server、前端 dev server 等。
2. **手動走一次 Main Flow**：確認資料在各端正確傳遞與渲染。
3. **手動製造異常情境**：API 逾時、服務斷線、回傳 4xx/5xx，驗證錯誤處理與提示是否符合預期。
4. 出錯時，優先查看各服務的終端機日誌 / console，定位問題發生在哪一層。

---

## Step 8 — 完成前的最終稽核（Definition of Done）

提交前逐一確認：

- [ ] 所有相關測試綠燈，覆蓋率達標
- [ ] 有跑過影響分析，且列出的依賴都已同步更新
- [ ] 程式碼行為與文件（API 規格 / Workflow / PRD）100% 吻合；若開發期間規格有調整，文件已同步更新
- [ ] 相依套件版本鎖定檔（lockfile）如有變更，已一併納入版控
- [ ] 若新增 migration，已在乾淨環境跑通
- [ ] Commit message 符合專案規範
- [ ] 沒有把無關變更混進同一次提交

任一項沒過 → 不算完成。

---

## 反模式（Anti-Patterns，禁止）

- 跳過規格文件確認，憑印象或憑猜測實作
- 修改既有程式碼前沒做影響分析
- 混用不同套件管理工具（例如同專案又用 pip 又用 poetry）
- 沒寫測試，或測試紅燈就回報完成
- 用全域搜尋取代做 rename，忽略呼叫關係
- 把不相關的變更混進同一個 commit
- 在程式碼中硬寫 secret / API Key / Token
- 敏感資料（token、session）存放在不安全的地方（如前端 localStorage）

---

## 套用到新專案時，記得替換的項目

- `[test-runner]` → 該專案的測試指令（`pytest` / `jest` / `go test` ...）
- 套件管理工具名稱
- 目錄結構對照表（依實際框架調整）
- 覆蓋率門檻數字
- Commit message 規範（若專案有自己的慣例）
