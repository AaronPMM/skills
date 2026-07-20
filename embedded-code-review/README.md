# embedded-code-review

嵌入式 C/C++ 代碼審核 skill,供 Claude Code 使用。

## 結構

```
embedded-code-review/
├── SKILL.md                        # 主入口:審核流程、嚴重度分級、報告格式
└── rules/
    ├── 01-memory-safety.md         # 緩衝區、堆疊、動態記憶體、指標、初始化
    ├── 02-interrupts-concurrency.md# ISR 紀律、共享資料、競態、記憶體順序
    ├── 03-types-arithmetic.md      # 有號/無號、整數提升、溢位、定時器回繞、浮點
    ├── 04-hardware-io.md           # 暫存器、輪詢超時、DMA、看門狗、Flash
    ├── 05-rtos.md                  # ISR-safe API、死鎖、堆疊、隊列、時間
    ├── 06-defensive-coding.md      # 返回值、控制流、宏、斷言(MISRA 風格)
    ├── 07-portability-cpp.md       # 對齊、字節序、編譯器依賴、嵌入式 C++
    ├── 08-embedded-linux.md        # Linux userspace(syscall/signal/pthreads/RT)
    │                               # 與 kernel 驅動(原子上下文/copy_from_user/devm)
    └── 09-performance.md           # WCET/即時路徑、RAM/Flash 佔用、功耗、I/O 批次化
```

支援三類平台:**裸機**、**RTOS**、**嵌入式 Linux**(userspace 與 kernel 驅動)。SKILL.md 會依代碼推斷平台並只載入適用的規則(例如 Linux userspace 不套用暫存器/RTOS 規則)。

## 安裝

二選一:

```bash
# 個人級(所有專案可用)
ln -s "$(pwd)/embedded-code-review" ~/.claude/skills/embedded-code-review

# 專案級(只在某個韌體專案中可用)
cp -r embedded-code-review /path/to/your/firmware/.claude/skills/
```

## 使用

在 Claude Code 中:

- `/embedded-code-review` 直接調用;或
- 自然語言觸發:「幫我審核這個驅動」「review 一下這段 ISR 代碼」等。

可在指令後附上範圍,例如 `/embedded-code-review drivers/uart.c` 或先 `git diff` 再要求審核當前變更。

## 客製化

- 專案有自己的編碼規範時,在 `rules/` 下新增 `08-project-specific.md` 並在 SKILL.md 的規則文件表格中登記。
- 各規則中的門檻值(如區域陣列 256 位元組上限)可依專案堆疊預算調整。
- 若專案正式遵循 MISRA C:2012 / CERT C,SKILL.md 已指示引用規則編號;可將完整的偏差 (deviation) 清單放進 `rules/` 供對照。
