#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* ================= GLOBALS ================= */
I2C_HandleTypeDef hi2c1;      // I2C handle for LCD
uint8_t lcd_addr = 0;          // I2C address of the LCD
