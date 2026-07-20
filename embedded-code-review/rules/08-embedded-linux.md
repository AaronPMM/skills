# 規則 08:嵌入式 Linux

適用於跑 Linux 的嵌入式平台(i.MX、AM335x、樹莓派、RK 系列等)。分 userspace 與 kernel 兩部分,按代碼所在層載入對應章節。注意:規則 04(裸機暫存器)與規則 05(RTOS)大部分不適用於 Linux userspace,不要誤套。

## A. Userspace

### 8.1 系統呼叫與錯誤處理

- 每個 syscall / libc 呼叫的返回值必查,錯誤時讀 `errno`。注意 `errno` 只在返回值指示失敗時才有意義。
- **EINTR**:`read`/`write`/`poll`/`sem_wait` 等可被 signal 打斷返回 EINTR,必須重試(或用 `SA_RESTART`,但它不覆蓋所有呼叫)。漏處理 EINTR 的代碼在接上 signal(定時器、子進程回收)後開始偶發失敗。
- **短讀/短寫 (partial read/write)**:`read`/`write`/`recv`/`send` 返回的字節數可能小於請求量(尤其 socket、pipe、串口),必須迴圈直到完成或用包裝函數。`write` 一次寫完是不能依賴的假設。
- 串口 (termios) 配置:檢查 `VMIN`/`VTIME` 語義與讀取邏輯是否匹配、raw 模式是否正確設置(`cfmakeraw`)、波特率設置後 `tcsetattr` 返回成功不代表所有欄位都生效。

### 8.2 Signal 與進程

- Signal handler 中只能呼叫 async-signal-safe 函數(`printf`/`malloc`/大部分庫函數都**不安全**);慣用法是 handler 只設 `volatile sig_atomic_t` 標誌或寫 self-pipe/`signalfd`,主循環處理。
- `SIGPIPE`:向已關閉的 socket/pipe 寫入默認殺死進程——長期運行的 daemon 必須忽略 SIGPIPE 或用 `MSG_NOSIGNAL`。
- fork/exec:fork 後子進程繼承所有 fd——打開 fd 時用 `O_CLOEXEC`/`SOCK_CLOEXEC`;多執行緒程式中 fork 後只能呼叫 async-signal-safe 函數直到 exec。
- 子進程必須回收(`waitpid`/`SIGCHLD`),否則殭屍進程累積。
- fd 洩漏:每條錯誤路徑檢查 close;長期運行 daemon 的 fd 洩漏最終耗盡 `RLIMIT_NOFILE`。

### 8.3 執行緒 (pthreads)

- 規則 02 的共享資料原則適用,但工具不同:用 `pthread_mutex`/C11 atomics,**不得**用「關中斷」思維;`volatile` 在多執行緒同步中無效(既不原子也無屏障),看到用 volatile 做執行緒同步就是問題。
- 條件變數:等待必須在迴圈中重查條件(虛假喚醒 spurious wakeup);signal 前是否持鎖與喚醒丟失的關係要檢查。
- 即時執行緒與普通執行緒共享 mutex 時,優先級反轉需要 `PTHREAD_PRIO_INHERIT` 屬性(Linux 默認不開)。
- 執行緒取消 (`pthread_cancel`) 幾乎總是錯誤的設計(取消點、資源清理難以正確),應改為標誌+喚醒。

### 8.4 即時性 (RT)

- Linux 默認調度不保證即時性;有時序要求的執行緒檢查:`SCHED_FIFO`/`SCHED_RR` + 合理優先級、`mlockall(MCL_CURRENT|MCL_FUTURE)` 防頁面錯誤、預觸碰堆疊/堆。
- SCHED_FIFO 執行緒中的死迴圈會餓死整個系統(包括 ssh)——RT 執行緒必須有阻塞點,建議保留 RT throttling 或看門狗。
- **時鐘選擇**:超時/週期計算必須用 `CLOCK_MONOTONIC`,不得用 `CLOCK_REALTIME`/`gettimeofday`(NTP 跳變、手動改時間會使定時器爆走)。`clock_nanosleep` + `TIMER_ABSTIME` 做週期任務(對應 RTOS 的 vTaskDelayUntil)。
- `usleep`/`nanosleep` 的實際精度受 tick/HZ 和調度影響,毫秒級以下的精確時序要質疑其可行性。

