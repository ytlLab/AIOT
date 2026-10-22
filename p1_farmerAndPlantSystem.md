# 智慧農夫與植物診斷系統
運作流程：
- ESP32 使用 Timer 定時讀取土壤濕度、DHT22 溫濕度與光照數據。
- ESP32 透過 Blynk.virtualWrite() 將數據寫入 Blynk 雲端顯示於手機 App。
- 當到達設定的時間（或土壤過乾時），ESP32 自動打包數據，經由 SSL 直連 HTTP POST 呼叫 Gemini API 取得植物養護建議。
- ESP32 收到 Gemini 回傳的 JSON 文字後，直接發送 Webhook 到 Discord 頻道。若使用者在 Blynk 上按「澆水」，ESP32 直接驅動繼電器開關(使用紅綠燈模組模擬澆水狀態，紅燈表示需澆水，黃燈表示澆水中，綠燈表示澆水完畢)。

## 基本程式範例
| 元件         |   ESP32 |
| ---------- | ------: |
| DHT11 DATA |  GPIO 4 |
| 土壤 S      | GPIO 34 |
| 光敏 S     | GPIO 35 |
| 🔴 RED     | GPIO 27 |
| 🟡 YELLOW  | GPIO 26 |
| 🟢 GREEN   | GPIO 25 |

### 紅綠燈模組測試

```
// ========================================
// ESP32 紅綠燈模組測試
// ========================================

#define RED_LED     27
#define YELLOW_LED  26
#define GREEN_LED   25

void setup() {

  Serial.begin(115200);

  pinMode(RED_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);

  // 一開始全部關閉
  digitalWrite(RED_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(GREEN_LED, LOW);

  Serial.println("Traffic Light Test Start");
}

void loop() {

  // -------------------------
  // 紅燈
  // -------------------------

  Serial.println("RED - 需要澆水");

  digitalWrite(RED_LED, HIGH);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(GREEN_LED, LOW);

  delay(3000);

  // -------------------------
  // 黃燈
  // -------------------------

  Serial.println("YELLOW - 澆水中");

  digitalWrite(RED_LED, LOW);
  digitalWrite(YELLOW_LED, HIGH);
  digitalWrite(GREEN_LED, LOW);

  delay(3000);

  // -------------------------
  // 綠燈
  // -------------------------

  Serial.println("GREEN - 澆水完畢");

  digitalWrite(RED_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  digitalWrite(GREEN_LED, HIGH);

  delay(3000);
}
```

### DHT11 溫濕度模組測試

```
#include "DHTesp.h"
#define dhtpin 4   // 將dhtpin 定義為GPIO 4

DHTesp dht;         // 創建一個名為dht的溫溼度感測物件

void setup() {
  Serial.begin(115200);
  dht.setup(dhtpin, DHTesp::DHT11);  // 設定dht(腳位,型號)  
  delay(2000);                       // 先等感測器穩定
}

void loop() {
  TempAndHumidity data = dht.getTempAndHumidity();  // 讀取溫濕度
  Serial.print("T: ");
  Serial.print(data.temperature,1);  // 取到小數點後第1位
  Serial.print(" H: ");
  Serial.println(data.humidity,0);   // 取整數
  delay(2000); 
}
```

### 土壤濕度模組測試
先確認感測器的原始數值，根據實際測量結果修改SOIL_DRY_VALUE與SOIL_WET_VALUE。
```
// ========================================
// ESP32 土壤濕度百分比測試
// ========================================

#define SOIL_PIN 34
// 根據實際測量結果修改
#define SOIL_DRY_VALUE 0
#define SOIL_WET_VALUE 1100

void setup() {

  Serial.begin(115200);

  Serial.println();
  Serial.println("======================");
  Serial.println("Soil Moisture Test");
  Serial.println("======================");
}
void loop() {

  int soilRaw = analogRead(SOIL_PIN);
  int soilPercent = map(soilRaw,SOIL_DRY_VALUE,SOIL_WET_VALUE,0,100);

  // 限制在 0～100
  soilPercent = constrain(soilPercent,0,100);

  Serial.print("ADC = ");
  Serial.print(soilRaw);
  Serial.print("Soil = ");
  Serial.print(soilPercent);
  Serial.println(" %");
  delay(1000);
}
```

