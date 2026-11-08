# Firmware

#include <Adafruit_Fingerprint.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

// A. KHAI BÁO CẤU HÌNH
// 1. AS608
HardwareSerial mySerial(2);
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);

// 3. LCD
LiquidCrystal_I2C lcd(0x27, 16, 2);

// 4. Servo
Servo servoIn;
Servo servoOut;
const int pinIn = 18;  // Servo Check-in
const int pinOut = 19; // Servo Check-out

// Biến trạng thái riêng cho từng cổng
unsigned long inOpenTime = 0;
unsigned long outOpenTime = 0;
bool inGateOpen = false;
bool outGateOpen = false;

const int tongsocho = 100;
int sochotrong = 100;

const unsigned long thoigiancho = 5000;

// B. KHỞI TẠO HỆ THỐNG
void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("SMART PARKING");
  lcd.setCursor(0, 1);
  lcd.print("System Booting..");
  delay(2000);

  mySerial.begin(57600, SERIAL_8N1, 16, 17);
  finger.begin(57600);
  delay(1000);

  // Khởi tạo Servo IN
  servoIn.attach(pinIn, 500, 2400);
  servoIn.write(0);
  delay(1000);
  servoIn.detach();

  // Khởi tạo Servo OUT
  servoOut.attach(pinOut, 500, 2400);
  servoOut.write(180);
  delay(1000);
  servoOut.detach();

  hienThiManHinhCho();
}

// C. MAIN LOOP
void loop() {
  int fingerID = getFingerprintID();

  if (fingerID < 0) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("TU CHOI VAO");
    lcd.setCursor(0, 1);
    if (fingerID == -1) lcd.print("Dat lai tay");
    if (fingerID == -2) lcd.print("Chua dang ky");

    delay(2000);
    hienThiManHinhCho();
  }

  if (fingerID > 0) {
    // LUÔN CHO QUÉT VÀ GỬI LÊN SERVER KIỂM TRA TRƯỚC
    Serial.println("VERIFY:" + String(fingerID));

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print(" Dang kiem tra  ");
    lcd.setCursor(0, 1);
    lcd.print("Xin cho chut");

    unsigned long thoiGianBatDauCho = millis();
    bool isApproved = false;
    bool isTimeout = true;
    int direction = 0;  // 1 là VÀO, 2 là RA

    String ketQua = "";
    while (millis() - thoiGianBatDauCho < 5000) {
      if (Serial.available() > 0) {
        ketQua = Serial.readStringUntil('\n');
        ketQua.trim();

        // BỘ LỌC MỚI: Sử dụng startsWith để chấp nhận các lệnh có kèm số (VD: IN:99)
        if (ketQua.startsWith("IN") || ketQua.startsWith("OUT") || ketQua.startsWith("ERR_") || ketQua == "FAIL") {
          if (ketQua.startsWith("IN")) {
            isApproved = true;
            direction = 1;
          } else if (ketQua.startsWith("OUT")) {
            isApproved = true;
            direction = 2;
          }
          isTimeout = false;
          break;
        }
      }
    }

    if (isApproved == true) {
      if (direction == 1)  // LOGIC CHO XE VÀO
      {
        if (sochotrong > 0) {
          sochotrong--;
          thucHienMoCong(fingerID, 1);  // Gọi hàm với cờ số 1 (VÀO)
        } else {
          // Bãi xe đã đầy, từ chối cho vào dù Server cho phép
          lcd.clear();
          lcd.setCursor(0, 0);
          lcd.print(" BAI XE FULL ");
          lcd.setCursor(0, 1);
          lcd.print(" Khong the vao! ");
          delay(2000);
          hienThiManHinhCho();
        }
      } else if (direction == 2)  // LOGIC CHO XE RA
      {
        if (sochotrong < tongsocho) {
          sochotrong++;  // Trả lại 1 chỗ trống
        }
        thucHienMoCong(fingerID, 2);  // Gọi hàm với cờ số 2 (RA)
      }
    } else {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print(" TU CHOI VAO");
      lcd.setCursor(0, 1);

      if (isTimeout) {
        lcd.print("Loi: Mat ket noi");
      } else if (ketQua == "ERR_READ") {
        lcd.print("Khong Thay Bien!");
      } else if (ketQua == "ERR_MONEY") {
        lcd.print("Loi: Het Tien!  ");
      } else if (ketQua == "ERR_PLATE") {
        lcd.print("Loi: Sai Bien So");
      } else if (ketQua == "ERR_USER") {
        lcd.print("ID Chua Dang Ky ");
      } else {
        lcd.print("Loi He Thong!   ");
      }

      delay(3000);
      hienThiManHinhCho();
    }
  }

  if (inGateOpen == true) {
    if (millis() - inOpenTime >= thoigiancho) {
      thucHienDongCong(1); // Gọi hàm đóng cổng IN
    }
  }

  // Kiểm tra đóng cổng RA
  if (outGateOpen == true) {
    if (millis() - outOpenTime >= thoigiancho) {
      thucHienDongCong(2); // Gọi hàm đóng cổng OUT
    }
  }
  // --- Lắng nghe lệnh gán ID TỰ ĐỘNG CÓ KÈM TÊN TỪ WEB ---
  if (Serial.available() > 0) {
    String command = Serial.readStringUntil('\n');
    command.trim();

    if (command.startsWith("ENROLL_NEW:")) {
      String studentName = command.substring(11);  // Cắt lấy cái tên phía sau

      // 1. Máy tự động tìm ô nhớ trống tiếp theo trong cảm biến
      int newID = getNextFreeID();

      if (newID == -1) {
        lcd.clear();
        lcd.print("BO NHO DAY!");
        delay(2000);
      } else {
        // 2. In tên người cần gán ra màn hình LCD
        lcd.clear();
        lcd.print("Dang ky cho:");
        lcd.setCursor(0, 1);
        if (studentName.length() > 16) studentName = studentName.substring(0, 16);
        lcd.print(studentName);
        delay(2500);  // Dừng 2.5s để sinh viên đọc tên mình

        // 3. Tiến hành quét tay cho ID mới này
        enrollAdminMode(newID);
      }
      hienThiManHinhCho();
    }
    // === THÊM ĐOẠN NÀY ĐỂ BẮT LỆNH MỞ CỔNG KHẨN CẤP TỪ WEB ===
    else if (command.startsWith("OUT:")) {
      if (sochotrong < tongsocho) sochotrong++; // Cộng lại 1 chỗ trống
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print(" XUAT KHAN CAP! ");
      lcd.setCursor(0, 1);
      lcd.print("   TAM BIET!    ");
      
      // Mở Servo cổng RA an toàn
      servoOut.attach(pinOut, 500, 2400);
      servoOut.write(180); // vị trí cũ
      delay(15);
      servoOut.write(90);  // Mở cổng
      
      outOpenTime = millis();
      outGateOpen = true; // Cờ này giúp vòng loop tự động đóng cổng sau 5s
    }
  }
}  // KẾT THÚC HÀM LOOP

