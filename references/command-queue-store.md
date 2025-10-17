# Command Queue 與 Store 參考指南

## MultiType Queue 的儲存佈局與背壓機制

| SQLite 資料表 | `MessageType` enum | 相關設定鍵 | 背壓策略 |
| --- | --- | --- | --- |
| `STATELESS` | `MessageType::STATELESS` | `agent.queue_size`（位元組）、`events.batch_interval`（毫秒） | 同步生產者在 `std::condition_variable` 上等待，直到 `QUEUE_STATUS_REFRESH_TIMER` 到期；協程生產者在佇列滿時會等待一個 `steady_timer` 後重試。 |
| `STATEFUL` | `MessageType::STATEFUL` | `agent.queue_size`（位元組）、`events.batch_interval`（毫秒） | 與上列相同。 |
| `COMMAND` | `MessageType::COMMAND` | `agent.queue_size`（位元組）、`events.batch_interval`（毫秒） | 與上列相同。 |

* `m_mapMessageTypeName` 會把每個 `MessageType` 對應到 `STATELESS_TABLE_NAME`、`STATEFUL_TABLE_NAME` 與 `COMMAND_TABLE_NAME` 宣告的 SQLite 資料表，確保查詢在整個實作中一致。【F:src/agent/multitype_queue/include/multitype_queue.hpp†L19-L123】
* 每個資料表可儲存的最大訊息數由 `agent.queue_size` 限制，透過 `GetBytesConfigInRangeOrDefault` 讀取；`events.batch_interval` 則會設定消費端的 `m_batchInterval` 以控制批次處理間隔。【F:src/agent/multitype_queue/src/multitype_queue.cpp†L17-L44】
* Push 端在 `shouldWait=true` 時透過 `std::condition_variable` 與 `config::agent::QUEUE_STATUS_REFRESH_TIMER` 進行等待；協程 Push 則使用相同逾時的 `steady_timer` 迴圈等待空間釋出。【F:src/agent/multitype_queue/src/multitype_queue.cpp†L45-L132】
* 批次消費者同時監看 `messageQuantity` 位元組與 `m_batchInterval` 毫秒的逾時，透過大小輪詢配合 `steady_timer` 確保低吞吐時仍能定期返回結果。【F:src/agent/multitype_queue/src/multitype_queue.cpp†L133-L210】

## 訊息信封與命令項目

* 佇列項目使用 `Message` 包裝器，除了 JSON payload，還帶有 `moduleName`、`moduleType` 與選用的 metadata，方便在持久化流程間往返。【F:src/agent/multitype_queue/include/message_entry/message.hpp†L7-L46】
* 命令沿用 `module_command::CommandEntry` 資料模型；除了 `Parameters` JSON 外，還記錄模組路由、執行模式、時間戳與執行結果（狀態與訊息）。【F:src/agent/command_handler/include/command_entry/command_entry.hpp†L8-L83】

### JSON 欄位慣例

| 欄位 | 來源 | 說明 |
| --- | --- | --- |
| `data` | 佇列持久化資料列 | 儲存在佇列中的原始事件或命令 payload。 |
| `moduleName` | 佇列持久化資料列 | 模組生產者的邏輯識別，用於依模組逐一取出。 |
| `moduleType` | 佇列持久化資料列 | 生產者類別（例如 collector 或 handler）。 |
| `metadata` | 佇列持久化資料列 | 傳輸相關 metadata，例如重試提示。 |
| `parameters` | `CommandEntry` | 命令專用的 JSON 參數，依命令定義驗證。 |
| `result` | `CommandEntry::ExecutionResult.Message` | 供人閱讀的執行訊息。 |
| `status` | `CommandEntry::ExecutionResult.ErrorCode` | 序列化後的 `module_command::Status` enum。 |
| `execution_mode` | `CommandEntry::ExecutionMode` | `SYNC` 或 `ASYNC` 的派送模式。 |
| `time` | `CommandEntry::Time` | 命令寫入 store 時的毫秒級時間戳。 |

## Command Store 的 Schema 與生命週期

### SQLite Schema

`command_store` 資料表位於 `command_store.db`，於需要時建立，欄位如下。所有欄位皆為 `NOT NULL`，`id` 為主鍵。【F:src/agent/command_handler/src/command_store.cpp†L16-L96】

| 欄位 | 型別 | 意義 |
| --- | --- | --- |
| `id` | TEXT PK | Command 的 UUID。 |
| `module` | TEXT | 驗證後決定的目標模組。 |
| `command` | TEXT | 命令動詞。 |
| `parameters` | TEXT (JSON) | 序列化的參數。 |
| `execution_mode` | INTEGER | `0` 表示同步、`1` 表示非同步。 |
| `result` | TEXT | 最近一次執行訊息。 |
| `status` | INTEGER | 編碼後的 `module_command::Status`。 |
| `time` | REAL | 佇列入庫時的 epoch 毫秒。 |

