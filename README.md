# adruino_4-

4位共阳数码管，可以开机动画，但是字符有限，可以正常写数字，字母切换有点问题，没加运算，没有加计时

#include "SevSeg.h"

SevSeg sevseg;

byte set = COMMON_ANODE;
byte number = 4;

byte COM[]  = {10, 11, 12, 13};
byte pins[] = {2, 3, 4, 5, 6, 7, 8, 9};

String buffer = "";
bool   isLetterMode = false;

long   displayNum = 0;
int    displayDecimal = -1;
String displayChars = "";

const String VALID_LETTERS = "AbCdEFGHIJLnOoPqrStUuy";


// =========================
// 滚动显示
// XXXX
// XXX0
// XX0P
// X0PE
// 0PEn
// PEn!
// En!X
// n!XX
// !XXX
// XXXX
// =========================
void scrollText(String text, int delayMs) {

  // 前后补空格
  String full = "    " + text + "    ";

  // 滑动窗口
  for (int i = 0; i <= full.length() - 4; i++) {

    String frame = full.substring(i, i + 4);

    unsigned long start = millis();

    while (millis() - start < delayMs) {

      // 先正常显示
      sevseg.setChars(frame.c_str());

      // 处理 !
      for (int j = 0; j < 4; j++) {

        if (frame[j] == '!') {

          // ! = 1 + DP
          sevseg.setSegmentsDigit(j, 0b10000110);
        }
      }

      sevseg.refreshDisplay();
    }
  }
}


// =========================
// 数字解析
// =========================
bool parseBuffer(long &outNum, int &outDecimal) {

  if (buffer.length() == 0)
    return false;

  bool neg = buffer.startsWith("-");

  String digits =
    neg ? buffer.substring(1) : buffer;

  int dotPos = digits.indexOf('.');

  String intPart =
    (dotPos == -1)
    ? digits
    : digits.substring(0, dotPos);

  String fracPart =
    (dotPos == -1)
    ? ""
    : digits.substring(dotPos + 1);

  String numStr = intPart + fracPart;

  if (numStr.length() == 0)
    numStr = "0";

  outNum = numStr.toInt();

  if (neg)
    outNum = -outNum;

  outDecimal =
    (dotPos == -1)
    ? -1
    : (int)fracPart.length();

  return true;
}


// =========================
// 更新显示缓存
// =========================
void updateDisplay() {

  if (isLetterMode) {

    if (buffer.length() > 0)
      displayChars = buffer;

  } else {

    long tmpNum;
    int tmpDec;

    if (parseBuffer(tmpNum, tmpDec)) {

      displayNum = tmpNum;
      displayDecimal = tmpDec;
    }
  }
}


// =========================
// setup
// =========================
void setup() {

  Serial.begin(9600);

  sevseg.begin(set, number, COM, pins);

  sevseg.setBrightness(90);

  buffer.reserve(10);

  displayChars.reserve(5);

  // 开机动画
  scrollText("OPEn! OnO", 500);

  // 最后显示 0
  displayNum = 0;
}


// =========================
// loop
// =========================
void loop() {

  while (Serial.available()) {

    char c = Serial.read();

    // 回车
    if (c == '\n' || c == '\r') {

      buffer = "";
      isLetterMode = false;
      displayChars = "";

      continue;
    }

    // 退格
    if (c == '\b' || c == 127) {

      if (buffer.length() > 0)
        buffer.remove(buffer.length() - 1);

      if (buffer.length() == 0) {

        isLetterMode = false;
        displayChars = "";
      }

      updateDisplay();

      continue;
    }

    // 字母
    if (VALID_LETTERS.indexOf(c) >= 0) {

      if (!isLetterMode) {

        buffer = "";
        isLetterMode = true;
      }

      if (buffer.length() < 4)
        buffer += c;

      updateDisplay();

      continue;
    }

    // 字母模式忽略数字
    if (isLetterMode)
      continue;

    // 负号
    if (c == '-') {

      if (buffer.length() == 0)
        buffer += '-';

      updateDisplay();

      continue;
    }

    // 小数点
    if (c == '.') {

      String noSign =
        buffer.startsWith("-")
        ? buffer.substring(1)
        : buffer;

      if (
        noSign.length() > 0 &&
        noSign.indexOf('.') == -1
      ) {
        buffer += '.';
      }

      updateDisplay();

      continue;
    }

    // 数字
    if (c >= '0' && c <= '9') {

      String trial = buffer + c;

      bool neg = trial.startsWith("-");

      String digits =
        neg ? trial.substring(1) : trial;

      int dotPos = digits.indexOf('.');

      String intPart =
        (dotPos == -1)
        ? digits
        : digits.substring(0, dotPos);

      String fracPart =
        (dotPos == -1)
        ? ""
        : digits.substring(dotPos + 1);

      int totalDigits =
        intPart.length() + fracPart.length();

      int maxDigits = neg ? 3 : 4;

      if (totalDigits <= maxDigits)
        buffer = trial;

      updateDisplay();
    }
  }

  // 实时显示
  if (displayChars.length() > 0) {

    sevseg.setChars(displayChars.c_str());

  } else {

    sevseg.setNumber(displayNum, displayDecimal);
  }

  sevseg.refreshDisplay();
}
