#include "stm32f4xx_hal.h"
#include <stdio.h>

// --- Global Variables ---
I2C_HandleTypeDef hi2c1;
uint8_t lcd_addr = 0; 
#define MAX_ASGN 5

typedef enum { PROGRESS, WARNING, OVERDUE, SUBMITTED } Status;

typedef struct {
    int id;
    int days_left;
    Status status;
} Assignment;

Assignment list[MAX_ASGN];
int current_view = 0;
int total_asgn = 0;

// --- 1. LCD SUBROUTINE (Fixed for Stability) ---
void LCD_Send(uint8_t val, uint8_t rs) {
    if (lcd_addr == 0) return;
    uint8_t high = (val & 0xf0) | rs | 0x08;
    uint8_t low = ((val << 4) & 0xf0) | rs | 0x08;
    uint8_t data[4] = {high|0x04, high, low|0x04, low};
    HAL_I2C_Master_Transmit(&hi2c1, lcd_addr, data, 4, 100);
    HAL_Delay(2); 
}

void LCD_Init(void) {
    HAL_Delay(100);
    LCD_Send(0x33, 0); HAL_Delay(5);
    LCD_Send(0x32, 0); HAL_Delay(5);
    LCD_Send(0x28, 0); // 2 lines, 5x8 font
    LCD_Send(0x0C, 0); // Display ON, Cursor OFF
    LCD_Send(0x01, 0); // Clear Screen
    HAL_Delay(5);
}

void LCD_Print(char *str) {
    while(*str) LCD_Send(*str++, 1);
}

// --- 2. KEYPAD SUBROUTINE (Moved here to fix Undefined Symbol error) ---
char Keypad_Scan(void) {
    char keys[4][4] = {{'D','#','0','*'},{'C','9','8','7'},{'B','6','5','4'},{'A','3','2','1'}};
    for (int i = 0; i < 4; i++) {
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_RESET);
        for (int j = 0; j < 4; j++) {
            if (HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_0 << j)) == GPIO_PIN_RESET) {
                HAL_Delay(50); // Debounce
                while(HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_0 << j)) == GPIO_PIN_RESET);
                HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
                return keys[i][j];
            }
        }
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
    }
    return 0;
}

// --- 3. MAIN FUNCTION ---
int main(void) {
    HAL_Init();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // Keypad Inputs (PA0-PA3) and Buttons (PA10-PA11)
    GPIO_InitStruct.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2|GPIO_PIN_3|GPIO_PIN_10|GPIO_PIN_11;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // Keypad Outputs (PA4-PA7)
    GPIO_InitStruct.Pin = GPIO_PIN_4|GPIO_PIN_5|GPIO_PIN_6|GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // LEDs (PB0, PB1, PB2)
    GPIO_InitStruct.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // I2C Pins (PB8, PB9)
    GPIO_InitStruct.Pin = GPIO_PIN_8|GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // I2C Peripheral Initialization (Very slow 10kHz for max compatibility)
    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 10000; 
    HAL_I2C_Init(&hi2c1);

    // Scan for LCD address
    uint8_t addrs[] = {0x27 << 1, 0x3F << 1};
    for(int i=0; i<2; i++) {
        if(HAL_I2C_IsDeviceReady(&hi2c1, addrs[i], 3, 100) == HAL_OK) {
            lcd_addr = addrs[i];
            break;
        }
    }

    LCD_Init();
    LCD_Print("READY GROUP 5");
    HAL_Delay(1000);

    while (1) {
        // Fast Heartbeat to show STM32 is alive
        HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_0);
        HAL_Delay(100);

        char key = Keypad_Scan();
        if (key != 0) {
            // Light Red LED when key is pressed
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET);
            
            if (key >= '1' && key <= '9' && total_asgn < MAX_ASGN) {
                list[total_asgn].id = total_asgn + 1;
                list[total_asgn].days_left = key - '0';
                list[total_asgn].status = (list[total_asgn].days_left <= 3) ? WARNING : PROGRESS;
                total_asgn++;
                
                LCD_Send(0x01, 0); // Clear
                LCD_Print("SAVING...");
                HAL_Delay(500);
            }
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_RESET);
        }

        // Button Logic: Scroll (PA10)
        if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_10) == GPIO_PIN_RESET) {
            HAL_Delay(200);
            if (total_asgn > 0) current_view = (current_view + 1) % total_asgn;
            LCD_Send(0x01, 0); 
        }

        // Display update
        if (total_asgn > 0) {
            char b1[16];
            sprintf(b1, "ID:%d Days:%d", list[current_view].id, list[current_view].days_left);
            LCD_Send(0x80, 0); 
            LCD_Print(b1);
        }
    }
}

// Required for HAL_Delay
void SysTick_Handler(void) { HAL_IncTick(); }
