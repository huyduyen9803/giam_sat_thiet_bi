#include "main.h"
#include "i2c.h"  // Dùng cho BH1750
#include "usart.h" // Debug qua UART
#include "gpio.h"  // Điều khiển thiết bị

#define LIGHT_THRESHOLD  300  // Ngưỡng ánh sáng (lux)
#define TEMP_THRESHOLD   30   // Ngưỡng nhiệt độ (°C)

// Biến toàn cục
uint8_t dht_data[5];  // Dữ liệu từ DHT11
uint16_t lux;         // Giá trị ánh sáng từ BH1750
uint8_t motion_detected; // Trạng thái từ PIR

// Hàm đọc BH1750
uint16_t BH1750_ReadLight() {
    uint8_t data[2];
    HAL_I2C_Master_Receive(&hi2c1, 0x23 << 1, data, 2, 100);
    return ((data[0] << 8) | data[1]) / 1.2;
}

// Hàm đọc DHT11
uint8_t DHT11_ReadData(uint8_t *temperature) {
    // Gửi tín hiệu khởi động
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_5, GPIO_PIN_RESET);
    HAL_Delay(18);
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_5, GPIO_PIN_SET);
    HAL_Delay(40);
    
    // Đọc dữ liệu (giả lập đơn giản, cần tối ưu với delay chính xác)
    for (int i = 0; i < 5; i++) {
        dht_data[i] = 0; // Giả lập dữ liệu
    }
    *temperature = dht_data[2];
    return 1;
}

// Main Loop
int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();
    MX_USART2_UART_Init();

    uint8_t temperature;
    
    while (1) {
        lux = BH1750_ReadLight();
        DHT11_ReadData(&temperature);
        motion_detected = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1); // PIR input
        
        // Điều khiển đèn theo ánh sáng và chuyển động
        if (lux < LIGHT_THRESHOLD && motion_detected) {
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);  // Bật đèn
        } else {
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET); // Tắt đèn
        }
        
        // Điều khiển quạt theo nhiệt độ
        if (temperature > TEMP_THRESHOLD) {
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET);  // Bật quạt
        } else {
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); // Tắt quạt
        }
        
        HAL_Delay(1000);
    }
}
