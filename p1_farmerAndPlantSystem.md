
# 智慧農夫與植物診斷系統
運作流程：
- ESP32 使用 Timer 定時讀取土壤濕度、DHT22 溫濕度與光照數據。
- ESP32 透過 Blynk.virtualWrite() 將數據寫入 Blynk 雲端顯示於手機 App。
- 當到達設定的時間（或土壤過乾時），ESP32 自動打包數據，經由 SSL 直連 HTTP POST 呼叫 Gemini API 取得植物養護建議。
- ESP32 收到 Gemini 回傳的 JSON 文字後，直接發送 Webhook 到 Discord 頻道。若使用者在 Blynk 上按「澆水」，ESP32 直接驅動繼電器開關(使用紅綠燈模組模擬澆水狀態，紅燈表示需澆水，黃燈表示澆水中，綠燈表示澆水完畢)。

## 基本程式範例

### 紅綠燈模組

```
// ========================================
// ESP32 紅綠燈模組測試
// ========================================

#define RED_LED     25
#define YELLOW_LED  26
#define GREEN_LED   27

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

### DHT11 溫濕度模組

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
