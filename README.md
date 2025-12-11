# MicroPython on Raspberry Pi Pico W Lab Manual

## I. Objective

By the end of this session, participants will be able to set up a **MicroPython Web Server** on the Raspberry Pi Pico W to create and consume a basic RESTful API for on-board sensor reading (temperature) and actuator control (LED).

## II. Prerequisites and Setup

### A. Hardware & Software

  * Raspberry Pi Pico W board.
  * Micro-USB cable for power and data.
  * Computer with a modern IDE (e.g., **Thonny**, recommended for MicroPython development).
  * The Pico W must be running the **latest MicroPython firmware** (or a version supporting the `_thread` module for multitasking).

### B. Installing MicroPython

1.  If not already done, hold the **BOOTSEL** button while connecting the Pico W to your computer.
2.  The Pico W should appear as a drive named **RPI-RP2**.
3.  Drag and drop the official MicroPython UF2 file onto this drive. The drive will disappear, and the Pico W will reboot running MicroPython.
4.  Open **Thonny** and ensure it is configured to talk to the Pico W.

## III. MicroPython REST API Setup

The RESTful API logic will be broken into two main files: `main.py` (the main program and API router) and `web_server.py` (the core HTTP server logic).

### A. Utility Code (`web_server.py`)

This file contains the simple socket server and should be uploaded to the Pico W first.

```python
# web_server.py
# Simple function to create an HTTP response string
def http_response(html):
    return "HTTP/1.1 200 OK\r\nContent-type: text/html\r\n\r\n" + html

# Function to serve the client connection
def serve_client(client_socket, temp_c):
    # Read client request
    request = client_socket.recv(1024).decode('utf-8')
    # Use a basic split to find the requested path, e.g., 'GET /temp HTTP/1.1'
    try:
        path = request.split(' ')[1]
    except IndexError:
        path = '/'

    # Set up LED and Temperature variables
    global led_state
    led_state = -1 # Sentinel value, changed by the main logic

    # --- Routing Logic ---
    if path == '/temp':
        response_html = f"<h1>Pico W Temperature</h1><p>Temperature: {temp_c:.2f} &deg;C</p>"
        response = http_response(response_html)
        led_state = 0  # Turn LED OFF on status read
    elif path == '/led/1':
        response_html = "<h1>LED Control</h1><p>LED is ON</p>"
        response = http_response(response_html)
        led_state = 1
    elif path == '/led/0':
        response_html = "<h1>LED Control</h1><p>LED is OFF</p>"
        response = http_response(response_html)
        led_state = 0
    else: # Default endpoint ('/')
        response_html = "<h1>Pico W API Root</h1><p>Available Endpoints: /temp, /led/1, /led/0</p>"
        response = http_response(response_html)
        led_state = -1 # No change

    # Send response and close connection
    client_socket.send(response.encode('utf-8'))
    client_socket.close()

    return led_state

```

### B. Main Application Logic (`main.py`)

This file connects to WiFi, sets up the peripherals, and starts the server loop.

```python
# main.py
import network
import socket
import machine
import time
from web_server import serve_client

# WiFi Credentials (CHANGE THESE!)
SSID = "YOUR_WIFI_SSID"
PASSWORD = "YOUR_WIFI_PASSWORD"

# Pico W Peripherals
onboard_led = machine.Pin("LED", machine.Pin.OUT)
temp_sensor = machine.ADC(4)
conversion_factor = 3.3 / (65535)

def wifi_connect():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(SSID, PASSWORD)

    # Wait for connect or fail
    max_wait = 10
    while max_wait > 0:
        if wlan.status() < 0 or wlan.status() >= 3:
            break
        max_wait -= 1
        print('waiting for connection...')
        time.sleep(1)

    if wlan.status() != 3:
        raise RuntimeError('Network connection failed')
    else:
        status = wlan.ifconfig()
        print('Connected! IP Address:', status[0])
        return status[0]

def read_temperature():
    # Pico W internal temperature sensor reading logic
    voltage = temp_sensor.read_u16() * conversion_factor
    temperature = 27 - (voltage - 0.706) / 0.001721
    return temperature

# --- Main Program ---
try:
    ip_address = wifi_connect()
except Exception as e:
    print(f"Error connecting to WiFi: {e}")
    # Flash LED to indicate failure
    for i in range(5):
        onboard_led.toggle()
        time.sleep(0.2)
    raise

# Create Socket
addr = socket.getaddrinfo('0.0.0.0', 80)[0][-1]
s = socket.socket()
s.bind(addr)
s.listen(1)

print('Listening on:', addr)

# Server Loop
while True:
    try:
        # Check for client connection
        client_conn, client_addr = s.accept()
        print('Client connected from', client_addr)

        # Get current temperature before serving
        current_temp_c = read_temperature()

        # Serve the request and get the desired LED state
        new_led_state = serve_client(client_conn, current_temp_c)

        # Apply actuator change if requested
        if new_led_state == 0:
            onboard_led.value(0) # LED OFF
        elif new_led_state == 1:
            onboard_led.value(1) # LED ON

    except OSError as e:
        # Catch connection errors and continue loop
        if e.args[0] == 110: # ETIMEDOUT (non-blocking socket)
            pass
        else:
            print('Connection error:', e)
            continue
```

## IV. Lab Exercise: Testing the RESTful API

### 1\. Upload Code

  * Upload both `web_server.py` and `main.py` to the Raspberry Pi Pico W using Thonny.
  * Ensure you have updated the `SSID` and `PASSWORD` variables in `main.py` with your local WiFi credentials.

### 2\. Run and Connect

  * Run `main.py` on the Pico W.
  * Wait for the "Connected\! IP Address:" message in the Thonny shell. Note down the displayed IP address (e.g., `192.168.1.100`).

### 3\. Test Endpoints

Use a web browser or a tool like Postman to interact with your Pico W using its IP address.

| Endpoint (URL) | HTTP Method | Expected Action | Expected Output |
| :--- | :--- | :--- | :--- |
| `http://<IP_ADDRESS>/` | GET | Root/Info | Displays available endpoints. |
| `http://<IP_ADDRESS>/temp` | GET | Reads temperature sensor. | Displays on-board temperature in a simple HTML page. **LED turns OFF.** |
| `http://<IP_ADDRESS>/led/1` | GET | Turns on the Pico W's on-board LED. | Displays "LED is ON" page. **LED should turn ON.** |
| `http://<IP_ADDRESS>/led/0` | GET | Turns off the Pico W's on-board LED. | Displays "LED is OFF" page. **LED should turn OFF.** |

-----

Would you like me to adapt the **Lab Assignment** from the Arduino manual (adding accelerometer, gyroscope, and buzzer functionality) into a corresponding MicroPython challenge?
