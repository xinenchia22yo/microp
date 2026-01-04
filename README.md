# microp
ASSIGNMENT REMINDER
/* * Project: Smart Student Assignment Reminder and Management System
 * Platform: STM32F103C8T6 (Keil uVision)
 */

#include "main.h"
#include <stdio.h>
#include <string.h>

/* --- Private Constants & Macros --- */
#define MAX_ASSIGNMENTS 5
#define LCD_ADDR (0x27 << 1)  // Address may vary (0x27 or 0x3F)
#define DAY_TICK_THRESHOLD 5  // Simulated day = 5 timer interrupts

/* --- Typedefs --- */
typedef enum {
    STATUS_EMPTY,
    STATUS_ACTIVE,
    STATUS_SUBMITTED,
    STATUS_OVERDUE
} AssignStatus;

typedef struct {
    int id;
    int days_remaining;
    AssignStatus status;
    int overdue_counter;
} Assignment;

/* --- Global Variables --- */
I2C_HandleTypeDef hi2c1;
TIM_HandleTypeDef htim2;

Assignment assignments[MAX_ASSIGNMENTS];
int current_slot = 0;
volatile int timer_count = 0;

/* --- Function Prototypes --- */
void SystemClock_Config(void);
static void MX_GPIO_Init(void);
static void MX_I2C1_Init(void);
static void MX_TIM2_Init(void);

// Logic & UI Subroutines [cite: 20-25]
void LCD_Init(void);
void LCD_SendCommand(uint8_t cmd);
void LCD_SendData(uint8_t data);
void LCD_SendString(char *str);
void LCD_SetCursor(uint8_t row, uint8_t col);
char Keypad_Scan(void);
void Update_Alerts(void);
void Display_Menu(void);

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_I2C1_Init();
    MX_TIM2_Init();

    LCD_Init();
    LCD_SetCursor(0, 2);
    LCD_SendString("SYSTEM READY");
    HAL_Delay(1000);

    // Initialize slots to empty [cite: 5]
    for(int i = 0; i < MAX_ASSIGNMENTS; i++) {
        assignments[i].status = STATUS_EMPTY;
    }

    HAL_TIM_Base_Start_IT(&htim2); // Begin time tracking [cite: 9]

    char key;
    char input_val[3] = {0};
    int input_idx = 0;

    while (1) {
        Display_Menu();
        Update_Alerts();

        key = Keypad_Scan(); // [cite: 23]
        if (key != 0) {
            if (key >= '0' && key <= '9' && input_idx < 2) {
                input_val[input_idx++] = key;
                LCD_SetCursor(1, 10);
                LCD_SendString(input_val);
            } 
            else if (key == '#') { // Save assignment [cite: 7]
                int days;
                sscanf(input_val, "%d", &days);
                if (assignments[current_slot].status == STATUS_EMPTY) {
                    assignments[current_slot].id = current_slot + 1;
                    assignments[current_slot].days_remaining = days;
                    assignments[current_slot].status = STATUS_ACTIVE;
                    assignments[current_slot].overdue_counter = 0;
                }
                memset(input_val, 0, 3);
                input_idx = 0;
            } 
            else if (key == '*') { // Clear entry
                memset(input_val, 0, 3);
                input_idx = 0;
            }
        }

        // Scroll Button Logic (PB12) [cite: 17]
        if (HAL_GPIO_ReadPin(GPIOB, GPIO_PIN_12) == GPIO_PIN_RESET) {
            HAL_Delay(200); // Debounce
            current_slot = (current_slot + 1) % MAX_ASSIGNMENTS;
            LCD_SendCommand(0x01); // Clear screen for next slot
        }
    }
}

/* --- Interrupt Callbacks --- */

// Submission via Push Button 2 (PB13) [cite: 14, 17]
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == GPIO_PIN_13) {
        if (assignments[current_slot].status != STATUS_EMPTY) {
            assignments[current_slot].status = STATUS_SUBMITTED;
        }
    }
}

// Deadline Countdown logic [cite: 9, 12, 13]
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        timer_count++;
        if (timer_count >= DAY_TICK_THRESHOLD) {
            timer_count = 0;
            for (int i = 0; i < MAX_ASSIGNMENTS; i++) {
                if (assignments[i].status == STATUS_ACTIVE) {
                    if (assignments[i].days_remaining > 0) {
                        assignments[i].days_remaining--;
                    } else {
                        assignments[i].status = STATUS_OVERDUE;
                    }
                } else if (assignments[i].status == STATUS_OVERDUE) {
                    assignments[i].overdue_counter++;
                    if (assignments[i].overdue_counter > 5) { // Auto-deletion 
                        assignments[i].status = STATUS_EMPTY;
                    }
                }
            }
        }
    }
}

/* --- Core Subroutines --- */

void Update_Alerts(void) {
    // Reset all outputs [cite: 19]
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_10|GPIO_PIN_11, GPIO_PIN_RESET);

    if (assignments[current_slot].status == STATUS_EMPTY) return;

    if (assignments[current_slot].status == STATUS_SUBMITTED) {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET); // GREEN
    } 
    else if (assignments[current_slot].status == STATUS_OVERDUE) {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_10, GPIO_PIN_SET); // RED
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_11, GPIO_PIN_SET); // BUZZER [cite: 19]
    } 
    else if (assignments[current_slot].days_remaining <= 3) {
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET);  // YELLOW [cite: 11]
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_11, GPIO_PIN_SET); // WARNING BUZZER
    }
}

