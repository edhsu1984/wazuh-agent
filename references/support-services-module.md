# 支援服務模組說明

支援服務模組包含 Configuration Parser 與 Task Manager。兩者在 Agent 啟動時先行就緒，為核心通訊與功能模組提供組態來源與非同步執行環境。【F:src/agent/src/agent.cpp†L21-L198】

## Configuration Parser

1. **初始化與容錯**：建構子會讀取預設的 `wazuh-agent.yml`，若 YAML 結構無效則退回空節點並記錄警告，確保 Agent 不因組態錯誤而停止。【F:src/agent/configuration_parser/src/configuration_parser.cpp†L20-L63】
2. **集中化組態合併**：透過 `SetGetGroupIdsFunction` 注入群組查詢函式後，`LoadSharedConfig` 會逐一載入共享檔案並以 `MergeYamlNodes` 合併；`ReloadConfiguration` 會清空快取、重新讀取本地與共享設定。【F:src/agent/configuration_parser/src/configuration_parser.cpp†L97-L158】
3. **查詢介面**：`GetConfigOrDefault`、`GetConfigInRangeOrDefault`、`GetTimeConfig*` 與 `GetBytesConfigInRangeOrDefault` 提供型別安全的查詢與範圍驗證，遇到格式錯誤時回傳預設值並寫入警告。【F:src/agent/configuration_parser/include/configuration_parser.hpp†L45-L152】
4. **與核心整合**：`Agent` 啟動時會設定群組查詢、讀取執行緒數與路徑等參數，Module Manager 與各模組在 `Setup` 階段直接使用解析後的值。【F:src/agent/src/agent.cpp†L55-L198】【F:src/modules/src/moduleManager.cpp†L39-L160】

## Task Manager

1. **執行緒池管理**：`StartThreadPool` 會建立 `io_context` 工作守護與指定數量的執行緒；`Stop` 在終止時確保工作物件重設並等待所有執行緒結束。【F:src/agent/task_manager/src/task_manager.cpp†L16-L91】
2. **任務派送**：同步任務透過 `EnqueueTask(std::function)` 包裝例外處理並計數執行緒工作量；協程任務則以 `co_spawn` 啟動並在捕捉例外時記錄錯誤，維持背景作業穩定。【F:src/agent/task_manager/src/task_manager.cpp†L93-L144】
3. **輕量執行**：模組可呼叫 `RunSingleThread` 與 `CreateSteadyTimer` 建立單執行緒事件迴圈與逾時器，適合需自有計時或輪詢的模組（如 Logcollector）。【F:src/agent/task_manager/src/task_manager.cpp†L43-L163】【F:src/modules/logcollector/src/logcollector.cpp†L32-L169】

## 與其他元件的關聯

* Configuration Parser 為 Communicator、Command Handler、Module Manager 提供統一的設定入口，並在 Instance Communicator 觸發重載時更新模組行為。【F:src/agent/src/agent.cpp†L93-L206】
* Task Manager 同時被 Agent 核心與各模組使用：Communicator 的 HTTP/2 協程、命令處理迴圈與模組任務都透過它排程，確保關閉流程能統一停止所有背景工作。【F:src/agent/src/agent.cpp†L134-L220】

這兩個支援模組確保 Agent 的設定、排程與協程運作具備一致性與可管理性。
