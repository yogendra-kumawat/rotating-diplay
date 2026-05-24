<h1 align="center">High-Speed POV (Persistence of Vision) Video Display</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/STM32-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white" alt="STM32">
  <img src="https://img.shields.io/badge/ESP8266-000000?style=for-the-badge&logo=espressif&logoColor=white" alt="ESP8266">
  <img src="https://img.shields.io/badge/OpenCV-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white" alt="OpenCV">
  <img src="https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white" alt="C">
</p>

<p>
  A custom-built, high-performance Persistence of Vision (POV) display capable of streaming live video frames from a PC directly to a spinning LED array. 
</p>
<p>
  This project uses <strong>OpenCV</strong> to parse video frames into polar coordinates, packs the pixel data into a highly optimized 4-byte raw binary payload, and transmits it at <strong>230,400 baud</strong> to an <strong>STM32F103</strong> microcontroller. The STM32 utilizes its hardware timers to generate precise PWM signals for the RGB LEDs and the BLDC motor.
</p>

<hr>
<h2> hardware structure</h2>
<img width="1600" height="1204" alt="image" src="https://github.com/user-attachments/assets/59e1821d-1554-4ed2-8585-ad54104596aa" />
<h2>basically the project is undergoing</h2>
<img width="400" height="225" alt="WhatsApp Video 2026-05-24 at 7 29 12 PM" src="https://github.com/user-attachments/assets/bce04942-c39b-4d1b-9365-3711640be53b" />

<h2>🏗️ System Architecture</h2>


<h3>1. The Brain (Python / PC)</h3>
<ul>
  <li>Reads video frames using <code>cv2.VideoCapture</code>.</li>
  <li>Maps the flat image into a 360-degree circle (16 LEDs per ray).</li>
  <li>Extracts exact BGR intensity values.</li>
  <li>Packs the Channel ID and Colors into a strict 4-byte format.</li>
  <li>Blasts the data over serial via <code>pyserial</code>.</li>
</ul>

<h3>2. The Bridge (USB-to-TTL / ESP8266)</h3>
<ul>
  <li>Routes the 230,400 baud data stream from the PC to the STM32.</li>
  <li>Can be implemented via a direct CH340/CP2102 adapter or wirelessly via an ESP8266 running as a transparent serial bridge.</li>
</ul>

<h3>3. The Engine (STM32F103 Firmware)</h3>
<ul>
  <li>Receives the 4-byte packets via an interrupt-driven UART receiver (<code>HAL_UART_Receive_IT</code>).</li>
  <li>Routes the data based on Channel ID:
    <ul>
      <li><strong>Channel 1, 2, 3:</strong> Updates shift registers and sets the exact Pulse Width Modulation (0-799) for the Red, Green, and Blue LED channels.</li>
      <li><strong>Channel 4:</strong> Hijacks the blue byte slot to set the physical rotation speed of the DC motor.</li>
    </ul>
  </li>
</ul>

<hr>

<h2>🚀 The 4-Byte Optimization</h2>
<p>
  To achieve video framerates on a spinning display, data transmission must be microscopic. This project uses a custom binary packing strategy:
</p>
<p>
  Instead of sending text like <code>"A255128000"</code>, the data is packed into a raw <code>uint8_t</code> array:
</p>
<pre>
Byte 0: Channel/Target ID (0, 1, 2 for LEDs, 4 for Motor)
Byte 1: Blue Intensity (0-255)
Byte 2: Green Intensity (0-255)
Byte 3: Red Intensity (0-255)
</pre>
<p>
  <strong>The Result:</strong> Data size is reduced by 66%, and parsing overhead on the STM32 is completely eliminated. Combined with the 230,400 baud rate, the system updates 72x faster than a standard 9600-baud text-based setup.
</p>

<hr>

<h2>🛠️ Hardware Requirements</h2>
<ul>
  <li>STM32F103 Microcontroller</li>
  <li>High-Speed USB-to-TTL Adapter (CH340G or CP2102) OR ESP8266 (ESP-12F)</li>
  <li>RGB LEDs & Shift Registers (74HC595 or similar)</li>
  <li>BLDC Motor with PWM Speed Controller</li>
  <li>Slip Ring (to pass power/data to the spinning arm)</li>
  <li>3.3V / 5V Power Supply</li>
</ul>

<hr>

<h2>⚠️ Known Limitations & Future Upgrades</h2>
<ul>
  <li><strong>Physical Synchronization:</strong> Currently, the system runs open-loop. Adding a <strong>Hall Effect Sensor</strong> or an IR Optocoupler to the spinning arm will allow the STM32 to lock the "Angle 0" data packet to the physical top of the circle, creating a perfectly stable image.</li>
  <li><strong>Direct Wi-Fi Streaming:</strong> Bypassing the STM32 entirely and wiring the LEDs directly to an ESP8266 running at 160MHz would allow raw UDP video streaming over a local network, vastly increasing the maximum framerate.</li>
</ul>
