#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// LED ve ses ters çalışıyorsa (yaklaşınca yeşil/sessiz, uzaklaşınca kırmızı/alarm
// oluyorsa) bunu true yap:
const bool MESAFE_TERS = false;

LiquidCrystal_I2C* lcd;        // adres otomatik bulunacağı için pointer olarak tanımlandı
byte lcdAdresi = 0x27;         // bulunamazsa varsayılan

// HC-SR04 pinleri
const int trigPin = 2;
const int echoPin = 4;

// LED pinleri (her biri 220/330ohm direnç ile, katot GND'ye)
const int yesilPin   = 5;
const int sariPin    = 6;
const int kirmiziPin = 10;

// Pasif buzzer pini
const int buzzerPin = 9;

// Mesafe eşikleri (cm)
const int UZAK_MESAFE  = 50;
const int ORTA_MESAFE  = 25;
const int YAKIN_MESAFE = 10;

// LCD'nin I2C hattında hangi adreste cevap verdiğini bulur
byte lcdAdresBul() {
  for (byte adres = 1; adres < 127; adres++) {
    Wire.beginTransmission(adres);
    if (Wire.endTransmission() == 0) {
      return adres;
    }
  }
  return 0x27; // hiçbir cihaz bulunamazsa varsayılana dön
}

void setup() {
  Serial.begin(9600);
  Wire.begin();

  lcdAdresi = lcdAdresBul();
  Serial.print("LCD adresi bulundu: 0x");
  Serial.println(lcdAdresi, HEX);

  lcd = new LiquidCrystal_I2C(lcdAdresi, 16, 2);
  lcd->init();
  lcd->backlight();
  lcd->setCursor(0, 0);
  lcd->print("Park Sensoru");
  lcd->setCursor(0, 1);
  lcd->print("Hazirlaniyor...");
  delay(1500);
  lcd->clear();

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(yesilPin, OUTPUT);
  pinMode(sariPin, OUTPUT);
  pinMode(kirmiziPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);
}

float mesafeOlc() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long sureUs = pulseIn(echoPin, HIGH, 30000);
  if (sureUs == 0) return -1;

  return sureUs * 0.0343 / 2.0;
}

void ledAyarla(bool yesil, bool sari, bool kirmizi) {
  digitalWrite(yesilPin, yesil ? HIGH : LOW);
  digitalWrite(sariPin, sari ? HIGH : LOW);
  digitalWrite(kirmiziPin, kirmizi ? HIGH : LOW);
}

void lcdGuncelle(float mesafe, const char* durum) {
  lcd->setCursor(0, 0);
  lcd->print("Mesafe: ");
  if (mesafe < 0) {
    lcd->print("---     ");
  } else {
    lcd->print(mesafe, 0);
    lcd->print(" cm     ");
  }

  lcd->setCursor(0, 1);
  lcd->print("Durum: ");
  lcd->print(durum);
  int uzunluk = strlen(durum);
  for (int i = uzunluk; i < 9; i++) lcd->print(" ");
}

void loop() {
  float mesafe = mesafeOlc();

  Serial.print("Olculen mesafe: ");
  Serial.print(mesafe);
  Serial.println(" cm");

  if (mesafe < 0 || mesafe > 400) {
    // ölçüm alınamadı - ne güvenli ne tehlikeli say, LED/sesi kapat
    ledAyarla(false, false, false);
    noTone(buzzerPin);
    lcdGuncelle(-1, "Olcum yok");
    delay(30);
    return;
  }

  // durum: 0 = guvenli, 1 = dikkat, 2 = yaklasti, 3 = cok yakin
  int durum;
  if (mesafe > UZAK_MESAFE)      durum = 0;
  else if (mesafe > ORTA_MESAFE) durum = 1;
  else if (mesafe > YAKIN_MESAFE) durum = 2;
  else                             durum = 3;

  if (MESAFE_TERS) durum = 3 - durum; // ters çalışıyorsa sırayı çevir

  switch (durum) {
    case 0:
      ledAyarla(true, false, false);
      noTone(buzzerPin);
      lcdGuncelle(mesafe, "Guvenli");
      break;
    case 1: {
      ledAyarla(false, true, false);
      int frekans = map(mesafe, ORTA_MESAFE, UZAK_MESAFE, 1800, 900);
      tone(buzzerPin, frekans, 100);
      delay(300);
      noTone(buzzerPin);
      lcdGuncelle(mesafe, "Dikkat");
      break;
    }
    case 2: {
      ledAyarla(false, false, true);
      int frekans = map(mesafe, YAKIN_MESAFE, ORTA_MESAFE, 2800, 1800);
      tone(buzzerPin, frekans, 60);
      delay(100);
      noTone(buzzerPin);
      lcdGuncelle(mesafe, "Yaklasti");
      break;
    }
    case 3:
      ledAyarla(false, false, true);
      tone(buzzerPin, 3000);
      lcdGuncelle(mesafe, "DUR!");
      break;
  }

  delay(30);
}
