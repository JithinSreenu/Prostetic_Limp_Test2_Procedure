# Phase 4 Upgrade: Analog Signal Acquisition with DSP Filtering
**Target System:** B-U585I-IOT02A Discovery Kit  
**Hardware Channel:** ADC1 (Channel 5, mapped physically to Pin PA0)  
**Verification Goal:** Periodically sample an analog sensor signal, suppress noise using an Exponential Moving Average (EMA) filter, and stream calculated data over UART.

---

## 1. Engineering Calculations & Filter Theory

### 1.1 Fixed-Point ADC Voltage Translation
The STM32U585 ADC will be configured to operate at **12-bit resolution**. This divides the analog voltage spectrum into $2^{12} = 4096$ discrete digital quantization steps (ranging from `0` to `4095`).

Given a reference voltage ($V_{REF}$) of **3.3V**, the formula to calculate the actual analog input voltage from the raw digital reading is:
$$V_{Input} = \frac{\text{Raw ADC Value}}{4095} \times 3.3\text{V}$$

* *Example calculation:* If your sensor yields a raw quantization reading of `2500`:
  $$V_{Input} = \frac{2500}{4095} \times 3.3\text{V} = 2.0146\text{V}$$

### 1.2 Exponential Moving Average (EMA) Low-Pass Filter Architecture
To filter out random high-frequency thermal noise from long sensor wires, we implement a discrete-time first-order Infinite Impulse Response (IIR) filter, commonly called an EMA filter.

The mathematical difference equation is defined as:
$$Y[n] = \alpha \cdot X[n] + (1 - \alpha) \cdot Y[n-1]$$

Where:
* $X[n]$ = The current raw input sample from the ADC.
* $Y[n]$ = The new filtered output value.
* $Y[n-1]$ = The previous filtered output value (historical feedback).
* $\alpha$ (Alpha) = The smoothing factor parameter ($0 < \alpha \le 1$).

### 1.3 Calculating the Cutoff Frequency ($f_c$)
The smoothing factor $\alpha$ is directly bound to your desired system cutoff frequency ($f_c$) and the sampling loop interval period ($\Delta t$) by the equation:
$$\alpha = \frac{2\pi \cdot f_c \cdot \Delta t}{2\pi \cdot f_c \cdot \Delta t + 1}$$

**Our Design Scenario Parameters:**
* We sample the sensor inside a FreeRTOS task running every 20ms $\rightarrow \Delta t = 0.02\text{ seconds}$ (Sampling frequency $f_s = 50\text{Hz}$).
* We want to cut off random high-frequency electrical hum above **2Hz** $\rightarrow f_c = 2.0\text{Hz}$.

**Calculating $\alpha$:**
$$\alpha = \frac{2 \times 3.14159 \times 2.0 \times 0.02}{(2 \times 3.14159 \times 2.0 \times 0.02) + 1} = \frac{0.25132}{0.25132 + 1} \approx \mathbf{0.20}$$

* **Result:** Setting $\alpha = 0.20$ means each new sample contributes 20% to the output, while 80% is driven by the historical signal profile. This smooths out spikes immediately.

---

## 2. Implementation Code Blocks (`Core/Src/main.c`)

Open your workspace **`Core/Src/main.c`** file and integrate these code additions inside your user slots:

### Step 2.1: Global Object Handle Allotments
Declare your standard hardware configuration structures for the ADC peripheral layer:
```c
/* USER CODE BEGIN PV */
ADC_HandleTypeDef hadc1;
/* USER CODE END PV */
```

### Step 2.2: Forward Function Signatures
Register your upcoming initialisation blocks and the updated ADC reading task profile:
```c
/* USER CODE BEGIN PFP */
void ADC_Sensor_Task(void *pvParameters);
void Manual_ADC1_Init(void);
/* USER CODE END PFP */
```

### Step 2.3: Task Deployment inside `main()`
Launch the dedicated signal acquisition thread alongside your blinking and UART streaming tasks:
```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();

  /* USER CODE BEGIN 2 */
  Manual_LED_GPIO_Init();
  Manual_UART1_DMA_Init();
  Manual_ADC1_Init(); /* Fire up internal ADC1 routing blocks */

  /* Spin up concurrent thread processes */
  xTaskCreate(LED_Blink_Task, "LED_Blink", 128, NULL, tskIDLE_PRIORITY + 1, NULL);
  xTaskCreate(UART_Logging_Task, "UART_Log", 256, NULL, tskIDLE_PRIORITY + 1, NULL);
  xTaskCreate(ADC_Sensor_Task, "ADC_Sensor", 256, NULL, tskIDLE_PRIORITY + 2, NULL); /* Higher priority for stable sampling intervals */

  vTaskStartScheduler();
  /* USER CODE END 2 */

  while (1)
  {
  }
}
```

