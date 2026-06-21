# Phase 2 Test: Bidirectional UART Communication over ST-LINK VCP
**Target System:** B-U585I-IOT02A Discovery Kit (or equivalent simulation core)  
**Hardware Channel:** USART1 (Pins PA9-TX / PA10-RX hardwired to ST-LINK USB)  
**Verification Goal:** Stream diagnostic data to a PC terminal and receive interactive keyboard commands to control system tasks.

---

## 1. Architectural Strategy
The B-U585I-IOT02A architecture hardwires the microcontroller's **USART1** peripheral block directly into an on-board ST-LINK circuit debugger. This connection serves as a **Virtual COM Port (VCP)**, which provides full bidirectional data streaming through your standard USB interface cable.

This architecture can be evaluated in two separate modes:
* **Virtually (No Physical Hardware):** Emulators (such as QEMU, Proteus, or Wokwi) read the raw `Manual_UART1_Init()` peripheral registers and spawn an interactive console box. Typing into this console simulates incoming voltage transitions on the `PA10` (RX) channel.
* **Physically (With Hardware):** Characters typed into your PC terminal client flow down the USB cable. The physical ST-LINK microcontroller acts as a bridge, driving the hardware pin state changes dynamically.

We will expand our FreeRTOS logging thread to routinely check the hardware Receive Data Register (RDR) for keyboard flags. If you send a `'1'`, the system drives the LED pin high; if you send a `'0'`, it drives it low.

---

## 2. Implementation Code Blocks (`Core/Src/main.c`)

Open your workspace **`Core/Src/main.c`** file and integrate these updates into the matching user blocks:

### Step 2.1: Variable Allocations
Ensure your global hardware control structure handle is declared near the top:
```c
/* USER CODE BEGIN PV */
UART_HandleTypeDef huart1;
/* USER CODE END PV */
```

### Step 2.2: Forward Function Prototypes
Verify that the task handles and helper initialisation signatures are properly declared:
```c
/* USER CODE BEGIN PFP */
void UART_Logging_Task(void *pvParameters);
void Manual_UART1_Init(void);
/* USER CODE END PFP */
```

### Step 2.3: System Configuration Inside `main()`
Verify your concurrent runtime configurations right before firing up the kernel loop:
```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();

  /* USER CODE BEGIN 2 */
  /* Initialize the peripheral hardware channels */
  Manual_LED_GPIO_Init();
  Manual_UART1_Init();

  /* Instantiate your real-time processing threads */
  xTaskCreate(LED_Blink_Task, "LED_Blink", 128, NULL, tskIDLE_PRIORITY + 1, NULL);
  xTaskCreate(UART_Logging_Task, "UART_Log", 256, NULL, tskIDLE_PRIORITY + 1, NULL);

  /* Start the FreeRTOS Scheduler engine */
  vTaskStartScheduler();
  /* USER CODE END 2 */

  while (1)
  {
  }
}
```

