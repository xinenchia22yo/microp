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
void LCD_Init(void) {
    HAL_Delay(100); // Wait for LCD to power up
    LCD_Send(0x33, 0); LCD_Send(0x32, 0); // Initialize LCD in 4-bit mode
    LCD_Send(0x28, 0); // Function Set: 2 lines, 5x8 pixel font matrix
    LCD_Send(0x0C, 0); // Display ON, cursor OFF
    LCD_Send(0x01, 0); // Clear Display command
    HAL_Delay(5);       //Clear display takes longer, so wait 5ms
}

void LCD_Print(char *str) {
    while(*str) LCD_Send(*str++, 1);
}

void LCD_Clear(void) {
    LCD_Send(0x01, 0); // Clear LCD command
    HAL_Delay(2);
}