### 光敏電阻模組測試
```
// ========================================
// ESP32 光敏電阻模組測試
// ========================================
#define LIGHT_PIN 35

void setup() {

  Serial.begin(115200);

  Serial.println();
  Serial.println("======================");
  Serial.println("Light Sensor Test");
  Serial.println("======================");
}

void loop() {

  int lightRaw =analogRead(LIGHT_PIN);
  Serial.print("Light ADC = ");
  Serial.println(lightRaw);

  delay(500);
}
```
綜合測試
```
// ======================================================
// ESP32 AIoT 智慧農夫 - 植物環境監控系統
//
// 功能：
// 1. DHT11：溫度、空氣濕度
// 2. 土壤濕度：0~100%
// 3. 光敏電阻：光照 ADC
// 4. 紅綠燈：植物澆水狀態
//
// 紅燈：土壤過乾，需要澆水
// 黃燈：澆水中
// 綠燈：澆水完成 / 不需要澆水
//
// ======================================================

// ======================================================
// GPIO 腳位設定
// ======================================================

// 紅綠燈
#define RED_LED     27
#define YELLOW_LED  26
#define GREEN_LED   25
// DHT11
#define DHT_PIN     4
// 土壤濕度
#define SOIL_PIN    34
// 光敏電阻
#define LIGHT_PIN   35

// ======================================================
// DHT11
// ======================================================
#include "DHTesp.h"
DHTesp dht;

// ======================================================
// 土壤濕度校正值
// ======================================================
// 依照你的實際測試結果設定
#define SOIL_DRY_VALUE  0
#define SOIL_WET_VALUE  1100

// ======================================================
// 土壤過乾判斷值
// ======================================================
// 小於 30% → 需要澆水
#define SOIL_DRY_LEVEL 30

// ======================================================
// 讀取時間
// ======================================================
// 感測器每 2 秒讀取一次
unsigned long previousMillis = 0;
const unsigned long SENSOR_INTERVAL = 2000;

// ======================================================
// 系統狀態
// ======================================================
enum PlantState {
  NEED_WATER,     // 需要澆水
  WATERING,       // 澆水中
  WATERED         // 澆水完成 / 不需要澆水
};
PlantState plantState = WATERED;

// ======================================================
// LED 函式
// ======================================================
void redLight() {
  digitalWrite(RED_LED,HIGH);
  digitalWrite(YELLOW_LED,LOW);
  digitalWrite(GREEN_LED,LOW);
}
void yellowLight() {
  digitalWrite(RED_LED,LOW);
  digitalWrite(YELLOW_LED,HIGH);
  digitalWrite(GREEN_LED,LOW);
}
void greenLight() {
  digitalWrite(RED_LED,LOW);
  digitalWrite(YELLOW_LED,LOW);
  digitalWrite(GREEN_LED,HIGH);
}

// ======================================================
// 顯示植物狀態
// ======================================================

void showPlantState() {
  Serial.print("植物狀態：");
  switch (plantState) {
    case NEED_WATER:
      Serial.println("需要澆水");
      redLight();
      break;

    case WATERING:
      Serial.println("澆水中");
      yellowLight();
      break;

    case WATERED:
      Serial.println("澆水完成 / 不需要澆水");
      greenLight();
      break;
  }
}


// ======================================================
// 讀取所有感測器
// ======================================================
void readSensors() {
  // ====================================================
  // DHT11
  // ====================================================
  TempAndHumidity data = dht.getTempAndHumidity();
  float temperature = data.temperature;
  float humidity = data.humidity;

  // ====================================================
  // 土壤濕度
  // ====================================================
  int soilRaw =analogRead(SOIL_PIN);
  int soilPercent = map(soilRaw,SOIL_DRY_VALUE,SOIL_WET_VALUE,0,100);
  // 限制在 0～100%
  soilPercent =constrain(soilPercent,0,100);

  // ====================================================
  // 光照
  // ====================================================
  int lightRaw = analogRead(LIGHT_PIN);

  // ====================================================
  // 顯示資料
  // ====================================================
  Serial.println();
  Serial.println("==========================================");
  Serial.println("🌱 植物環境監控系統");
  Serial.println("==========================================");

  // 溫度
  Serial.print("🌡 溫度：");
  Serial.print(temperature,1);
  Serial.println(" °C");

  // 空氣濕度
  Serial.print("💧 空氣濕度：");
  Serial.print(humidity,0);
  Serial.println(" %");

  // 土壤 ADC
  Serial.print("🌱 土壤 ADC：");
  Serial.println(soilRaw);

  // 土壤濕度
  Serial.print("🌱 土壤濕度：");
  Serial.print(soilPercent);
  Serial.println(" %");

  // 光照
  Serial.print("☀️ 光照 ADC：");
  Serial.println(lightRaw);

  // ====================================================
  // 土壤狀態判斷
  // ====================================================
  /*
   * 如果目前不是澆水狀態，
   * 才根據土壤濕度決定紅燈或綠燈。
   */
  if (plantState != WATERING) {
    if (soilPercent < SOIL_DRY_LEVEL    ) {
      // 土壤過乾
      plantState = NEED_WATER;
    }
    else {
      // 土壤正常
      plantState = WATERED;
    }
  }

  // ====================================================
  // 顯示 LED 狀態
  // ====================================================
  showPlantState();
  Serial.println("==========================================");
}

void setup() {
  Serial.begin(115200);
  // ====================================================
  // 紅綠燈 GPIO
  // ====================================================
  pinMode(RED_LED,OUTPUT);
  pinMode(YELLOW_LED,OUTPUT);
  pinMode(GREEN_LED,OUTPUT);
  // 一開始全部關閉
  digitalWrite(RED_LED,LOW);
  digitalWrite(YELLOW_LED,LOW);
  digitalWrite(GREEN_LED,LOW);

  // ====================================================
  // DHT11
  // ====================================================
  dht.setup(DHT_PIN,DHTesp::DHT11);
  // 等待 DHT11 穩定
  delay(2000);

  // ====================================================
  // 啟動訊息
  // ====================================================
  Serial.println();
  Serial.println(
    "=========================================="
  );

  Serial.println("ESP32 植物環境監控系統");
  Serial.println("==========================================");
  Serial.println("DHT11      → GPIO 4");
  Serial.println("Soil      → GPIO 34");
  Serial.println("Light     → GPIO 35");
  Serial.println("RED       → GPIO 25");
  Serial.println("YELLOW    → GPIO 26");
  Serial.println("GREEN     → GPIO 27");
  Serial.println("==========================================");
  // 預設綠燈
  greenLight();
}

void loop() {
  unsigned long currentMillis =  millis();
  // ====================================================
  // 每 2 秒讀取一次
  // ====================================================

  if (currentMillis - previousMillis >= SENSOR_INTERVAL) 
  {
    previousMillis = currentMillis;
    readSensors();
  }
}
```
## Blynk
Blynk 是一個專為物聯網（IoT）設計的雲端平台與應用程式開發工具，能讓使用者快速建立硬體專案的遠端控制與監測介面。

