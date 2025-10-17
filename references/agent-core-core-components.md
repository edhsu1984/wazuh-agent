# Wazuh Agent 核心元件實作指南

本指南聚焦於 Agent 啟動時綁定的 HTTP/2 Client、Communicator、MultiType Queue 與 Command Handler，說明它們的責任、互動與資料流細節。

## 啟動與組態綁定

1. **建構階段**：`Agent` 會建立或注入 Configuration Parser、AgentInfo、MultiType Queue、Communicator、Module Manager 與 Command Handler 等元件，並驗證 enrollment 狀態。【F:src/agent/src/agent.cpp†L21-L90】
2. **組態來源**：Configuration Parser 提供 thread count、資料路徑與群組資訊；MultiType Queue 以 Parser 讀到的容量與批次時間進行初始化，確保後續背壓與批次行為一致。【F:src/agent/src/agent.cpp†L78-L86】【F:src/agent/multitype_queue/src/multitype_queue.cpp†L17-L100】
3. **執行緒池啟動**：`Run()` 會啟動 Task Manager 執行緒池，接著安排認證、命令抓取、事件上傳、模組啟動與命令處理等協程任務。【F:src/agent/src/agent.cpp†L134-L198】

## 元件責任區分

### HTTP/2 Client

HTTP/2 Client 封裝 TLS 連線、HTTP/2 交握與請求發送，供 Communicator 與其他背景任務重複使用。【F:src/agent/http_client/src/http_client.cpp†L81-L181】

### Communicator

* 管理對 Manager 的認證、token 更新與逾時控制，若失敗會套用重試與重新認證策略。【F:src/agent/communicator/src/communicator.cpp†L94-L210】【F:src/agent/communicator/src/communicator.cpp†L295-L355】
* 以 `ExecuteRequestLoop` 建立常駐協程，負責命令輪詢以及 STATEFUL/STATELESS 事件的批次上傳，並在需要時向 Module Manager 提供集中化設定檔案。【F:src/agent/src/agent.cpp†L141-L170】【F:src/agent/communicator/src/communicator.cpp†L257-L355】

### MultiType Queue

* 維護 STATELESS、STATEFUL 與 COMMAND 三種表格，並根據容量限制與 `QUEUE_STATUS_REFRESH_TIMER` 應用背壓；支援同步與協程 API，以服務不同生產/消費者。【F:src/agent/multitype_queue/include/multitype_queue.hpp†L19-L123】【F:src/agent/multitype_queue/src/multitype_queue.cpp†L17-L210】
* Module Manager 與 Command Handler 皆透過 Queue 交換訊息，Collectors/Executors 將事件與命令結果 push 入隊，HTTP/2 任務則批次抓取後上傳。【F:src/agent/src/agent.cpp†L143-L189】【F:src/modules/src/moduleManager.cpp†L39-L151】

### Command Handler

* `CommandsProcessingTask` 持續從 Queue 取得命令：驗證參數、寫入 `command_store`、在同步模式下等待 dispatcher 回覆，非同步模式則啟動協程後返回，最後更新結果並回報。【F:src/agent/command_handler/src/command_handler.cpp†L67-L143】
* `CleanUpInProgressCommands` 於啟動時恢復未完成的命令，將重啟流程標記成功、其餘標記為失敗，確保狀態一致性。【F:src/agent/command_handler/src/command_handler.cpp†L145-L168】
* `Agent::Run()` 中將命令交由 Module Manager 或內建處理器（中央化設定、Restart Handler）分派，並在停止時同步關閉 Command Handler、Communicator 與 Module Manager。【F:src/agent/src/agent.cpp†L174-L220】

## 資料流概覽

1. **事件產生**：Module Manager 管理的 Collectors 將資料寫入 MultiType Queue；`Stateful/StatelessMessageProcessingTask` 會以批次方式拉取並呼叫 Communicator 上傳。【F:src/modules/src/moduleManager.cpp†L39-L151】【F:src/agent/src/agent.cpp†L147-L170】
2. **命令回應**：Communicator 從 Manager 拉取命令後，透過 `PushCommandsToQueue` 寫入 COMMAND 表；Command Handler peek 佇列、驗證並落盤，再交由模組或系統指派執行，結果回寫 Queue 供上傳。【F:src/agent/src/agent.cpp†L141-L196】【F:src/agent/command_handler/src/command_handler.cpp†L91-L143】
3. **重載與集中化設定**：Instance Communicator 監聽 UNIX socket 訊息觸發模組重載；Communicator 則可下載集中化設定並透過 Module Manager 更新，維持設定一致。【F:src/agent/src/agent.cpp†L171-L206】【F:src/agent/communicator/src/communicator.cpp†L316-L353】

透過上述協同流程，Agent 能夠同時維持下行命令可靠執行與上行事件穩定傳輸。
