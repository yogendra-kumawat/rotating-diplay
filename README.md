# 🌀 High-Speed POV (Persistence of Vision) Video Display

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/STM32-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white" alt="STM32">
  <img src="https://img.shields.io/badge/ESP8266-000000?style=for-the-badge&logo=espressif&logoColor=white" alt="ESP8266">
  <img src="https://img.shields.io/badge/OpenCV-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white" alt="OpenCV">
  <img src="https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white" alt="C">
</p>

A custom-built,  **Persistence of Vision (POV)** display capable of streaming **live video frames** from a PC directly to a spinning RGB LED array. The system uses OpenCV to convert frames into polar coordinates, packs pixel data into an optimized **4-byte binary payload**, and transmits it at **230,400 baud** to an STM32F103 microcontroller that drives the LEDs and motor via hardware PWM.

> ⚠️ **Project Status: In Progress** — Physical synchronization (Hall effect sensor) is the next milestone.

---

## 🧭 Table of Contents

- [How It Works](#-how-it-works)
- [Hardware Structure](#-hardware-structure)
- [System Architecture](#-system-architecture)
- [The 4-Byte Protocol](#-the-4-byte-protocol)
- [Firmware Deep Dive](#-firmware-deep-dive)
- [Timer Configuration](#-timer-configuration)
- [Setup & Installation](#-setup--installation)
- [Known Limitations & Roadmap](#-known-limitations--roadmap)

---

## 💡 How It Works

A POV display works by spinning a line of LEDs fast enough that the human eye perceives a full 2D image — the same way a fan blade appears solid when spinning. This project takes that concept further by streaming **live video** to the spinning arm in real time.

```
PC (Python)                         STM32F103 (Spinning Arm)
───────────────                     ────────────────────────
Video Frame                         4-byte UART packet received
    │                                       │
    ▼                                       ▼
Polar Coordinate Mapping            Channel ID routing
(flat image → 360° circle)              │           │
    │                               LEDs (RGB PWM)  Motor (PWM)
    ▼                                   │
Pack into 4-byte binary payload     Shift registers clock out
    │                               to correct LED position
    ▼
Transmit at 230,400 baud
via USB-TTL / ESP8266
```

---

## 🔧 Hardware Structure

> Hardware block diagram:

<img width="1600" height="1204" alt="Hardware Structure" src="https://github.com/user-attachments/assets/59e1821d-1554-4ed2-8585-ad54104596aa" />

> Current build in progress:

<img width="400" height="225" alt="Build in progress" src="https://github.com/user-attachments/assets/bce04942-c39b-4d1b-9365-3711640be53b" />

### Bill of Materials

| Component | Purpose |
|-----------|---------|
| STM32F103 ("Blue Pill") | Main microcontroller — PWM, UART, GPIO |
| RGB LEDs (×16 per arm) | Persistence of vision display elements |
| 74HC595 Shift Registers | Extend GPIO to address 16 LED positions |
| BLDC Motor + ESC | Spins the LED arm at constant speed |
| Slip Ring | Passes power and data to the spinning arm |
| CH340G / CP2102 **or** ESP8266 | USB-to-TTL bridge (wired or wireless) |
| 3.3V / 5V Power Supply | Logic and LED power |

---

## 🏗️ System Architecture

The system has three layers working in tandem:

### 1. 🧠 The Brain — Python / PC (`rotating.ipynb`)

- Reads video frames using `cv2.VideoCapture`
- Maps the flat rectangular frame into a **360° polar grid** (16 LEDs × 360 rays)
- Extracts exact BGR intensity values at each polar coordinate
- Packs **Channel ID + B + G + R** into a strict **4-byte binary payload**
- Blasts data over serial at **230,400 baud** via `pyserial`

### 2. 🔗 The Bridge — USB-TTL / ESP8266

Routes the high-speed data stream from PC to STM32. Two options:

| Option | Method | Pros |
|--------|--------|------|
| **Wired** | CH340G or CP2102 USB-to-TTL | Simple, reliable, low latency |
| **Wireless** | ESP8266 as transparent serial bridge | No slip ring needed for data |

### 3. ⚙️ The Engine — STM32F103 Firmware (`main.c`)

- Receives 4-byte packets via **interrupt-driven UART** (`HAL_UART_Receive_IT`)
- Routes data by Channel ID:
  - **Channel 1 / 2** → Clock data into shift registers (selects LED position A or B)
  - **Channel 4** → Sets motor speed via TIM2 PWM (uses the Blue byte slot)
- Sets exact RGB PWM values (0–799) on TIM1 Channels 2, 3, 4

---

## 📦 The 4-Byte Protocol

Instead of sending ASCII text like `"A255128000"` (10 bytes, parsing overhead), this project uses a **raw binary packet**:

```
┌──────────┬──────────────┬───────────────┬──────────────┐
│  Byte 0  │    Byte 1    │    Byte 2     │    Byte 3    │
│ Channel  │ Blue (0–255) │ Green (0–255) │  Red (0–255) │
│  ID      │              │               │              │
└──────────┴──────────────┴───────────────┴──────────────┘
```

| Channel ID | Meaning |
|------------|---------|
| `1` | LED group A — clock shift register A |
| `2` | LED group B — clock shift register B |
| `4` | Motor speed command (Blue byte = speed 0–99) |

### Why This Matters

| Method | Bytes per command | Relative speed |
|--------|------------------|----------------|
| ASCII text (`"A255128000"`) | 10 bytes | 1× |
| **4-byte binary (this project)** | **4 bytes** | **2.5× faster** |

Combined with the 230,400 baud rate (vs a standard 9,600 baud setup), the system is **~72× faster** than a naive text-based approach — essential for video framerates on a spinning display.

---

## 🛠️ Firmware Deep Dive

### Initialization (`main()`)

```c
// Start RGB PWM on TIM1 channels 2, 3, 4
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_2);  // Red
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_3);  // Green
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_4);  // Blue

// Start motor PWM on TIM2 channel 1
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);

// Arm the interrupt listener (4-byte packets)
HAL_UART_Receive_IT(&huart1, rx_data, 4);

// ESC arming sequence: full throttle → zero throttle
motor_start();

// Keep shift register latch HIGH (data retained, active-low latch)
HAL_GPIO_WritePin(GPIOA, register_clc_Pin, GPIO_PIN_SET);
```

### UART Interrupt Callback

Every 4-byte packet triggers this callback. The system routes the command, clocks the shift register, and re-arms the interrupt:

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        int channel = rx_data[0];
        int clr_b   = rx_data[1];   // Blue
        int clr_g   = rx_data[2];   // Green
        int clr_r   = rx_data[3];   // Red

        if (channel == 4) {
            motor_speed(clr_b);     // Blue byte repurposed as speed (0–99)
        }
        if (channel == 1) {
            HAL_GPIO_WritePin(GPIOA, data_a_Pin, GPIO_PIN_RESET);  // Prep register A
        }
        else if (channel == 2) {
            HAL_GPIO_WritePin(GPIOA, data_b_Pin, GPIO_PIN_RESET);  // Prep register B
        }

        // Clock pulse → shifts bit into the selected register
        HAL_GPIO_WritePin(GPIOA, register_clk_Pin, GPIO_PIN_RESET);
        setcolor(clr_r, clr_g, clr_b);   // Set PWM (also acts as clock delay)
        HAL_GPIO_WritePin(GPIOA, register_clk_Pin, GPIO_PIN_SET);

        // Return data lines HIGH (shift registers propagate this 1 forward)
        HAL_GPIO_WritePin(GPIOA, data_a_Pin, GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOA, data_b_Pin, GPIO_PIN_SET);

        // Re-arm interrupt for next packet
        HAL_UART_Receive_IT(&huart1, rx_data, 4);
    }
}
```

### RGB Color Setting

Maps 8-bit color (0–255) to timer compare register range (0–799):

```c
void setcolor(int clr_r, int clr_g, int clr_b) {
    int pulse_r = ((clr_r * 799) / 255);
    int pulse_g = ((clr_g * 799) / 255);
    int pulse_b = ((clr_b * 799) / 255);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_2, pulse_r);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_3, pulse_g);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_4, pulse_b);
}
```

### Motor Speed Control

Maps speed (0–99) to ESC PWM pulse range (100–200 timer ticks = 1ms–2ms):

```c
void motor_speed(int speed) {
    int pulse = ((speed * 100) / 99) + 100;  // 100–200
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pulse);
}
```

---

## ⏱️ Timer Configuration

### TIM1 — RGB LED PWM

| Parameter | Value | Notes |
|-----------|-------|-------|
| Clock | 8 MHz (HSI) | No PLL |
| Prescaler | 0 | Full speed |
| Period (ARR) | 799 | PWM resolution = 800 steps |
| PWM Frequency | 8MHz / 800 = **10 kHz** | Flicker-free at full rotation |
| Channels | 2, 3, 4 | Red, Green, Blue |

### TIM2 — Motor ESC PWM

| Parameter | Value | Notes |
|-----------|-------|-------|
| Clock | 8 MHz (HSI) | |
| Prescaler | 79 | Divides to 100 kHz |
| Period (ARR) | 1999 | 20 ms period |
| PWM Frequency | 100kHz / 2000 = **50 Hz** | Standard ESC signal |
| Pulse range | 100–200 ticks | 1ms–2ms (ESC throttle range) |

---

## 🚀 Setup & Installation

### Prerequisites

- MATLAB / STM32CubeIDE for flashing firmware
- Python 3.x with the following packages:

```bash
pip install opencv-python pyserial numpy
```

### Flashing the Firmware

1. Open `main.c` in **STM32CubeIDE**
2. Connect STM32 via ST-Link programmer
3. Build and flash (`Run → Debug` or `Run → Run`)
4. Confirm the onboard LED (PC13) blinks on each received packet

### Running the Python Streamer

1. Connect your USB-to-TTL adapter and note the COM port
2. Open `rotating.ipynb` in Jupyter:
   ```bash
   jupyter notebook rotating.ipynb
   ```
3. Set your serial port and video source in the notebook:
   ```python
   SERIAL_PORT = "COM3"       # Windows: COMx | Linux: /dev/ttyUSBx
   BAUD_RATE   = 230400
   VIDEO_SOURCE = 0           # 0 = webcam, or path to video file
   LED_COUNT    = 16          # LEDs per arm
   ```
4. Run all cells — the POV display will begin rendering the live video feed

### GPIO Pin Map

| STM32 Pin | Signal | Connected To |
|-----------|--------|--------------|
| PA8 (TIM1_CH1) | — | (reserved) |
| PA9 (TIM1_CH2) | Red PWM | RGB LED Red |
| PA10 (TIM1_CH3) | Green PWM | RGB LED Green |
| PA11 (TIM1_CH4) | Blue PWM | RGB LED Blue |
| PA0 (TIM2_CH1) | Motor PWM | ESC signal input |
| PA (register_clk) | Shift register clock | 74HC595 SRCLK |
| PA (register_clc) | Shift register latch | 74HC595 RCLK (active low) |
| PA (data_a) | Data line A | 74HC595 A serial input |
| PA (data_b) | Data line B | 74HC595 B serial input |
| PC13 | Status LED | Onboard LED (toggles per packet) |

---


## 📁 Project Structure

```
pov-display/
├── firmware/
│   └── main.c              # STM32F103 firmware (HAL — PWM, UART, GPIO)
├── python/
│   └── rotating.ipynb      # Host PC: video → polar → serial stream
└── README.md
```

---

*Built with ❤️ using STM32 HAL, OpenCV, and PySerial.*
