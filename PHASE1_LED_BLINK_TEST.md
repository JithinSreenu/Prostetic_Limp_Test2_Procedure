# Phase 1 Test: Manual GPIO & FreeRTOS LED Blink Test
**Target System:** STM32U585 Microcontroller  
**Environment Constraints:** Offline Workstation (No `.ioc` Code Generator Access)  
**Verification Goal:** Validate kernel task creation, heap layout stability, and scheduler context switching.

---

## 1. Architectural Strategy
Because this workflow operates entirely offline without access to the graphical pin configurator tool, we bypass automatic hardware initializations. Instead, we use **manual bare-metal C code configurations** within the protected user code blocks. 

Blinking an LED verifies that the following sub-systems are operating in unison:
1. **The Kernel Memory Allocation Engine (`heap_4.c`)** is successfully carving out the required Stack depth for new threads.
2. **The Processor Core Port Layer (`ARM_CM33_NTZ`)** is successfully executing context switches.
3. **The System Timebase Vector overrides** are properly driving the real-time scheduling engine clocks.

---

## 2. Target Hardware Pin Reference Matrix
Identify your specific physical development board configuration to target the correct hardware peripheral channels inside your code:

| STM32 Board Family Profile | Physical Hardware LED Pin | Target GPIO Peripheral Block |
| :--- | :--- | :--- |
| Standard Nucleo-U575 / U585 Boards | **PA5** (User LED 1 / Green) | `GPIOA` / `GPIO_PIN_5` |
| Advanced Nucleo-U575 Boards | **PB7** (User LED 2 / Blue) | `GPIOB` / `GPIO_PIN_7` |
| B-U585I-IOT02A Discovery Kits | **PG11** (On-board Green LED) | `GPIOG` / `GPIO_PIN_11` |

*Note: The implementation instructions below use **PA5**. If your board uses a different pin profile, swap out the `GPIOX` and `GPIO_PIN_X` definitions to match.*

---

## 3. Implementation Code Code Blocks (`Core/Src/main.c`)

Open your workspace **`Core/Src/main.c`** module file and integrate these code segments inside the marked code blocks:

### Step 3.1: Kernel Header File Links
Add the real-time scheduling extensions right below your main application dependencies:
```c
/* USER CODE BEGIN Includes */
#include "FreeRTOS.h"
#include "task.h"
/* USER CODE END Includes */
```

### Step 3.2: Task Forward Function Prototypes
Declare your application worker thread functions and hardware initialization signatures:
```c
/* USER CODE BEGIN PFP */
void LED_Blink_Task(void *pvParameters);
void Manual_LED_GPIO_Init(void);
/* USER CODE END PFP */
```

### Step 3.3: Task Creation and Scheduler Hook Integration
Update your primary initialization sequence inside `int main(void)` to start the peripheral hardware clocks and pass control over to the operating system loop:
```c
int main(void)
{
  HAL_Init();
  SystemClock_Config();

  /* USER CODE BEGIN 2 */
  /* 1. Manually configure the targeted LED GPIO peripheral hardware channel */
  Manual_LED_GPIO_Init();

  /* 2. Instantiate the real-time blinking thread task */
  xTaskCreate(
    LED_Blink_Task,        /* Function tracking pointer reference */
    "LED_Blink",           /* Thread identification name for kernel debugging */
    128,                   /* Stack Depth memory allocation value (words) */
    NULL,                  /* Optional parameters passed to the thread */
    tskIDLE_PRIORITY + 1,  /* Context scheduling processing priority rank */
    NULL                   /* Optional tracking task hook handle reference */
  );

  /* 3. Relinquish bare-metal control to the FreeRTOS Scheduler engine */
  vTaskStartScheduler();
  /* USER CODE END 2 */

  while (1)
  {
    /* Code tracking flow should never enter this segment if allocations are stable */
  }
}
```

### Step 3.4: Manual Peripheral Initialization & Thread Routines
Scroll to the bottom of `main.c` and insert your low-level GPIO register initialization functions and active OS loop logic:
```c
/* USER CODE BEGIN 4 */
/**
  * @brief  Manually initializes the GPIO Hardware Pin 5 on Port A as a Digital Output.
  */
void Manual_LED_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* 1. Enable the peripheral register clock for GPIO Port A */
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* 2. Enforce a safe default starting state (Drive the output pin LOW) */
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);

  /* 3. Fill the initialization structure to map the pin parameters */
  GPIO_InitStruct.Pin = GPIO_PIN_5;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;   /* Push-Pull active voltage driving */
  GPIO_InitStruct.Pull = GPIO_NOPULL;          /* No internal pull-up/pull-down resistors */
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;  /* Low speed operation reduces EMI noise */
  
  /* 4. Commit settings to the chip hardware registers */
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}

/**
  * @brief  FreeRTOS Worker Task Thread that continuously switches the targeted LED pin.
  */
void LED_Blink_Task(void *pvParameters)
{
  while(1)
  {
    /* 1. Toggle the logic output state of the hardware pin */
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);

    /* 2. Execute a non-blocking operating system kernel thread sleep.
          This tells the engine to run other tasks while this thread pauses. */
    vTaskDelay(pdMS_TO_TICKS(500)); /* Puts thread into Blocked state for 500ms */
  }
}
/* USER CODE END 4 */
```

---

## 4. Post-Setup Compilation Protocol
1. Press **`Ctrl + B`** (or click the **Hammer Icon** in the top-left menu panel).
2. Confirm the build logs complete cleanly with `0 errors`.
3. Flash the binary file directly to your microcontroller hardware board to verify the LED alternates states exactly every 500 milliseconds.
