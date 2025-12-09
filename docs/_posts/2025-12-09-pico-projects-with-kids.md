---
layout: post
title: "Raspberry Pi Pico Projects with Kids"
date: 2025-12-09 00:00:00 -0000
categories:
tags: [raspberrypi,pico,micropython,electronics,projects]
---

I've been working on a few small electronics projects with my children recently using the Raspberry Pi Pico W. The Pico W is a great choice for introducing kids to electronics and programming - it's inexpensive, runs MicroPython, and the built-in Wi-Fi opens up possibilities for fun IoT projects.

This post documents three projects we built together, each one building on concepts from the previous. We started with simple LED control, moved on to reading sensor data, and finished by combining both into a basic thermostat system.

## What You'll Need

For all three projects, you'll need:

- Raspberry Pi Pico W
- Micro USB cable
- Breadboard and jumper wires
- LED and 330Ω resistor (Projects 1 and 3)
- TMP36 temperature sensor (Projects 2 and 3)

You'll also need the Thonny IDE installed on your computer for uploading the MicroPython code to the Pico.

## Project 1: Web-Controlled LED

Our first project was a simple web server that lets you turn an LED on and off from any device on your local network. This introduces the basics of GPIO output and web servers.

### Circuit Diagram

```
Pico W                       LED Circuit
┌─────────────┐
│             │
│        GP15 ├────────┬────[330Ω]────[LED>]────┐
│             │        │                        │
│         GND ├────────┴────────────────────────┘
│             │
└─────────────┘
```

### How It Works

The Pico W connects to your Wi-Fi network and starts a simple web server on port 80. When you visit the Pico's IP address in a browser, you see two buttons. Clicking them sends requests to `/light/on` or `/light/off`, which the Pico interprets to control the LED.

### Code

```python
import network
import socket
import time
from machine import Pin

# --- Configuration ---
ssid = 'WIFI_SSID'
password = 'WIFI_PASSWORD'

# Set up the LED pin
led = Pin(15, Pin.OUT)

# --- Connect to Wi-Fi ---
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(ssid, password)

# Wait for connection
max_wait = 10
while max_wait > 0:
    if wlan.status() < 0 or wlan.status() >= 3:
        break
    max_wait -= 1
    print('Waiting for connection...')
    time.sleep(1)

# Handle connection error
if wlan.status() != 3:
    raise RuntimeError('Network connection failed')
else:
    print('Connected')
    status = wlan.ifconfig()
    print('IP Address: ' + status[0])

# --- HTML Template ---
# This is the webpage the Pico will send to your browser
html = """<!DOCTYPE html>
<html>
<head> <title>Pico W Control</title> </head>
<body> <h1>Pico W LED Control</h1>
<p><a href="/light/on"><button style="font-size: 24px;">Turn ON</button></a></p>
<p><a href="/light/off"><button style="font-size: 24px;">Turn OFF</button></a></p>
</body>
</html>
"""

# --- Start Web Server ---
addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]
s = socket.socket()
s.bind(addr)
s.listen(1)

print('Listening on', addr)

while True:
    try:
        cl, addr = s.accept()
        print('Client connected from', addr)

        # Get the request
        request = cl.recv(1024)
        request = str(request)

        # Look for the command in the request
        led_on = request.find('/light/on')
        led_off = request.find('/light/off')

        if led_on == 6:
            print("LED ON")
            led.value(1)
        if led_off == 6:
            print("LED OFF")
            led.value(0)

        # Send the HTML response
        cl.send('HTTP/1.0 200 OK\r\nContent-type: text/html\r\n\r\n')
        cl.send(html)
        cl.close()

    except OSError as e:
        cl.close()
        print('Connection closed')
```

### What We Learned

- How to set up a GPIO pin as an output
- Basic web server concepts (sockets, HTTP requests)
- How URLs can carry commands (`/light/on`, `/light/off`)

## Project 2: Web Temperature Monitor

For the second project, we connected a TMP36 temperature sensor and displayed the reading on a webpage that auto-refreshes every 5 seconds.

### Circuit Diagram

