#include "stm32f4xx_hal.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

/* ================= GLOBALS ================= */
I2C_HandleTypeDef hi2c1;      // I2C handle for LCD
uint8_t lcd_addr = 0;          // I2C address of the LCD

// Assignment structure
typedef struct {
    char code[10];   // Course code
    char days[5];    // Days left to submit
    uint8_t submitted; // 1 if submitted, 0 if not
} Assignment;

Assignment myAssignments[5];   // Store up to 5 assignments
int assignment_count = 0;      // Number of assignments saved
int display_idx = 0;           // Which assignment is currently displayed

// Program state
typedef enum { INPUT_COURSE, INPUT_DAYS, LIST_FULL, DISPLAY_MODE } InputState;
InputState current_state = INPUT_COURSE;

char temp_course[10] = {0};    // Temporary input for course code
char temp_days[5] = {0};       // Temporary input for days
uint8_t str_idx = 0;           // Index for typing input

#define COUNTDOWN_INTERVAL 60000 // 1 minute in milliseconds
uint32_t last_countdown_tick = 0; // Tracks last countdown update

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






/* ================= MAIN ================= */
int main(void){
    HAL_Init();

    // Enable GPIO and I2C clocks
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();
    __HAL_RCC_I2C1_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};

    // Keypad rows PA0-PA3 input
    GPIO_InitStruct.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2|GPIO_PIN_3;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT; GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA,&GPIO_InitStruct);

    // Keypad columns PA4-PA7 output
    GPIO_InitStruct.Pin = GPIO_PIN_4|GPIO_PIN_5|GPIO_PIN_6|GPIO_PIN_7;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(GPIOA,&GPIO_InitStruct);

    // Submission button PA10 input
    GPIO_InitStruct.Pin = GPIO_PIN_10; GPIO_InitStruct.Mode = GPIO_MODE_INPUT; GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA,&GPIO_InitStruct);

    // LEDs PB0-2 output
    GPIO_InitStruct.Pin = GPIO_PIN_0|GPIO_PIN_1|GPIO_PIN_2; GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    HAL_GPIO_Init(GPIOB,&GPIO_InitStruct);

    // I2C pins PB8-9
    GPIO_InitStruct.Pin = GPIO_PIN_8|GPIO_PIN_9; GPIO_InitStruct.Mode=GPIO_MODE_AF_OD; GPIO_InitStruct.Pull=GPIO_PULLUP; GPIO_InitStruct.Alternate=GPIO_AF4_I2C1;
    HAL_GPIO_Init(GPIOB,&GPIO_InitStruct);

    // Initialize I2C
    hi2c1.Instance=I2C1; hi2c1.Init.ClockSpeed=100000; hi2c1.Init.AddressingMode=I2C_ADDRESSINGMODE_7BIT; HAL_I2C_Init(&hi2c1);

    // Detect LCD address
    uint8_t addrs[]={0x27<<1,0x3F<<1};
    for(int i=0;i<2;i++){
        if(HAL_I2C_IsDeviceReady(&hi2c1,addrs[i],3,100)==HAL_OK){ lcd_addr=addrs[i]; break; }
    }

    // Initialize LCD
    LCD_Init(); LCD_Clear(); LCD_Print("Asgn #1 Code:"); LCD_Send(0xC0,0);
    last_countdown_tick = HAL_GetTick(); // Start countdown timer

    while(1){
        // ------------------- 1. Countdown -------------------
        if(HAL_GetTick()-last_countdown_tick >= COUNTDOWN_INTERVAL){
            last_countdown_tick = HAL_GetTick();
            for(int i=0;i<assignment_count;i++){
                if(!myAssignments[i].submitted){
                    int d = atoi(myAssignments[i].days);
                    if(d>0) sprintf(myAssignments[i].days,"%d",d-1);
                }
            }
            if(current_state==DISPLAY_MODE) Update_Display_And_LEDs();
        }








void SysTick_Handler(void){ HAL_IncTick(); } // Required for HAL_GetTick and HAL_Delay
