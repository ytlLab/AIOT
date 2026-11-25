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
**介紹**

1. SW-420 是一款無方向性的滾珠/金屬彈簧式震動開關感測器，具有以下特性：
- 靜止狀態：內部滾珠或導電體接觸良好，開關呈閉合導通狀態，模組數位輸出端（DO）輸出 低電位 (LOW)。
- 震動狀態：當受到外力搖晃或振動時，內部開關會瞬間斷開，模組數位輸出端（DO）會輸出 高電位 (HIGH)。
- 應用場景：車輛防盜、地震警報、智能小車碰撞檢測、設備震動監測等。
<img width="800" height="200" alt="image" src="https://github.com/user-attachments/assets/08d37fb7-d331-4a5a-87a5-d53f86b69fde" />
  
2. 板載靈敏度調節電位器
- SW-420 模組板載設有一個藍色十字形可變電阻（電位器），旋轉它可以根據實際環境需求微調震動觸發的靈敏度（順時針或逆時針微調可改變觸發難易度）。

**重點**
- 確認腳位 DO -> GPIO 4、VCC -> VCC、GND -> GND。
- 感測到震動輸出 `HIGH` → OLED 顯示 `door open`，並停留 2 秒；否則顯示 `door closed`。
- 若偵測失敗會執行 `for(;;);` 卡住程式，避免繼續跑。

**操作**：輕敲或搖動感測器，觀察 OLED 與序列埠是否出現 `震動偵測！Door Open`。

> 若靈敏度太高或太低，可用模組上的藍色小螺絲（可變電阻）調整。

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

> 這一步全部通過，再進入下一步。

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
## 3. Discord 註冊與申請機器人
### 3-1 註冊帳號並建立伺服器
1. 前往 <https://discord.com> 註冊並登入（手機 App 建議也安裝，才能收到推播）。
2. 左側點「**＋**」→「建立我的」→ 選擇「僅供我和我的朋友使用」，建立自己的伺服器。
3. 可新增一個文字頻道，例如 `#home-alert`。

### 3-2 建立 Webhook（給第 4、5 步使用）
1. 在頻道旁點齒輪 **編輯頻道** → **整合** → **Webhook** → **新 Webhook**。
2. 取名（如 `ESP32 通報`）→ **複製 Webhook 網址**。
3. 網址格式：`https://discord.com/api/webhooks/<ID>/<TOKEN>`

### 3-3 申請機器人（給第 6、7 步使用）
1. 前往 <https://discord.com/developers/applications> → **New Application**，輸入名稱。
2. 左側 **Bot** 頁面 → **Reset Token** → **複製並妥善保存 Token**（只會顯示一次）。
3. 同一頁往下找到 **Privileged Gateway Intents**，**開啟 `MESSAGE CONTENT INTENT`**，並儲存。
   > 沒開這項，機器人讀取到的訊息內容會是空字串，`!alarm` 等指令將無法辨識。
4. 左側 **OAuth2 → URL Generator**：
   - Scopes 勾選 `bot`
   - Bot Permissions 勾選：`檢視頻道(View Channels)`、`傳送訊息(Send Messages)`、`讀取訊息歷史紀錄(Read Message History)`
5. 複製下方產生的網址，貼到瀏覽器，選擇你的伺服器 → 授權，機器人即加入伺服器。

### 3-4 取得頻道 ID
1. Discord 設定 → **進階** → 開啟 **開發者模式**。
2. 在目標頻道上按右鍵 → **複製頻道 ID**。

### 3-5 本步驟你需要準備的資料

| 資料 | 用途 |
|---|---|
| Wi-Fi 名稱 / 密碼 | 所有連網程式（2.4GHz，ESP32 不支援 5GHz） |
| Webhook 網址 | p02_6、p02_7 |
| Bot Token | p02_8、p02_9 |
| Channel ID | p02_8、p02_9 |

> **資安提醒**
> - Webhook 網址與 Bot Token 等同「密碼」，**不要貼到公開的聊天室、GitHub 或作業截圖**。
> - 若不慎外洩，請立刻到 Developer Portal 按 **Reset Token** / 刪除並重建 Webhook。
> - 程式範例中請以 `"你的WiFi名稱"`、`"你的_BOT_TOKEN"` 等佔位文字替代，上課分享檔案前先檢查。

