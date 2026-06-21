# Phase 3 Upgrade: High-Efficiency UART Transmission using GPDMA
**Target System:** B-U585I-IOT02A Discovery Kit  
**Hardware Channels:** USART1 (Pins PA9-TX) mapped to GPDMA1 Channel 0  
**Verification Goal:** Offload UART transmissions from the CPU to the hardware DMA engine within a FreeRTOS environment.

---

## 1. Architectural Strategy
By default, standard polling configurations (`HAL_UART_Transmit`) force the CPU core to lock up inside loop blocks while waiting for data bytes to clear the register. This wastes processing cycles. 

By upgrading to **Direct Memory Access (DMA)**:
1. The FreeRTOS task passes the string buffer pointer to the `HAL_UART_Transmit_DMA` API.
2. The **GPDMA1** controller takes over, pulling bytes straight out of SRAM and pushing them into the `USART1->TDR` register over a dedicated hardware bus.
3. The CPU is instantly released to handle scheduling tasks while data transmission happens in the background.

---

## 2. Implementation Code Blocks (`Core/Src/main.c`)

Open your project workspace **`Core/Src/main.c`** file and integrate these updates into the corresponding user code blocks:

### Step 2.1: Global Structure Allocations
Declare both the standard UART handle and the new GPDMA channel structure handle:
```c
/* USER CODE BEGIN PV */
UART_HandleTypeDef huart1;
DMA_HandleTypeDef hdma_usart1_tx; /* Dedicated GPDMA channel structure */
/* USER CODE END PV */
```

### Step 2.2: Forward Function Prototypes
Ensure your updated initialization routing prototypes are registered:
```c
/* USER CODE BEGIN PFP */
void UART_Logging_Task(void *pvParameters);
void Manual_UART1_DMA_Init(void);
/* USER CODE END PFP */
```

### Step 2.3: Task Creation Inside `main()`
Switch out the old initialization call for the new DMA-enabled function call inside your `main()` loop setup:
```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();

  /* USER CODE BEGIN 2 */
  /* Initialize the peripheral hardware channels with GPDMA */
  Manual_LED_GPIO_Init();
  Manual_UART1_DMA_Init();

  /* Instantiate concurrent processing threads */
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

### Step 2.4: GPDMA Core Configuration & Non-Blocking Thread Logic
Scroll to the bottom of the module and place your low-level hardware register setups inside the `USER CODE 4` section:
```c
/* USER CODE BEGIN 4 */
/**
  * @brief  Manually configures USART1 on Pins PA9/PA10 and links it to GPDMA1 Channel 0.
  */
void Manual_UART1_DMA_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInit = {0};

  /* 1. Direct the USART1 Clock Source to run off PCLK2 */
  PeriphClkInit.PeriphClockSelection = RCC_PERIPHCLK_USART1;
  PeriphClkInit.Usart1ClockSelection = RCC_USART1CLKSOURCE_PCLK2;
  HAL_RCCEx_PeriphCLKConfig(&PeriphClkInit);

  /* 2. Enable GPDMA, USART1, and GPIOA Hardware Module Clocks */
  __HAL_RCC_GPDMA1_CLK_ENABLE();
  __HAL_RCC_USART1_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* 3. Map Pins PA9 (TX) and PA10 (RX) to Alternate Function 7 (USART1) */
  GPIO_InitStruct.Pin = GPIO_PIN_9 | GPIO_PIN_10;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 4. Configure GPDMA1 Channel 0 for UART TX Streaming */
  hdma_usart1_tx.Instance = GPDMA1_Channel0;
  hdma_usart1_tx.Init.Request = GPDMA_REQUEST_USART1_TX; /* Route to USART1 TX hardware trigger */
  hdma_usart1_tx.Init.BlkHWRequest = DMA_BREQ_SINGLE_BURST;
  hdma_usart1_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;  /* Streaming direction configuration */
  hdma_usart1_tx.Init.SrcInc = DMA_SINC_INCREMENTED;     /* Increment through memory string array */
  hdma_usart1_tx.Init.DestInc = DMA_DINC_FIXED;          /* UART TDR register target remains fixed */
  hdma_usart1_tx.Init.SrcDataWidth = DMA_SRC_DATAWIDTH_BYTE;
  hdma_usart1_tx.Init.DestDataWidth = DMA_DEST_DATAWIDTH_BYTE;
  hdma_usart1_tx.Init.Priority = DMA_HIGH_PRIORITY;
  
  HAL_DMA_Init(&hdma_usart1_tx);

  /* Link the initialized GPDMA channel handle to our UART configuration matrix */
  __HAL_LINKDMA(&huart1, hdmatx, hdma_usart1_tx);

  /* 5. Configure UART Basic Frame Parameters */
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
  * @brief  FreeRTOS Task that streams real-time telemetry utilizing non-blocking DMA.
  */