void Display_Menu(void) {
    char l1[16], l2[16];
    LCD_SetCursor(0, 0);
    
    if (assignments[current_slot].status == STATUS_EMPTY) {
        sprintf(l1, "Slot %d: Empty   ", current_slot + 1);
        LCD_SendString(l1);
        LCD_SetCursor(1, 0);
        LCD_SendString("Set Days: --    ");
    } else {
        sprintf(l1, "Asgn %d: %2d Days", assignments[current_slot].id, assignments[current_slot].days_remaining);
        LCD_SendString(l1);
        LCD_SetCursor(1, 0);
        switch(assignments[current_slot].status) {
            case STATUS_ACTIVE:    LCD_SendString("Status: Active  "); break;
            case STATUS_SUBMITTED: LCD_SendString("Status: DONE    "); break;
            case STATUS_OVERDUE:   LCD_SendString("Status: OVERDUE "); break;
            default: break;
        }
    }
}

/* --- Hardware Drivers --- */

char Keypad_Scan(void) {
    char keys[4][4] = {{'1','2','3','A'},{'4','5','6','B'},{'7','8','9','C'},{'*','0','#','D'}};
    for (int r = 0; r < 4; r++) {
        HAL_GPIO_WritePin(GPIOA, 0x0F, GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOA, (1 << r), GPIO_PIN_RESET);
        for (int c = 0; c < 4; c++) {
            if (HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_4 << c)) == GPIO_PIN_RESET) {
                HAL_Delay(150); // Debounce [cite: 21]
                while(HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_4 << c)) == GPIO_PIN_RESET);
                return keys[r][c];
            }
        }
    }
    return 0;
}

void LCD_SendCommand(uint8_t cmd) {
    uint8_t u = cmd & 0xF0, l = (cmd << 4) & 0xF0;
    uint8_t d[4] = {u|0x0C, u|0x08, l|0x0C, l|0x08};
    HAL_I2C_Master_Transmit(&hi2c1, LCD_ADDR, d, 4, 10);
}
void LCD_SendData(uint8_t data) {
    uint8_t u = data & 0xF0, l = (data << 4) & 0xF0;
    uint8_t d[4] = {u|0x0D, u|0x09, l|0x0D, l|0x09};
    HAL_I2C_Master_Transmit(&hi2c1, LCD_ADDR, d, 4, 10);
}
void LCD_Init(void) {
    HAL_Delay(50); LCD_SendCommand(0x30); HAL_Delay(5); LCD_SendCommand(0x30);
    HAL_Delay(1); LCD_SendCommand(0x30); LCD_SendCommand(0x20); LCD_SendCommand(0x28);
    LCD_SendCommand(0x08); LCD_SendCommand(0x01); HAL_Delay(2); LCD_SendCommand(0x0C);
}
void LCD_SendString(char *str) { while(*str) LCD_SendData(*str++); }
void LCD_SetCursor(uint8_t row, uint8_t col) { LCD_SendCommand((row == 0 ? 0x80 : 0xC0) + col); }

/* --- Auto-Generated Configs --- */
void SystemClock_Config(void) {
    // Standard Blue Pill 72MHz Clock Config
}
static void MX_I2C1_Init(void) {
    hi2c1.Instance = I2C1; hi2c1.Init.ClockSpeed = 100000; HAL_I2C_Init(&hi2c1);
}
static void MX_TIM2_Init(void) {
    htim2.Instance = TIM2; htim2.Init.Prescaler = 7199; htim2.Init.Period = 9999; HAL_TIM_Base_Init(&htim2);
}
static void MX_GPIO_Init(void) {
    __HAL_RCC_GPIOA_CLK_ENABLE(); __HAL_RCC_GPIOB_CLK_ENABLE();
    GPIO_InitTypeDef G = {0};
    // Keypad Rows
    G.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2|GPIO_PIN_3; G.Mode = GPIO_MODE_OUTPUT_PP; HAL_GPIO_Init(GPIOA, &G);
    // Keypad Cols
    G.Pin = GPIO_PIN_4|GPIO_PIN_5|GPIO_PIN_6|GPIO_PIN_7; G.Mode = GPIO_MODE_INPUT; G.Pull = GPIO_PULLUP; HAL_GPIO_Init(GPIOA, &G);
    // LEDs & Buzzer
    G.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_10|GPIO_PIN_11; G.Mode = GPIO_MODE_OUTPUT_PP; HAL_GPIO_Init(GPIOB, &G);
    // Scroll Button
    G.Pin = GPIO_PIN_12; G.Mode = GPIO_MODE_INPUT; G.Pull = GPIO_PULLUP; HAL_GPIO_Init(GPIOB, &G);
    // Submit Button (EXTI)
    G.Pin = GPIO_PIN_13; G.Mode = GPIO_MODE_IT_FALLING; G.Pull = GPIO_PULLUP; HAL_GPIO_Init(GPIOB, &G);
    HAL_NVIC_SetPriority(EXTI15_10_IRQn, 0, 0); HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}
