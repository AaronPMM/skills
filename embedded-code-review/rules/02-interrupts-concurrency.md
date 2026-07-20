# 規則 02:中斷與並發

中斷相關 bug 是嵌入式最難重現的一類:低機率、時序相關、現場才爆發。審核時對「ISR 與其他上下文共享的每一個變數」逐一過帳,不要抽查。

## 2.1 ISR 基本紀律(BLOCKER 級)

- ISR 必須短:只做取數據、清標誌、通知(設事件/發訊息隊列),重活交給任務/主循環。ISR 中出現迴圈等待、`printf`、浮點運算(無硬體 FPU 或未做 lazy stacking 配置時)、長計算都要標記。
- ISR 中禁止:`malloc`/`free`、阻塞式 API(mutex 獲取、無 timeout=0 的隊列操作)、非 ISR-safe 的 RTOS API(FreeRTOS 中必須用 `xxxFromISR` 版本)。
- **中斷標誌清除**:確認 ISR 正確清除了硬體中斷標誌,且清除時機正確(過早清可能丟事件,不清會無限重入)。注意某些外設是「讀暫存器即清除」,重複讀會丟資料。
- ISR 是否可能重入 / 是否被同優先級中斷打斷,依平台確認(Cortex-M 同優先級不搶佔,但要檢查 NVIC 優先級配置)。

## 2.2 共享資料保護

對每個被「ISR 與任務」或「多任務」共享的變數,回答三個問題:**誰寫?誰讀?讀-改-寫是否原子?**

- **讀-改-寫非原子**:`flag |= BIT;`、`count++` 在 C 層面是一條語句,在指令層面是 load-modify-store,中間可被中斷。共享變數的 RMW 必須在臨界區內,或使用原子操作(C11 `_Atomic`、LDREX/STREX、或關中斷)。
- **超過字寬的資料**:32 位 MCU 上讀寫 64 位變數、結構體、多個關聯變數(如 `head` 和 `count`)不是原子的,撕裂讀 (torn read) 必須用臨界區保護。
- **volatile 的正確角色**:ISR 與任務共享的變數必須 `volatile`(防止編譯器快取到暫存器/優化掉輪詢迴圈),但 **volatile 不提供原子性、不提供記憶體屏障**。「加了 volatile 所以安全」是需要駁回的常見誤解。
- 臨界區實現檢查:
  - 關中斷保護:確認恢復的是「先前狀態」而非無條件開中斷(嵌套臨界區會被破壞)——應使用 save/restore 模式(如 `PRIMASK` 保存恢復)。
  - 臨界區內不得呼叫可能阻塞或耗時的函數。
  - 關中斷時長影響中斷延遲,超過幾微秒量級的臨界區要質疑。

## 2.3 競態的典型形態(逐一比對)

- **check-then-act**:`if (buf_not_full) { put(buf); }` 檢查和動作之間狀態被改變。
- **回呼註冊競態**:`callback = NULL;` 與 ISR 中 `if (callback) callback();` ——ISR 可能在判斷後、呼叫前被打斷?單寫單讀且指標寬度原子時安全,但「先設 context 再設 callback」的順序必須正確且有屏障。
- **標誌位溝通**:任務清標誌與 ISR 設標誌若用 RMW 操作同一個位元組/字,互相覆蓋。
- **DMA 與 CPU 共享緩衝區**:DMA 進行中 CPU 讀寫同一緩衝區;有 D-Cache 的平台(Cortex-M7 等)必須檢查 cache 清理/失效操作(`clean` before DMA-TX、`invalidate` before讀 DMA-RX 結果),以及緩衝區是否 cache line 對齊。
- **雙核/多核**(ESP32、RP2040 等):關中斷只擋得住本核,跨核共享必須用 spinlock/硬體互斥。

## 2.4 記憶體順序與編譯器優化

- 編譯器和 CPU 都可能重排。以「資料就緒標誌」通知另一上下文時,資料寫入與標誌設置之間需要屏障(`__DMB()`、C11 release/acquire、或依賴 volatile 順序——注意 volatile 只約束 volatile 之間的順序,不約束普通變數)。
- busy-wait 輪詢的變數若非 volatile,`-O2` 下迴圈會被優化成死迴圈——這種 bug 在 Debug 版 (-O0) 不出現,是「Release 才壞」的經典原因。