---
## 4. Discord Webhook 測試（p02_6_discordWebhook）

**介紹**：Webhook 是一種「事件驅動」的自動化通知機制：當某個系統發生特定事件（例如付款成功、收到新訊息、程式碼被推送）時，它會主動向你指定的網址（URL）發送一個 HTTP 請求（通常是 POST 並帶上 JSON 資料），不需要你的系統一直去詢問「有新資料了嗎？」。

Webhook 與輪詢（Polling）的差別
- Webhook（被動接收、主動推送）： 事件發生才傳送，即時、省流量，適合即時通知與自動化串接。
- 輪詢（主動詢問）： 每隔幾秒或幾分鐘就發一次請求問「有沒有更新？」，沒事也會浪費資源，且不夠即時。

常見運作流程

1. 在來源系統（如金流、GitHub、LINE）設定接收端網址（Webhook URL）。
2. 使用者觸發事件（例如完成結帳）。
3. 來源系統將事件資料打包成 JSON，發送到該網址。
4. 你的伺服器收到後，自動執行後續動作（例如寄出確認信、更新訂單狀態）。

常見應用場景

- 金流通知： 刷卡成功後，金流商立刻通知你的商店後台出貨。
- CI/CD 與開發： 推送程式碼到 GitHub 後，自動觸發建置與部署。
- 聊天機器人： 使用者在通訊軟體傳訊息時，立刻把訊息轉給你的後端處理並回覆。
- 
**目的**：確認 ESP32 能連上 Wi-Fi 並成功送訊息到 Discord。

**修改位置**
```cpp
char ssid[] = "你的WiFi名稱";
char pass[] = "你的WiFi密碼";
const char* discordWebhookUrl = "https://discord.com/api/webhooks/你的ID/你的TOKEN";
```

**核心：發送函式**
```cpp
WiFiClientSecure client;
client.setInsecure();               // 略過 SSL 憑證驗證，讓連線更簡單穩定
HTTPClient https;
https.begin(client, discordWebhookUrl);
https.addHeader("Content-Type", "application/json");
String jsonPayload = "{\"content\": \"" + message + "\"}";   // Discord 規定格式
int code = https.POST(jsonPayload);                           // 成功通常回 204
```

**預期結果**
- 序列埠：`Wi-Fi 連線成功！` → `發送成功！HTTP 回應碼: 204`
- Discord 頻道收到：`🤖 ESP32 機器人測試連線成功！系統已上線。`

**常見問題**

| 現象 | 可能原因 |
|---|---|
| 一直印 `.....` | Wi-Fi 帳密錯誤，或是 5GHz 網路 |
| 回應碼 401 / 404 | Webhook 網址貼錯或已被刪除 |
| 回應碼負數 | 網路或 HTTPS 連線失敗 |

> `loop()` 刻意留空，開機只發一次，避免短時間大量發送被 Discord 限制（rate limit）。

---
## 5. Webhook 通知感測器狀態（p02_7_discord_Vib_PIR）

**目的**：感測器一觸發，就自動推播到 Discord。

**設計重點**

1. **異常判斷**
   ```cpp
   bool isAbnormal = (vibVal == HIGH) || (pirVal > 0);   // 任一觸發即異常
   ```
   若希望「震動 **且** 有人」才通報，把 `||` 改成 `&&`。

2. **防洗頻機制**：僅在「剛由正常轉為異常」或「持續異常且超過 10 秒」才發送。
   ```cpp
   if (!wasAbnormal || (now - lastNotifyTime >= notifyCooldown)) { ... }
   ```

3. **訊息排版**：Discord 支援 Markdown，`**粗體**`、`` `等寬字` ``。
   JSON 內的換行要寫成 `\\n`（C++ 字串中的 `\\n` 送出後為 JSON 的 `\n`）。

**預期通知內容**
```
防盜系統異常警報！
• 震動感測器數值: 1 → 異常 (door open)
• PIR 人體感測值: 0 → 正常 (no one)
```

**測試方式**：搖動震動感測器或在 PIR 前揮手，約 1 秒內應收到通知。

---


## 6. Discord 機器人雙向告警（p02_8_discordBOT_Vib_PIR）

