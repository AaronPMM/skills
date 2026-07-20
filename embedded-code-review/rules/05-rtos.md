# 規則 05:RTOS 使用

適用於 FreeRTOS、Zephyr、RT-Thread、ThreadX、embOS 等。API 名稱以 FreeRTOS 為例,審核時對應到專案實際使用的 RTOS。

## 5.1 上下文正確性(BLOCKER 級)

- **ISR 中只能用 ISR-safe API**:FreeRTOS 的 `xQueueSendFromISR`/`xSemaphoreGiveFromISR` 等;在 ISR 中呼叫普通版本(可能觸發任務切換或阻塞)是 BLOCKER。
- `FromISR` API 的 `xHigherPriorityTaskWoken` 是否正確傳出並在 ISR 尾部呼叫 `portYIELD_FROM_ISR()`——漏掉會造成高優先級任務延遲響應(到下一個 tick 才切換)。
- 呼叫 RTOS API 的中斷,其 NVIC 優先級必須在 RTOS 管理範圍內(FreeRTOS: 數值 ≥ `configMAX_SYSCALL_INTERRUPT_PRIORITY`)。配置錯誤會造成偶發的資料結構損毀,極難排查——看到中斷優先級設定代碼必查。
- 調度器啟動前不得呼叫依賴調度器的 API(阻塞延遲、互斥鎖獲取)。

## 5.2 互斥與死鎖

- **mutex vs 二值信號量**:保護共享資源必須用 mutex(有優先級繼承);用二值信號量做互斥會有優先級反轉風險。反之,ISR→任務的同步用信號量/任務通知,不能用 mutex(ISR 不能拿 mutex)。
- 死鎖模式:兩個任務以不同順序獲取兩把鎖;持鎖時呼叫可能阻塞在另一把鎖上的函數;持鎖時等待需要該鎖才能完成的事件。要求全專案一致的鎖獲取順序。
- 每條路徑(含錯誤提前返回)都要釋放鎖。C++ 中優先 RAII 鎖守衛;C 中檢查所有 return/goto 路徑。
- 遞迴獲取非遞迴 mutex 是死鎖。
- 臨界區/持鎖區內呼叫阻塞 API、耗時操作、`printf`(內部可能有鎖或阻塞)要標記。

## 5.3 阻塞與超時

- 阻塞 API 一律要求顯式超時策略:`portMAX_DELAY`(永久等待)只允許用在「等不到就沒有意義」的場合,且需說明理由;等待硬體/通訊相關事件必須有限超時 + 失敗處理。
- 超時返回值必須檢查:`xQueueReceive` 超時返回後使用未填充的接收緩衝區是常見 bug。
- 高優先級任務中的長阻塞/長計算會餓死低優先級任務;檢查任務優先級分配與各任務的執行時長是否匹配(CPU 密集的任務應在低優先級)。

## 5.4 堆疊與資源

- 每個任務的堆疊大小依據:函數呼叫深度、區域變數(含 `printf` 系列的大量堆疊消耗)、ISR 疊加。新增任務/在任務中新增深呼叫鏈時,質疑堆疊是否重新評估過。建議使用高水位檢測 (`uxTaskGetStackHighWaterMark`)。
- 任務動態創建/刪除:`vTaskDelete` 不會釋放任務自行分配的資源;被刪任務持有的鎖永不釋放。長期運行系統建議任務只建不刪。
- 隊列滿的行為:發送方阻塞、丟棄、還是覆寫?ISR 中隊列滿丟資料是否可接受、是否需要計數上報。

## 5.5 時間與延遲

- `vTaskDelay`(相對延遲)vs `vTaskDelayUntil`(絕對週期):週期性任務必須用後者,否則週期漂移。
- tick 精度限制:`vTaskDelay(1)` 實際延遲在 0~1 tick 之間;短於 1 tick 的精確延時要用硬體定時器/DWT。
- 忙等待 (`while` + 查詢) 在任務中浪費 CPU 且可能餓死低優先級任務,應改為阻塞等待事件。

## 5.6 資料傳遞

- 透過隊列傳指標:指標指向的資料在接收方使用完之前必須保持有效——傳送區域變數位址是 BLOCKER;傳 static 緩衝區位址要檢查覆寫競態。
- 任務通知 (task notification) 比隊列/信號量輕量,但只有一個槽位:多事件源通知同一任務時會合併/丟失,檢查是否可接受。
- 事件組 (event group) 的清除時機:`xEventGroupWaitBits` 的 clear-on-exit 與多個等待者的交互。