列舉轉換器維持 enum 映射穩定：`StatusFromInt` 將 `0..3` 映射為 `SUCCESS`、`FAILURE`、`IN_PROGRESS`、`TIMEOUT`，其餘值視為 `UNKNOWN`；`ExecutionModeFromInt` 會把 `0` 視為 `SYNC`，其他值視為 `ASYNC`。【F:src/agent/command_handler/src/command_store.cpp†L28-L63】

### 處理狀態機

`CommandHandler::CommandsProcessingTask` 協調佇列讀取與狀態持久化，主要轉換如下：

1. **Idle → Fetch**：等待佇列命令；若無資料則睡眠 `WAIT_TIME`（1 秒）後重試。【F:src/agent/command_handler/src/command_handler.cpp†L47-L88】
2. **Fetch → Validate**：`CheckCommand` 逐一驗證命令與參數，從 `VALID_COMMANDS_MAP` 取得模組路由與執行模式；若驗證失敗，將狀態設為 `FAILURE` 並回報「Command is not valid」，再丟棄佇列項目。【F:src/agent/command_handler/src/command_handler.cpp†L18-L104】
3. **Validate → Persist (IN_PROGRESS)**：有效命令透過 `StoreCommand` 寫入 `command_store`；若發生資料庫錯誤，回報 `FAILURE` 與「Agent's database failure」後刪除佇列訊息。【F:src/agent/command_handler/src/command_handler.cpp†L90-L119】
4. **Persist → Dispatch**：
   * **同步命令**：等待 `dispatchCommand` 完成，並以回傳的 `ExecutionResult` 更新資料列後記錄完成訊息。【F:src/agent/command_handler/src/command_handler.cpp†L121-L131】
   * **非同步命令**：啟動分離的協程，在 dispatcher 完成後更新資料列並記錄結果。【F:src/agent/command_handler/src/command_handler.cpp†L132-L143】
5. **啟動清理**：在開始處理前，`CleanUpInProgressCommands` 會檢查殘留的 `IN_PROGRESS` 記錄——重新啟動的命令標記為 `SUCCESS`，其餘則標記為 `FAILURE`，維持至少一次的語意。【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】

### 與佇列的同步

* `getCommandFromQueue` 與 `popCommandFromQueue` 形成佇列與持久層的橋樑：命令會先被 peek 用於驗證與持久化，只有在寫入成功後才會真正 pop，避免驗證失敗時遺失訊息。【F:src/agent/command_handler/src/command_handler.cpp†L67-L120】
* `reportCommandResult` 會接收所有終態或清理後的更新，以便呼叫端（通常是佇列管理者）發出回應或指標。佇列依 `moduleName` 與 `moduleType` 儲存訊息，使消費者可以在取回時依模組篩選。【F:src/agent/multitype_queue/src/multitype_queue.cpp†L151-L210】

## 替換持久層時的 API 契約

若要以其他儲存引擎取代 SQLite，必須遵守以下保證：

1. **MultiType Queue 儲存 API**：任何 `IStorage` 實作都必須維持每張資料表的原子 `Store`、`Retrieve*` 與 `RemoveMultiple` 行為，確保訊息數量能正確驅動背壓。容量檢查（`GetElementCount`、`GetElementsStoredSize`）也要準確，才能正確套用 `shouldWait` 阻塞與協程計時器，維持生產端至少一次投遞。【F:src/agent/multitype_queue/include/istorage.hpp†L9-L87】【F:src/agent/multitype_queue/src/multitype_queue.cpp†L45-L210】
2. **Command Store 耐久性**：`ICommandStore` 後端需實作 `StoreCommand`、`UpdateCommand` 與 `GetCommandByStatus` 並具備交易式一致性，確保 `IN_PROGRESS` 紀錄能跨重啟保留並在新流程開始前清理。若回傳過期或不完整資料，會破壞 `CleanUpInProgressCommands` 的恢復流程。【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】【F:src/agent/command_handler/src/command_store.cpp†L98-L218】
3. **至少一次派送**：只有在持久化成功後才能從佇列移除命令，dispatcher 也必須在每次執行後恰好更新一次狀態與結果。替代儲存要支援以命令 `id` 為鍵的冪等更新，並確保重啟清理能辨識未完成工作（例如依 `status = IN_PROGRESS`）。【F:src/agent/command_handler/src/command_handler.cpp†L90-L171】
4. **重啟清理**：任何後端都必須提供列舉與更新 `IN_PROGRESS` 命令的能力，啟動時要能重新嘗試或標記失敗，對齊現行行為以清理懸掛的操作並向上游回報結果。【F:src/agent/command_handler/src/command_handler.cpp†L145-L171】

遵守上述契約，就能在不犧牲傳遞保證與命令處理流程的前提下導入其他儲存解決方案。
