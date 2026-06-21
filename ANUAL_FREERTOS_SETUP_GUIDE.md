# Complete Master Reference: Manual FreeRTOS Integration Guide
**Target Microcontroller:** STM32U585 (ARM Cortex-M33 Core)  
**IDE & Toolchain:** STM32CubeIDE (GCC Compiler)  
**Architecture Deployment:** Non-TrustZone Monolithic Portfolio (`ARM_CM33_NTZ`)

---

## Phase 1: Acquiring the Official Source Pack
1. Open your web browser and navigate to the official [STMicroelectronics X-CUBE-FREERTOS Expansion Pack Page](https://st.com).
2. Click **Get Software**, accept the terms, log into your `myST` account, and download the `.zip` archive.
3. Extract the downloaded ZIP file to a known temporary location on your local machine (e.g., your Desktop or Downloads folder).

---

## Phase 2: Workspace Isolation & Pruning
When importing the source files into a clean, bare-metal STM32CubeIDE project, you must delete all unused architectures to prevent multi-definition compiler crashes.

### Step 2.1: Copying Files to Your Project Root
1. Open your machine's File Explorer and go to the unzipped FreeRTOS package directory.
2. Locate the folder at: `.../Middlewares/Third_Party/FreeRTOS/`
3. Copy that entire `FreeRTOS` folder.
4. Open your active STM32CubeIDE project directory on your computer, create a top-level directory in the root named `Third_Party`, and paste the folder inside.

### Step 2.2: Pruning the Codebase
Navigate into your project's `Third_Party/FreeRTOS/Source/portable/` folder and perform the following cleanup actions:
1. **Delete** these directories entirely: `Common`, `IAR`, `Keil`, `RVDS`, `Tasking`, `ThirdParty`. Keep only **`GCC`** and **`MemMang`**.
2. Open the **`MemMang`** directory: Delete `heap_1.c`, `heap_2.c`, `heap_3.c`, and `heap_5.c`. **Keep only `heap_4.c`** (provides automatic adjacent-block memory consolidation to eliminate fragmentation during dynamic deletion).
3. Open the **`GCC`** directory: Delete all architecture listings except **`ARM_CM33_NTZ`** (this is the dedicated port for the Cortex-M33 running without active TrustZone multi-binary security splits).

---

## Phase 3: IDE Compiler Path Configuration
The GCC compiler cannot find manually added headers automatically. You must link them within the IDE properties:
1. Right-click your project name in the STM32CubeIDE **Project Explorer** tree and select **Properties**.
2. Expand **C/C++ Build** and select **Settings**.
3. Under the **Tool Settings** tab, navigate to **MCU GCC Compiler** $\rightarrow$ **Include paths**.
4. Click the **Add... (Plus icon)** button, select **Workspace...**, and add these two specific folders:
   * `"${workspace_loc:/${ProjName}/Third_Party/FreeRTOS/Source/include}"`
   * `"${workspace_loc:/${ProjName}/Third_Party/FreeRTOS/Source/portable/GCC/ARM_CM33_NTZ}"`
5. Click **Apply and Close**.

---

## Phase 4: Fetching and Injecting `FreeRTOSConfig.h`
Every FreeRTOS project requires a configuration profile to set up kernel limits and task variables.
1. Go back to your unzipped expansion package on your computer and navigate to:  
   `Projects/NUCLEO-U575ZI-Q/Applications/FreeRTOS/FreeRTOS_Mutex/Config/`
2. Copy the file named **`FreeRTOSConfig.h`**.
3. Return to STM32CubeIDE, right-click on your project's **`Core/Inc/`** folder, and select **Paste**.
4. Open your newly pasted `FreeRTOSConfig.h` file and make sure the following properties are configured to disable TrustZone features:
   ```c
   #define configENABLE_TRUSTZONE            0
   #define configRUN_FREE_RTOS_SECURE_ONLY   0
   ```

---

## Phase 5: Hardware Interrupt Mapping & Header Sanitation
To let the FreeRTOS scheduling engine run your threads, you must override the default bare-metal handlers. 

### Step 5.1: Clean the Header File (`stm32u5xx_it.h`)
Ensure that all executable function logic is removed from `stm32u5xx_it.h` to prevent multi-definition linker crashes. It should only contain declarations:
```c
/* Ensure these prototypes sit cleanly in stm32u5xx_it.h */
void NMI_Handler(void);
void HardFault_Handler(void);
void MemManage_Handler(void);
void BusFault_Handler(void);
void UsageFault_Handler(void);
void SVC_Handler(void);
void DebugMon_Handler(void);
void PendSV_Handler(void);
void SysTick_Handler(void);
```

### Step 5.2: Safely Remap Code inside the Source File (`stm32u5xx_it.c`)
Open `Core/Src/stm32u5xx_it.c`. You must place all modifications inside the `/* USER CODE */` comment zones so they are not wiped out when modifying your `.ioc` file in the future.

1. **Add Assembly Externs to the Top:**
   ```c
   /* USER CODE BEGIN 0 */
   extern void vPortSVCHandler(void);
   extern void xPortPendSVHandler(void);
   extern void xPortSysTickHandler(void);
   /* USER CODE END 0 */
   ```

2. **Intercept System Core Calls:**
   ```c
   void SVC_Handler(void)
   {
     /* USER CODE BEGIN SVC_IRQn 0 */
     vPortSVCHandler(); 
     /* USER CODE END SVC_IRQn 0 */
     /* USER CODE BEGIN SVC_IRQn 1 */
     /* USER CODE END SVC_IRQn 1 */
   }

   void PendSV_Handler(void)
   {
     /* USER CODE BEGIN PendSV_IRQn 0 */
     xPortPendSVHandler(); 
     /* USER CODE END PendSV_IRQn 0 */
     /* USER CODE BEGIN PendSV_IRQn 1 */
     /* USER CODE END PendSV_IRQn 1 */
   }

   void SysTick_Handler(void)
   {
     /* USER CODE BEGIN SysTick_IRQn 0 */
     HAL_IncTick();         /* Services standard STM32 HAL peripheral timeouts */
     xPortSysTickHandler(); /* Drives the operational FreeRTOS time increments */
     /* USER CODE END SysTick_IRQn 0 */
     /* USER CODE BEGIN SysTick_IRQn 1 */
     /* USER CODE END SysTick_IRQn 1 */
   }
   ```

---

## Phase 6: Writing the Basic Test Application Code
Open **`Core/Src/main.c`** to initialize your test task environment.

1. **Include headers at the top of the file:**
   ```c
   /* USER CODE BEGIN Includes */
   #include "FreeRTOS.h"
   #include "task.h"
   /* USER CODE END Includes */
   ```

2. **Define your task prototype and execution logic:**
   ```c
   /* USER CODE BEGIN PFP */
   void Heartbeat_Task(void *pvParameters);
   /* USER CODE END PFP */

   /* USER CODE BEGIN 4 */
   void Heartbeat_Task(void *pvParameters)
   {
     while(1)
     {
       /* Insert your test application logic or GPIO pin toggles here */
       
       /* Block the thread efficiently for exactly 500 milliseconds */
       vTaskDelay(pdMS_TO_TICKS(500)); 
     }
   }
   /* USER CODE END 4 */
   ```

3. **Instantiate and start the kernel scheduler inside `main()`:**
   ```c
   int main(void)
   {
     HAL_Init();
     SystemClock_Config();
     /* (Initialize any configured peripherals here) */

     /* USER CODE BEGIN 2 */
     xTaskCreate(Heartbeat_Task, "Heartbeat", 128, NULL, tskIDLE_PRIORITY + 1, NULL);
     vTaskStartScheduler();
     /* USER CODE END 2 */

     while (1)
     {
        /* Code execution should never drop here unless RAM allocations fail */
     }
   }
   ```

---

## Phase 7: Post-Setup Verification
1. Right-click your project root name in the sidebar and select **Index** $\rightarrow$ **Rebuild** to refresh the workspace file indexer.
2. Change a setting in your `.ioc` layout and click save to generate code. Verify that none of your modifications inside the `/* USER CODE */` slots were erased.
3. Press **Ctrl + B** (or click the **Hammer Icon**). The workspace will compile completely with `0 errors`.
