# 功能模組（Collectors / Executors）運作筆記

功能模組由 Module Manager 統一註冊、套用設定並啟動；Collectors 專職資料蒐集並寫入 MultiType Queue，Executors 則處理由 Manager 下達的命令或長時作業。

## 模組生命週期

1. **註冊**：`ModuleManager::AddModules()` 會根據建置選項建立 Inventory、Logcollector、SCA 等模組實例，並將 Agent UUID 傳遞給需要的模組。【F:src/modules/src/moduleManager.cpp†L39-L58】
2. **依賴注入**：每個模組啟動前都會設定 pushMessage 函式與共用的 Configuration Parser，確保後續能寫入 MultiType Queue 並存取組態。【F:src/modules/src/moduleManager.cpp†L63-L160】
3. **啟動監控**：Module Manager 以 Task Manager 執行緒池跑每個模組的 `Run()`，並等待所有模組在指定時間內回報啟動狀態；若超時則記錄錯誤。【F:src/modules/src/moduleManager.cpp†L112-L151】
4. **重載與關閉**：模組可在集中化設定更新時熱重載；停用時 Module Manager 會呼叫每個模組的 `Stop()` 並關閉執行緒池。【F:src/modules/src/moduleManager.cpp†L86-L171】

## Collectors

Collectors 會將事件序列化為 `Message` 後推送至 MultiType Queue，通常包含 STATELESS 或 STATEFUL 負載。

| 模組 | 核心責任 | 組態/實作重點 |
| --- | --- | --- |
| Logcollector | 根據設定建立多個檔案讀取器，透過協程輪詢來源並推送 stateless 訊息。 | `Setup` 解析啟用旗標、讀取/重新載入間隔與來源列表；每個 reader 透過 `PushMessage` 將資料與 metadata 打包後送入 Queue，Task Manager 追蹤協程生命週期。【F:src/modules/logcollector/src/logcollector.cpp†L24-L169】 |
| File Integrity Monitoring (FIM) | 監控檔案與目錄，支援排程掃描與 inotify 實時模式。 | `main` 讀取設定、初始化 rootcheck 與訊息佇列；根據資源狀況決定是否啟用即時監控，並在迴圈中處理事件。【F:src/modules/fim/src/main.c†L55-L220】 |
| Inventory | 收集系統、硬體、網路等資產資訊，產生 stateful 與 stateless 差異訊息。 | `Setup` 讀取各項掃描旗標與資料庫路徑；`Run` 建立 `SysInfo` 並註冊差異回呼，`PushMessage` 會同時寫入 STATEFUL/STATELESS 事件以描述資產變更。【F:src/modules/inventory/src/inventory.cpp†L9-L153】 |
| Security Configuration Assessment (SCA) | 排程政策掃描並回報檢查結果。 | 建構子建立 SQLite 儲存；`Setup` 載入政策、建立 Task Manager 並註冊 `SCAEventHandler` 回寫 Queue，掃描任務以協程週期執行。【F:src/modules/sca/src/sca.cpp†L35-L146】 |

## Executors / 主動反應模組

Executors 主要透過命令佇列觸發，執行動態任務或主動回應。

| 模組 | 核心責任 | 佇列/執行細節 |
| --- | --- | --- |
| Active Response (execd) | 接收命令佇列指令並啟動對應腳本，處理重複觸發與子程序管理。 | 主迴圈 `ExecdStart` 監聽 UNIX 佇列、分派命令、追蹤子程序數量並處理逾時，確保重複事件會套用節流邏輯。【F:src/modules/active_response/src/execd.c†L415-L520】 |
| Module Executors（例如模組的 `ExecuteCommand`） | 讓模組回應 Command Handler 派發的同步/非同步命令。 | 每個模組實作 `ExecuteCommand` 時會檢查啟用狀態與 Task Manager 是否運作中，並回傳 `CommandExecutionResult` 供 Command Handler 更新結果。【F:src/modules/logcollector/src/logcollector.cpp†L105-L121】【F:src/modules/inventory/src/inventory.cpp†L83-L98】【F:src/modules/sca/src/sca.cpp†L119-L131】 |

## 與 Command Handler / MultiType Queue 的整合

1. Module Manager 啟動時會把 `pushMessage` 函式注入所有模組，使其可直接呼叫 MultiType Queue 寫入事件或命令結果。【F:src/modules/src/moduleManager.cpp†L21-L128】
2. Command Handler 接獲命令後會以模組名稱查詢 Module Manager，並透過模組的 `ExecuteCommand` 回傳結果；若模組停止則回報失敗並記錄訊息。【F:src/agent/src/agent.cpp†L174-L195】【F:src/agent/command_handler/src/command_handler.cpp†L91-L143】
3. Collectors/Executors 都可使用相同的 Queue 基礎設施，享有背壓、批次上傳與持久化保護，確保事件與命令結果在重啟後不遺失。【F:src/agent/multitype_queue/src/multitype_queue.cpp†L45-L210】

藉由上述流程，功能模組能在獨立執行緒中運行，同時透過統一的佇列與命令協調機制與核心元件協作。
