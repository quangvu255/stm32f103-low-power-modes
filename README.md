# STM32F103C8T6 Low-Power Modes: Sleep, Stop và Standby

> Repository tổng hợp mã nguồn firmware, cấu hình STM32CubeMX, tài liệu thiết kế, sơ đồ mạch và kết quả đo kiểm thực nghiệm 3 chế độ tiết kiệm năng lượng (**Sleep**, **Stop**, **Standby**) trên vi điều khiển **STM32F103C8T6 (Blue Pill)**.

---

## Mục lục

1. [Tổng quan dự án](#tổng-quan-dự-án)
2. [Mục tiêu kỹ thuật](#mục-tiêu-kỹ-thuật)
3. [Cấu trúc thư mục và hướng dẫn sử dụng từng folder](#cấu-trúc-thư-mục-và-hướng-dẫn-sử-dụng-từng-folder)
4. [Phần cứng và công cụ phát triển](#phần-cứng-và-công-cụ-phát-triển)
5. [Cơ chế hoạt động của 3 chế độ Low-Power](#cơ-chế-hoạt-động-của-3-chế-độ-low-power)
6. [Ánh xạ GPIO và ngoại vi](#ánh-xạ-gpio-và-ngoại-vi)
7. [Hướng dẫn mở project, build, nạp firmware và kiểm thử](#hướng-dẫn-mở-project-build-nạp-firmware-và-kiểm-thử)
8. [Kết quả đo dòng tiêu thụ và phân tích thực nghiệm](#kết-quả-đo-dòng-tiêu-thụ-và-phân-tích-thực-nghiệm)
9. [Video demo và chạy thực tế](#video-demo-và-chạy-thực-tế)
10. [Nhóm thực hiện và thời gian thực hiện](#nhóm-thực-hiện-và-thời-gian-thực-hiện)

---

## Tổng quan dự án

Repository gồm **3 project firmware Keil µVision / STM32CubeMX độc lập** cho từng chế độ tiết kiệm năng lượng trên bo mạch **STM32F103C8T6 Blue Pill**:

- **Sleep mode**: Ngắt xung nhịp lõi CPU Cortex-M3 bằng `WFI` trong khi ngoại vi vẫn duy trì hoạt động; tạm dừng `SysTick` trước khi ngủ để tránh bị đánh thức định kỳ mỗi 1 ms.
- **Stop mode**: Dừng CPU và các nguồn xung nhịp cao tần (HSE, PLL); bộ điều áp chính vẫn duy trì SRAM và nội dung thanh ghi; sau khi thức dậy bằng `EXTI1`, firmware gọi lại `SystemClock_Config()` và `HAL_ResumeTick()` để khôi phục xung hệ thống 64 MHz.
- **Standby mode**: Ngắt bộ điều áp nội (miền 1.8 V mất nguồn, SRAM/thanh ghi không được giữ); trước khi vào chế độ, firmware xóa cờ `PWR_FLAG_WU`, bật chân `PA0/WKUP` (`PWR_WAKEUP_PIN1`) rồi gọi `HAL_PWR_EnterSTANDBYMode()`. Khi được đánh thức, MCU khởi động lại từ đầu tương tự reset cứng.

Trong mỗi bài thử nghiệm, nút nhấn điều khiển việc vào chế độ công suất thấp và đánh thức vi điều khiển, hai LED hiển thị trạng thái thực thi của vòng lặp chính / ngắt Timer (`TIM1`), và đồng hồ vạn năng đo dòng tiêu thụ ở cấp bo mạch.

---

## Mục tiêu kỹ thuật

- Cấu hình và triển khai đầy đủ 3 mức tiết kiệm năng lượng (`Sleep`, `Stop`, `Standby`) trên dòng STM32F1 thông qua STM32 HAL và ngắt ngoài `EXTI`.
- Thực hành xử lý các vấn đề hậu đánh thức (post-wakeup recovery): quản lý `SysTick`, khôi phục cây clock PLL 64 MHz sau `Stop`, kiểm tra/xóa cờ trạng thái `PWR_FLAG_WU` và `PWR_FLAG_SB` trong `Standby`.
- Đo kiểm dòng tiêu thụ thực tế trên phần cứng Blue Pill và đối chiếu nguyên nhân sai lệch giữa dòng tiêu thụ lý thuyết của riêng chip MCU (datasheet) với dòng đo thực tế của cả bo mạch phát triển (LED nguồn, chân GPIO giữ mức `HIGH`).

---

## Cấu trúc thư mục và hướng dẫn sử dụng từng folder

```text
LOW-POWER/
├── .gitignore
├── README.md
├── Images/
│   ├── hardware/                    # Ảnh sơ đồ nguyên lý trích xuất từ Proteus
│   └── results/                     # Ảnh đấu nối phần cứng và ảnh đồng hồ đo dòng thực tế
├── Sleep_mode/
│   ├── SLEEP_MODE/                  # Project STM32CubeMX & Keil MDK-ARM cho chế độ Sleep
│   │   ├── Core/                    # Mã nguồn ứng dụng (main.c, stm32f1xx_it.c, hal_msp.c, Inc/)
│   │   ├── Drivers/                 # Thư viện CMSIS và STM32F1xx_HAL_Driver đi kèm
│   │   ├── MDK-ARM/                 # Project Keil (SLEEP_MODE.uvprojx, startup, RTE, DebugConfig)
│   │   ├── SLEEP_MODE.ioc           # File cấu hình chân, clock, NVIC trong STM32CubeMX
│   │   └── .mxproject               # Metadata sinh code của STM32CubeMX
│   ├── docs/                        # Báo cáo PDF, sơ đồ mạch PDF và file link YouTube (Sleep)
│   └── Video/                       # (Ignored trong .gitignore) Video .mp4 cục bộ, xem qua YouTube bên dưới
├── Stop_mode/
│   ├── Stop_Mode/                   # Project STM32CubeMX & Keil MDK-ARM cho chế độ Stop
│   │   ├── Core/                    # Mã nguồn ứng dụng (main.c, stm32f1xx_it.c, hal_msp.c, Inc/)
│   │   ├── Drivers/                 # Thư viện CMSIS (subset) và STM32F1xx_HAL_Driver
│   │   ├── MDK-ARM/                 # Project Keil (stopmode.uvprojx, startup, RTE, DebugConfig)
│   │   ├── stopmode.ioc             # File cấu hình chân, RCC (HSE), TIM1, NVIC trong STM32CubeMX
│   │   └── .mxproject               # Metadata sinh code của STM32CubeMX
│   ├── docs/                        # Báo cáo PDF, sơ đồ mạch PDF và file link YouTube (Stop)
│   └── Video/                       # (Ignored trong .gitignore) Video .mp4 cục bộ, xem qua YouTube bên dưới
└── Standby_mode/
    ├── Standby_Mode/                # Project STM32CubeMX & Keil MDK-ARM cho chế độ Standby
    │   ├── Core/                    # Mã nguồn ứng dụng (main.c, stm32f1xx_it.c, hal_msp.c, Inc/)
    │   ├── Drivers/                 # Thư viện CMSIS (subset) và STM32F1xx_HAL_Driver
    │   ├── MDK-ARM/                 # Project Keil (stanby_mode.uvprojx, startup, RTE, DebugConfig)
    │   ├── stanby_mode.ioc          # File cấu hình chân, SYS_WKUP, RCC (HSE), TIM1 trong CubeMX
    │   └── .mxproject               # Metadata sinh code của STM32CubeMX
    ├── docs/                        # Báo cáo PDF, sơ đồ mạch PDF và file link YouTube (Standby)
    └── Video/                       # (Ignored trong .gitignore) Video .mp4 cục bộ, xem qua YouTube bên dưới
```

### Hướng dẫn công dụng và cách sử dụng từng thư mục

| Thư mục | Mục đích | Cách sử dụng |
|---|---|---|
| `Sleep_mode/SLEEP_MODE/`<br>`Stop_mode/Stop_Mode/`<br>`Standby_mode/Standby_Mode/` | Chứa toàn bộ project firmware độc lập cho từng chế độ. | Mỗi lần thử nghiệm chỉ mở, build và nạp **một** trong ba project lên bo Blue Pill. |
| `.../Core/Src/` và `.../Core/Inc/` | Chứa logic chính do người phát triển viết trong các khối `USER CODE` (`main.c`, `stm32f1xx_it.c`, `stm32f1xx_hal_msp.c`, `main.h`). | Đọc hoặc chỉnh sửa logic vào/thoát chế độ, cấu hình LED/nút nhấn tại [`main.c` (Sleep)](Sleep_mode/SLEEP_MODE/Core/Src/main.c), [`main.c` (Stop)](Stop_mode/Stop_Mode/Core/Src/main.c), [`main.c` (Standby)](Standby_mode/Standby_Mode/Core/Src/main.c). |
| `.../*.ioc` | File cấu hình phần cứng STM32CubeMX (`PINOUT`, `RCC`, `NVIC`, `TIM1`, `SYS`). | Mở bằng **STM32CubeMX** để xem trực quan sơ đồ chân, cây xung nhịp 64 MHz hoặc tái sinh mã nguồn khi thay đổi ngoại vi. |
| `.../MDK-ARM/` | Chứa file project Keil µVision (`*.uvprojx`), file cấu hình debug (`*.uvoptx`), startup assembly (`startup_stm32f103xb.s`) và cấu hình RTE. | Nhấp đúp file `.uvprojx` tương ứng để mở bằng **Keil µVision**, biên dịch và nạp qua ST-LINK. Thư mục build output và file `*.uvguix.*`, `*.lst` đã được tự động bỏ qua trong [`.gitignore`](.gitignore). |
| `.../Drivers/` | Chứa CMSIS và STM32F1xx HAL Driver đi kèm để project có thể build ngay sau khi clone mà không cần tải lại gói firmware. | Không chỉnh sửa trực tiếp các file trong thư mục này. Lưu ý: project `Sleep_mode` dùng chế độ sao chép toàn bộ thư viện của CubeMX (`LibraryCopy=0`) nên `Drivers/CMSIS/` chứa thêm các module DSP/NN/RTOS; hai project `Stop_mode` và `Standby_mode` chỉ giữ tập con cần thiết (`LibraryCopy=1`). |
| `Sleep_mode/docs/`<br>`Stop_mode/docs/`<br>`Standby_mode/docs/` | Lưu tài liệu kỹ thuật gốc gồm báo cáo cuối kỳ (PDF), bản vẽ schematic Proteus (PDF) và file chứa link video YouTube (`Link_video_youtube.docx` / `Link_Video_youtube.docx`). | Mở các file `.pdf` để xem cơ sở lý thuyết, lưu đồ thuật toán, ảnh cấu hình CubeMX và phân tích đo kiểm chi tiết của từng chế độ. |
| `Images/` | Tập hợp ảnh bằng chứng phục vụ hiển thị trực tiếp trên GitHub (`Images/hardware/` là schematic đã trích xuất; `Images/results/` là ảnh mạch breadboard và đồng hồ đo dòng). | Dùng làm tài liệu đối chiếu nhanh phần cứng và kết quả đo dòng mà không cần mở file PDF lớn. |
| `*/Video/` | Chứa các bản ghi video `.mp4` cục bộ (~128.5 MB tổng cộng). | Đã được loại trừ trong [`.gitignore`](.gitignore) để giữ repository gọn nhẹ khi đẩy lên GitHub; xem trực tiếp qua các link YouTube tại mục [Video demo và chạy thực tế](#video-demo-và-chạy-thực-tế). |

### Truy cập nhanh các file chính của từng project

| Chế độ | Project Keil (`MDK-ARM`) | Cấu hình CubeMX (`.ioc`) | Mã nguồn chính (`main.c`) | Tài liệu & Schematic (`docs/`) |
|---|---|---|---|---|
| **Sleep** | [SLEEP_MODE.uvprojx](Sleep_mode/SLEEP_MODE/MDK-ARM/SLEEP_MODE.uvprojx) | [SLEEP_MODE.ioc](Sleep_mode/SLEEP_MODE/SLEEP_MODE.ioc) | [main.c](Sleep_mode/SLEEP_MODE/Core/Src/main.c) | [sleep_mode_report.pdf](Sleep_mode/docs/sleep_mode_report.pdf) · [Sleep_Mode_Schematic.PDF](Sleep_mode/docs/Sleep_Mode_Schematic.PDF) |
| **Stop** | [stopmode.uvprojx](Stop_mode/Stop_Mode/MDK-ARM/stopmode.uvprojx) | [stopmode.ioc](Stop_mode/Stop_Mode/stopmode.ioc) | [main.c](Stop_mode/Stop_Mode/Core/Src/main.c) | [Stop_mode_report.pdf](Stop_mode/docs/Stop_mode_report.pdf) · [Stop_Mode_Schematic.PDF](Stop_mode/docs/Stop_Mode_Schematic.PDF) |
| **Standby** | [stanby_mode.uvprojx](Standby_mode/Standby_Mode/MDK-ARM/stanby_mode.uvprojx) | [stanby_mode.ioc](Standby_mode/Standby_Mode/stanby_mode.ioc) | [main.c](Standby_mode/Standby_Mode/Core/Src/main.c) | [Standby_mode_report.pdf](Standby_mode/docs/Standby_mode_report.pdf) · [Standby_Mode_Schematic.PDF](Standby_mode/docs/Standby_Mode_Schematic.PDF) |

---

## Phần cứng và công cụ phát triển

### Phần cứng sử dụng

- Bo mạch phát triển **STM32F103C8T6 Blue Pill** (lõi ARM Cortex-M3, LQFP48)
- Bộ nạp và gỡ lỗi **ST-LINK** (kết nối qua giao tiếp SWD: `PA13` / `SYS_JTMS-SWDIO`, `PA14` / `SYS_JTCK-SWCLK`)
- Breadboard, 2 nút nhấn tích cực mức cao (kèm điện trở kéo xuống và tụ chống dội), 2 LED đơn và điện trở hạn dòng
- Đồng hồ vạn năng số **HABOTEST HT118D** mắc nối tiếp trên đường cấp nguồn để đo dòng tiêu thụ bo mạch

### Công cụ phần mềm và thông số build

| Hạng mục | Chi tiết cấu hình thực tế trong repository |
|---|---|
| Ngôn ngữ & thư viện | C, STM32Cube HAL (`STM32F1xx_HAL_Driver`), CMSIS, GPIO, EXTI, SysTick, PWR và TIM1 (Stop, Standby) |
| Cấu hình CubeMX | Sleep tạo bằng STM32CubeMX `6.16.1`; Stop và Standby tạo bằng STM32CubeMX `6.15.0`; gói firmware `STM32Cube FW_F1 V1.8.6` |
| IDE & Device Pack | Keil µVision / MDK-ARM, target device `STM32F103C8`, device pack `Keil.STM32F1xx_DFP 2.4.1` |
| Compiler cấu hình | Sleep dùng Arm Compiler 5 (`uAC6=0`); Stop và Standby khai báo Arm Compiler 6.24 (`ARMCLANG`, `uAC6=1`) |
| Vẽ mạch nguyên lý | Proteus (xuất PDF trong các thư mục `docs/` và ảnh PNG trong `Images/hardware/`) |

---

## Cơ chế hoạt động của 3 chế độ Low-Power

Cả ba project đều cấu hình xung nhịp hệ thống `SYSCLK = 64 MHz` (`AHB = 64 MHz`, `APB1 = 32 MHz`, `APB2 = 64 MHz`) và bật clock `AFIO`, `PWR` cùng cấu hình SWD trong `HAL_MspInit()` tại `stm32f1xx_hal_msp.c`.

| Tiêu chí | Sleep Mode | Stop Mode | Standby Mode |
|---|---|---|---|
| Nguồn clock hệ thống | `HSI/2` × PLL 16 (`64 MHz`) | `HSE 8 MHz` × PLL 8 (`64 MHz`) | `HSE 8 MHz` × PLL 8 (`64 MHz`) |
| Điều khiển vào chế độ | Nhấn nút `PA0` (`EXTI0` cạnh lên) đặt `EnterSleepFlag = 1` | Nhấn nút `PA0` (`EXTI0` cạnh lên) đặt `EnterStopFlag = 1` | Nhấn nút `PA1` (`EXTI1` cạnh lên) đặt `EnterStbFlag = 1` |
| Trình tự vào trong `main.c` | `HAL_SuspendTick()` → `HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI)` | `HAL_SuspendTick()` → `HAL_PWR_EnterSTOPMode(PWR_MAINREGULATOR_ON, PWR_STOPENTRY_WFI)` | `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU)` → `HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1)` → `HAL_PWR_EnterSTANDBYMode()` |
| Nguồn đánh thức | Nhấn lại `PA0` (`EXTI0`) | Nhấn nút `PA1` (`EXTI1` cạnh lên) | Cạnh lên trên `PA0 / SYS_WKUP` (hoặc reset `NRST`) |
| Xử lý sau khi thức dậy | Trong `HAL_GPIO_EXTI_Callback()` gọi `HAL_ResumeTick()`, xóa `SleepMode_Activated = 0` và chạy tiếp vòng lặp | Trong `main()` gọi `SystemClock_Config()` để bật lại HSE/PLL 64 MHz, gọi `HAL_ResumeTick()`, xóa cờ | MCU reset cứng và chạy lại từ đầu `main()`; xóa cờ `PWR_FLAG_SB` nếu được set |
| Bộ điều áp & SRAM | Bộ điều áp chính ON; giữ nguyên toàn bộ SRAM và thanh ghi | Bộ điều áp chính ON (`PWR_MAINREGULATOR_ON`); giữ nguyên SRAM và thanh ghi | Bộ điều áp OFF; **mất dữ liệu SRAM** và trạng thái thanh ghi thông thường |
| Tác vụ nền quan sát | `blink_led()` nhấp nháy `PA3` (50 ms bật / 950 ms tắt); `PA4` phản ánh trạng thái nút `PA1` | `blinkled()` nhấp nháy `PA4` (50 ms bật / 950 ms tắt); `TIM1` ngắt mỗi 500 ms đảo `PA3` | `blink_led()` đếm phần mềm (`counter >= 1000000`) đảo `PA4`; `TIM1` ngắt mỗi 500 ms đảo `PA3` |

---

## Ánh xạ GPIO và ngoại vi

| Chân MCU | Project Sleep | Project Stop | Project Standby |
|---|---|---|---|
| `PA0` | Ngắt ngoài `EXTI0` cạnh lên: nhấn lần 1 vào Sleep, nhấn lần 2 đánh thức | Ngắt ngoài `EXTI0` cạnh lên: vào Stop | Chân đánh thức phần cứng `SYS_WKUP` (`PWR_WAKEUP_PIN1`): đánh thức khỏi Standby |
| `PA1` | `GPIO_Input`: điều khiển trực tiếp LED `PA4` trong vòng lặp `while(1)` | Ngắt ngoài `EXTI1` cạnh lên: đánh thức khỏi Stop | Ngắt ngoài `EXTI1` cạnh lên: vào Standby |
| `PA3` | `GPIO_Output`: LED báo trạng thái chạy (`blink_led()` trong `while(1)`) | `GPIO_Output`: LED đảo trạng thái trong ngắt `TIM1` (chu kỳ ngắt 500 ms) | `GPIO_Output`: LED đảo trạng thái trong ngắt `TIM1` (`HAL_TIM_PeriodElapsedCallback`) |
| `PA4` | `GPIO_Output`: LED bật/tắt theo mức logic tại `PA1` | `GPIO_Output`: LED báo trạng thái chạy (`blinkled()` trong `while(1)`) | `GPIO_Output`: LED đảo trạng thái bằng vòng đếm trong `blink_led()` |
| `PD0` / `PD1` | Không dùng (dùng dao động nội HSI) | `RCC_OSC_IN` / `RCC_OSC_OUT` (thạch anh ngoài HSE 8 MHz) | `RCC_OSC_IN` / `RCC_OSC_OUT` (thạch anh ngoài HSE 8 MHz) |
| `PA13` / `PA14` | Serial Wire Debug (`SWDIO` / `SWCLK`) | Serial Wire Debug (`SWDIO` / `SWCLK`) | Serial Wire Debug (`SWDIO` / `SWCLK`) |

- **Thông số Timer `TIM1` (trong Stop và Standby)**: `Prescaler = 63999`, `Period = 499`, nguồn clock nội `64 MHz` → tần số đếm `1 kHz`, ngắt cập nhật (`TIM1_UP_IRQn`) mỗi `500 ms` tạo nhịp nháy `1 Hz`.

---

## Hướng dẫn mở project, build, nạp firmware và kiểm thử

> Các bước dưới đây được tổng hợp từ cấu hình thực tế của các file `.ioc`, `.uvprojx` và mã nguồn trong repository.

### 1. Xem hoặc chỉnh sửa cấu hình bằng STM32CubeMX (tùy chọn)

1. Mở phần mềm **STM32CubeMX** (khuyến nghị phiên bản `6.15.0` trở lên và gói `STM32Cube FW_F1 V1.8.6`).
2. Mở file `.ioc` tương ứng với chế độ cần khảo sát:
   - Sleep: [`Sleep_mode/SLEEP_MODE/SLEEP_MODE.ioc`](Sleep_mode/SLEEP_MODE/SLEEP_MODE.ioc)
   - Stop: [`Stop_mode/Stop_Mode/stopmode.ioc`](Stop_mode/Stop_Mode/stopmode.ioc)
   - Standby: [`Standby_mode/Standby_Mode/stanby_mode.ioc`](Standby_mode/Standby_Mode/stanby_mode.ioc)
3. Kiểm tra cấu hình chân `PA0`, `PA1`, `PA3`, `PA4`, cấu hình `RCC`, `SYS (Serial Wire)` và `NVIC`. Vì các project đã có sẵn mã nguồn sinh hoàn chỉnh trong `Core/` và `Drivers/`, bạn không cần sinh lại code nếu chỉ build và nạp thử.

### 2. Build và nạp firmware bằng Keil µVision (MDK-ARM)

1. Cài đặt **Keil µVision / MDK-ARM** và Device Family Pack **Keil.STM32F1xx_DFP 2.4.1**.
2. Mở một trong ba file project `.uvprojx`:
   - [`SLEEP_MODE.uvprojx`](Sleep_mode/SLEEP_MODE/MDK-ARM/SLEEP_MODE.uvprojx) — kiểm tra target dùng **Arm Compiler 5** (`uAC6=0`).
   - [`stopmode.uvprojx`](Stop_mode/Stop_Mode/MDK-ARM/stopmode.uvprojx) — kiểm tra target dùng **Arm Compiler 6.24** (`ARMCLANG`).
   - [`stanby_mode.uvprojx`](Standby_mode/Standby_Mode/MDK-ARM/stanby_mode.uvprojx) — kiểm tra target dùng **Arm Compiler 6.24** (`ARMCLANG`).
3. Nhấn **Build (F7)** để biên dịch chương trình.
4. Kết nối mạch Blue Pill với **ST-LINK** qua các chân SWD (`3.3V`, `GND`, `SWDIO/PA13`, `SWCLK/PA14`), vào **Options for Target → Debug → ST-Link Debugger** để kiểm tra kết nối chip `STM32F103C8`.
5. Nhấn **Load (F8)** để nạp firmware vào Flash (lưu ý: mỗi lần chỉ nạp **một** project chế độ lên bo mạch).

### 3. Quy trình kiểm thử trên phần cứng và đo dòng

1. Đấu nối 2 nút nhấn (có điện trở kéo xuống GND và tụ chống dội nếu cần) vào `PA0` và `PA1`, đấu 2 LED (qua điện trở hạn dòng) vào `PA3` và `PA4` theo bảng ánh xạ GPIO ở trên.
2. Để đo dòng tiêu thụ chính xác hơn: rút dây cấp nguồn/ST-LINK sau khi nạp xong hoặc mắc đồng hồ vạn năng ở thang `mA DC` nối tiếp vào đường cấp nguồn của mạch test (như ảnh thực nghiệm trong `Images/results/`).
3. Quan sát hoạt động của từng chế độ:
   - **Sleep**: Khi đang chạy, LED `PA3` nháy ngắn mỗi 1 s; nhấn `PA0` → LED dừng, dòng giảm xuống mức Sleep (~`7.97 mA`); nhấn `PA0` lần nữa → CPU thức dậy và nháy LED tiếp tục ngay tại vị trí cũ. Có thể giữ `PA1` ở mức `HIGH` trước/trong khi ngủ để quan sát ảnh hưởng của chân GPIO mức cao lên dòng tiêu thụ.
   - **Stop**: Khi đang chạy, cả `PA3` và `PA4` đều nháy; nhấn `PA0` → toàn bộ xung clock dừng, LED ngừng nháy, dòng giảm xuống ~`2.02 mA`; nhấn `PA1` → MCU thức dậy, tự khôi phục PLL 64 MHz và 2 LED hoạt động bình thường trở lại.
   - **Standby**: Khi đang chạy, cả `PA3` và `PA4` đều nháy; nhấn `PA1` → MCU vào Standby, dòng giảm xuống ~`1.61 – 1.67 mA`; nhấn `PA0` (`WKUP`) → MCU khởi động lại hoàn toàn từ đầu `main()`.

---

## Kết quả đo dòng tiêu thụ và phân tích thực nghiệm

Các số liệu dưới đây được ghi nhận trực tiếp trên bo mạch **STM32F103C8T6 Blue Pill** bằng đồng hồ vạn năng **HABOTEST HT118D** ở cấp độ **toàn bo mạch + mạch thử nghiệm**, không phải dòng tiêu thụ cô lập của riêng chip MCU. Do cấu hình xung nhịp, ngoại vi (`TIM1`) và trạng thái LED tải khác nhau giữa 3 bài thí nghiệm, các giá trị dòng ở chế độ Run không giống nhau.

| Thí nghiệm | Trạng thái hệ thống | Dòng điện đo trên mạch (ảnh đo / báo cáo) | Tài liệu gốc |
|---|---|---|---|
| **Sleep** | Run mode | `25.67 mA` | [sleep_mode_report.pdf](Sleep_mode/docs/sleep_mode_report.pdf) (tr. 35) |
| **Sleep** | Sleep mode | `7.97 mA` | [sleep_mode_report.pdf](Sleep_mode/docs/sleep_mode_report.pdf) (tr. 36) |
| **Sleep** | Sleep mode + giữ 1 GPIO mức `HIGH` | `10.90 mA` | [sleep_mode_report.pdf](Sleep_mode/docs/sleep_mode_report.pdf) (tr. 36–37) |
| **Stop** | Trước khi vào Stop (Run mode) | `30.41 mA` | [Stop_mode_report.pdf](Stop_mode/docs/Stop_mode_report.pdf) (tr. 27) |
| **Stop** | Trong Stop mode | `2.02 mA` (báo cáo kết luận ≈ `2 mA`) | [Stop_mode_report.pdf](Stop_mode/docs/Stop_mode_report.pdf) (tr. 27, 29) |
| **Stop** | Sau khi Wake-up khỏi Stop | `29.42 mA` | [Stop_mode_report.pdf](Stop_mode/docs/Stop_mode_report.pdf) (tr. 28) |
| **Standby** | Run mode | `32.62 mA` | [Standby_mode_report.pdf](Standby_mode/docs/Standby_mode_report.pdf) (tr. 27–28) |
| **Standby** | Standby mode | `1.61 mA` (trên đồng hồ ảnh đo) / `~1.67 mA` (báo cáo ghi nhận) | [Standby_mode_report.pdf](Standby_mode/docs/Standby_mode_report.pdf) (tr. 28) |

### Phân tích kỹ thuật

1. **Chênh lệch giữa đo bo mạch thực tế và datasheet MCU**:
   - Theo datasheet STM32F103x8/xB, ở điều kiện `3.3 V` và tắt các ngoại vi không cần thiết, dòng MCU ở chế độ Standby chỉ khoảng vài `µA` (~`2 µA`), Stop ở mức hàng chục `µA` và Sleep ở mức vài `mA`.
   - Khi đo trên bo **Blue Pill**, LED nguồn (`PWR LED`) được nối cố định qua điện trở hạn dòng trên đường `3.3 V` và luôn sáng kể cả khi MCU vào Stop hoặc Standby, tiêu thụ khoảng `1 – 1.3 mA`, cộng thêm mạch ổn áp trên bo tạo nên dòng nền ~`1.61 – 2.02 mA` như ghi nhận trên đồng hồ.
2. **Ảnh hưởng của trạng thái I/O khi ngủ**:
   - Ở thí nghiệm Sleep, khi giữ thêm một ngõ ra GPIO ở mức `HIGH` cấp tải LED, dòng bo mạch tăng từ `7.97 mA` lên `10.90 mA`. Điều này minh họa tầm quan trọng của việc cấu hình các chân không dùng sang Analog Input hoặc ngắt tải ngoại vi trước khi đưa MCU vào chế độ tiết kiệm năng lượng.
3. **Độ trễ đánh thức (Wake-up Latency)**:
   - Theo cơ chế kiến trúc, Sleep thức dậy nhanh nhất (vài chu kỳ xung nhịp do không tắt dao động); Stop cần thời gian ổn định lại HSE + khóa pha PLL thông qua `SystemClock_Config()`; Standby tương đương reset hệ thống nên có thời gian khởi động lại dài nhất. Repository hiện tại chưa thực hiện phép đo định lượng độ trễ bằng máy hiện sóng (oscilloscope) hoặc logic analyzer.

<details>
<summary>Xem ảnh đấu nối phần cứng, sơ đồ nguyên lý và ảnh đồng hồ đo dòng (Images/)</summary>

#### Ảnh đo dòng thực nghiệm

| Sleep: Run (`25.67 mA`) | Sleep: Sleep (`7.97 mA`) | Sleep + GPIO HIGH (`10.90 mA`) |
|---|---|---|
| <img src="Images/results/sleep-run.jpg" alt="Dòng ở trạng thái Run trong thí nghiệm Sleep" width="240"> | <img src="Images/results/sleep-mode.jpg" alt="Dòng ở chế độ Sleep" width="240"> | <img src="Images/results/sleep-gpio-high.jpg" alt="Dòng ở chế độ Sleep với GPIO ở mức HIGH" width="240"> |

| Stop: Trước khi vào (`30.41 mA`) | Stop: Trong chế độ (`2.02 mA`) | Stop: Sau khi đánh thức (`29.42 mA`) |
|---|---|---|
| <img src="Images/results/stop-run.jpg" alt="Dòng trước khi vào Stop" width="240"> | <img src="Images/results/stop-mode.jpg" alt="Dòng trong chế độ Stop" width="240"> | <img src="Images/results/stop-wakeup.jpg" alt="Dòng sau khi đánh thức khỏi Stop" width="240"> |

| Standby: Run (`32.62 mA`) | Standby: Standby (`1.61 mA`) |
|---|---|
| <img src="Images/results/standby-run.jpg" alt="Dòng ở trạng thái Run trước khi vào Standby" width="320"> | <img src="Images/results/standby-mode.jpg" alt="Dòng trong chế độ Standby" width="320"> |

#### Ảnh mạch thử nghiệm và sơ đồ tham khảo

| Mạch thực tế Stop Mode | Mạch thực tế Standby Mode | Schematic Proteus tham khảo |
|---|---|---|
| <img src="Images/results/stop-hardware.jpg" alt="Mạch breadboard thử nghiệm Stop mode" width="260"> | <img src="Images/results/standby-hardware.jpg" alt="Mạch breadboard thử nghiệm Standby mode" width="260"> | <img src="Images/hardware/sleep-schematic.png" alt="Sơ đồ nguyên lý Proteus tham khảo" width="260"> |

</details>

---

## Video demo và chạy thực tế

Các file `.mp4` cục bộ trong thư mục `Video/` được loại trừ khỏi Git qua [`.gitignore`](.gitignore) để tối ưu dung lượng repository; toàn bộ video đã được lưu trữ trên YouTube (trích từ các file `Link_video_youtube.docx` trong từng thư mục `docs/`):

| Chế độ | Video Demo (giải thích & thao tác) | Video Chạy thực tế (đo dòng phần cứng) | File ghi chú gốc |
|---|---|---|---|
| **Sleep Mode** | [Xem Demo Sleep Mode](https://youtu.be/zpPMImgsR6M) | [Xem Chạy thực tế Sleep Mode](https://youtu.be/kWmwyme6-zA) | [Link_video_youtube.docx](Sleep_mode/docs/Link_video_youtube.docx) |
| **Stop Mode** | [Xem Demo Stop Mode](https://youtu.be/HYJBpMe65hQ) | [Xem Chạy thực tế Stop Mode](https://youtu.be/FOSVa_1vRFc) | [Link_Video_youtube.docx](Stop_mode/docs/Link_Video_youtube.docx) |
| **Standby Mode** | [Xem Demo Standby Mode](https://youtu.be/Gtf8XyCK0gI) | [Xem Chạy thực tế Standby Mode](https://youtu.be/wraBephHXPU) | [Link_video_youtube.docx](Standby_mode/docs/Link_video_youtube.docx) |

---

## Nhóm thực hiện và thời gian thực hiện

- **Nhóm thực hiện**: Nhóm 18, Khoa Điện - Điện tử, Trường Đại học Công nghệ Kỹ thuật TP. HCM (HCM-UTE)
  - Nguyễn Ngọc Trường Phi
  - Hà Quang Huy
  - Võ Trung Nguyên ([`@NguyenVo-09`](https://github.com/NguyenVo-09))
  - Vũ Minh Quang ([`@quangvu255`](https://github.com/quangvu255))
- **Thời gian thực hiện**: Học kỳ 1, năm học 2025–2026 (hoàn thành tháng 01/2026)