### Step 2.4: Hardware Configs & Digital Signal Processing Task
Scroll to the bottom of the module and place your raw peripheral configurations and math execution loops inside `USER CODE 4`:
```c
/* USER CODE BEGIN 4 */
/**
  * @brief  Manually configures ADC1 Channel 5 on Pin PA0 for 12-bit single-ended operation.
  */
void Manual_ADC1_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  ADC_ChannelConfTypeDef sConfig = {0};

  /* 1. Enable Clocks for GPIOA and ADC1 Modules */
  __HAL_RCC_GPIOA_CLK_ENABLE();
  __HAL_RCC_ADC1_CLK_ENABLE();

  /* 2. Configure Pin PA0 as an Analog Input Pin */
  GPIO_InitStruct.Pin = GPIO_PIN_0;
  GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 3. Configure Base ADC1 Basic Operating Architecture */
  hadc1.Instance = ADC1;
  hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
  hadc1.Init.Resolution = ADC_RESOLUTION_12B;            /* 12-Bit Quantization resolution */
  hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
  hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;             /* Single channel sampling mode */
  hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
  hadc1.Init.LowPowerAutoWait = DISABLE;
  hadc1.Init.ContinuousConvMode = DISABLE;                /* Triggered explicitly on demand */
  hadc1.Init.NbrOfConversion = 1;
  hadc1.Init.DiscontinuousConvMode = DISABLE;
  hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;       /* Manual software polling triggers */
  hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
  HAL_ADC_Init(&hadc1);

  /* 4. Map Channel 5 (Hardwired to PA0) as the Primary Conversion Target */
  sConfig.Channel = ADC_CHANNEL_5;
  sConfig.Rank = ADC_REGULAR_RANK_1;
  sConfig.SamplingTime = ADC_SAMPLETIME_12CYCLES_5;       /* Provide adequate sample hold window */
  sConfig.SingleDiff = ADC_SINGLE_ENDED;
  sConfig.OffsetNumber = ADC_OFFSET_NONE;
  sConfig.Offset = 0;
  HAL_ADC_ConfigChannel(&hadc1, &sConfig);
}

/**
  * @brief  FreeRTOS task that precisely samples raw sensor voltages, filters 
  *         out electrical high-frequency noise, and telemetry prints outputs.
  */
void ADC_Sensor_Task(void *pvParameters)
{
  uint32_t raw_adc_value = 0;
  float current_voltage = 0.0f;
  
  /* Filter Memory Variables */
  float filtered_voltage = 0.0f; 
  const float ALPHA = 0.20f;       /* Calculated smoothing factor for 2Hz cutoff at 50Hz sampling */
  
  /* First-run safety tracking flag to initialize filter baseline memory */
  uint8_t is_first_run = 1;

  while(1)
  {
    /* 1. Software trigger a fresh single conversion conversion */
    HAL_ADC_Start(&hadc1);
    
    /* 2. Wait up to 10ms for conversion complete */
    if (HAL_ADC_PollForConversion(&hadc1, 10) == HAL_OK)
    {
      raw_adc_value = HAL_ADC_GetValue(&hadc1);
      
      /* 3. Execute Voltage Conversion Formula */
      current_voltage = ((float)raw_adc_value / 4095.0f) * 3.3f;
      
      /* 4. Process Signal Through EMA Low-Pass Filter Formula */
      if (is_first_run)
      {
        filtered_voltage = current_voltage; /* Seed the baseline filter array history */
        is_first_run = 0;
      }
      else
      {
        filtered_voltage = (ALPHA * current_voltage) + ((1.0f - ALPHA) * filtered_voltage);
      }
    }
    HAL_ADC_Stop(&hadc1); /* Reset hardware registers conversion state safely */

    /* Optional Diagnosis: You can observe these values via a terminal by combining strings */
    // printf("Raw Volts: %.2f V, Filtered Volts: %.2f V\r\n", current_voltage, filtered_voltage);

    /* 5. Precise Sample Rate Enforcer: Block for exactly 20 milliseconds (50Hz Sampling Rate) */
    vTaskDelay(pdMS_TO_TICKS(20));
  }
}
/* USER CODE END 4 */
```
Step 2: Sync and Build Your FirmwareOpen your physical project files (main.c) and copy the structural code segments from the markdown file user sections above into their designated spots.
Press Ctrl + B to compile the complete firmware layout manually.The firmware will build into a unified execution block. 
The ADC processing layer runs dynamically on an independent execution timeline, completely isolated from your asynchronous LED blink loops and UART telemetry tasks.
