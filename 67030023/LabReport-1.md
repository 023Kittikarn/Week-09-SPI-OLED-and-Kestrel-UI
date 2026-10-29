# รายงานการทดลองที่ 9.1 (Lab 9.1)
## การประกอบสร้างตัวขับจอแสดงผล SSD1306 ทีละชิ้นส่วน (Deconstructed Bring-up) สู่ Hello World และการตรวจสอบความจำภาพเชิงนิติวิทยาศาสตร์ (Framebuffer Forensics)

---

## 1. ซอร์สโค้ดฉบับสมบูรณ์ (`main/main.c`)

```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/spi_master.h"
#include "esp_log.h"
#include "font5x7.h"

static const char *TAG = "OLED_LAB9_1";

// Macro สำหรับแปลงค่า 1 ไบต์เป็นบิต 0/1 ในการทำ Hex Dump
#define BYTE_TO_BINARY_PATTERN "%c%c%c%c%c%c%c%c"
#define BYTE_TO_BINARY(byte)  \
  ((byte) & 0x80 ? '1' : '0'), \
  ((byte) & 0x40 ? '1' : '0'), \
  ((byte) & 0x20 ? '1' : '0'), \
  ((byte) & 0x10 ? '1' : '0'), \
  ((byte) & 0x08 ? '1' : '0'), \
  ((byte) & 0x04 ? '1' : '0'), \
  ((byte) & 0x02 ? '1' : '0'), \
  ((byte) & 0x01 ? '1' : '0')

// 1. กำหนดขาเชื่อมต่อตามแผนภาพวงจรจริง
#define OLED_PIN_SCK    (GPIO_NUM_18) // D0 (SPI Clock)
#define OLED_PIN_MOSI   (GPIO_NUM_23) // D1 (SPI MOSI Data)
#define OLED_PIN_RES    (GPIO_NUM_4)  // RES (Hardware Reset)
#define OLED_PIN_DC     (GPIO_NUM_2)  // DC (0 = Command, 1 = Data)
#define OLED_PIN_CS     (GPIO_NUM_5)  // CS (Chip Select - Active LOW)

static spi_device_handle_t s_spi_handle = NULL;
static uint8_t s_oled_buffer[1024]; // Framebuffer ขนาด 1KB (128x64 pixels)

// 2. ฟังก์ชันกำหนดค่าเริ่มต้นพิน GPIO และบัส SPI2
esp_err_t oled_spi_init(void)
{
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << OLED_PIN_DC) | (1ULL << OLED_PIN_RES),
        .mode = GPIO_MODE_OUTPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };
    gpio_config(&io_conf);

    spi_bus_config_t buscfg = {
        .miso_io_num = -1,
        .mosi_io_num = OLED_PIN_MOSI,
        .sclk_io_num = OLED_PIN_SCK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 1024 + 16,
    };

    esp_err_t ret = spi_bus_initialize(SPI2_HOST, &buscfg, SPI_DMA_CH_AUTO);
    if (ret != ESP_OK) return ret;

    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = 10 * 1000 * 1000, // 10 MHz
        .mode = 0,                          // Mode 0
        .spics_io_num = OLED_PIN_CS,
        .queue_size = 7,
    };

    return spi_bus_add_device(SPI2_HOST, &devcfg, &s_spi_handle);
}

// 3. ฟังก์ชันส่งคำสั่ง (DC = 0)
void oled_send_cmd(uint8_t cmd)
{
    gpio_set_level(OLED_PIN_DC, 0);
    spi_transaction_t t;
    memset(&t, 0, sizeof(t));
    t.length = 8;
    t.tx_buffer = &cmd;
    spi_device_polling_transmit(s_spi_handle, &t);
}

// 4. ฟังก์ชันส่งข้อมูลพิกเซล (DC = 1)
void oled_send_data(const uint8_t *data, size_t len)
{
    if (len == 0) return;
    gpio_set_level(OLED_PIN_DC, 1);
    spi_transaction_t t;
    memset(&t, 0, sizeof(t));
    t.length = len * 8;
    t.tx_buffer = data;
    spi_device_polling_transmit(s_spi_handle, &t);
}

// 5. ฟังก์ชันล้าง Framebuffer
void oled_clear(void)
{
    memset(s_oled_buffer, 0x00, sizeof(s_oled_buffer));
}

// 6. ฟังก์ชันวาดพิกเซลด้วย Bitwise Manipulation
void oled_draw_pixel(int x, int y, bool color)
{
    if (x < 0 || x >= 128 || y < 0 || y >= 64) return;

    int byte_index = x + (y / 8) * 128;
    int bit_offset = y % 8;

    if (color) {
        s_oled_buffer[byte_index] |= (1 << bit_offset);
    } else {
        s_oled_buffer[byte_index] &= ~(1 << bit_offset);
    }
}

// 7. ฟังก์ชันยิงส่ง Framebuffer สู่ GRAM (Flush)
void oled_flush(void)
{
    oled_send_cmd(0x21); // Set Column Address
    oled_send_cmd(0x00);
    oled_send_cmd(0x7F);

    oled_send_cmd(0x22); // Set Page Address
    oled_send_cmd(0x00);
    oled_send_cmd(0x07);

    oled_send_data(s_oled_buffer, sizeof(s_oled_buffer));
}

// 8. ฟังก์ชันวาดตัวอักษร 1 ตัวจาก Font Matrix 5x7
void oled_draw_char(int x, int y, char c, bool color)
{
    if (c < 32 || c > 126) c = '?';

    int font_idx = c - 32;

    for (int col = 0; col < 5; col++) {
        uint8_t line = font5x7[font_idx][col];
        for (int row = 0; row < 7; row++) {
            if (line & (1 << row)) {
                oled_draw_pixel(x + col, y + row, color);
            } else {
                oled_draw_pixel(x + col, y + row, !color);
            }
        }
    }
    // เว้นช่องไฟระหว่างตัวอักษร 1 พิกเซล
    for (int row = 0; row < 7; row++) {
        oled_draw_pixel(x + 5, y + row, !color);
    }
}

// 9. ฟังก์ชันแสดงข้อความสตริง
void oled_draw_string(int x, int y, const char *str, bool color)
{
    while (*str) {
        oled_draw_char(x, y, *str, color);
        x += 6;
        if (x + 6 > 128) break;
        str++;
    }
}

// --- MAIN FUNCTION ---
void app_main(void)
{
    // ขั้นตอนที่ 1: เริ่มต้นระบบ SPI2
    ESP_ERROR_CHECK(oled_spi_init());

    // ขั้นตอนที่ 2: Hardware Reset
    gpio_set_level(OLED_PIN_RES, 0);
    vTaskDelay(pdMS_TO_TICKS(15));
    gpio_set_level(OLED_PIN_RES, 1);
    vTaskDelay(pdMS_TO_TICKS(15));

    // ขั้นตอนที่ 3: Magic Sequence เปิดวงจร Charge Pump และเปิดจอ
    oled_send_cmd(0xAE); // Display OFF
    oled_send_cmd(0x8D); // Charge Pump Setting
    oled_send_cmd(0x14); // Enable Charge Pump (7.5V)
    oled_send_cmd(0x20); // Set Memory Addressing Mode
    oled_send_cmd(0x00); // Horizontal Addressing Mode
    oled_send_cmd(0xAF); // Display ON

    // ขั้นตอนที่ 4: ทดสอบถมหน้าจอ (สว่างทั้งจอ 1.5 วินาที)
    uint8_t fill_buffer[128];
    memset(fill_buffer, 0xFF, sizeof(fill_buffer));
    oled_send_cmd(0x21); oled_send_cmd(0x00); oled_send_cmd(0x7F);
    oled_send_cmd(0x22); oled_send_cmd(0x00); oled_send_cmd(0x07);
    for (int page = 0; page < 8; page++) {
        oled_send_data(fill_buffer, sizeof(fill_buffer));
    }
    vTaskDelay(pdMS_TO_TICKS(1500));

    // ขั้นตอนที่ 5: ทดสอบจุด 4 มุมจอ (1.5 วินาที)
    oled_clear();
    oled_draw_pixel(0, 0, true);     // มุมบนซ้าย
    oled_draw_pixel(127, 0, true);   // มุมบนขวา
    oled_draw_pixel(0, 63, true);    // มุมล่างซ้าย
    oled_draw_pixel(127, 63, true);  // มุมล่างขวา
    oled_flush();
    vTaskDelay(pdMS_TO_TICKS(1500));

    // ขั้นตอนที่ 6: พิมพ์ข้อความ HELLO WORLD และ รหัสนักศึกษา 67030023
    oled_clear();
    oled_draw_string(30, 4, "HELLO WORLD", true);   // โซนสีเหลือง
    oled_draw_string(24, 32, "ID: 67030023", true);  // โซนสีฟ้า
    oled_flush();

    // กิจกรรมนิติวิทยาศาสตร์ 1.1: Dump ค่า Framebuffer ในบริเวณตัวอักษร 'H'
    ESP_LOGI(TAG, "=== DUMPING FRAMEBUFFER PAGE 0 (Col 30 to 34 for 'H') ===");
    for (int i = 30; i <= 34; i++) {
        printf("Byte[%2d] (Col %2d): 0x%02X  [Binary: " BYTE_TO_BINARY_PATTERN "]\n", 
               i, i, s_oled_buffer[i], BYTE_TO_BINARY(s_oled_buffer[i]));
    }
}
2. การสังเกตการณ์เชิงลึกทางกายภาพ (Forensic Visual Observation)
1.แถบสีของหน้าจอ (Dual-Color Zone): แถบสีเหลืองปรากฏอยู่โซนด้านล่าง และแถบสีฟ้าปรากฏอยู่โซนด้านบน

2.ทิศทางของตัวอักษร (Upside-down Display): ข้อความ "HELLO WORLD" และ "ID: 67030023" แสดงผลกลับหัว 180 องศา

3.สาเหตุทางฮาร์ดแวร์: เกิดจากค่า Default ของชิป SSD1306 คือ 0xA0 (Segment Remap) และ 0xC0 (COM Scan Direction) ร่วมกับการติดตั้งแผงจอ OLED กลับหัวทางกายภาพจากโรงงานผู้ผลิต
3. ขั้นตอนการตรวจสอบเชิงนิติวิทยาศาสตร์ (Framebuffer Forensics)
กิจกรรมนิติวิทยาศาสตร์ 1.1 Hex Dump Memory Inspection
ESP_LOGI("FORENSIC", "=== DUMPING FRAMEBUFFER PAGE 0 (Col 30 to 34 for 'H') ===");
for (int i = 30; i <= 34; i++) {
    printf("Byte[%2d] (Col %2d): 0x%02X  [Binary: " BYTE_TO_BINARY_PATTERN "]\n", 
           i, i, s_oled_buffer[i], BYTE_TO_BINARY(s_oled_buffer[i]));
}
กิจกรรมนิติวิทยาศาสตร์ 1.2 Bit-to-Pixel Forensic Reconstructionตารางบันทึกค่า Binary และการถอดรหัสบิตข้อมูลของตัวอักษร 'H' (คอลัมน์ $x=30$ ถึง $x=34$ ใน Page 0):คอลัมน์ (x)ดัชนีไบต์ในแรมค่า Hexค่า Binaryบิตที่เป็น 1ความหมายทางพิกเซลและรูปร่างตัวอักษรCol 30s_oled_buffer[30]0x7F01111111Bit 0, 1, 2, 3, 4, 5, 6เส้นตรงแนวตั้งด้านซ้ายของตัว 'H' (พิกเซลสว่างแนวตั้ง 7 แถว)Col 31s_oled_buffer[31]0x0800001000Bit 3แถบขีดกลางของตัว 'H' (พิกเซลสว่างเฉพาะจุดกลาง)Col 32s_oled_buffer[32]0x0800001000Bit 3แถบขีดกลางของตัว 'H' (พิกเซลสว่างเฉพาะจุดกลาง)Col 33s_oled_buffer[33]0x0800001000Bit 3แถบขีดกลางของตัว 'H' (พิกเซลสว่างเฉพาะจุดกลาง)Col 34s_oled_buffer[34]0x7F01111111Bit 0, 1, 2, 3, 4, 5, 6
4. ภาพจำลองผลลัพธ์บนหน้าจอจริง (Physical Outputs)
1) สว่างทั้งหน้าจอ (Full Fill Test)
+--------------------------------------------------+
|##################################################| <- [Yellow Zone]
|##################################################|
|==================================================|
|##################################################| <- [Blue Zone]
|##################################################|
|##################################################|
+--------------------------------------------------+
2) จุด 4 มุมจอ (Corner Pixels Test)
+--------------------------------------------------+
|*                                                *| <- (0,0) และ (127,0) [Yellow Zone]
|                                                  |
|==================================================|
|                                                  |
|                                                  |
|*                                                *| <- (0,63) และ (127,63) [Blue Zone]
+--------------------------------------------------+
3) แสดงผลข้อความ (Hello World & ID - สภาพกลับหัว)
+--------------------------------------------------+
|                                                  |
|                  32003076 :DI                    | <- [Yellow Zone - ID: 67030023 กลับหัว]
|==================================================|
|                                                  |
|                   DLROW OLLEH                    | <- [Blue Zone - HELLO WORLD กลับหัว]
|                                                  |
+--------------------------------------------------+
5. คำถามท้ายการทดลองเพื่อการประเมินผล (Review Questions)
1: จากการทำ Hex Dump ในกิจกรรมนิติวิทยาศาสตร์ จงอธิบายว่าทำไมตัวอักษร 'H' จึงใช้ข้อมูลจำนวน 5 ไบต์ และแต่ละไบต์ทำหน้าที่ควบคุมพิกเซลในทิศทางใด?คำตอบ:ฟอนต์ที่ใช้เป็นฟอนต์มาตรฐานขนาด $5 \times 7$ พิกเซล (กว้าง 5 พิกเซล สูง 7 พิกเซล) โดยสถาปัตยกรรมชิป SSD1306 จะจัดเก็บแรม 1 ไบต์ (8 บิต) ใน แนวตั้ง (Vertical Column) ของแต่ละ Page ดังนั้น ความกว้างของตัวอักษรจำนวน 5 พิกเซล จึงต้องใช้ข้อมูลจำนวน 5 คอลัมน์ (รวมเป็น 5 ไบต์) โดยแต่ละไบต์ทำหน้าที่ควบคุมพิกเซล ในแนวตั้ง (จาก Bit 0 ด้านบนสุด ถึง Bit 7 ด้านล่างสุด) ของคอลัมน์นั้นๆ
2: หากเราสลับสายไฟระหว่างขา D0 และ D1 จะเกิดผลอย่างไรกับสัญญาณ SPI และหน้าจอจะติดหรือไม่?คำตอบ:หน้าจอจะ ไม่ติดและดับสนิท เนื่องจากขา D0 คือ SCLK (SPI Serial Clock) ทำหน้าที่ส่งสัญญาณนาฬิกาควบคุมจังหวะ และขา D1 คือ MOSI (Serial Data) ทำหน้าที่ส่งคำสั่งและข้อมูลพิกเซล หากสลับสายกัน สัญญาณ Clock และ Data จะถูกส่งผิดช่องทาง ทำให้ฮาร์ดแวร์ SSD1306 ไม่สามารถรับจังหวะ Clock เพื่อตีความข้อมูลได้ และไม่สามารถรับชุดคำสั่งเปิดวงจรทวีแรงดัน (Charge Pump Initialization) เพื่อปลุกหน้าจอให้ติดได้
3: เหตุใดการแก้ไขพิกัด $(x, y)$ บน s_oled_buffer จึงไม่ทำให้ภาพบนหน้าจอจริงเปลี่ยนทันที จนกว่าจะมีการเรียกคำสั่ง oled_flush()?คำตอบ:เนื่องจาก s_oled_buffer เป็นเพียง Back Buffer ที่อยู่ในหน่วยความจำ RAM ของไมโครคอนโทรลเลอร์ ESP32 ไม่ใช่ GRAM บนชิปคอนโทรลเลอร์ SSD1306 การสั่งวาดพิกเซลด้วย oled_draw_pixel() เป็นเพียงการคำนวณเปลี่ยนบิตใน RAM ของ ESP32 เพื่อความเร็วในการประมวลผล ภาพบนหน้าจอจริงจะเปลี่ยนก็ต่อเมื่อเรียกคำสั่ง oled_flush() เพื่อส่งถ่ายข้อมูลทั้งหมด 1,024 ไบต์จาก RAM ของ ESP32 ผ่านบัส SPI เข้าสู่ GRAM ของชิป SSD1306 พร้อมกันในคราวเดียว