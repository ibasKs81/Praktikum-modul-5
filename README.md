# Praktikum Mikrokontroler - Modul I: Real-Time Operating System: Multitasking dan Komunikasi Task


**Nama:** Ibnu Abbas  
**NIM:** H1H024038  
**Mata Kuliah:** TK244005 - Praktikum Mikrokontroler  
**Program Studi:** Informatika, Universitas Jenderal Soedirman  

---

## 📌 Deskripsi Repositori
Repositori ini berisi source code, skematik rangkaian, dan dokumentasi hasil praktikum Modul 3 yang berfokus pada implementasi protokol komunikasi serial menggunakan antarmuka UART (Universal Asynchronous Receiver-Transmitter) dan I2C (Inter-Integrated Circuit) pada platform mikrokontroler Arduino Uno.

Praktikum ini dirancang untuk memahami bagaimana mikrokontroler dapat menerima komando interaktif dari antarmuka komputer melalui Serial Monitor (UART), serta bagaimana membaca nilai sensor analog (potensiometer) dan mentransmisikan datanya secara sinkron ke dalam modul display eksternal seperti LCD I2C.

---
## 🔬 Analisis Percobaan 1 (ADC)
### 1. Apakah ketiga task berjalan secara bersamaan atau bergantian? Jelaskan mekanismenya!
Seluruh tugas operasional tersebut dieksekusi secara bergantian dan tidak benar-benar simultan pada satu waktu yang sama. Kecepatan transisi antar tugas yang sangat tinggi menciptakan impresi seolah-olah proses berlangsung secara serentak (concurrent). Fenomena ini dikelola secara sistematis oleh FreeRTOS Scheduler melalui algoritma preemptive scheduling yang mengacu pada tingkat urgensi tertentu. Mengingat elemen TaskBlink1, TaskBlink2, serta Taskprint dikonfigurasi dengan level prioritas yang identik, maka unit pemrosesan pusat akan didistribusikan menggunakan metode round-robin. Saat sebuah instruksi memicu fungsi vTaskDelay(), statusnya segera beralih menjadi blocked, sehingga sumber daya komputasi dapat dialokasikan pada tugas lain yang telah siap. Hal ini memungkinkan pemanfaatan CPU yang efisien selama masa tunggu, yang pada akhirnya mewujudkan fungsionalitas multitasking dalam sistem.
### 2. Bagaimana cara menambahkan task keempat? Jelaskan langkahnya!
Prosedur penambahan task keempat diawali dengan deklarasi prototipe fungsi pada bagian hulu program, seperti void TaskKeempat(void *pvParameters);. Implementasi dilanjutkan pada fungsi setup() dengan menginisialisasi xTaskCreate() yang mencakup konfigurasi nama, alokasi stack, serta tingkat prioritas. Definisi fungsi kemudian disusun menggunakan struktur iterasi while(1) yang mengintegrasikan perintah vTaskDelay() demi efisiensi penggunaan sumber daya CPU. Hal ini selaras dengan prinsip manajemen memori dan penjadwalan sistem mikrokontroler agar operasional antar komponen tetap stabil dan terkoordinasi secara optimal.
### 3.   Modifikasilah program dengan menambah sensor (misalnya potensiometer), lalu gunakan nilainya untuk mengontrol kecepatan LED! Bagaimana hasilnya?
```/*
 * Modul 5A — Multitasking FreeRTOS dengan Kontrol Kecepatan LED via Potensiometer
 *
 * Deskripsi:
 *   Program ini menjalankan 4 task secara concurrent menggunakan FreeRTOS.
 *   Kecepatan kedip LED dikontrol secara dinamis oleh nilai potensiometer
 *   yang dibaca melalui task tersendiri dengan prioritas lebih tinggi.
 *   Mutex digunakan untuk melindungi akses ke variabel global blinkSpeed
 *   agar tidak terjadi race condition antar task.
 *
 * Wiring:
 *   Potensiometer  -> A0
 *   LED 1          -> Pin 8
 *   LED 2          -> Pin 7
 */

#include <Arduino_FreeRTOS.h>
#include <semphr.h>

// ── Definisi Pin ───────────────────────────────────────────────
#define POT_PIN   A0   // Pin analog untuk potensiometer
#define LED1_PIN   8   // Pin digital untuk LED 1
#define LED2_PIN   7   // Pin digital untuk LED 2

// ── Handle Mutex dan Variabel Global ──────────────────────────
SemaphoreHandle_t xMutex;             // Mutex untuk proteksi blinkSpeed
volatile int blinkSpeed = 200;        // Nilai default delay LED dalam ms

// ── Prototipe Fungsi Task ──────────────────────────────────────
void TaskReadPot(void *pvParameters); // Task pembaca potensiometer
void TaskBlink1(void *pvParameters);  // Task kedip LED 1
void TaskBlink2(void *pvParameters);  // Task kedip LED 2
void Taskprint(void *pvParameters);   // Task cetak counter ke Serial

// ──────────────────────────────────────────────────────────────
void setup() {
  Serial.begin(9600);
  pinMode(POT_PIN, INPUT);

  // Buat mutex sebelum task dibuat agar tersedia saat task pertama kali berjalan
  xMutex = xSemaphoreCreateMutex();

  // Buat semua task; TaskReadPot diberi prioritas 2 agar pembacaan sensor
  // selalu diperbarui sebelum task LED menggunakannya
  xTaskCreate(TaskReadPot, "ReadPot", 128, NULL, 2, NULL);
  xTaskCreate(TaskBlink1,  "task1",   128, NULL, 1, NULL);
  xTaskCreate(TaskBlink2,  "task2",   128, NULL, 1, NULL);
  xTaskCreate(Taskprint,   "task3",   128, NULL, 1, NULL);

  // Serahkan kontrol ke FreeRTOS Scheduler; fungsi ini tidak pernah return
  vTaskStartScheduler();
}

// loop() dibiarkan kosong karena FreeRTOS Scheduler yang mengelola eksekusi
void loop() {}

// ──────────────────────────────────────────────────────────────
// Task 1 — TaskReadPot
// Membaca nilai potensiometer setiap 100 ms dan memetakan hasilnya
// ke rentang delay 50–1000 ms, lalu menyimpannya ke blinkSpeed.
// Prioritas lebih tinggi (2) memastikan nilai selalu diperbarui tepat waktu.
// ──────────────────────────────────────────────────────────────
void TaskReadPot(void *pvParameters) {
  while (1) {
    int potValue = analogRead(POT_PIN);                   // Baca ADC 0–1023
    int newSpeed = map(potValue, 0, 1023, 50, 1000);      // Petakan ke 50–1000 ms

    // Ambil mutex sebelum menulis ke variabel global
    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      blinkSpeed = newSpeed;   // Perbarui kecepatan kedip
      xSemaphoreGive(xMutex);  // Lepas mutex agar task lain bisa mengakses
    }

    vTaskDelay(100 / portTICK_PERIOD_MS); // Tunggu 100 ms sebelum baca ulang
  }
}

// ──────────────────────────────────────────────────────────────
// Task 2 — TaskBlink1
// Mengedipkan LED 1 dengan delay yang diambil dari blinkSpeed.
// Kecepatan kedip berubah sesuai posisi potensiometer.
// ──────────────────────────────────────────────────────────────
void TaskBlink1(void *pvParameters) {
  pinMode(LED1_PIN, OUTPUT);
  while (1) {
    int speed;

    // Baca blinkSpeed secara aman menggunakan mutex
    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      speed = blinkSpeed;
      xSemaphoreGive(xMutex);
    }

    Serial.println("Task1 - LED1 Blink");
    digitalWrite(LED1_PIN, HIGH);
    vTaskDelay(speed / portTICK_PERIOD_MS); // ON selama 'speed' ms
    digitalWrite(LED1_PIN, LOW);
    vTaskDelay(speed / portTICK_PERIOD_MS); // OFF selama 'speed' ms
  }
}

// ──────────────────────────────────────────────────────────────
// Task 3 — TaskBlink2
// Mengedipkan LED 2 dengan delay 1.5x lebih lambat dari LED 1,
// sehingga kedua LED terlihat berbeda ritmenya.
// ──────────────────────────────────────────────────────────────
void TaskBlink2(void *pvParameters) {
  pinMode(LED2_PIN, OUTPUT);
  while (1) {
    int speed;

    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      speed = blinkSpeed;
      xSemaphoreGive(xMutex);
    }

    Serial.println("Task2 - LED2 Blink");
    digitalWrite(LED2_PIN, HIGH);
    vTaskDelay((speed * 1.5) / portTICK_PERIOD_MS); // ON 1.5x lebih lambat
    digitalWrite(LED2_PIN, LOW);
    vTaskDelay((speed * 1.5) / portTICK_PERIOD_MS); // OFF 1.5x lebih lambat
  }
}

// ──────────────────────────────────────────────────────────────
// Task 4 — Taskprint
// Mencetak nilai counter dan kecepatan kedip saat ini ke Serial Monitor
// setiap 500 ms sebagai informasi debug dan monitoring sistem.
// ──────────────────────────────────────────────────────────────
void Taskprint(void *pvParameters) {
  int counter = 0;
  while (1) {
    int speed;

    if (xSemaphoreTake(xMutex, portMAX_DELAY) == pdTRUE) {
      speed = blinkSpeed;
      xSemaphoreGive(xMutex);
    }

    counter++;
    Serial.print("Counter: ");
    Serial.print(counter);
    Serial.print(" | Blink Speed: ");
    Serial.print(speed);
    Serial.println(" ms");

    vTaskDelay(500 / portTICK_PERIOD_MS); // Cetak setiap 500 ms
  }
}
```