與 Webhook 的差異：Webhook 只能「單向送出」；機器人除了發訊息，還能**讀取頻道訊息**，達成遠端控制。

### 6-1 通訊架構

```
        ┌── 異常告警 / 回覆 ──▶
 ESP32 ─┤                         Discord 頻道 ◀── 你輸入 !alarm / !stop / !status
        └◀── 每 3 秒輪詢指令 ──┘
```

### 6-2 修改位置
```cpp
char ssid[] = "你的WiFi名稱";
char pass[] = "你的WiFi密碼";
const char* botToken  = "你的_BOT_TOKEN";
const char* channelId = "你的_CHANNEL_ID";
```

### 6-3 新增的硬體：蜂鳴器（GPIO 32）
蜂鳴器改為**非阻塞式**控制（用 `millis()` 計時，不用 `delay()`），因此感測、OLED、網路輪詢不會被卡住。警報為 C6 / G5 兩音交替，每 300ms 切換。

### 6-4 兩個 API 呼叫

| 方向 | 方法 | 網址 |
|---|---|---|
| ESP32 → Discord | `POST` | `https://discord.com/api/v10/channels/<ChannelID>/messages` |
| Discord → ESP32 | `GET` | `https://discord.com/api/v10/channels/<ChannelID>/messages?limit=1` |

驗證標頭：
```cpp
https.addHeader("Authorization", "Bot " + String(botToken));
```

### 6-5 指令處理流程（pollDiscordCommands）
1. 每 3 秒抓取頻道**最新 1 則**訊息（`limit=1`，省記憶體）。
2. 用 `ArduinoJson` 解析 `id`、`content`、`author.bot`。
3. 以 `lastProcessedMessageId` 避免同一則指令重複執行。
4. 忽略機器人自己發出的訊息（`author.bot`），避免自問自答。

### 6-6 可用指令

| 指令 | 功能 |
|---|---|
| `!alarm` | 遠端啟動蜂鳴器告警 |
| `!stop` | 解除告警並關閉蜂鳴器 |
| `!status` | 回報震動 / PIR 數值與蜂鳴器狀態 |

### 6-7 測試步驟
1. 上傳後觀察 OLED 顯示 `Connecting WiFi...`，之後顯示門 / 人員狀態。
2. Discord 收到上線通知與指令說明。
3. 在頻道輸入 `!status` → 約 3 秒內機器人回覆目前狀態。
4. 輸入 `!alarm` → 蜂鳴器鳴響；輸入 `!stop` → 停止。
5. 觸發感測器 → 收到異常告警。

> 📝 程式中 `//autoAlarmEndTime = now + 3000;` 被註解掉，表示「感測器觸發時**不**自動響蜂鳴器」，只發 Discord 通知；若想要本地同步響 3 秒，把註解拿掉即可。

### 6-8 常見問題

| 現象 | 解決方式 |
|---|---|
| 發送回應碼 403 | 機器人沒有該頻道的發言 / 檢視權限 |
| 發送回應碼 401 | Token 錯誤或已被重置 |
| 指令都沒反應，`content` 為空 | **沒開啟 `MESSAGE CONTENT INTENT`** |
| 編譯錯誤 `JsonDocument` 未定義 | ArduinoJson 版本太舊，請升級至 7.x |
| 指令延遲 | 屬正常，輪詢間隔為 3 秒（`pollInterval`） |

---
## 7. 加上時間標記（p02_9_discordBOT_time）

**新增內容**：以 NTP 網路校時，在通知與指令回覆中加入時間。

### 7-1 設定校時
```cpp
#include <time.h>
configTime(8 * 3600, 0, "pool.ntp.org", "time.nist.gov");   // 台灣 UTC+8 = 28800 秒
```
- 第一個參數：時區偏移（秒）；第二個：日光節約時間偏移（台灣無，填 0）。

### 7-2 取得格式化時間
```cpp
String getFormattedTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo, 100)) return "時間同步中...";
  char buf[30];
  strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", &timeinfo);
  return String(buf);
}
```

### 7-3 時間出現在哪裡
- 開機通知：`連線時間`
- 異常告警：`發生時間`
- `!status` 回覆：`查詢時間`

### 7-4 其他調整
- 蜂鳴器改為「**僅由 `!alarm` 指令控制**」，感測異常只發通知，不再自動響。
- 發送失敗時（回應碼非 200）會印出 Discord 回傳的錯誤內容，方便除錯。

