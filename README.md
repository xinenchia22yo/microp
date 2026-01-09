#include "stm32f4xx_hal.h"
#include <stdio.h>

/* ================= GLOBALS ================= */

I2C_HandleTypeDef hi2c1;
uint8_t lcd_addr = 0;

uint8_t lcd_pos = 0;

/* ================= LCD FUNCTIONS ================= */

void LCD_Send(uint8_t val, uint8_t rs)
{
    if (lcd_addr == 0) return;

    uint8_t high = (val & 0xF0) | rs | 0x08;
    uint8_t low  = ((val << 4) & 0xF0) | rs | 0x08;

    uint8_t data[4] = {
        high | 0x04,
        high,
        low | 0x04,
        low
    };

    HAL_I2C_Master_Transmit(&hi2c1, lcd_addr, data, 4, 100);
    HAL_Delay(2);
}

void LCD_Init(void)
{
    HAL_Delay(100);
    LCD_Send(0x33, 0);
    LCD_Send(0x32, 0);
    LCD_Send(0x28, 0);   // 4-bit, 2-line
    LCD_Send(0x0C, 0);   // Display ON
    LCD_Send(0x01, 0);   // Clear
    HAL_Delay(5);
}

void LCD_Clear(void)
{
    LCD_Send(0x01, 0);
    HAL_Delay(2);
    LCD_Send(0x80, 0);
    lcd_pos = 0;
}

/* ================= KEYPAD ================= */

char Keypad_Scan(void)
{
    char keys[4][4] = {
        {'D','#','0','*'},
        {'C','9','8','7'},
        {'B','6','5','4'},
        {'A','3','2','1'}
    };

    for (int i = 0; i < 4; i++) {
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_RESET);

        for (int j = 0; j < 4; j++) {
            if (HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_0 << j)) == GPIO_PIN_RESET) {
                HAL_Delay(20); // debounce
                HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
                return keys[i][j];
            }
        }
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
    }
    return 0;
}

/* ================= MAIN ================= */

int main(void)
{
    HAL_Init();

    /* ---------- CLOCK ENABLE ---------- */
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct;

    /* ---------- KEYPAD INPUTS PA0–PA3 ---------- */
    GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    GPIO_InitStruct.Alternate = 0;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* ---------- KEYPAD OUTPUTS PA4–PA7 ---------- */
    GPIO_InitStruct.Pin = GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    GPIO_InitStruct.Alternate = 0;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* ---------- LEDs PB0–PB2 ---------- */
    GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    GPIO_InitStruct.Alternate = 0;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    /* ---------- I2C PINS PB8 PB9 ---------- */
    GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
    GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    /* ---------- I2C INIT (COMPLETE) ---------- */
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 100000;
    hi2c1.Init.DutyCycle = I2C_DUTYCYCLE_2;
    hi2c1.Init.OwnAddress1 = 0;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
    HAL_I2C_Init(&hi2c1);

    /* ---------- LCD ADDRESS SCAN ---------- */
    uint8_t addrs[] = {0x27 << 1, 0x3F << 1};
    for (int i = 0; i < 2; i++) {
        if (HAL_I2C_IsDeviceReady(&hi2c1, addrs[i], 3, 100) == HAL_OK) {
            lcd_addr = addrs[i];
            break;
        }
    }

    /* ---------- LCD START ---------- */
    LCD_Init();
    LCD_Clear();
    LCD_Send('>', 1);

    /* ================= LOOP ================= */

    while (1)
    {
        /* Heartbeat LED */
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
        HAL_Delay(300);

        char key = Keypad_Scan();

        if (key)
        {
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET);

            LCD_Send(0x80 + lcd_pos + 1, 0);
            LCD_Send(key, 1);
            lcd_pos++;

            HAL_Delay(200);
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_RESET);
        }
    }
}

/* ================= SYSTICK ================= */

void SysTick_Handler(void)
{
    HAL_IncTick();
}
