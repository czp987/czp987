---
name: yg-paper-matching
description: "Implement, extend, or audit the YG Quant local paper-trading engine, including persistent orders and balances, price ticks, order-book matching, cancel/partial-fill transitions, isolated USDT perpetuals, WebSocket updates, and automated tests. Use when modifying a paper-trading simulator or its exchange-free matching lifecycle."
---

# YG 本地模擬撮合工作流程

## 目標與邊界

本技能針對 YG量化的 LOCAL_PAPER 核心；預設服務端 MySQL/Drizzle、tRPC、Node/Express。只處理虛擬資產，不呼叫交易所私有端點、不接受真實訂單、不儲存交易所密鑰。不要將本地行情、成交、費率或 PnL 說成交易所實際數據。

## 工作順序

1. 先讀專案 `plan.md`、`TODO.md`、目前 `drizzle/schema.ts`、遷移 journal、模擬 service、router、server bootstrap 與測試。確認實際環境/`DATABASE_URL`，不可輸出秘密；本專案 managed DB 可能與 production 共用，只提交可回滾但非破壞的增量 migration，禁止以測試清除或重置資料。
2. 保留登入帳戶隔離：每個讀/寫以已驗證 `ctx.user.id` 定位帳戶和訂單；撤單、平倉、WebSocket 訂閱都需所有權檢查。對帳戶及訂單行採交易內 `FOR UPDATE`，防止重複扣款、雙重成交和撤單競態。
3. 以固定精度整數/decimal 字串處理數量、價格、手續費、保證金和損益；不可用 binary float 累積資產。明示 tick、費率與風險限額是可設定的本地示例值。
4. 將訂單狀態轉換明確實作並集中保護：`open → partially_filled → filled`；`open/partially_filled → canceled`；終態 `filled/canceled/rejected` 不可重複轉移。部分成交須同步剩餘數量、預留額、成交、流水和事件。
5. 市價 tick/深度撮合須在 server worker 產生或取得可信伺服器價格，依價位/深度逐層撮合；行情更新呼叫和多個 worker 重入要有鎖或單例防護。限價 BUY 僅於 asks 可在限價內成交，SELL 僅於 bids 可在限價上成交。tick 遇重啟應從資料庫價位續行，不以瀏覽器提供的價格觸發。
6. 逐倉 USDT 合約分開建模方向、數量、入場均價、槓桿、isolated margin、開/平倉費與 realized PnL。先預檢最大槓桿、最低保證金、可用餘額、單筆及持倉名義本金；風險拒單不可改變資產。平倉量不得超過對向持倉，部分平倉按比例釋放保證金並原子結算 PnL/費用。未實作的清算/資金費功能應明確列為限制，不能暗示已有完整永續合約風控。
7. WebSocket 只向已認證且已授權連線傳送自己帳戶的訂單事件；市場行情只傳 LOCAL_PAPER 標記。處理 Origin、無效 JSON、訂閱白名單、連線逾時/關閉、慢消費者、心跳與服務關閉；WS 是通知管道，資料庫仍是唯一真相來源。
8. 每個狀態轉換以自動化測試覆蓋：成功撤單與雙重撤單、跨帳戶 IDOR、價位排序、深度全成/部分成/未觸價、可注入 seed 的隨機行情界限與限價觸發、開倉費/保證金邊界、槓桿超限、資金不足、平倉盈虧和部分平倉、重複 worker/tick。WebSocket 另以確定性 burst/Promise.all 壓力測試多符號、多帳戶推送、無跨帳戶資料外洩及事件無漏送；不要用脆弱的單機毫秒門檻代替 correctness assertions。先跑 targeted tests，再跑 `pnpm check && pnpm test && pnpm build`。
9. 先生成、審閱新增的 Drizzle migration，再於取得授權的環境執行 `pnpm db:migrate`；核對唯讀 schema 查詢結果。更新 README/TODO，清楚區分已完成、未完成和依賴真實行情/外部服務的功能。

## 專案慣例

- 精度與撮合純函式放 `server/simulation/` 並用 Vitest 測試；資料庫副作用集中於 service；tRPC 僅做登入授權、輸入驗證與呼叫 service。
- migration 一律加法；不可在訂單流水表更新/刪除既有紀錄作為「修正」。需要沖正時寫明確的反向流水。
- 開發背景 tick worker 用單一程序生命週期啟動一次，使用 `unref()`/受控 shutdown；不可啟動每個 request 一個 timer。多副本部署時需資料庫租約或分散式鎖，不能假設單例。
- WebSocket 事件 payload 不含憑證、內部錯誤 stack 或其他帳戶資料；掉線客戶端需重抓 REST/tRPC snapshot 追平。

## 回覆標準

說明實際修改檔案、執行過的檢查與 migration 是否已套用；分開標示背景 tick、訂單自動撮合、逐倉開/平倉、WebSocket 等已接通與僅有雛形/待完成內容。絕不把型別通過當成實盤安全審查。
