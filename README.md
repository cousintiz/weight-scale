# weight-scale

## Overview

This project implements a digital weight scale using an STM32L4R5ZI microcontroller. It reads data from a load cell via an ADC, processes the readings, and displays the calculated weight on an I2C LCD. The weight is also transmitted over UART for debugging or data logging.

**Note:** This project was originally developed in 2023 and the code was uploaded to this repository in 2025.

## Hardware

*   **Microcontroller:** STM32L4R5ZI (Based on `.ioc` file and common STM32 naming)
*   **Load Cell:** Connected to an ADC input (ADC1, Channel 1 inferred from `main.c`). An HX711 amplifier module is commonly used with load cells, but its presence isn't explicitly confirmed in the C code provided.
*   **Display:** Character LCD connected via I2C (I2C1 inferred from `main.c` and `liquidcrystal_i2c.c`).
*   **Tare Button:** An external button connected to a GPIO pin with an interrupt (`HAL_GPIO_EXTI_Callback` in `main.c`) to trigger the tare function.
*   **UART:** LPUART1 used for communication (e.g., debugging output).

## Firmware (`Core/Src/main.c`)

*   **Initialization:** Configures System Clock, GPIOs, ADC (10-bit resolution), I2C1, and LPUART1 using STM32 HAL libraries.
*   **Calibration:**
    *   Performs initial ADC self-calibration (`HAL_ADCEx_Calibration_Start`).
    *   Runs a `calibrate` function on startup. This likely determines the zero-offset by averaging `CAL_SAMPLES` (64) ADC readings.
*   **Measurement Loop:**
    *   Continuously checks if the ADC is ready using `is_ready()` (details of this function are not fully visible in the snippet but might involve checking ADC flags or a timer).
    *   Reads the ADC `MEAN_SAMPLES` (1024) times using `read_adc()`. This function includes a simple low-pass filter (`filter = filter * (1 - ALFA) + adc_value * ALFA;` with ALFA=0.01).
    *   Calculates the average of these readings in `get_adc_average()`.
    *   Converts the average ADC reading to weight using `get_weight()`. This function likely applies a linear conversion factor (slope) determined offline and the offset calculated during calibration. The exact formula isn't shown but uses the `offset` variable.
    *   Displays the calculated weight (in grams, formatted to two decimal places) on the LCD using `show_weight()`.
    *   Sends the weight value over LPUART1 as a string (e.g., "\r\n Weight: 123.45 g").
*   **Tare Function:**
    *   A GPIO interrupt (`HAL_GPIO_EXTI_Callback`) likely triggers the tare functionality.
    *   Sets a `tara` flag and potentially captures the current weight as `tare_weight`. The `show_weight` function probably subtracts `tare_weight` when the `tara` flag is active.
*   **Libraries:**
    *   STM32 HAL Library
    *   Custom I2C LCD library (`liquidcrystal_i2c.h`/`.c`)

## Calibration Script (`calibration_script.py`)

*   This Python script is a **development utility** and is **not** part of the embedded firmware running on the STM32.
*   It uses `numpy`, `scipy`, and `matplotlib`.
*   Takes known ADC readings and corresponding real weights.
*   Performs linear regression (`np.polyfit`) to find the calibration coefficients (slope and intercept) that relate ADC values to weight.
*   Can be used to visualize the calibration data and interpolation.
*   **Usage:** The coefficients (slope) derived from this script are likely hardcoded into the `get_weight` function within the C code. The offset is determined dynamically by the `calibrate` function on the microcontroller.

## Building and Running

*   This project appears to be configured for STM32CubeIDE or a similar Eclipse-based environment (`.cproject`, `.project`, `.ioc`).
*   Build the project using the IDE.
*   Flash the firmware onto the STM32L4R5ZI target board using a debugger/programmer (like ST-Link).
*   Connect a serial terminal (e.g., PuTTY, Tera Term) to LPUART1 to view the weight output.

## Usage

1.  **Power On:** Apply power to the STM32 board.
2.  **Initialization:** The scale will automatically perform an initial calibration to determine the zero point. The LCD should show "0.00 g" (or similar).
3.  **Weighing:** Place an object on the load cell platform. The LCD will update to show the measured weight in grams. The same value is sent via LPUART1.
4.  **Tare:** To zero out the current weight (e.g., if using a container), press the Tare button. The display should return to "0.00 g". Any subsequent weight added will be measured relative to the tared weight. Pressing Tare again might reset the tare value, depending on the exact implementation in `HAL_GPIO_EXTI_Callback`.

## Dependencies (Python Script)

Install necessary Python libraries:
```bash
pip install numpy scipy matplotlib
```

## License

Refer to the `LICENSE` file (MIT License).