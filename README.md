# HIIT 運動間歇計時器 - 專案規格書

## 1.情境故事
- **誰、在哪、遇到什麼麻煩，裝置怎麼幫忙**：使用者在家進行高強度間歇訓練（HIIT）時，常因桌面上擺滿多個雜亂計時器、盯著手機碼錶而無法專心訓練。本裝置透過板載按鍵一鍵啟動，搭配雙色 LED（綠燈運動、紅燈休息）與蜂鳴器即時聲光提示，讓使用者無需分心看螢幕即可精準掌握節奏，並透過電腦端 UART 終端機自動記錄與監控每秒狀態。

---

## 2.替身對照
- 雲端 App / 藍牙數據連線 <-> 電腦端 UART 序列埠 (pyserial miniterm)
- 雙色 LED 指示燈 / RGB 模組 <-> 外接紅色 LED + 綠色 LED (GPIO 控制)
- 商業級開關 / 鍵盤組元件 <-> 板載 B1 按鈕 (PC13) 或外接按鈕 (軟體去彈跳)
- DS3231 RTC 模組 / 外部晶脈 <-> STM32 TIM2 定時器中斷 (精準時脈)
- 主動式蜂鳴器 / 音訊放大電路 <-> Timer PWM 輸出 (經 NPN 電晶體動蜂鳴器)
- 觸控屏幕界面 / 旋鈕開關 <-> Python 終端機指令輸入 (UART 接收解析)
- microSD 卡讀寫模組 <-> 電腦端 Python 指令與 pyserial 日誌紀錄

---

## 3.功能清單
- 1. 按鍵控制與狀態切換：按下 B1 按鈕（PC13）觸發外部中斷，透過軟體去彈跳（ticks - t_btn_last >= 50ms），實現「按一下只算一次」的開始、暫停與狀態切換。
- 2. 雙色燈號與蜂鳴器提示：運動狀態亮綠燈（PA5）；休息狀態亮紅燈（PA6），休息結束前 10 秒紅燈開始連續閃爍，休息結束前 3 秒蜂鳴器（PA0）發出提醒逼逼聲。
- 3. UART 參數動態設定：透過電腦端輸入 `set <sets> <exercise> <rest>` 指令（例如 `set 2 10 10`），透過 UART 中斷與環形緩衝區解析，動態調整組數與秒數；輸入 `new` 則可重置系統回 IDLE 狀態。
- 4. 即時數據監控回報：每次計時事件觸發時，透過 `USART2` (`115200 8N1`) 即時傳送 `[REPORT]` 狀態、當前組數與剩餘秒數至電腦端終端機。

---

## 4.技術與暫存器
- 計時器基准：使用 TIM2 定時器，設定 PSC = 169、ARR = 999，產生 1 ms 中斷 (UIE)，搭配 1000 次計數精準計算 1 秒倒數，並啟用 NVIC 中斷。
- 外部中斷 (EXTI)：設定 EXTICR4 選定 PC13（B1 按鈕），配置為下降緣觸發（RTSR1/FTSR1、IMR1、PR1 寫 1 清除），並致能 EXTI15_10_IRQn。
- GPIO 控制：運動綠燈（PA5）與休息紅燈（PA6）使用 MODER = 01（Output），透過 BSRR 暫存器進行設值與清除。
- 序列埠通訊 (UART)：使用 USART2 (PA2/PA3，AFR 選 AF7)，波特率 115200，啟用 RXNE 接收中斷與主迴圈出列解析。

---

## 5.腳位表
- PA5 = 運動狀態指示燈（綠燈），GPIO_Output，Push-Pull, No pull, Low
- PA6 = 休息狀態指示燈（紅燈），GPIO_Output，Push-Pull, No pull, Low
- PA0 = 蜂鳴器提醒，GPIO_Output，Push-Pull, No pull, Low
- PC13 = B1 按鈕，GPIO_EXTI13，Falling Edge, Pull-up（觸發開始/暫停與狀態切換）
- PA2 / PA3 = 序列埠通訊 (USART2 TX/RX)，波特率 115200 8N1，致能 RX 中斷 (AF7)

---

