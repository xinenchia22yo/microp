#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* ================= GLOBALS ================= */
I2C_HandleTypeDef hi2c1;
uint8_t lcd_addr = 0;

typedef struct {
    char code[10];
    char days[5];
    uint8_t submitted; 
} Assignment;

Assignment myAssignments[5]; 
int assignment_count = 0;
int display_idx = 0;

typedef enum { INPUT_COURSE, INPUT_DAYS, LIST_FULL, DISPLAY_MODE } InputState;
InputState current_state = INPUT_COURSE;

char temp_course[10] = {0};
char temp_days[5] = {0};
uint8_t str_idx = 0;

// Timer for 1-minute countdown
uint32_t last_countdown_tick = 0;
#define COUNTDOWN_INTERVAL 60000 // 60,000ms = 1 minute

/* ================= LCD FUNCTIONS ================= */
void LCD_Send(uint8_t val, uint8_t rs) {
    if (lcd_addr == 0) return;
    uint8_t high = (val & 0xF0) | rs | 0x08;
    uint8_t low  = ((val << 4) & 0xF0) | rs | 0x08;
    uint8_t data[4] = { high | 0x04, high, low | 0x04, low };
    HAL_I2C_Master_Transmit(&hi2c1, lcd_addr, data, 4, 100);
    HAL_Delay(2);
}

void LCD_Init(void) {
    HAL_Delay(100);
    LCD_Send(0x33, 0); LCD_Send(0x32, 0);
    LCD_Send(0x28, 0); LCD_Send(0x0C, 0);
    LCD_Send(0x01, 0); HAL_Delay(5);
}

void LCD_Print(char *str) {
    while(*str) LCD_Send(*str++, 1);
}

void LCD_Clear(void) {
    LCD_Send(0x01, 0);
    HAL_Delay(2);
}

/* ================= HELPERS ================= */
void Update_Display_And_LEDs(void) {
    if (current_state != DISPLAY_MODE) return;

    // Turn off all LEDs (PB0=G, PB1=Y, PB2=R)
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2, GPIO_PIN_RESET);
    LCD_Clear();
    
    char line1[16];
    int days_val = atoi(myAssignments[display_idx].days);

    if (myAssignments[display_idx].submitted) {
        // STATE: SUBMITTED (Green LED)
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET); 
        sprintf(line1, "#%d: %s [OK]", display_idx + 1, myAssignments[display_idx].code);
        LCD_Print(line1);
        LCD_Send(0xC0, 0);
        LCD_Print("SUBMITTED");
    } 
    else if (days_val == 0) {
        // STATE: OVERDUE (Red LED)
        HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET);
        sprintf(line1, "#%d: %s [!]", display_idx + 1, myAssignments[display_idx].code);
        LCD_Print(line1);
        LCD_Send(0xC0, 0);
        LCD_Print("OVERDUE");
    }
    else {
        // STATE: PENDING
        if (days_val <= 2) {
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_2, GPIO_PIN_SET); // Red
        } else {
            HAL_GPIO_WritePin(GPIOB, GPIO_PIN_1, GPIO_PIN_SET); // Yellow
        }

        sprintf(line1, "#%d: %s", display_idx + 1, myAssignments[display_idx].code);
        char line2[16];
        sprintf(line2, "Days Left: %d", days_val);
        LCD_Print(line1);
        LCD_Send(0xC0, 0);
        LCD_Print(line2);
    }
}

/* ================= KEYPAD ================= */
char Keypad_Scan(void) {
    char keys[4][4] = {
        {'D','#','0','*'}, {'C','9','8','7'},
        {'B','6','5','4'}, {'A','3','2','1'}
    };
    for (int i = 0; i < 4; i++) {
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_RESET);
        for (int j = 0; j < 4; j++) {
            if (HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_0 << j)) == GPIO_PIN_RESET) {
                HAL_Delay(20); 
                while(HAL_GPIO_ReadPin(GPIOA, (GPIO_PIN_0 << j)) == GPIO_PIN_RESET);
                HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
                return keys[i][j];
            }
        }
        HAL_GPIO_WritePin(GPIOA, (GPIO_PIN_4 << i), GPIO_PIN_SET);
    }
    return 0;
}

