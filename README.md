```c

#include <WiFiClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <NTPClient.h>
#include <WiFi.h> // Thư viện WiFi cho ESP32
#include <RTClib.h>

#include <PubSubClient.h>

#define DEC 10

#define SSID "sometime"
#define PASSWORD "zxcvbnmm"




#define lightSensA1 34
#define lightSensA16 35
#define lightSensCT 32

#define ledA1 15
#define ledA16 2
#define ledCT 4 

#define SDA 21
#define SCL 22

#define pirInA1 13
#define pirOutA1 12
#define pirInA16 14 
#define pirOutA16 27
#define pirInCT 26
#define pirOutCT 25

RTC_DS3231 rtc;

LiquidCrystal_I2C lcd(0x27, 16, 2);
WiFiUDP ntpUDP;

const int ngay = 22;
const int thang = 11;
const int nam = 2024;

const int gio = 3;
const int phut = 30;
const int giay = 0;

bool isSummerMode = false;
bool isWinterMode = false;

bool TurnOn3Duong = false;

bool TurnOnA1AndA16 =false;
bool TurnOnA1AndCT = false;
bool TurnOnCTAndA16 = false; 

bool TurnOnA1 = false;
bool TurnOnA16 = false;
bool TurnOnCT = false;


bool isA1On = false;
bool isA16On = false;
bool isCTOn = false;

bool isA1Motion = false;
bool isA16Motion = false;
bool isCTMotion = false;

unsigned long preMillisA1 = 0;
unsigned long preMillisA16 = 0;
unsigned long preMillisCT = 0;

bool isA1MotionAndDark = false;
bool isA16MotionAndDark = false;
bool isCTMotionAndDark = false;

unsigned long preMillisA1_2 = 0;
unsigned long preMillisA16_2 = 0;
unsigned long preMillisCT_2 = 0;

const int nguongSang = 3000;
const int setSeconds = 5000;
// TEST LUNG TUNG
unsigned long mode = 3;
unsigned long previousMillis = 0; // Biến để lưu trữ thời gian trước đó
const unsigned long interval = 60000; // Khoảng thời gian (milliseconds) giữa mỗi lần tăng, 1 phút = 60000 milliseconds
// ----
void turnOnA1Led() 
{
  digitalWrite(ledA1, HIGH);
  isA1On = true;
}
void turnOffA1Led() 
{
  digitalWrite(ledA1, LOW);
  isA1On = false;
}
void turnOnA16Led() 
{
  digitalWrite(ledA16, HIGH);
  isA16On = true;
}
void turnOffA16Led() 
{
  digitalWrite(ledA16, LOW);
  isA16On = false;
}
void turnOnCTLed()
{
  digitalWrite(ledCT, HIGH);
  isCTOn = true;
}
void turnOffCTLed()
{
  digitalWrite(ledCT, LOW);
  isCTOn = false;
}
void turnOnAllLed() 
{
  digitalWrite(ledA1, HIGH);
  digitalWrite(ledA16, HIGH);
  digitalWrite(ledCT, HIGH); 
  isA1On = true;
  isA16On = true;
  isCTOn =true;
}
void turnOffAllLed() 
{
  digitalWrite(ledA1, LOW);
  digitalWrite(ledA16, LOW);
  digitalWrite(ledCT, LOW); // Tắt LED C
  isA1On = false;
  isA16On = false;
  isCTOn = false;
}
bool motionDetectedA1() 
{
  if (digitalRead(pirInA1) == HIGH) 
  {
    return true;
  } 
  else if (digitalRead(pirInA1) == LOW && digitalRead(pirOutA1) == HIGH)
  {
    return true;
  }
  else
  {
    return false;
  }
}
bool motionDetectedA16() 
{
  if (digitalRead(pirInA16) == HIGH) 
  {
    return true;
  } 
  else if (digitalRead(pirInA16) == LOW && digitalRead(pirOutA16) == HIGH)
  {
    return true;
  }
  else
  {
    return false;
  }
}
bool motionDetectedCT() 
{
  if (digitalRead(pirInCT) == HIGH) 
  {
    return true;
  } 
  else if (digitalRead(pirInCT) == LOW && digitalRead(pirOutCT) == HIGH)
  {
    return true;
  }
  else
  {
    return false;
  }
}


bool isA1Dark(int lightValueA1, int nguongSang) 
{
  if (lightValueA1 > nguongSang) {
    return true;
  } else {
    return false;
  }
}

bool isA16Dark(int lightValueA16, int nguongSang) 
{
  if (lightValueA16 > nguongSang) {
    return true;
  } else {
    return false;
  }
}

bool isCTDark(int lightValueCT, int nguongSang) 
{
  if (lightValueCT > nguongSang) {
    return true;
  } else {
    return false;
  }
}
// ----------------------------------SETUP--------------------------------------------
void setup() 
{
  pinMode(pirInA1, INPUT);
  pinMode(pirOutA1, INPUT);
  pinMode(pirInA16, INPUT);
  pinMode(pirOutA16, INPUT);
  pinMode(pirInCT, INPUT);
  pinMode(pirOutCT, INPUT);

  pinMode(ledA1, OUTPUT);
  pinMode(ledA16, OUTPUT);
  pinMode(ledCT, OUTPUT); 

  WiFi.begin(SSID, PASSWORD);

  Wire.begin(21, 22);
  lcd.init();
  lcd.begin(16, 2);
  lcd.backlight();

  Serial.begin(115200);

  // Kiểm tra xem RTC có kết nối đúng không
  if (!rtc.begin()) {
    Serial.println("Không tìm thấy RTC");
    while (1);
  }

  rtc.adjust(DateTime(nam, thang, ngay, gio, phut, giay));
  // if (rtc.lostPower()) {
  //   Serial.println("RTC mất nguồn, thiết lập lại thời gian!");
  //   // Đặt lại thời gian cho RTC (năm, tháng, ngày, giờ, phút, giây)
  //   rtc.adjust(DateTime(2024, 5, 22, 11, 30, 0)); // Ví dụ: Đặt thời gian là 11:30:00 ngày 22/05/2024
  // }

  if(thang > 4 && thang < 9) 
  {
    isSummerMode = true;
    isWinterMode = false;
  }
  else
  {
    isSummerMode = false;
    isWinterMode = true;
  }

}
// ---------------------------------- LOOP--------------------------------------------
void loop() 
{
  //TEST LUNG TUNG
  unsigned long currentMillis = millis(); // Lấy thời gian hiện tại

  // Kiểm tra nếu đã đủ khoảng thời gian mong muốn
  if (currentMillis - previousMillis >= interval) {
    previousMillis = currentMillis; // Cập nhật thời gian trước đó

    mode++; // Tăng giá trị của mode
    turnOffAllLed();

    // Kiểm tra nếu mode vượt quá 6, quay lại 1
    if (mode > 6) mode = 3;
  }
  
  if(mode == 1){
    lcd.setCursor(10, 1);
    lcd.print("summer");
  }else if(mode == 2){
    lcd.setCursor(10, 1);
    lcd.print("winter");
  }else if (mode == 3){
    lcd.setCursor(10, 1);
    lcd.print("AllLed");
  }else if (mode == 4){
    lcd.setCursor(10, 1);
    lcd.print("A1&A16");
  }else if(mode == 5){
    lcd.setCursor(10, 1);
    lcd.print("A1&CT");
  }else if(mode == 6){
    lcd.setCursor(10, 1);
    lcd.print("CT&A16");
  }
  






  DateTime now = rtc.now();
  lcd.setCursor(0, 0);

  if(now.hour() < 10) lcd.print('0');
  lcd.print(now.hour(), DEC);
  lcd.print(':');

  if(now.minute() < 10) lcd.print('0');
  lcd.print(now.minute(), DEC);

  lcd.print(':');
  if(now.second() < 10) lcd.print('0');
  lcd.print(now.second(), DEC);

  int nguongSang = 3000;
  int lightValueA1 = analogRead(lightSensA1);
  int lightValueA16 = analogRead(lightSensA16);
  int lightValueCT = analogRead(lightSensCT);

  lcd.setCursor(10, 0);
  lcd.print(lightValueCT);
  lcd.print("    ");

  int hrs = now.hour();
  int min = now.minute();

  // ------------------------------Summer-----------------------------------------
  if (isSummerMode && mode == 1) 
  {
    // lcd.setCursor(14, 1);
    // lcd.print("S");

    if ((hrs == 18 && min >= 30) || (hrs > 18 && hrs < 23)) 
    {
      turnOnAllLed();
    } 
    else if ((hrs >= 23) || (hrs < 5) || (hrs == 5 && min < 50)) 
    {
      unsigned long curMillis = millis();

      if (motionDetectedA1()) 
      {
        lcd.setCursor(0, 1);
        lcd.print("A1");
        isA1Motion = true;
        preMillisA1 = curMillis;
        turnOnA1Led(); 
      }
      if (isA1Motion && (curMillis - preMillisA1 >= setSeconds)) 
      {
        isA1Motion = false;
        turnOffA1Led();
        lcd.setCursor(0, 1);
        lcd.print("  ");
      }

      if (motionDetectedA16()) 
      {
        lcd.setCursor(2, 1);
        lcd.print("16");
        isA16Motion = true;
        preMillisA16 = curMillis;
        turnOnA16Led(); 
      }
      if (isA16Motion && (curMillis - preMillisA16 >= setSeconds)) 
      {
        isA16Motion = false;
        turnOffA16Led();
        lcd.setCursor(2, 1);
        lcd.print("   ");
      }

      if (motionDetectedCT()) 
      {
        lcd.setCursor(4, 1);
        lcd.print("CT");
        isCTMotion = true;
        preMillisCT = curMillis;
        turnOnCTLed(); 
      }
      if (isCTMotion && (curMillis - preMillisCT >= setSeconds)) 
      {
        isCTMotion = false;
        turnOffCTLed();
        lcd.setCursor(4, 1);
        lcd.print("  ");
      }
    } 

    else if ((hrs == 5 && min >= 50) || (hrs > 5 && hrs < 18) || (hrs == 18 && min < 30)) 
    {
      unsigned long curMillis_2 = millis();

      if (motionDetectedA1() && isA1Dark(lightValueA1, nguongSang)) 
      {
        lcd.setCursor(0, 1);
        lcd.print("A1");
        isA1MotionAndDark = true;
        preMillisA1_2 = curMillis_2;
        turnOnA1Led();
      } 
      if (isA1MotionAndDark && (curMillis_2 - preMillisA1_2 >= setSeconds)) 
      {
        isA1MotionAndDark = false;
        turnOffA1Led();
        lcd.setCursor(0, 1);
        lcd.print("  ");
      }

      if (motionDetectedA16() && isA16Dark(lightValueA16, nguongSang)) 
      {
        lcd.setCursor(2, 1);
        lcd.print("16");
        isA16MotionAndDark = true;
        preMillisA16_2 = curMillis_2;
        turnOnA16Led();
      } 
      if (isA16MotionAndDark && (curMillis_2 - preMillisA16_2 >= setSeconds))
      {
        isA16MotionAndDark = false;
        turnOffA16Led();        
        lcd.setCursor(2, 1);
        lcd.print("  ");
      }

      if(motionDetectedCT() && isCTDark(lightValueCT, nguongSang))
      {
        lcd.setCursor(4, 1);
        lcd.print("CT");
        isCTMotionAndDark = true;
        preMillisCT_2 = curMillis_2;
        turnOnCTLed();
      }
      if(isCTMotionAndDark && (curMillis_2 - preMillisCT_2 >= setSeconds))
      {
        isCTMotionAndDark = false;
        turnOffCTLed();
        lcd.setCursor(4, 1);
        lcd.print("  ");
      }
    }
  }
  // -----------------------------Winter--------------------------------------
  if (isWinterMode && mode == 2) 
  {
    // lcd.setCursor(15, 1);
    // lcd.print("W");

    if ((hrs == 17 && min >= 45) || (hrs > 17 && hrs < 23)) 
    {
      turnOnAllLed();
    } 
    else if ((hrs >= 23) || (hrs < 5) || (hrs == 5 && min < 20)) 
    {
      unsigned long curMillis = millis();

      if (motionDetectedA1()) 
      {
        lcd.setCursor(0, 1);
        lcd.print("A1");
        isA1Motion = true;
        preMillisA1 = curMillis;
        turnOnA1Led(); 
      }
      if (isA1Motion && (curMillis - preMillisA1 >= setSeconds)) 
      {
        isA1Motion = false;
        turnOffA1Led();
        lcd.setCursor(0, 1);
        lcd.print("  ");
      }

      if (motionDetectedA16()) 
      {
        lcd.setCursor(2, 1);
        lcd.print("16");
        isA16Motion = true;
        preMillisA16 = curMillis;
        turnOnA16Led(); 
      }
      if (isA16Motion && (curMillis - preMillisA16 >= setSeconds)) 
      {
        isA16Motion = false;
        turnOffA16Led();
        lcd.setCursor(2, 1);
        lcd.print("   ");
      }

      if (motionDetectedCT()) 
      {
        lcd.setCursor(4, 1);
        lcd.print("CT");
        isCTMotion = true;
        preMillisCT = curMillis;
        turnOnCTLed(); 
      }
      if (isCTMotion && (curMillis - preMillisCT >= setSeconds)) 
      {
        isCTMotion = false;
        turnOffCTLed();
        lcd.setCursor(4, 1);
        lcd.print("  ");
      }
    } 

    else if ((hrs == 5 && min >= 20) || (hrs > 5 && hrs < 17) || (hrs == 17 && min < 45)) 
    {
      unsigned long curMillis_2 = millis();

      if (motionDetectedA1() && isA1Dark(lightValueA1, nguongSang)) 
      {
        lcd.setCursor(0, 1);
        lcd.print("A1");
        isA1MotionAndDark = true;
        preMillisA1_2 = curMillis_2;
        turnOnA1Led();
      } 
      if (isA1MotionAndDark && (curMillis_2 - preMillisA1_2 >= setSeconds)) 
      {
        isA1MotionAndDark = false;
        turnOffA1Led();
        lcd.setCursor(0, 1);
        lcd.print("  ");
      }

      if (motionDetectedA16() && isA16Dark(lightValueA16, nguongSang)) 
      {
        lcd.setCursor(2, 1);
        lcd.print("16");
        isA16MotionAndDark = true;
        preMillisA16_2 = curMillis_2;
        turnOnA16Led();
      } 
      if (isA16MotionAndDark && (curMillis_2 - preMillisA16_2 >= setSeconds))
      {
        isA16MotionAndDark = false;
        turnOffA16Led();        
        lcd.setCursor(2, 1);
        lcd.print("  ");
      }

      if(motionDetectedCT() && isCTDark(lightValueCT, nguongSang))
      {
        lcd.setCursor(4, 1);
        lcd.print("CT");
        isCTMotionAndDark = true;
        preMillisCT_2 = curMillis_2;
        turnOnCTLed();
      }
      if(isCTMotionAndDark && (curMillis_2 - preMillisCT_2 >= setSeconds))
      {
        isCTMotionAndDark = false;
        turnOffCTLed();
        lcd.setCursor(4, 1);
        lcd.print("  ");
      }
    }                  
  }
  if (mode == 3)
  {
    turnOnAllLed();
  }
  if(mode == 4)
  {
    turnOnA1Led();
    turnOnA16Led();
  }
  if(mode == 5)
  {
    turnOnA1Led();
    turnOnCTLed();
  }
  if(mode == 6)
  {
    turnOnCTLed();
    turnOnA16Led();
  }
}
```