### Step 2.4: Bidirectional Routine Implementation
Scroll to the bottom of the module and paste the complete initialisation blocks and interactive control structures inside `USER CODE 4`:
```c
/* USER CODE BEGIN 4 */
/**
  * @brief  Manually configures USART1 on Pins PA9 (TX) and PA10 (RX) at 115200 Baud.
  */
void Manual_UART1_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};

  /* 1. Direct the USART1 Clock Source to run off the Main System Clock (PCLK2) */
  PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_USART1;
  PeriphClkInit.Usart1ClockSelection = RCC_USART1CLKSOURCE_PCLK2;
  HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit);

  /* 2. Enable Peripheral Driver Clocks */
  __HAL_RCC_USART1_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* 3. Map Pins PA9 (TX) and PA10 (RX) to Alternate Function 7 (AF7 - USART1) */
  GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 4. Configure UART Frame Structure Parameters */
  huart1.Instance = USART1;
  huart1.Init.BaudRate = 115200;
  huart1.Init.WordLength = UART_WORDLENGTH_8B;
  huart1.Init.StopBits = UART_STOPBITS_1;
  huart1.Init.Parity = UART_PARITY_NONE;
  huart1.Init.Mode = UART_MODE_TX_RX;
  huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
  huart1.Init.OverSampling = UART_OVERSAMPLING_16;
  huart1.Init.OneBitSampling = UART_ONE_BIT_SAMPLE_DISABLE;
  huart1.AdvancedInit.AdvFeatureInit = UART_ADVFEATURE_NO_INIT;
  
  HAL_UART_Init(&huart1);
}

/**
  * @brief  FreeRTOS Thread that manages dynamic data output streams and monitors
  *         the hardware registers for interactive keyboard control inputs.
  */
void UART_Logging_Task(void *pvParameters)
{
  char tx_buffer[64];
  uint8_t rx_byte = 0;
  uint32_t cycle_counter = 0;

  /* Print a helpful system initialization prompt to the PC screen */
  char *welcome = "\r\n=== Bidirectional UART Initialized ===\r\nType '1' to turn LED ON, '0' to turn LED OFF\r\n\r\n";
  HAL_UART_Transmit(&huart1, (uint8_t*)welcome, (uint16_t)strlen(welcome), 100);

  while(1)
  {
    /* 1. DATA TRANSMISSION PATHWAY (Microcontroller -> PC) */
    int len = snprintf(tx_buffer, sizeof(tx_buffer), "FreeRTOS Active. Counts: %lu\r\n", cycle_counter++);
    HAL_UART_Transmit(&huart1, (uint8_t*)tx_buffer, (uint16_t)len, 50);

    /* 2. DATA RECEIVING PATHWAY (PC -> Microcontroller)
       Listen for exactly 1 byte. We use a tiny 50ms timeout window so that
       the thread can resume and loop smoothly even if no key is pressed. */
    if (HAL_UART_Receive(&huart1, &rx_byte, 1, 50) == HAL_OK)
    {
      /* Echo the typed character back to the screen so you can verify your keystroke */
      HAL_UART_Transmit(&huart1, &rx_byte, 1, 10);
      
      /* Process the received terminal configuration command */
      if (rx_byte == '1')
      {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET); /* Turn LED ON */
        char *msg = " -> [LED Status: ON]\r\n";
        HAL_UART_Transmit(&huart1, (uint8_t*)msg, (uint16_t)strlen(msg), 50);
      }
      else if (rx_byte == '0')
      {
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET); /* Turn LED OFF */
        char *msg = " -> [LED Status: OFF]\r\n";
        HAL_UART_Transmit(&huart1, (uint8_t*)msg, (uint16_t)strlen(msg), 50);
      }
    }

    /* Block thread cleanly for 1 second to release processing resources */
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
/* USER CODE END 4 */
```

---

## 3. Evaluation and Testing Setup

### A. Testing Virtually (In a Simulator)
1. Load your newly compiled `.elf` or `.hex` firmware binary file into your simulation layout tool.
2. Link the simulator's built-in virtual serial terminal component to your simulated `USART1` pins (`PA9` and `PA10`).
3. Start the simulation runtime window. You will see the runtime count text rendering dynamically inside your virtual console box.
4. Focus your keyboard pointer on the console window and press **`1`** or **`0`** to toggle the simulated state of pin `PA5`.

### B. Testing Physically (With Your B-U585I-IOT02A Board)
1. Use a standard USB Type-C cable to connect your laptop directly to the port marked **ST-LINK** on your discovery kit.
2. Flash the compiled firmware onto your target board using the built-in programmer.
3. Open your desktop Serial Monitor software (such as PuTTY, Tera Term, or the built-in STM32 Terminal).
4. Configure your communication interface settings to match the target specification:
   * **Baud Rate:** `115200`
   * **Data Bits:** `8`
   * **Parity:** `None`
   * **Stop Bits:** `1`
5. **Crucial Terminal Step:** Open your terminal settings page and enable **Local Echo**. This forces the terminal to display your characters as you type them.
6. Open the port connection. Press **`1`** or **`0`** on your keyboard to toggle the physical LED state on your board instantly!