## 6.驗收條件
- **UART 指令驗收**：打 `set 2 10 10`，看 pyserial 終端機發送設定指令 -> 顯示 `[CONFIG OK] Sets:2, Ex:10s, Rest:10s`。
- **按鍵與回報驗收**：按 B1 按鈕一次，看綠燈（PA5）亮起，終端機印出 `[INFO] Start HIIT! Set 1/2` 且每秒印出狀態。
- **燈號閃爍驗收**：休息倒數至 10 秒時，紅燈（PA6）以 1 秒為週期連續閃爍。
- **蜂鳴器驗收**：休息倒數至 3 秒時，聽見蜂鳴器（PA0）發出提醒逼逼聲，紅燈持續閃爍。
- **完成與重置驗收**：全部組數執行完畢時，紅綠燈與蜂鳴器全關，終端機顯示 `[FINISH] HIIT Workout Complete!`；打 `new` 能將計時器重置為待機，印出 `[RESET] HIIT Reset to IDLE`。
- **通訊穩定驗收**：115200 8N1 鮑率設定下，終端機回傳文字全程顯示清晰無亂碼。

---

## 📁 專案目錄結構 (Project Directory)
專案的檔案組織結構如下，採用 STM32CubeMX 標準架構進行模組化開發：

HIIT/
│
├── Core/
│   ├── Inc/                    # 標頭檔資料夾
│   └── Src/                    # 原始程式碼資料夾
│       ├── main.c              # 主程式、狀態機與中斷回調
│       ├── stm32g4xx_hal_msp.c # MCU 外部週邊底層初始化
│       ├── stm32g4xx_it.c      # 中斷服務函數 (ISR)
│       ├── syscalls.c          # 系統呼叫支援
│       ├── sysmem.c            # 記憶體管理支援
│       └── system_stm32g4xx.c  # 系統時脈設定
│
├── Drivers/                    # STM32G4xx HAL 驅動與 CMSIS 檔案
├── Startup/                    # 啟動組合語言檔案 (.s)
├── Debug/                      # 編譯輸出的目標與偵錯檔案
├── HIIT.ioc                    # STM32CubeMX 專案設定檔
├── HIIT Debug.launch           # 偵錯設定檔
├── STM32G474RETX_FLASH.ld      # Flash 連結器腳本
└── README.md                   # 專案說明與規格書

## 💻 核心程式碼部分 (Core Code Snippets)

1. 按鍵去彈跳處理 (EXTI 外部中斷)
確保按下 B1 按鈕（PC13）時「按一下只算一次」，並切換運行與暫停狀態：
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_13) {
        uint32_t ticks = HAL_GetTick();
        if ((ticks - t_btn_last) >= 50) { // 軟體去彈跳 50ms
            t_btn_last = ticks;
            if (is_running == 0) {
                is_running = 1;
                if (current_state == STATE_IDLE) {
                    current_set = 1;
                    current_state = STATE_EXERCISE;
                    remaining_time = exercise_time;
                }
            } else {
                is_running = 0;
            }
        }
    }
}

2. 1ms 定時中斷與倒數計時邏輯 (TIM2)
利用 TIM2 產生 1ms 基準，精準計算每秒倒數，並於休息結束前 10 秒閃爍紅燈、3 秒前觸發蜂鳴器：
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        if (!is_running) return;

        ms_counter++;
        if (ms_counter >= 1000) { // 滿 1 秒
            ms_counter = 0;
            if (remaining_time > 0) {
                remaining_time--;
                
                // 狀態機邏輯與聲光提示
                if (current_state == STATE_EXERCISE) {
                    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);   // 綠燈亮
                    if (remaining_time == 0) {
                        current_state = STATE_REST;
                        remaining_time = rest_time;
                    }
                } else if (current_state == STATE_REST) {
                    // 休息倒數 10 秒紅燈閃爍、3 秒蜂鳴器響起...
                }
            }
        }
    }
}


## 🔀 系統狀態流程圖與說明 (State Machine Flow)

[ STATE_IDLE ]（待機狀態）
  │
  ▼ [按 B1 / 接收指令]
[ STATE_EXERCISE ]（運動狀態：綠燈恆亮）
  │
  ▼ [時間歸零]
[ STATE_REST ]（休息狀態：紅燈 / 閃爍 / 蜂鳴器）
  ├── [尚未完成組數] ──> 回到運動狀態
  └── [組數執行完畢] ──> 結束訓練切回 IDLE

### 流程圖文字說明：
1. **IDLE（待機狀態）**：系統初始狀態，等待使用者透過電腦端發送指令或按下板載 `B1` 按鈕（PC13）啟動。
2. **EXERCISE（運動狀態）**：按下按鈕後進入此狀態，綠燈（PA5）恆亮，開始倒數運動秒數。
3. **REST（休息狀態）**：運動倒數結束後自動切換，紅燈（PA6）恆亮；當倒數剩餘 10 秒時紅燈開始連續閃爍，剩餘 3 秒時蜂鳴器（PA0）發出提醒逼逼聲。
4. **迴圈與完成**：若組數尚未完成，自動切回運動狀態；若已達到總組數，則結束訓練並重置回 IDLE 狀態。