```
Pico W                       TMP36 Sensor
┌─────────────┐              ┌───────────┐
│             │              │  TMP36    │
│        3.3V ├──────────────┤ VCC       │
│             │              │           │
│  GP26 (ADC) ├──────────────┤ OUT       │
│             │              │           │
│         GND ├──────────────┤ GND       │
│             │              └───────────┘
└─────────────┘

TMP36 Pinout (flat side facing you):
  ┌─────┐
  │     │
 ─┴─┬─┬─┴─
   1 2 3
   │ │ │
   │ │ └── GND
   │ └──── OUT (signal)
   └────── VCC (3.3V)
```

### How It Works

The TMP36 is an analog temperature sensor. The Pico reads the voltage on its ADC pin, converts it to a temperature value using the TMP36's formula, and displays it on a styled webpage. The page automatically refreshes to show updated readings.

### Code

```python
import machine
import socket
import time
import network

# --- WI-FI SETUP ---
ssid = 'WIFI_SSID'
password = 'WIFI_PASSWORD'

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(ssid, password)

# Wait for connection
max_wait = 10
while max_wait > 0:
    if wlan.status() < 0 or wlan.status() >= 3:
        break
    max_wait -= 1
    print('Waiting for connection...')
    time.sleep(1)

if wlan.status() != 3:
    raise RuntimeError('Network connection failed')
else:
    print('Connected')
    status = wlan.ifconfig()
    print('IP Address: ' + status[0])

# --- SENSOR SETUP ---
# We are using GP26 (ADC0) for the signal pin
sensor_pin = machine.ADC(26)

def read_temp():
    # 1. Read the raw ADC value (0 to 65535)
    raw_value = sensor_pin.read_u16()

    # 2. Convert to Voltage (3.3V is the reference voltage)
    voltage = raw_value * (3.3 / 65535)

    # 3. Convert Voltage to Celsius (TMP36 formula)
    # (Voltage - 0.5V offset) * 100
    temperature = (voltage - 0.5) * 100

    return temperature

# --- WEB SERVER ---
addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(addr)
s.listen(1)

print('Listening on', addr)

while True:
    try:
        cl, addr = s.accept()
        print('Client connected from', addr)
        request = cl.recv(1024)

        # Get the reading
        temp_c = read_temp()

        # Optional: Convert to Fahrenheit if you prefer
        # temp_f = (temp_c * 9/5) + 32

        html = f"""<!DOCTYPE html>
            <html>
            <head>
                <title>Room Temp</title>
                <meta http-equiv="refresh" content="5">
                <style>
                    body {{ font-family: sans-serif; text-align: center; margin-top: 50px; }}
                    h1 {{ color: #333; }}
                    .temp {{ font-size: 48px; color: #007bff; font-weight: bold; }}
                </style>
            </head>
            <body>
                <h1>Current Temperature</h1>
                <p class="temp">{temp_c:.1f} &deg;C</p>
                <p>Sensor: TMP36</p>
            </body>
            </html>
            """

        cl.send('HTTP/1.0 200 OK\r\nContent-type: text/html\r\n\r\n')
        cl.send(html)
        cl.close()

    except OSError as e:
        cl.close()
        print('Connection closed')
```

### What We Learned

- How analog-to-digital conversion (ADC) works
- Reading sensor datasheets to understand voltage-to-temperature conversion
- Using f-strings to embed dynamic data in HTML
- Auto-refreshing webpages with the `<meta http-equiv="refresh">` tag

## Project 3: Smart Thermostat

The final project combines everything: reading the temperature sensor and automatically controlling the LED based on a threshold. When the temperature exceeds the limit, the LED turns on (simulating a cooling system activating).

### Circuit Diagram

```
Pico W                       Components
┌─────────────┐
│             │              ┌───────────┐
│        3.3V ├──────────────┤ VCC       │
│             │              │  TMP36    │
│  GP26 (ADC) ├──────────────┤ OUT       │
│             │              │           │
│         GND ├──────┬───────┤ GND       │
│             │      │       └───────────┘
│             │      │
│        GP15 ├──────┼────[330Ω]────[LED>]────┐
│             │      │                        │
│         GND ├──────┴────────────────────────┘
│             │
└─────────────┘
```

