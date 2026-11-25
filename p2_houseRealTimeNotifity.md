# 居家雲端即時通報
> 目標：用 ESP32 結合震動感測器、PIR 人體感測器、OLED、蜂鳴器，
> 打造「門窗異常」使用 Discord 即時通知，並可遠端控制」的居家防盜系統。

## 硬體接腳說明
| 元件 | 元件腳位 | ESP32 腳位 | 備註 |
|---|---|---|---|
| 蜂鳴器 Buzzer | 訊號 (S / +) | **GPIO 32** | GND 接 GND |
| OLED SSD1306 (I2C) | SDA | **GPIO 21** | ESP32 預設 I2C |
| | SCL | **GPIO 22** | ESP32 預設 I2C |
| | VCC / GND | 3.3V / GND | I2C 位址 `0x3C` |
| 震動感測器 | DO | **GPIO 4** | VCC 接 3.3V，GND 接 GND |
| PIR 人體感測器 | OUT | **GPIO 13** | **VCC 請接 5V（VIN）**，GND 接 GND |

> 注意：
> - PIR 感測器多數需 5V 供電，接 3.3V 可能無法正常偵測。
> - PIR 上電後需 **約 30~60 秒暖機**，期間數值可能亂跳，屬正常現象。
> - 若 OLED 螢幕無畫面，請先確認 SDA / SCL 沒有接反。

## 開發環境與函式庫（執行前務必安裝）
| 函式庫 | 搜尋關鍵字 | 使用於 |
|---|---|---|
| Adafruit GFX Library | `Adafruit GFX` | p02_2、3、5、8、9 |
| Adafruit SSD1306 | `Adafruit SSD1306` | p02_2、3、5、8、9 |
| ArduinoJson | `ArduinoJson`（**請安裝 7.x 版**，程式使用 `JsonDocument`） | p02_8、9 |
| Blynk | `Blynk`（by Volodymyr Shymanskyy） | p02_10、11 |

> `WiFi.h`、`HTTPClient.h`、`WiFiClientSecure.h`、`Wire.h`、`time.h` 皆為 ESP32 套件內建，**不用另外安裝**。
> 安裝 Adafruit SSD1306 時，若詢問是否一併安裝相依函式庫（Adafruit BusIO 等），請選 **Install all**。

## 1. 感測器測試（p02_1 ~ p02_4）

先逐一確認每個元件都正常，之後整合時出問題才好排查。

### 1-1 蜂鳴器（p02_1_Buzzer）

**重點**
- 確認腳位 I/O -> GPIO 32、VCC -> VCC、GND -> GND。
- `Buzzer_PIN` 定義為 GPIO 32。
- 用 `#define` 定義音符頻率（C5=523Hz … C6=1047Hz）。
- `tone(腳位, 頻率, 持續時間)` 發出指定音高。

```cpp
tone(Buzzer_PIN, C5, duration);   // 發出 C5 音，持續 100ms
```

**操作**
1.上傳後開啟序列埠監控視窗（鮑率 **115200**）。
3. 輸入 `1` → 發出單音 C5。
4. 輸入 `2` → 依序播放 C5~C6 八個音階。

>  序列埠視窗下方請設定為「**沒有行結尾**」或「換行」皆可，`Serial.parseInt()` 會把輸入轉成整數。

### 1-2 OLED 螢幕（p02_2_OLED）

**重點**
-確認腳位接擴充板右下方 pin 腳，注意接腳順序。
- `display.begin(SSD1306_SWITCHCAPVCC, 0x3c)` 啟動 OLED。
- 繪圖流程固定為：`clearDisplay()` → 設定游標/字體 → `println()` → `display()`。
- 文字尺寸 1 = 6x8、2 = 12x16、3 = 18x24 像素。
- `setTextColor(BLACK, WHITE)` 為「黑字白底」（反白標題）。

**預期結果**：螢幕顯示 `AIoT`、`Hi bro!`，以及每秒加 1 的 `time: N`。

### 1-3 震動感測器（p02_3_Vibrator）

**重點**
- 確認腳位 DO -> GPIO 4、VCC -> VCC、GND -> GND。
- 感測到震動輸出 `HIGH` → OLED 顯示 `door open`，並停留 2 秒；否則顯示 `door closed`。
- 若偵測失敗會執行 `for(;;);` 卡住程式，避免繼續跑。

**操作**：輕敲或搖動感測器，觀察 OLED 與序列埠是否出現 `震動偵測！Door Open`。

> 💡 若靈敏度太高或太低，可用模組上的藍色小螺絲（可變電阻）調整。

### 1-4 PIR 人體感測器（p02_4_PIR）

**重點**
- 確認腳位 DO -> GPIO 13、VCC -> 需接 5V電壓、GND -> GND。
- PIR 接 GPIO 13，內建 LED（GPIO 2）當作指示燈。
- 有人經過 → `digitalRead` 為 1 → LED 亮並印出「有人經過」。
- 每秒偵測一次。

```cpp
int pir = digitalRead(PIR);
if (pir > 0) { digitalWrite(led, HIGH); }
```

**操作**：手在感測器前揮動，觀察序列埠 `pir: 1` 與板載 LED。

> ✅ 這一步全部通過，再進入下一步。

---

## 2. 地端整合 OLED（p02_5_Vib_PIR）

把震動 + PIR + OLED + LED 整合成一個「地端防盜面板」。

**畫面配置**

| 區域 | Y 座標 | 內容 |
|---|---|---|
| 標題列 | 0 | ` AIoT Security System `（反白） |
| 門狀態 | 16 | 震動 → `door open` / `door close` |
| 人員狀態 | 42 | PIR → `someOne` / `noOne` |

**關鍵邏輯**
```cpp
int isVibrating = digitalRead(VIBRATION_PIN);
int pirState    = digitalRead(PIR_PIN);
```
- 偵測到開門震動時，畫面停留 1.5 秒方便閱讀；平常每 0.1 秒偵測一次，避免漏掉瞬間震動。

**預期結果**：OLED 即時顯示門與人員狀態，PIR 觸發時板載 LED 亮起。

---