---
## Analisa Percobaan 2
### 1. Apakah kedua task berjalan secara bersamaan atau bergantian? Jelaskan mekanismenya!
Implementasi sistem ini melibatkan dua unit tugas utama, yakni task read_data dan display, yang pengoperasiannya diatur secara bergantian oleh penjadwal FreeRTOS. Proses pertukaran informasi dilakukan melalui mekanisme xQueueCreate sebagai media penampung sementara berbasis FIFO. Dalam alurnya, task read_data mengirimkan entri data menggunakan instruksi xQueueSend() sebelum akhirnya memasuki fase penundaan melalui vTaskDelay() yang mengubah statusnya menjadi blocked. Pada kondisi tersebut, sumber daya prosesor dialihkan ke task display untuk menarik data lewat xQueueReceive() dengan durasi tunggu portMAX_DELAY. Setelah data berhasil diperoleh, task terkait akan memvisualisasikan hasilnya pada terminal Serial Monitor. Sinergi ini menunjukkan efektivitas sinkronisasi antar elemen sistem melalui pemanfaatan antrean data.
### 2.  Apakah program ini berpotensi mengalami race condition? Jelaskan!
Implementasi sistem ini tergolong aman dari potensi race condition yang berarti, mengingat pertukaran data antar task telah diatur melalui fitur Queue pada FreeRTOS. Sifat thread-safe dari antrean tersebut memberikan jaminan bahwa eksekusi fungsi xQueueSend() serta xQueueReceive() berlangsung secara atomic tanpa gangguan dari proses lainnya. Hal ini mencegah terjadinya akses data simultan pada sumber daya yang sama tanpa adanya koordinasi. Secara umum, konflik data muncul apabila beberapa unit proses memanipulasi variabel global secara bebas tanpa adanya kendali mutex ataupun semaphore. Oleh karena kode ini mengandalkan queue sebagai jalur tunggal pembagian informasi, maka risiko kegagalan sinkronisasi dapat dieliminasi secara efektif.
### 3.program dengan menggunakan sensor DHT sesungguhnya sehingga informasi yang ditampilkan dinamis.
```/*
 * Modul 5B — Komunikasi Task FreeRTOS dengan Sensor DHT22
 *
 * Deskripsi:
 *   Program ini menerapkan komunikasi antar task menggunakan Queue FreeRTOS.
 *   Task read_data membaca data dari sensor DHT22 secara periodik dan
 *   mengirimkannya ke queue. Task display menerima data dari queue dan
 *   menampilkannya ke Serial Monitor termasuk perhitungan Heat Index.
 *   Pendekatan queue memastikan tidak ada race condition karena FreeRTOS
 *   menjamin thread-safety pada operasi xQueueSend dan xQueueReceive.
 *
 * Wiring:
 *   DHT22 DATA -> Pin 2
 *   DHT22 VCC  -> 5V
 *   DHT22 GND  -> GND
 */

#include <Arduino_FreeRTOS.h>
#include <queue.h>
#include <DHT.h>

// ── Konfigurasi Sensor DHT ─────────────────────────────────────
#define DHTPIN   2        // Pin data sensor DHT22
#define DHTTYPE  DHT22    // Tipe sensor: DHT22 (AM2302)

DHT dht(DHTPIN, DHTTYPE); // Inisialisasi objek sensor DHT

// ── Struktur Data untuk Queue ──────────────────────────────────
// Menyimpan satu set pembacaan sensor: suhu, kelembaban, dan status validitas
struct readings {
  float temp;   // Suhu dalam derajat Celsius
  float h;      // Kelembaban relatif dalam persen
  bool  valid;  // true jika pembacaan berhasil, false jika sensor error
};

// ── Handle Queue ───────────────────────────────────────────────
QueueHandle_t my_queue; // Queue dengan kapasitas 1 slot (size = 1)

// ── Prototipe Fungsi Task ──────────────────────────────────────
void read_data(void *pvParameters); // Task pembaca sensor DHT22
void display(void *pvParameters);   // Task penampil data ke Serial Monitor

// ──────────────────────────────────────────────────────────────
void setup() {
  Serial.begin(9600);
  dht.begin(); // Inisialisasi sensor DHT22

  // Buat queue dengan kapasitas 1 elemen bertipe struct readings
  // Kapasitas 1 cukup karena task display selalu mengonsumsi sebelum data baru datang
  my_queue = xQueueCreate(1, sizeof(struct readings));

  // Buat task dengan prioritas sama; scheduler akan mengatur eksekusi round-robin
  xTaskCreate(read_data, "read sensors", 128, NULL, 1, NULL);
  xTaskCreate(display,   "display",      128, NULL, 1, NULL);

  // Serahkan kontrol ke FreeRTOS Scheduler
  vTaskStartScheduler();
}

void loop() {} // Dikosongkan karena FreeRTOS yang mengelola eksekusi

// ──────────────────────────────────────────────────────────────
// Task 1 — read_data
// Membaca suhu dan kelembaban dari sensor DHT22 setiap 2000 ms,
// mengemas hasilnya ke dalam struct readings, lalu mengirim ke queue.
// Flag valid digunakan untuk memberi tahu task display apakah data dapat dipercaya.
// ──────────────────────────────────────────────────────────────
void read_data(void *pvParameters) {
  struct readings data;

  for (;;) {
    // Baca data dari sensor DHT22
    data.h    = dht.readHumidity();    // Kelembaban relatif (%)
    data.temp = dht.readTemperature(); // Suhu dalam Celsius

    // Validasi: isnan() mendeteksi kegagalan pembacaan sensor
    data.valid = !(isnan(data.h) || isnan(data.temp));

    // Kirim data ke queue; portMAX_DELAY artinya tunggu selamanya jika queue penuh
    // Dalam kasus ini queue berkapasitas 1 sehingga harus menunggu display mengonsumsi dulu
    xQueueSend(my_queue, &data, portMAX_DELAY);

    vTaskDelay(2000 / portTICK_PERIOD_MS); // Baca sensor setiap 2 detik
  }
}

// ──────────────────────────────────────────────────────────────
// Task 2 — display
// Menunggu data dari queue menggunakan portMAX_DELAY (blocked sampai ada data).
// Setelah data diterima, menampilkan suhu, kelembaban, dan Heat Index ke Serial Monitor.
// Jika pembacaan tidak valid, menampilkan pesan error.
// ──────────────────────────────────────────────────────────────
void display(void *pvParameters) {
  struct readings data;

  for (;;) {
    // Tunggu data dari queue; task ini blocked sampai read_data mengirim data
    if (xQueueReceive(my_queue, &data, portMAX_DELAY) == pdPASS) {

      if (data.valid) {
        // Tampilkan suhu
        Serial.print(F("Suhu      : "));
        Serial.print(data.temp, 1);   // 1 angka desimal
        Serial.println(F(" C"));

        // Tampilkan kelembaban
        Serial.print(F("Kelembaban: "));
        Serial.print(data.h, 1);
        Serial.println(F(" %"));

        // Hitung dan tampilkan Heat Index (indeks kenyamanan termal)
        // Parameter false = hasil dalam Celsius
        float hi = dht.computeHeatIndex(data.temp, data.h, false);
        Serial.print(F("Heat Index: "));
        Serial.print(hi, 1);
        Serial.println(F(" C"));

        Serial.println(F("--------------------")); // Pemisah tiap set data
      } else {
        // Tampilkan pesan error jika sensor gagal dibaca
        Serial.println(F("ERROR: Gagal membaca sensor DHT!"));
        Serial.println(F("Periksa kabel dan koneksi sensor."));
        Serial.println(F("--------------------"));
      }
    }
  }
}
```





---

## 📸 Dokumentasi Praktikum
### 1. Dokumentasi Percobaan
Percobaan 1:
![Lampiran percobaan 1 Praktikum](587878913-d09394eb-b248-451f-9853-12f4221aa1a4.jpg)
Percobaan 2: 
![Lampiran percobaan 2 Praktikum](587879293-8a848cbe-2a8f-4c07-9303-132b897876d6.jpg)