### How It Works

This project introduces conditional logic - the "brain" of any automation system. The Pico continuously reads the temperature and compares it to a threshold value. If the temperature is too high, it turns on the LED and displays "COOLING ACTIVE" on the webpage. Otherwise, it shows "Standby".

### Code

```python
import machine
import socket
import time
import network

# --- CONFIGURATION ---
ssid = 'WIFI_SSID'
password = 'WIFI_PASSWORD'
TEMP_THRESHOLD = 27.0  # The target temperature in Celsius

# --- HARDWARE SETUP ---
# LED on GP15 (Pin 20)
led = machine.Pin(15, machine.Pin.OUT)

# TMP36 on GP26 (Pin 31)
sensor_pin = machine.ADC(26)

def read_temp():
    raw_value = sensor_pin.read_u16()
    voltage = raw_value * (3.3 / 65535)
    temperature = (voltage - 0.5) * 100
    return temperature

# --- WIFI CONNECT ---
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(ssid, password)

max_wait = 10
while max_wait > 0:
    if wlan.status() < 0 or wlan.status() >= 3:
        break
    max_wait -= 1
    print('Waiting for connection...')
    time.sleep(1)

if wlan.status() != 3:
    raise RuntimeError('Network connection failed')
else:
    print('Connected:', wlan.ifconfig()[0])

# --- WEB SERVER ---
addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(addr)
s.listen(1)

print('Listening on', addr)

while True:
    try:
        cl, addr = s.accept()
        print('Client connected')
        request = cl.recv(1024)

        # 1. Get the current Reality
        current_temp = read_temp()

        # 2. Make the Decision (The "Thermostat" Logic)
        if current_temp > TEMP_THRESHOLD:
            led.value(1) # Turn LED ON
            state = "COOLING ACTIVE"
            color = "red" # For the web text
        else:
            led.value(0) # Turn LED OFF
            state = "Standby"
            color = "green"

        # 3. Create the HTML
        html = f"""<!DOCTYPE html>
            <html>
            <head>
                <title>Pico Thermostat</title>
                <meta http-equiv="refresh" content="3">
                <style>
                    body {{ font-family: sans-serif; text-align: center; margin-top: 50px; }}
                    .temp {{ font-size: 48px; font-weight: bold; }}
                    .status {{ font-size: 24px; color: {color}; border: 2px solid {color}; padding: 10px; display: inline-block; }}
                </style>
            </head>
            <body>
                <h1>Home Thermostat</h1>
                <p>Current Temp:</p>
                <p class="temp">{current_temp:.1f} &deg;C</p>
                <p>Target Limit: {TEMP_THRESHOLD} &deg;C</p>
                <br>
                <div class="status">System: {state}</div>
            </body>
            </html>
            """

        cl.send('HTTP/1.0 200 OK\r\nContent-type: text/html\r\n\r\n')
        cl.send(html)
        cl.close()

    except OSError as e:
        cl.close()
        print('Connection closed')
```

### What We Learned

- Conditional logic (`if`/`else` statements) for automation
- Combining inputs (sensors) and outputs (LEDs) in one system
- How real-world systems like thermostats work
- The concept of feedback loops in control systems

## Wrapping Up

These three projects provided a great introduction to electronics and programming. Starting with simple LED control, progressing to reading sensors, and finally combining both with decision logic gave a solid foundation in:

- **Hardware**: GPIO, ADC, breadboard wiring
- **Networking**: Wi-Fi, web servers, HTTP
- **Programming**: Variables, functions, conditionals, loops

The Raspberry Pi Pico W is an excellent platform for learning - the projects are achievable in a single session, and seeing a webpage control physical hardware or display real sensor data is genuinely exciting for kids (and adults!).

### Next Steps

Some ideas for extending these projects:

- Add multiple LEDs or RGB LEDs for different temperature ranges
- Log temperature data over time
- Add a physical button to override the thermostat
- Use a relay to control an actual fan instead of an LED
- Add email or push notifications when temperature exceeds threshold