/* ================= MAIN ================= */
int main(void) {
    HAL_Init();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    // Keypad Input Rows (PA0-PA3)
    GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2 | GPIO_PIN_3;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // Keypad Output Columns (PA4-PA7)
    GPIO_InitStruct.Pin = GPIO_PIN_4 | GPIO_PIN_5 | GPIO_PIN_6 | GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // Submission Button (PA10)
    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // LEDs (PB0=Green, PB1=Yellow, PB2=Red)
    GPIO_InitStruct.Pin = GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    // I2C SCL/SDA (PB8, PB9)
    GPIO_InitStruct.Pin = GPIO_PIN_8 | GPIO_PIN_9;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_OD;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    GPIO_InitStruct.Alternate = GPIO_AF4_I2C1;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);

    hi2c1.Instance = I2C1;
    hi2c1.Init.ClockSpeed = 100000;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    HAL_I2C_Init(&hi2c1);

    uint8_t addrs[] = {0x27 << 1, 0x3F << 1};
    for (int i = 0; i < 2; i++) {
        if (HAL_I2C_IsDeviceReady(&hi2c1, addrs[i], 3, 100) == HAL_OK) {
            lcd_addr = addrs[i];
            break;
        }
    }

    LCD_Init();
    LCD_Clear();
    LCD_Print("Asgn #1 Code:");
    LCD_Send(0xC0, 0);

    last_countdown_tick = HAL_GetTick();

    while (1) {
        // --- 1. AUTOMATIC 1-MINUTE COUNTDOWN ---
        if (HAL_GetTick() - last_countdown_tick >= COUNTDOWN_INTERVAL) {
            last_countdown_tick = HAL_GetTick();
            for (int i = 0; i < assignment_count; i++) {
                if (!myAssignments[i].submitted) {
                    int d = atoi(myAssignments[i].days);
                    if (d > 0) {
                        d--; 
                        sprintf(myAssignments[i].days, "%d", d);
                    }
                }
            }
            if (current_state == DISPLAY_MODE) Update_Display_And_LEDs();
        }

        // --- 2. SUBMISSION BUTTON (PA10) ---
        if (current_state == DISPLAY_MODE && HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_10) == GPIO_PIN_RESET) {
            HAL_Delay(50); 
            if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_10) == GPIO_PIN_RESET) {
                myAssignments[display_idx].submitted = 1;
                Update_Display_And_LEDs();
            }
            while(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_10) == GPIO_PIN_RESET);
        }

        // --- 3. KEYPAD HANDLING ---
        char key = Keypad_Scan();
        if (key) {

            // --- KEY 'A' ---
            if (key == 'A') {
                if (assignment_count == 0) {
                    LCD_Clear(); LCD_Print("No Data!"); HAL_Delay(800);
                    LCD_Clear(); LCD_Print("Asgn #1 Code:"); LCD_Send(0xC0, 0);
                } else {
                    if (current_state != DISPLAY_MODE) {
                        // First press -> enter display mode
                        current_state = DISPLAY_MODE;
                        display_idx = 0; // start from #1
                    } else {
                        // Scroll to next assignment
                        display_idx = (display_idx + 1) % assignment_count;
                    }
                    Update_Display_And_LEDs();
                }
            }

            // --- EXIT VIEW ('*') ---
            else if (key == '*' && current_state == DISPLAY_MODE) {
                current_state = (assignment_count >= 5) ? LIST_FULL : INPUT_COURSE;
                HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2, GPIO_PIN_RESET);
                LCD_Clear();
                if (current_state == LIST_FULL) LCD_Print("STORAGE FULL");
                else {
                    char msg[16]; sprintf(msg, "Asgn #%d Code:", assignment_count + 1);
                    LCD_Print(msg); LCD_Send(0xC0, 0);
                }
            }

            // --- DATA ENTRY MODE ---
            else if (current_state != DISPLAY_MODE && current_state != LIST_FULL) {

                // ENTER / SAVE (*)
                if (key == '*') {
                    if (current_state == INPUT_COURSE && str_idx > 0) {
                        current_state = INPUT_DAYS; str_idx = 0;
                        LCD_Clear(); LCD_Print("Days Left:"); LCD_Send(0xC0, 0);
                    } 
                    else if (current_state == INPUT_DAYS && str_idx > 0) {
                        // Save assignment
                        strcpy(myAssignments[assignment_count].code, temp_course);
                        strcpy(myAssignments[assignment_count].days, temp_days);
                        myAssignments[assignment_count].submitted = 0;
                        assignment_count++;

                        LCD_Clear();
                        if (assignment_count >= 5) {
                            current_state = LIST_FULL;
                            LCD_Print("STORAGE FULL");
                        } else {
                            current_state = INPUT_COURSE; str_idx = 0;
                            memset(temp_course, 0, sizeof(temp_course));
                            memset(temp_days, 0, sizeof(temp_days));
                            char msg[16]; sprintf(msg, "Asgn #%d Code:", assignment_count+1);
                            LCD_Print(msg); LCD_Send(0xC0, 0);
                        }
                    }
                }

                // BACKSPACE (#)
                else if (key == '#') {
                    if (str_idx > 0) {
                        str_idx--;
                        if (current_state == INPUT_COURSE) temp_course[str_idx] = 0;
                        else temp_days[str_idx] = 0;
                        LCD_Send(0x10, 0); LCD_Send(' ', 1); LCD_Send(0x10, 0);
                    }
                }

                // TYPING
                else {
                    if (current_state == INPUT_COURSE && str_idx < 9) {
                        temp_course[str_idx++] = key; LCD_Send(key, 1);
                    } else if (current_state == INPUT_DAYS && str_idx < 4) {
                        temp_days[str_idx++] = key; LCD_Send(key, 1);
                    }
                }
            }

            HAL_Delay(200); // simple debounce
        }
    }
}

void SysTick_Handler(void) { HAL_IncTick(); }
