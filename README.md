#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* ================= GLOBALS ================= */
I2C_HandleTypeDef hi2c1;      // I2C handle for LCD
uint8_t lcd_addr = 0;          // I2C address of the LCD

/* ================= LCD FUNCTIONS ================= */
void LCD_Send(uint8_t val, uint8_t rs) {
    if (!lcd_addr) return; // If LCD not found, do nothing

    uint8_t high = (val & 0xF0) | rs | 0x08;
    uint8_t low  = ((val << 4) & 0xF0) | rs | 0x08;
    uint8_t data[4] = { high | 0x04, high, low | 0x04, low };

    HAL_I2C_Master_Transmit(&hi2c1, lcd_addr, data, 4, 100);
    HAL_Delay(2); // Small delay for LCD to process
}

