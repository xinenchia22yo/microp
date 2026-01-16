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

/* ================= LED HELPERS ================= */
void LEDs_Off(void) {
    HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0 | GPIO_PIN_1 | GPIO_PIN_2, GPIO_PIN_RESET);
}

void LED_Set(uint8_t pin) {
    LEDs_Off();
    HAL_GPIO_WritePin(GPIOB, pin, GPIO_PIN_SET);
}

/* ================= DISPLAY FUNCTION ================= */
void Update_Display_And_LEDs(void) {
    if (current_state != DISPLAY_MODE) return;

    LEDs_Off();
    LCD_Clear();

    int days_val = atoi(myAssignments[display_idx].days); // Convert days string to number
    char line1[16], line2[16];

    if (myAssignments[display_idx].submitted) {
        LED_Set(GPIO_PIN_0); // Green LED
        sprintf(line1,"#%d: %s [OK]", display_idx+1, myAssignments[display_idx].code);
        strcpy(line2,"SUBMITTED");
    } else if (days_val == 0) {
        LED_Set(GPIO_PIN_2); // Red LED
        sprintf(line1,"#%d: %s [!]", display_idx+1, myAssignments[display_idx].code);
        strcpy(line2,"OVERDUE");
    } else {
        LED_Set(days_val <= 2 ? GPIO_PIN_2 : GPIO_PIN_1); // Red if <=2 days, Yellow otherwise
        sprintf(line1,"#%d: %s", display_idx+1, myAssignments[display_idx].code);
        sprintf(line2,"Days Left: %d", days_val);
    }

    LCD_Print(line1);
    LCD_Send(0xC0,0); // Move to second line
    LCD_Print(line2);
}

/* ================= KEYPAD ================= */
char Keypad_Scan(void) {
    char keys[4][4] = {
        {'D','#','0','*'},
        {'C','9','8','7'},
        {'B','6','5','4'},
        {'A','3','2','1'}
    };

    for(int i=0;i<4;i++){
        HAL_GPIO_WritePin(GPIOA,(GPIO_PIN_4<<i),GPIO_PIN_RESET); // Drive column low
        for(int j=0;j<4;j++){
            if(HAL_GPIO_ReadPin(GPIOA,(GPIO_PIN_0<<j))==GPIO_PIN_RESET){ // Check row
                HAL_Delay(20); 
                while(HAL_GPIO_ReadPin(GPIOA,(GPIO_PIN_0<<j))==GPIO_PIN_RESET); // Wait release
                HAL_GPIO_WritePin(GPIOA,(GPIO_PIN_4<<i),GPIO_PIN_SET);
                return keys[i][j];
            }
        }
        HAL_GPIO_WritePin(GPIOA,(GPIO_PIN_4<<i),GPIO_PIN_SET); // Set column high again
    }
    return 0; // No key pressed
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

       // ------------------- 2. Submission button -------------------
        if(current_state==DISPLAY_MODE && HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_10)==GPIO_PIN_RESET){
            HAL_Delay(50);
            if(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_10)==GPIO_PIN_RESET){
                myAssignments[display_idx].submitted=1;
                Update_Display_And_LEDs();
            }
            while(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_10)==GPIO_PIN_RESET);
        }
        
        // ------------------- 3. Keypad -------------------
        char key = Keypad_Scan();
        if(key){
            // --- Press A ---
            if(key=='A'){
                if(assignment_count==0){
                    LCD_Clear(); LCD_Print("No Data!"); HAL_Delay(800);
                    LCD_Clear(); LCD_Print("Asgn #1 Code:"); LCD_Send(0xC0,0);
                } else {
                    if(current_state!=DISPLAY_MODE) { current_state=DISPLAY_MODE; display_idx=0; }
                    else display_idx=(display_idx+1)%assignment_count; // Scroll
                    Update_Display_And_LEDs();
                }
            }

            // --- Press '*' to exit display ---
            else if(key=='*' && current_state==DISPLAY_MODE){
                current_state = (assignment_count>=5)? LIST_FULL: INPUT_COURSE;
                LEDs_Off(); LCD_Clear();
                if(current_state==LIST_FULL) LCD_Print("STORAGE FULL");
                else { char msg[16]; sprintf(msg,"Asgn #%d Code:",assignment_count+1); LCD_Print(msg); LCD_Send(0xC0,0);}
            }

            // --- Input mode typing ---
            else if(current_state!=DISPLAY_MODE && current_state!=LIST_FULL){
                if(key=='*'){ // Save / Enter
                    if(current_state==INPUT_COURSE && str_idx>0){ 
                        current_state=INPUT_DAYS; str_idx=0; LCD_Clear(); LCD_Print("Days Left:"); LCD_Send(0xC0,0); 
                    }
                    else if(current_state==INPUT_DAYS && str_idx>0){
                        strcpy(myAssignments[assignment_count].code,temp_course);
                        strcpy(myAssignments[assignment_count].days,temp_days);
                        myAssignments[assignment_count].submitted=0; assignment_count++;
                        LCD_Clear();
                        if(assignment_count>=5){ current_state=LIST_FULL; LCD_Print("STORAGE FULL"); }
                        else{
                            current_state=INPUT_COURSE; str_idx=0; memset(temp_course,0,sizeof(temp_course)); memset(temp_days,0,sizeof(temp_days));
                            char msg[16]; sprintf(msg,"Asgn #%d Code:",assignment_count+1); LCD_Print(msg); LCD_Send(0xC0,0);
                        }
                    }
                }
                else if(key=='#'){ // Backspace
                    if(str_idx>0){ str_idx--; if(current_state==INPUT_COURSE) temp_course[str_idx]=0; else temp_days[str_idx]=0; LCD_Send(0x10,0); LCD_Send(' ',1); LCD_Send(0x10,0);}
                }
                else{ // Typing keys
                    if(current_state==INPUT_COURSE && str_idx<9){ temp_course[str_idx++]=key; LCD_Send(key,1); }
                    else if(current_state==INPUT_DAYS && str_idx<4){ temp_days[str_idx++]=key; LCD_Send(key,1); }
                }
            }

            HAL_Delay(200); // Key debounce
        }
    }
}

void SysTick_Handler(void){ HAL_IncTick(); } // Required for HAL_GetTick and HAL_Delay