void UART_Logging_Task(void *pvParameters)
{
  /* CRITICAL NOTE: When using DMA, data buffers must remain valid during the 
     entire transmission window. Using static allocation ensures memory isn't 
     corrupted by context switches mid-run. */
  static char tx_buffer[64];
  uint32_t cycle_counter = 0;

  while(1)
  {
    /* Format your logging tracking string output */
    int len = snprintf(tx_buffer, sizeof(tx_buffer), "GPDMA Streaming Active! Cycle: %lu\r\n", cycle_counter++);

    /* Dispatch memory buffer straight to GPDMA. 
       This function returns INSTANTLY without locking the CPU thread context. */
    HAL_UART_Transmit_DMA(&huart1, (uint8_t*)tx_buffer, (uint16_t)len);

    /* Suspend task cleanly for 1 second. The GPDMA engine continues 
       shuttling bytes in the background while this task sleeps. */
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}
/* USER CODE END 4 */
```

---

## 3. Advanced Step: Routing the Interrupt Vector (`stm32u5xx_it.c`)
Because GPDMA runs independently, it must inform the system when it has finished transmitting all string bytes so that it can reset its internal pipeline states cleanly.

Open **`Core/Src/stm32u5xx_it.c`**, look for the global variables block, and reference your GPDMA handles safely inside the user code slots:

### Step 3.1: Add GPDMA Interrupt Prototype Declarations
```c
/* USER CODE BEGIN 0 */
extern DMA_HandleTypeDef hdma_usart1_tx;
/* USER CODE END 0 */
```

### Step 3.2: Insert the GPDMA Global Channel Interrupt Handler
Scroll to the very bottom of `stm32u5xx_it.c`, find the `USER CODE 1` block, and paste the hardware interrupt routing routine:
```c
/* USER CODE BEGIN 1 */
/**
  * @brief This function handles GPDMA1 Channel 0 global interrupt vector.
  */
void GPDMA1_Channel0_IRQHandler(void)
{
  /* Routes the hardware flag back to the ST HAL library to clear active flags */
  HAL_DMA_IRQHandler(&hdma_usart1_tx);
}
/* USER CODE END 1 */
```





Step 2: Update Your Active Workspace Code FilesTo complete this manual integration upgrade on your computer:
Open your real main.c file and apply the modifications detailed in Phase 2 of the markdown guide above.

Open your real stm32u5xx_it.c file and insert the GPDMA1 Interrupt Handler shown in Phase 3 into your user slots.

Save all your workspace tracks and press Ctrl + B to run a fresh build.


Why Using static is Mandatory Here (Crucial Tip)Take a look at line 98 in the guide code: 
static char tx_buffer[64];.
When using standard polling mode, a local variable like char tx_buffer[64]; 
works fine because the code stalls until transmission completes. However, with DMA, the function triggers the transfer and instantly leaves the block. 
If you don't use the static keyword, the memory location on the stack will immediately clean itself up and overwrite with trash data while the DMA hardware is still actively reading from it! 
Making it static locks its memory position safely in place.