### 8.5 硬體存取(userspace 方式)

- GPIO:sysfs GPIO (`/sys/class/gpio`) 已棄用,新代碼應用 libgpiod(字符設備 `/dev/gpiochipN`)。
- `/dev/mem` + mmap 直接操作暫存器:指標必須 volatile;這種方式繞過核心驅動,與同外設的核心驅動並存時是衝突源,審核時質疑是否應改用 UIO 或正式驅動。
- i2c-dev/spidev:ioctl 返回值必查;I2C 交易的組合讀寫要用 `I2C_RDWR` 一次完成,分開的 write+read 之間可被其他進程插入。
- 設備熱插拔:設備節點可能消失(USB 串口拔出),read 返回 0/錯誤後的重連邏輯。

### 8.6 資源與穩健性

- Linux 預設 overcommit:`malloc` 幾乎不返回 NULL,真正的失敗是 OOM killer 直接殺進程——關鍵 daemon 檢查是否需要調整 `oom_score_adj`、限制自身記憶體增長(隊列有界、快取有上限)。
- 長期運行洩漏:除記憶體外,還有 fd、mmap 區域、pthread(未 join 且未 detach)、System V IPC。
- **掉電安全寫檔**:配置/資料落盤必須「寫臨時檔 → fsync(檔) → rename → fsync(目錄)」;直接覆寫原檔在掉電時損毀。flash 檔案系統(ubifs 等)與 SD 卡的寫入耐久也要納入考慮(高頻寫日誌要質疑)。
- daemon 與 systemd:服務異常退出的 `Restart` 策略、`WatchdogSec`(`sd_notify` 餵狗)是否配置;啟動順序依賴(網路、設備節點就緒)不得用 sleep 硬等。

## B. Kernel(驅動/模組代碼)

### 8.7 上下文規則(BLOCKER 級)

- **原子上下文不得睡眠**:中斷處理器、持 spinlock 期間、`preempt_disable` 區間內,禁止呼叫可能睡眠的函數——`mutex_lock`、`kmalloc(GFP_KERNEL)`、`copy_from_user`、`msleep` 等。原子上下文分配用 `GFP_ATOMIC`。
- 與中斷處理器共享資料的 spinlock 必須用 `spin_lock_irqsave`(否則同 CPU 上被 IRQ 搶佔造成死鎖)。
- 耗時工作放 threaded IRQ / workqueue,hard IRQ handler 保持最短。

### 8.8 用戶指標與資料

- 來自 userspace 的指標**永不直接解引用**,必須 `copy_from_user`/`copy_to_user` 並檢查返回值;長度欄位先驗證上限再拷貝(ioctl 處理器是重災區)。
- ioctl 的 cmd/arg 驗證、compat(32 位 userspace 對 64 位 kernel)結構體佈局差異。

### 8.9 資源與生命週期

- 優先使用 `devm_*` 管理資源(devm_kzalloc、devm_request_irq…),否則 probe 失敗路徑必須以 goto 鏈精確反向解除已申請資源——probe 錯誤路徑是 kernel 驅動洩漏的高發區。
- 模組卸載/設備移除競態:remove 時必須確保中斷已釋放、workqueue/timer 已 cancel(用 `cancel_work_sync`/`del_timer_sync` 同步版本)、無在途的回呼仍會觸發。
- 暫存器存取用 `ioremap` + `readl`/`writel` 訪問器,不得直接解引用物理位址強轉的指標。
- DMA 用 DMA API(`dma_alloc_coherent`/`dma_map_single` + sync),不得對 `kmalloc` 緩衝區直接給硬體物理位址;方向參數與實際傳輸方向一致。
- Device tree:驅動讀取的屬性要有缺失時的默認值或明確報錯,不得默默用垃圾值。