void thucHienMoCong(int id, int type) {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("XIN CHAO: ID #");
  lcd.print(id);
  lcd.setCursor(0, 1);

  if (type == 1) { // Mở cổng IN
    lcd.print("   MOI VAO!   ");
    servoIn.attach(pinIn, 500, 2400); // 1. Gọi đủ tham số
    servoIn.write(0);                 // 2. Ép phát xung tại vị trí đang đóng (0 độ)
    delay(15);                        // 3. Chờ 15ms cho xung mồi ổn định, triệt tiêu giật
    servoIn.write(90);                // 4. Mới thực hiện mở 90 độ
    
    inOpenTime = millis();
    inGateOpen = true;
  } else {        // Mở cổng OUT
    lcd.print("   TAM BIET!  ");
    servoOut.attach(pinOut, 500, 2400); // 1. Gọi đủ tham số
    servoOut.write(180);                // 2. Ép phát xung tại vị trí đang đóng (180 độ)
    delay(15);                          // 3. Chờ 15ms cho xung mồi ổn định
    servoOut.write(90);                 // 4. Mới thực hiện mở 90 độ
    
    outOpenTime = millis();
    outGateOpen = true;
  }
}

void thucHienDongCong(int type) {
  if (type == 1) { // Đóng cổng IN
    servoIn.write(0);
    delay(500);
    servoIn.detach();
    inGateOpen = false;
  } else {        // Đóng cổng OUT
    servoOut.write(180);
    delay(500);
    servoOut.detach();
    outGateOpen = false;
  }
  hienThiManHinhCho();
}

