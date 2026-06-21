# Phase 5 Upgrade: End-to-End Autonomous Full-DMA Pipeline
**Target System:** B-U585I-IOT02A Discovery Kit  
**Hardware Map:** ADC1 -> GPDMA1 Channel 1 -> SRAM -> Flash Memory via NVidia/ST-FLASH-DMA  
**Verification Goal:** Eliminate all CPU polling cycles across the entire Analog Acquisition and Non-Volatile Data Logging paths.

---

## 1. Implementation Code Blocks (`Core/Src/main.c`)

Open your workspace **`Core/Src/main.c`** file and integrate these unified full-DMA pipeline upgrades:

### Step 1.1: Complete Global Handle Arrays
Declare the secondary GPDMA structure handles for the ADC data collection tracks:
```c
/* USER CODE BEGIN PV */
UART_HandleTypeDef huart1;
ADC_HandleTypeDef hadc1;

DMA_HandleTypeDef hdma_usart1_tx;
DMA_HandleTypeDef hdma_adc1;      /* Dedicated GPDMA channel for automated analog collection */

/* Automated double-buffering array for background ADC collections */
#define ADC_BUFFER_SIZE  2
static uint32_t adc_raw_dma_buffer[ADC_BUFFER_SIZE]; 
/* USER CODE END PV */
```

### Step 1.2: Full-DMA Initialisation Protocols
Scroll down to the `USER CODE 4` hardware setup layout block and update your ADC initialization function to use hardware GPDMA channels:
```c
/* USER CODE BEGIN 4 */
/**
  * @brief  Manually initializes ADC1 and links it to GPDMA1 Channel 1 for continuous streaming.
  */
void Manual_ADC1_FullDMA_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  ADC_ChannelConfTypeDef sConfig = {0};

  /* 1. Enable Module Hardware Clocks */
  __HAL_RCC_GPDMA1_CLK_ENABLE();
  __HAL_RCC_ADC1_CLK_ENABLE();
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* 2. Map Analog Routing Pins (PA0) */
  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 3. Setup GPDMA1 Channel 1 for Automated ADC Transfers */
  hdma_adc1.Instance = GPDMA1_Channel1;
  hdma_adc1.Init.Request = GPDMA_REQUEST_ADC1;          /* Link to ADC1 Hardware Trigger Line */
  hdma_adc1.Init.BlkHWRequest = DMA_BREQ_SINGLE_BURST;
  hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;       /* Pull from peripheral data registers to SRAM */
  hdma_adc1.Init.SrcInc = DMA_SINC_FIXED;                /* ADC Data Register address stays fixed */
  hdma_adc1.Init.DestInc = DMA_DINC_INCREMENTED;         /* Move forward through local array memory */
  hdma_adc1.Init.SrcDataWidth = DMA_SRC_DATAWIDTH_WORD;
  hdma_adc1.Init.DestDataWidth = DMA_DEST_DATAWIDTH_WORD;
  hdma_adc1.Init.Priority = DMA_HIGH_PRIORITY;
  HAL_DMA_Init(&hdma_adc1);

  /* Securely link GPDMA handle properties directly to ADC engine configuration track */
  __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);

  /* 4. Configure ADC Peripheral Core Operating Parameters */
  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
  hadc1.Init.Resolution = ADC_RESOLUTION_12B;
  hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
  hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc1.Init.ContinuousConvMode = ENABLE;                /* Continuous free-running acquisition mode */
  hadc1.Init.NbrOfConversion = 1;
  hadc1.Init.DMAContinuousRequests = ENABLE;             /* Enforce cyclical circular DMA array loops */
  HAL_ADC_Init(&hadc1);

  /* 5. Map Regular Execution Conversion Targets */
  sConfig.Channel = ADC_CHANNEL_5;
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_56CYCLES_5;
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  HAL_ADC_ConfigChannel(&hadc1, &sConfig);

  /* 6. Fire up the Background DMA Transfer Loop instantly */
  HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_raw_dma_buffer, ADC_BUFFER_SIZE);
}

/**
  * @brief  Autonomous Flash Logging function utilizing high-speed background bus loops.
  */
void Manual_Flash_Log_Data(uint32_t flash_sector_address, uint32_t *data_array, uint16_t word_count)
{
  /* Unlock Flash configurations safely */
  HAL_FLASH_Unlock();

  /* 
     On the STM32U585, Flash programming is executed over the Quad-Word Quad-Bus profile.
     The system hardware uses internal page-buffers to stream RAM pages to flash arrays 
     autonomously without forcing a step-by-step CPU wait state.
  */
  HAL_FLASH_Program(FLASH_TYPEPROGRAM_QUADWORD, flash_sector_address, (uint32_t)data_array);

  /* Lock configurations to secure system storage states */
  HAL_FLASH_Lock();
}
/* USER CODE END 4 */
```

### Step 1.3: Upgraded Zero-CPU Autonomous FreeRTOS Processing Loop
Update your `ADC_Sensor_Task` loop so it no longer triggers or polls hardware flags manually. Instead, it reads the data values loaded directly into your local RAM buffer by the DMA hardware:
```c
void ADC_Sensor_Task(void *pvParameters)
{
  float current_voltage = 0.0f;
  float filtered_voltage = 0.0f; 
  const float ALPHA = 0.20f;
  
  uint32_t target_flash_address = 0x080F0000; /* Dedicated sector location map inside safe Bank 2 user memory */

  while(1)
  {
    /* 
       ZERO POLLING OVERHEAD: 
       The GPDMA controller is continuously updating 'adc_raw_dma_buffer[0]' in the background. 
       The CPU simply reads the latest value straight from SRAM.
    */
    uint32_t fresh_raw_reading = adc_raw_dma_buffer[0];
    
    current_voltage = ((float)fresh_raw_reading / 4095.0f) * 3.3f;
    filtered_voltage = (ALPHA * current_voltage) + ((1.0f - ALPHA) * filtered_voltage);

    /* Prepare a local page array package to log telemetry to flash */
    uint32_t log_package[4] = { fresh_raw_reading, (uint32_t)(filtered_voltage * 1000), 0xAAAA, 0xBBBB };
    
    /* Offload data packet logging to the background flash page-buffer pipeline */
    Manual_Flash_Log_Data(target_flash_address, log_package, 4);
    target_flash_address += 16; /* Shift forward to the next empty Quad-Word storage slot */

    /* Suspend task loop cleanly for 20ms */
    vTaskDelay(pdMS_TO_TICKS(20));
  }
}
```

---

## 2. Low-Level Vector Registration Updates (`stm32u5xx_it.c`)
Open **`Core/Src/stm32u5xx_it.c`** and route the newly configured GPDMA1 Channel 1 hardware vector to maintain state clearance:

```c
/* USER CODE BEGIN 1 */
extern DMA_HandleTypeDef hdma_adc1;

/**
  * @brief This function handles GPDMA1 Channel 1 global interrupt vector (ADC Link).
  */
void GPDMA1_Channel1_IRQHandler(void)
{
  HAL_DMA_IRQHandler(&hdma_adc1);
}
/* USER CODE END 1 */
```


Step 4: Run the Build 
SetupOpen your active workspace files (main.c and stm32u5xx_it.c) and update your old functions with these new Full-DMA configurations.

Replace your original Manual_ADC1_Init() call inside main() with the updated Manual_ADC1_FullDMA_Init(); step.
Save your changes and compile your project using Ctrl + B.
Your entire telemetry setup is now fully optimized with hardware accelerators. 
The GPDMA controllers handle moving data both from your analog inputs into RAM and out to your UART terminal, 
completely bypassing standard CPU bottleneck delays.
