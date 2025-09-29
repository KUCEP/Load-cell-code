#include "HX711.h"

#define DT 3
#define SCK 2

HX711 scale;
float calibration_factor = 1.0;
float known_weight = 1000.0; // g, 분동 무게(수정가능)
// 일정 시간 동안 무게를 읽어서 평균값 반환
float averageReading(int seconds) {
  float sum = 0;
  for (int i = 0; i < seconds; i++) {
    sum += scale.get_units();
    delay(1000);
  }
  return sum / seconds;
}

void setup() {
  Serial.begin(9600);
  scale.begin(DT, SCK);
  Serial.println("HX711 Ready");
}

void loop() {
  // 시리얼 명령 처리
  if (Serial.available()) {
    String command = Serial.readStringUntil('\n');
    command.trim();

    if (command == "tare") {
      float avgZero = averageReading(60);
      scale.set_offset(avgZero);
      Serial.println("TARE_DONE");
    }
    else if (command == "cal") {
      float avgWeight = averageReading(60);
      calibration_factor = avgWeight / known_weight;  // 분동 기준 보정
      scale.set_scale(calibration_factor);
      Serial.println("CAL_DONE");
    }
  }

  // 1초마다 라즈베리파이로 무게값 전송
  float current_weight = scale.get_units();
  Serial.println(current_weight, 2);
  delay(1000);  // 1초 대기
}