void hienThiManHinhCho() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("  MOI QUET TAY  ");
  lcd.setCursor(0, 1);
  lcd.print("Trong: ");
  lcd.print(sochotrong);
  lcd.print("/");
  lcd.print(tongsocho);
}

int getFingerprintID() {
  uint8_t p = finger.getImage();
  if (p == FINGERPRINT_NOFINGER) return 0;
  if (p != FINGERPRINT_OK) return -1;

  p = finger.image2Tz();
  if (p != FINGERPRINT_OK) return -1;

  p = finger.fingerFastSearch();
  if (p != FINGERPRINT_OK) return -2;

  return finger.fingerID;
}

int getNextFreeID() {
  for (int id = 1; id < 127; id++) {
    uint8_t p = finger.loadModel(id);
    if (p != FINGERPRINT_OK) {
      return id;  // Trả về ID đầu tiên bị trống
    }
  }
  return -1;  // Trả về -1 nếu bộ nhớ đã chứa đủ 127 vân tay
}

// -----------------------------------------------------------------
// Hàm 2: Quy trình Đăng ký Vân tay (ĐÃ THÊM CHỐNG TRÙNG LẶP VÂN TAY)
// -----------------------------------------------------------------
void enrollAdminMode(int id_from_web) {
  int id = id_from_web;
  if (id == -1) {
    id = getNextFreeID();
  }
  if (id == -1) {
    lcd.clear();
    lcd.print("BO NHO DAY!");
    delay(2000);
    return;
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("DANG KY: ID " + String(id));
  lcd.setCursor(0, 1);
  lcd.print("Moi dat tay...");

  unsigned long startTime = millis();
  uint8_t p = -1;

  // ==========================================
  // LẦN 1: Chờ đặt tay
  // ==========================================
  while (p != FINGERPRINT_OK) {
    p = finger.getImage();
    if (millis() - startTime > 10000) {
      lcd.clear();
      lcd.print(" HET GIO!");
      delay(2000);
      return;
    }
  }

  p = finger.image2Tz(1);
  if (p != FINGERPRINT_OK) {
    lcd.clear();
    lcd.print("Loi doc tay!");
    delay(2000);
    return;
  }

  // ------------------------------------------------------------------
  // TÍNH NĂNG MỚI: KIỂM TRA TRÙNG LẶP VÂN TAY NGAY SAU LẦN ĐẶT ĐẦU TIÊN
  // ------------------------------------------------------------------
  p = finger.fingerFastSearch();
  if (p == FINGERPRINT_OK) {
    // Nếu tìm thấy vân tay này đã có trong bộ nhớ
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print(" DA DANG KY ROI ");
    lcd.setCursor(0, 1);
    lcd.print("Trung ID so: ");
    lcd.print(finger.fingerID);
    delay(3000);
    return;  // Đuổi ra ngoài, hủy bỏ luôn quá trình đăng ký
  }
  // ------------------------------------------------------------------

  lcd.clear();
  lcd.print("Nho tay ra...");
  delay(2000);

  // Chờ người dùng rút tay ra
  while (p != FINGERPRINT_NOFINGER) {
    p = finger.getImage();
  }

  lcd.clear();
  lcd.print("Dat lai lan 2");
  startTime = millis();  // Reset đồng hồ bấm giờ
  p = -1;

  // ==========================================
  // LẦN 2: Chờ đặt lại
  // ==========================================
  while (p != FINGERPRINT_OK) {

    p = finger.getImage();
    if (millis() - startTime > 10000) {
      lcd.clear();
      lcd.print("HET GIO!");
      delay(2000);
      return;
    }
  }
  p = finger.image2Tz(2);
  if (p != FINGERPRINT_OK) {
    lcd.clear();
    lcd.print("Loi doc tay!");
    delay(2000);
    return;
  }

  // GỘP 2 MẪU VÀ LƯU VÀO BỘ NHỚ
  p = finger.createModel();
  if (p == FINGERPRINT_OK) {
    p = finger.storeModel(id);
    if (p == FINGERPRINT_OK) {
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("LUU THANH CONG");
      lcd.setCursor(0, 1);
      lcd.print("Van tay ID: " + String(id));
      Serial.println("ENROLL_OK:" + String(id));
      delay(3000);
      return;
    }    
  }
  lcd.clear();
  lcd.print("LUU THAT BAI");
  delay(2000);
}