1. 主要功能與組成元件
- Blynk IoT App (iOS / Android)：提供手機端的操作介面，可透過拖曳方式快速建立按鈕、滑桿、圖表與指示燈。
- Blynk.Console：強大的網頁管理後台，可用來管理裝置、使用者帳號、檢視歷史數據以及執行遠端 OTA（空中下載）韌體更新。
- Blynk.Cloud：提供安全託管、雲端基礎架構、24/7 事件監控與端對端加密。
- 虛擬腳位（Virtual Pins）：硬體與雲端、App 之間傳遞資料的核心機制（例如 V0、V1、V2 等），可用來傳送開關狀態、感測器數值等。

2. 支援的硬體與連線方式
- 支援硬體：相容超過 400 種開發板與微控制器，包含 ESP32、ESP8266、Arduino、Raspberry Pi（樹莓派）、Seeed Wio Terminal 等。
- 連線協定：支援 Wi-Fi、乙太網路、蜂巢網路（GSM/LTE）、LoRaWAN、HTTP(s) 及 MQTT。

### Blynk Web 端設定
登入 [Blynk](https://www.blynk.io/) 控制台進行設定

### 建立 Template (模板)
1. 點擊左側 Developer Zone -> Templates -> + New Template。
2. Name 設定為 Smart Plant，Hardware 選擇 ESP32，Connection Type 選擇 WiFi，完成後點擊 Done。

<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/7a82993c-f5fd-491b-8588-ca9b5b7ded8a" />
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/89c5ad47-e8bb-4906-a3ec-4cb1892620db" />
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/a5d577d2-2904-4072-bdd7-b2bf959a84da" />
<img width="400" height="200" alt="image" src="https://github.com/user-attachments/assets/5164c42c-1b79-4373-a046-6314e07333ea" />

系統會提示以下訊息，請先複製到記事本或WORD暫存。</br>
<img width="300" height="150" alt="image" src="https://github.com/user-attachments/assets/f16466d8-d53e-4030-bbde-8372e1ac67da" />



### 設定 Datastreams (資料流)
進入 Template 後點選 Datastreams 標籤頁，建立以下 5 個 Virtual Pins：
<img width="800" height="400" alt="image" src="https://github.com/user-attachments/assets/887ab1ef-b081-4323-a4c5-8ff105e8fef9" />

| 名稱 | Virtual Pin | 資料類型 (Data Type) | 最小值 | 最大值 |
| :--- | :--- | :--- |:--- |:--- |
| 溫度 | V0 | Double / Float | 0 | 50 |
| 空氣濕度 | V1 | Double / Float | 0 | 100 |
| 土壤濕度 | V2 | Integer / Float | 0 | 100 |
| 光照度 | V3 | Integer / Float | 0 | 4096 |
| 澆水按鈕 | V4 | Integer / Float | 0 | 1 |

### 設定 Web / Mobile Dashboard (儀表板)