**預期告警範例**
```
🚨 AIoT 安全系統異常告警！
• 發生時間: 2026-11-25 14:30:12
• 震動感測器數值: 1 → 異常 (door open)
• PIR 人體感測值: 0 → 正常 (noOne)
```

> 💡 若時間顯示「時間同步中...」，代表 NTP 尚未完成，稍待幾秒；若持續如此請確認網路可連外。

---

## 8. 補充：Blynk 控制（p02_10_blynk_buzz、p02_11_blynkFinal）

### 8-1 為什麼補充 Blynk？
Discord 適合「文字通知 + 指令」；Blynk 則提供**手機儀表板**（按鈕、燈號、數值顯示），操作更直覺。

### 8-2 前置準備
1. 註冊 <https://blynk.cloud>，建立 **Template**（硬體選 ESP32、連線方式 WiFi）。
2. 建立 **Datastream**（Virtual Pin，以下為建議配置，請依實際程式調整）：

| Virtual Pin | 名稱 | 資料型態 | 用途 |
|---|---|---|---|
| V0 | Buzzer | Integer 0~1 | 遠端控制蜂鳴器 |
| V1 | Vibration | Integer 0~1 | 顯示門狀態 |
| V2 | PIR | Integer 0~1 | 顯示人員狀態 |

3. 建立 **Device**，取得 `BLYNK_TEMPLATE_ID`、`BLYNK_TEMPLATE_NAME`、`BLYNK_AUTH_TOKEN`。
4. 手機安裝 Blynk App，建立 Dashboard（按鈕綁 V0、LED/Label 綁 V1、V2）。
5. Arduino IDE 安裝 **Blynk** 函式庫（見 0.1）。

### 8-3 p02_10_blynk_buzz：遠端控制蜂鳴器
- 學習重點：`BLYNK_WRITE(V0)` 接收 App 按鈕、控制 GPIO 32。
- 📎 （程式碼與說明待補）

### 8-4 p02_11_blynkFinal：完整整合
- 學習重點：感測器數值上傳（`Blynk.virtualWrite`）+ 蜂鳴器控制 + OLED +（可能含）Discord。
- 📎 （程式碼與說明待補）

### 8-5 Discord 與 Blynk 比較

| 項目 | Discord | Blynk |
|---|---|---|
| 主要用途 | 文字通知、指令 | 圖形儀表板、按鈕 |
| 控制方式 | 輸入指令 `!alarm` | 點擊按鈕 |
| 即時性 | 輪詢（約 3 秒） | 近即時 |
| 歷史紀錄 | 聊天紀錄即為紀錄 | 需另設定圖表 |

---

## 單元總結

1. **地端**：感測器 → OLED / LED / 蜂鳴器即時反應。
2. **單向通知**：Webhook 簡單好用，只能「發」。
3. **雙向控制**：Bot API 可「發 + 收」，實現遠端指令。
4. **時間標記**：NTP 校時讓每筆紀錄可追溯。
5. **進階**：Blynk 提供視覺化操作介面。

## 課後練習

1. 把異常條件改成「震動 **且** PIR 同時觸發」，觀察通知頻率變化。
2. 新增指令 `!led`，遠端切換板載 LED。
3. 讓感測器觸發時，蜂鳴器自動響 3 秒（取消註解 `autoAlarmEndTime`）。
4. 調整 `notifyCooldown` 與 `pollInterval`，比較「反應速度」與「訊息量」的取捨。
5. 在 `!status` 回覆中加入 Wi-Fi 訊號強度 `WiFi.RSSI()`。

## 附錄：常見錯誤速查

| 現象 | 檢查項目 |
|---|---|
| 編譯找不到 `Adafruit_SSD1306.h` | 未安裝函式庫 |
| OLED 不亮 | 接線、I2C 位址 `0x3C`、供電 |
| `tone` 未定義 | ESP32 開發板套件版本過舊，請更新 |
| Wi-Fi 連不上 | 帳密、2.4GHz、距離 |
| Discord 無訊息 | Token / Webhook / 頻道權限 / Intent |
| PIR 一直誤報 | 暖機未完成、供電不足（需 5V） |
