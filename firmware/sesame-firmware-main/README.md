# esame Robot Firmware

This document provides technical information on the firmware architecture, control logic, and hardware abstraction layers used in the Sesame Robot.

> [!NOTE]
> The firmware is now organized into a modular structure with a main entry point and specialized header files for bitmaps, movement, and web assets. This makes customization much easier and the codebase cleaner.

## Table of Contents

- [How to Flash the Firmware](#how-to-flash-the-firmware)
- [Network Configuration](#network-configuration--connectivity)
- [API Reference](#api-reference)
  - [Legacy Web Endpoints](#legacy-web-endpoints)
  - [JSON API](#json-api-recommended-for-network-clients)
  - [Python Examples](#python-api-example)
  - [JavaScript Examples](#nodejsjavascript-example)
- [Advanced Integration Examples](#advanced-integration-examples)
  - [Voice Assistant Integration](#voice-assistant-integration)
  - [Home Automation](#home-automation-integration)
- [Idle Animation System](#idle-animation-system)
- [Firmware Architecture](#firmware-architecture)
- [Technical Implementation](#technical-implementation-overview)
- [Asset Pipeline &amp; Face Customization](#asset-pipeline--face-customization)
- [Hardware Abstraction Layer](#hardware-abstraction-layer-hal)

## How to Flash the Firmware

> [!NOTE]
> **Sesame Build Kit Users:** Your Distro Board V3 (and earlier V2) kits come pre-flashed with the latest firmware. You only need to flash firmware if you want to customize it or update to a newer version.

### Prerequisites

1. **Arduino IDE** (version 2.0 or higher recommended)
2. **ESP32 Board Support**: Install via Arduino IDE's Board Manager
   - Open Arduino IDE
   - Go to **File → Preferences**
   - Add to "Additional Board Manager URLs": `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - Go to **Tools → Board → Boards Manager**
   - Search for "ESP32" and install "ESP32 by Espressif Systems" (v2.0.0 or higher)
3. **Required Libraries** (install via Library Manager):

- `Adafruit PWM Servo Driver Library`
- `Adafruit SSD1306`
- `Adafruit GFX Library`

> [!IMPORTANT]
> The firmware expects a PCA9685 servo driver at I2C address `0x40`. The SSD1306 display shares the same SDA/SCL bus and is detected at `0x3C` or `0x3D`.

### Flashing Steps

1. **Connect your board** via USB to your computer
2. **Open the firmware**:

   - Open [sesame-firmware-main.ino](sesame-firmware-main.ino) in Arduino IDE
   - Make sure you have it in a folder with the same name.
   - Also include all of the .h header files.
3. **Select your board**:

   - Go to **Tools → Board**
   - For Lolin S2 Mini: Select "LOLIN S2 Mini"
   - For Sesame Distro Board V1: Select "ESP32 Dev Module"
   - For Sesame Distro Board V2 or V3: Select "ESP32S3 Dev Module"
4. **Configure board settings**:

   - **For Lolin S2 Mini**:
     - **Upload Speed**: 921600
     - **USB CDC On Boot**: "Enabled"
     - **Partition Scheme**: "Default 4MB with spiffs"
   - **For Distro Board V2 or V3 (ESP32-S3)**:
     - **USB CDC On Boot**: "Enabled" (Required for Serial Monitor)
     - **Flash Mode**: "QIO 80MHz"
     - **Partition Scheme**: "Default 4MB with spiffs"
5. **Select the correct port**:

   - Go to **Tools → Port** and select your ESP32's COM port
6. **Choose your board configuration** in the code:

   - Open [sesame-firmware-main.ino](sesame-firmware-main.ino)
   - The firmware uses `Wire.begin()` and the selected board's default I2C pins.
   - Connect servos 0-7 to PCA9685 channels 0-7. No individual ESP32 servo GPIO configuration is required.
7. **(Optional) Configure network mode**:

   - If you want the robot to connect to your WiFi network (for API access and remote control), edit the network configuration section (around line 17-22):

   ```cpp
   #define NETWORK_SSID "YourNetworkName"  // Your WiFi network name
   #define NETWORK_PASS "YourPassword"     // Your WiFi password
   #define ENABLE_NETWORK_MODE true        // Set to true to enable
   ```

   - Leave `ENABLE_NETWORK_MODE false` to use Access Point mode only
8. **Upload the firmware**:

   - Click the **Upload** button (→) in Arduino IDE
   - Wait for compilation and upload to complete
9. **Test the connection**:

   - Open Serial Monitor (**Tools → Serial Monitor**, set baud rate to 115200)
   - Reset the board - you should see startup messages and network information
   - The serial monitor provides individual motor control for testing and troubleshooting
   - **If using AP mode only**: Connect to the "Sesame-Controller-BETA" WiFi network (password: `12345678`) and navigate to any website
   - **If using network mode**: Look for the "Connected to network!" message in Serial Monitor, then access via `http://sesame-robot.local` or the displayed IP address

### Troubleshooting

- **Linux USB Permissions**: If your port isn't showing up or you get "Permission Denied", you likely need to add your user to the `dialout` group: `sudo usermod -a -G dialout $USER`. Log out and back in for changes to take effect.
- **Upload fails**: Try holding the BOOT button while uploading, or try a different USB cable. For S3-based boards, the "USB CDC On Boot" setting is critical for finding the port after a reset.
- **Port not found**: Install the appropriate USB drivers (CP210x for S2 Mini, CH340 for some ESP32 boards). Windows 10/11 usually includes these.
- **Robot not moving**: Check power supply and servo connections; increase `motorCurrentDelay` in web settings if brownouts occur
- **Can't connect to network**:
  - Verify `ENABLE_NETWORK_MODE` is set to `true` and SSID/password are correct
  - Check Serial Monitor for connection status - look for "Connected to network!" or error messages
  - Ensure your WiFi network is 2.4GHz (ESP32 does not support 5GHz)
  - Try increasing the connection timeout in the code (currently 10 seconds / 20 attempts)
- **mDNS hostname not resolving**:
  - On Windows, install [Bonjour Print Services](https://support.apple.com/kb/DL999)
  - On Linux, ensure `avahi-daemon` is installed and running
  - Try accessing via IP address instead (check Serial Monitor for the assigned IP)
  - Some routers block mDNS traffic - check router settings or use IP address
- **API commands not working**:
  - Ensure you're sending POST requests to `/api/command` (not GET)
  - Verify `Content-Type: application/json` header is set
  - Check Serial Monitor for error messages and parsed command output
  - Test with a simple cURL command first to verify connectivity

## Firmware Architecture

The firmware is split into several key files to keep the logic organized and assets easy to manage:

- **[sesame-firmware-main.ino](sesame-firmware-main.ino)**: The main entry point containing the `setup()`, `loop()`, core system logic, network configuration, and API endpoints.
- **[face-bitmaps.h](face-bitmaps.h)**: A dedicated header for OLED face macros and raw bitmap data. Now includes extensive conversational faces for voice assistant integration.
- **[movement-sequences.h](movement-sequences.h)**: Definitions for all procedural movement and pose animations.
- **[captive-portal.h](captive-portal.h)**: Contains the HTML, CSS, and JS for the web-based remote control interface.

## Technical Implementation Overview

The firmware is built on the Arduino-ESP32 framework. It runs on a single-core event loop and delegates servo PWM generation to a PCA9685 driver.

### PWM & Servo Kinematics

- **External PWM**: A PCA9685 at I2C address `0x40` generates all eight 50 Hz servo signals, avoiding ESP32 PWM timer allocation and direct servo GPIO wiring.
- **Pulse Width Mapping**: Leg servos are mapped from degrees (0-180) to microseconds (500us to 2500us), matching the tested PCA9685 Spider sketch. Edit `SERVO_MIN_US` and `SERVO_MAX_US` near the top of `sesame-firmware-main.ino` for a different servo range.
- **Staggered Activation**: To prevent VCC rail collapse (brownout) caused by simultaneous inductive loads, the `setServoAngle` helper introduces a mandatory `motorCurrentDelay` (default 20ms) between sequential pulses. This delay should be tweaked to your power setup. If you have a strong dedicated power supply you can try setting it to zero. It can also be changed while running through the AP controller settings menu.

### Communication & Networking Stack

- **Dual-Mode WiFi**: The ESP32 can simultaneously operate in both Access Point mode (for direct connections) and Station mode (connected to an existing network) using `WiFi.mode(WIFI_AP_STA)`.
- **SoftAP & Captive Portal**: The ESP32 initializes an Access Point using `WiFi.softAP()`. A `DNSServer` listens on UDP Port 53, using a wildcard "*" redirect to map all DNS queries to the internal gateway (`192.168.4.1`).
- **mDNS Service Discovery**: The firmware broadcasts `sesame-robot.local` via Multicast DNS (mDNS) using the `ESPmDNS` library, allowing network discovery without hardcoded IP addresses.
- **RESTful API Surface**: The `WebServer` handles both legacy URL-parameter endpoints and modern JSON-based API:
  - `/cmd?go=[dir]`: Legacy movement control
  - `/cmd?pose=[name]`: Legacy pose triggers
  - `/api/status`: JSON status endpoint (GET)
  - `/api/command`: JSON command endpoint (POST) - supports face-only updates and combined face+movement commands
  - `/getSettings` / `/setSettings`: Parameter configuration
- **Face-Only Command Support**: The `/api/command` endpoint intelligently detects face-only requests (no `command` field) and updates the display without triggering movement animations.
- **Non-Blocking Control Flow**: Instead of `delay()`, the firmware uses a custom `pressingCheck(String cmd, int ms)` function. This function polls `server.handleClient()` and `dnsServer.processNextRequest()` during animation frames, allowing for real-time interruptibility (e.g., immediate stop on button release). This pressingCheck protocol can be used for motion commands like walking to play each motion only when the button is held.

### Display & Graphics Subsystem

- **Shared I2C Bus**: The PCA9685 (`0x40`) and SSD1306 (`0x3C` or `0x3D`) share the ESP32 hardware I2C bus.
- **Memory Management (`PROGMEM`)**: Large 128x64 bitmap arrays (1024 bytes per frame) are stored in Flash memory using the `PROGMEM` attribute.
- **Macro-Based Asset Management**: The firmware uses a `FACE_LIST` macro in [face-bitmaps.h](face-bitmaps.h) to automatically register and handle new faces, reducing the boilerplate required when adding animations.
- **Rendering Pipeline**: The `updateAnimatedFace()` function manages frame rates and sequence looping outside of the main movement logic to ensure smooth visual feedback even during complex movements.
- **Dynamic WiFi Info Overlay**: The `updateWifiInfoScroll()` function composites scrolling connection information over the face bitmap during the first 30 seconds of operation (before first input), drawing the face as background with a black bar and white text overlay for readability.
- **Idle Animation System**: Implements realistic idle behavior with randomized blinking (including double-blinks) and boomerang face animations, triggered automatically when no input is detected.

## Prerequisites & Development Environment

- **Board Support**: ESP32 by Espressif Systems (v2.0.0+ recommended). Lolin S2 Mini and ESP32-WROOM32 DevKitC are best supported.
- **Libraries**:
  - `Adafruit_PWMServoDriver`: PCA9685 servo PWM control.
  - `Adafruit_SSD1306` & `Adafruit_GFX`: Buffer-based OLED rendering.
  - `ESPmDNS`: mDNS service discovery (included with ESP32 board support).
  - `DNSServer`: Captive portal DNS redirection (included with ESP32 board support).
  - `WebServer`: HTTP server implementation (included with ESP32 board support).
- **Tooling**: Arduino IDE 2.0+ recommended.

## Network Configuration & Connectivity

The firmware now supports **dual-mode WiFi operation**, allowing the robot to simultaneously act as an Access Point (for direct connections) and connect to your existing WiFi network (for integration with other devices and remote control).

### Access Point Mode (Default)

By default, the robot creates its own WiFi network:

- **SSID**: `Sesame-Controller-BETA`
- **Password**: `12345678` (Some devices may require you to change this if the default isn't connecting)
- **IP Address**: `192.168.4.1`

Connect to this network and navigate to any website to access the captive portal control interface. If you're having trouble connecting, try changing the password to something custom in the code's `#define AP_PASS` section.

### Network Mode (Optional)

To connect the robot to your home or office WiFi network:

1. **Enable Network Mode** in [sesame-firmware-main.ino](sesame-firmware-main.ino):

   ```cpp
   #define NETWORK_SSID "YourNetworkName"  // Your WiFi network name
   #define NETWORK_PASS "YourPassword"     // Your WiFi password
   #define ENABLE_NETWORK_MODE true        // Set to true to enable
   ```
2. **Flash the updated firmware** to your robot.
3. **Access via multiple methods**:

   - **mDNS Hostname**: `http://sesame-robot.local` (works on most devices)
   - **Network IP**: Check Serial Monitor at 115200 baud for the assigned IP address
   - **Still accessible via AP**: The robot maintains its Access Point even when connected to your network

### mDNS Discovery

The firmware includes an mDNS responder that broadcasts the hostname `sesame-robot.local` on your local network. This allows you to access the robot without knowing its IP address:

```bash
# Access from browser
http://sesame-robot.local

# Ping test
ping sesame-robot.local
```

**Note**: mDNS works natively on:

- macOS and iOS devices
- Linux with Avahi installed
- Windows 10+ (may require Bonjour service)

### Security Considerations

The firmware is designed for local network use and does not include authentication by default. When enabling network mode:

- **Local Network Only**: The robot responds to requests from any device on your network. Do not expose it to the internet without proper security measures.
- **No HTTPS**: Communication is unencrypted HTTP. Avoid transmitting sensitive data through the robot's API.
- **Access Point Password**: The default AP password is `12345678`. For production use, change `AP_PASS` in the firmware to a stronger password.
- **Trusted Networks**: Only connect your robot to trusted WiFi networks.

For enhanced security:

1. Change the default AP password to a strong, unique password
2. Use network segmentation (IoT VLAN) to isolate the robot from critical devices
3. Implement firewall rules to restrict access to specific IP addresses
4. Consider adding authentication headers in a custom firmware fork for production deployments

### WiFi Information Display

The robot's OLED screen now features an intelligent WiFi info system:

- **First 30 seconds**: If no input is received, WiFi connection information scrolls across the top of the display
- **After first input**: WiFi info disappears to show only faces
- **Dual-mode info**: When connected to both AP and network, displays both connection details

The scrolling text includes:

- Access Point SSID and IP
- Network name and IP (if connected)
- mDNS hostname
- Captive portal instructions

## API Reference

The firmware exposes both legacy web endpoints and a modern JSON-based API for programmatic control.

### Legacy Web Endpoints

These endpoints use URL parameters and are primarily used by the web interface:

#### Movement Control

```http
GET /cmd?go=forward
GET /cmd?go=backward
GET /cmd?go=left
GET /cmd?go=right
GET /cmd?stop
```

#### Pose Control

```http
GET /cmd?pose=wave
GET /cmd?pose=dance
GET /cmd?pose=rest
GET /cmd?pose=stand
# ... (see movement-sequences.h for all available poses)
```

#### Individual Motor Control

```http
GET /cmd?motor=1&value=90
# motor: 1-8 (motor number)
# value: 0-180 (angle in degrees)
```

#### Settings Management

```http
GET /getSettings
# Returns: {"frameDelay":100,"walkCycles":10,"motorCurrentDelay":20,"faceFps":8}

GET /setSettings?frameDelay=120&walkCycles=15&motorCurrentDelay=25&faceFps=10
```

### JSON API (Recommended for Network Clients)

The JSON API is designed for programmatic control from external devices, Python scripts, and IoT integrations.

#### Get Robot Status

```http
GET /api/status
```

**Response:**

```json
{
  "currentCommand": "forward",
  "currentFace": "walk",
  "networkConnected": true,
  "apIP": "192.168.4.1",
  "networkIP": "192.168.1.100"
}
```

#### Send Commands

```http
POST /api/command
Content-Type: application/json

{
  "command": "forward",
  "face": "walk"
}
```

**Response:**

```json
{
  "status": "ok",
  "message": "Command executed"
}
```

#### Face-Only Updates

Send a face change without triggering movement:

```http
POST /api/command
Content-Type: application/json

{
  "face": "happy"
}
```

**Response:**

```json
{
  "status": "ok",
  "message": "Face updated"
}
```

#### Stop Command

```http
POST /api/command
Content-Type: application/json

{
  "command": "stop"
}
```

### Available Commands

**Movement Commands:**

- `forward`, `backward`, `left`, `right` - Continuous movement (loops until stopped)
- `stop` - Immediately stop current movement

**Pose Commands (one-shot animations):**

- `rest`, `stand`, `wave`, `dance`, `swim`, `point`
- `pushup`, `bow`, `cute`, `freaky`, `worm`, `shake`
- `shrug`, `dead`, `crab`

**Available Faces:**

- Movement faces: `walk`, `rest`, `stand`, `dance`, `wave`, etc.
- Conversational faces: `happy`, `sad`, `angry`, `surprised`, `sleepy`, `love`, `excited`, `confused`, `thinking`
- Talk variants: `talk_happy`, `talk_sad`, `talk_angry`, etc.
- Special: `idle`, `idle_blink`, `default`

### Python API Example

```python
import requests
import time

# Robot IP or hostname
robot_url = "http://sesame-robot.local"

# Get status
response = requests.get(f"{robot_url}/api/status")
status = response.json()
print(f"Current face: {status['currentFace']}")

# Make robot wave and show happy face
requests.post(f"{robot_url}/api/command", json={
    "command": "wave",
    "face": "happy"
})

time.sleep(3)

# Just change face without moving
requests.post(f"{robot_url}/api/command", json={
    "face": "excited"
})

# Stop movement
requests.post(f"{robot_url}/api/command", json={
    "command": "stop"
})
```

### cURL Examples

```bash
# Get robot status
curl http://sesame-robot.local/api/status

# Make robot dance
curl -X POST http://sesame-robot.local/api/command \
  -H "Content-Type: application/json" \
  -d '{"command":"dance","face":"dance"}'

# Change face only
curl -X POST http://sesame-robot.local/api/command \
  -H "Content-Type: application/json" \
  -d '{"face":"happy"}'

# Stop movement
curl -X POST http://sesame-robot.local/api/command \
  -H "Content-Type: application/json" \
  -d '{"command":"stop"}'
```

### Node.js/JavaScript Example

```javascript
const robotURL = 'http://sesame-robot.local';

// Get status
fetch(`${robotURL}/api/status`)
  .then(res => res.json())
  .then(data => console.log('Robot status:', data));

// Send command
fetch(`${robotURL}/api/command`, {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    command: 'wave',
    face: 'happy'
  })
})
.then(res => res.json())
.then(data => console.log('Response:', data));

// Face-only update
fetch(`${robotURL}/api/command`, {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({face: 'surprised'})
})
.then(res => res.json())
.then(data => console.log('Response:', data));
```

## Advanced Integration Examples

### Voice Assistant Integration

The combination of conversational faces, the JSON API, and network mode makes the Sesame Robot perfect for voice assistant projects. Here's an example using Python with speech recognition:

```python
import requests
import speech_recognition as sr
from textblob import TextBlob
import time

robot_url = "http://sesame-robot.local"

def analyze_sentiment(text):
    """Analyze sentiment and return appropriate face"""
    blob = TextBlob(text)
    polarity = blob.sentiment.polarity
  
    if polarity > 0.5:
        return "excited"
    elif polarity > 0.2:
        return "happy"
    elif polarity < -0.5:
        return "angry"
    elif polarity < -0.2:
        return "sad"
    else:
        return "thinking"

def set_robot_face(face, talking=False):
    """Update robot's face expression"""
    if talking:
        face = f"talk_{face}"
  
    requests.post(f"{robot_url}/api/command", json={"face": face})

# Initialize speech recognition
recognizer = sr.Recognizer()

while True:
    with sr.Microphone() as source:
        print("Listening...")
        set_robot_face("idle")
    
        try:
            audio = recognizer.listen(source, timeout=5)
            set_robot_face("thinking")
        
            text = recognizer.recognize_google(audio)
            print(f"You said: {text}")
        
            # Determine emotion and show talking face
            emotion = analyze_sentiment(text)
            set_robot_face(emotion, talking=True)
        
            # Process command...
            time.sleep(2)
        
            # Return to neutral
            set_robot_face(emotion, talking=False)
        
        except sr.WaitTimeoutError:
            set_robot_face("sleepy")
        except sr.UnknownValueError:
            set_robot_face("confused")
```

### Home Automation Integration

Integrate with Home Assistant or other home automation platforms:

```python
# Home Assistant automation example
import requests

def robot_notification(message_type):
    """Display robot emotion based on notification type"""
    robot_url = "http://sesame-robot.local"
  
    emotion_map = {
        "doorbell": "surprised",
        "alarm": "angry",
        "reminder": "thinking",
        "success": "happy",
        "error": "sad"
    }
  
    face = emotion_map.get(message_type, "default")
    requests.post(f"{robot_url}/api/command", json={"face": face})
  
    # Optional: add movement
    if message_type == "doorbell":
        requests.post(f"{robot_url}/api/command", json={
            "command": "wave",
            "face": "happy"
        })
```

### WebSocket Streaming (Advanced)

For real-time control with minimal latency, you can poll the status endpoint or implement a simple state machine:

```javascript
// React/Vue.js real-time robot control
class RobotController {
  constructor(robotURL) {
    this.url = robotURL;
    this.currentFace = 'default';
  }
  
  async updateEmotion(emotion, isSpeaking) {
    const face = isSpeaking ? `talk_${emotion}` : emotion;
  
    if (face !== this.currentFace) {
      await fetch(`${this.url}/api/command`, {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({face})
      });
  
      this.currentFace = face;
    }
  }
  
  async getStatus() {
    const res = await fetch(`${this.url}/api/status`);
    return await res.json();
  }
  
  async performAction(action, emotion = null) {
    const payload = {command: action};
    if (emotion) payload.face = emotion;
  
    await fetch(`${this.url}/api/command`, {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify(payload)
    });
  }
}

// Usage
const robot = new RobotController('http://sesame-robot.local');

// Set emotion without moving
await robot.updateEmotion('happy', false);

// Set talking face
await robot.updateEmotion('excited', true);

// Perform action with face
await robot.performAction('dance', 'excited');

// Check status
const status = await robot.getStatus();
console.log(`Robot is ${status.currentFace} and doing ${status.currentCommand}`);
```

## Idle Animation System

The firmware includes an intelligent idle animation system that activates automatically:

### Behavior

- **Activation**: When the robot has received no commands for a period, it enters idle mode
- **Idle Face**: Displays a gentle "breathing" animation using the `idle` face in `FACE_ANIM_BOOMERANG` mode
- **Blinking**: Randomly triggers blink animations (3-7 second intervals)
- **Double Blinks**: 30% chance of a double blink for realistic behavior
- **Exit**: Any movement command or input immediately exits idle mode

### Technical Implementation

The idle system uses:

- `enterIdle()` - Activates idle face with boomerang animation
- `exitIdle()` - Returns to normal operation
- `updateIdleBlink()` - Manages random blink timing and double-blink logic
- `scheduleNextIdleBlink()` - Randomizes blink intervals for natural appearance

To customize idle behavior, modify the timing values in [sesame-firmware-main.ino](sesame-firmware-main.ino):

```cpp
scheduleNextIdleBlink(3000, 7000);  // Min and max ms between blinks
```

## Asset Pipeline & Face Customization

The firmware includes an intelligent idle animation system that activates automatically:

### Behavior

- **Activation**: When the robot has received no commands for a period, it enters idle mode
- **Idle Face**: Displays a gentle "breathing" animation using the `idle` face in `FACE_ANIM_BOOMERANG` mode
- **Blinking**: Randomly triggers blink animations (3-7 second intervals)
- **Double Blinks**: 30% chance of a double blink for realistic behavior
- **Exit**: Any movement command or input immediately exits idle mode

### Technical Implementation

The idle system uses:

- `enterIdle()` - Activates idle face with boomerang animation
- `exitIdle()` - Returns to normal operation
- `updateIdleBlink()` - Manages random blink timing and double-blink logic
- `scheduleNextIdleBlink()` - Randomizes blink intervals for natural appearance

To customize idle behavior, modify the timing values in [sesame-firmware-main.ino](sesame-firmware-main.ino):

```cpp
scheduleNextIdleBlink(3000, 7000);  // Min and max ms between blinks
```

## Hardware Abstraction Layer (HAL)

The firmware uses the selected ESP32 board's default I2C pins. Servo outputs come from PCA9685 channels 0-7, while the SSD1306 display is connected to the same SDA/SCL bus.

### PCA9685 Channel Map

| Servo | PCA9685 Channel | Leg |
| ----- | --------------- | --- |
| Motor 0 | 0 | R1 |
| Motor 1 | 1 | R2 |
| Motor 2 | 2 | L1 |
| Motor 3 | 3 | L2 |
| Motor 4 | 4 | R4 |
| Motor 5 | 5 | R3 |
| Motor 6 | 6 | L3 |
| Motor 7 | 7 | L4 |

### ESP32 I2C Pins

| Board | SDA | SCL |
| ----- | --- | --- |
| Lolin S2 Mini | 33 | 35 |
| Sesame Distro Board V1 | 21 | 22 |
| Sesame Distro Board V2 / V3 | 8 | 9 |

Connect both the PCA9685 and SSD1306 to these SDA/SCL pins. Grounds must be common between the ESP32, PCA9685 logic, display, and servo power supply. Power the servos through the PCA9685 `V+` rail from a suitable external supply, not from the ESP32 3.3 V pin.

### Porting to Other ESP32 Variants

The firmware calls `Wire.begin()` to use the default SDA/SCL pins supplied by the selected Arduino board definition. For custom wiring, replace it with `Wire.begin(SDA_PIN, SCL_PIN)`; both pins must support output because I2C data and clock are bidirectional.

## Asset Pipeline & Face Customization

To maintain a clean main source file and optimize performance, face bitmaps are decoupled from the primary logic. Faces are managed in [face-bitmaps.h](face-bitmaps.h) using a "Single Source of Truth" macro system.

### Face Library

The firmware includes an extensive library of faces organized into three categories:

#### Movement Faces

Synchronized with physical poses and animations:

- `walk`, `rest`, `stand`, `swim`, `dance`, `wave`, `point`
- `pushup`, `bow`, `cute`, `freaky`, `worm`, `shake`, `shrug`
- `dead`, `crab`, `idle`, `idle_blink`

#### Conversational Faces

Designed for expressive communication and voice assistant integration:

- **Base emotions**: `happy`, `sad`, `angry`, `surprised`, `sleepy`, `love`, `excited`, `confused`, `thinking`
- **Talk variants**: `talk_happy`, `talk_sad`, `talk_angry`, `talk_surprised`, `talk_sleepy`, `talk_love`, `talk_excited`, `talk_confused`, `talk_thinking`

The "talk_" variants feature open mouths for lip-sync and animated speech. These faces are perfect for:

- Voice assistant projects (Alexa, Google Assistant, custom TTS)
- Chatbot interfaces controlled via the JSON API
- Interactive storytelling and educational applications
- Remote-controlled performances

#### Special Faces

- `default` - The startup/fallback face
- `idle` - Gentle breathing animation for idle state
- `idle_blink` - Blink animation triggered randomly during idle

### Adding Custom Faces

### Workflow:

1. **Image Creation**: Find faces using Kaomoji or [Emojicombos](https://emojicombos.com/kaomoji). Create a `128x64` image in a tool like [jsPaint](https://jspaint.app/).
2. **Bitmap Conversion**: Use [image2cpp](https://javl.github.io/image2cpp/) with `Horizontal` scaling, `128x64` resolution, and `Arduino Code` output.
3. **Registration**:
   - Add your face name to the `FACE_LIST` macro in [face-bitmaps.h](face-bitmaps.h).
   - Paste the generated C array into [face-bitmaps.h](face-bitmaps.h) right after the last bitmap in the list.
   - (Optional) For animations, add numbered suffixes (e.g., `_1`, `_2`) and register the FPS in the `faceFpsEntries` in [sesame-firmware-main.ino](sesame-firmware-main.ino).

### Animating Faces

For an animation to be recognized by the `MAKE_FACE_FRAMES` macro, your array names in [face-bitmaps.h](face-bitmaps.h) must follow a strict naming convention:

- **Root Frame**: `epd_bitmap_myface` (This is required and acts as frame 0).
- **Subsequent Frames**: `epd_bitmap_myface_1`, `epd_bitmap_myface_2`, etc.
- **Limit**: The default system supports up to 6 frames per face (Root + 5 numbered frames).
- **Animation Modes**: Animations can be configured to play as a `LOOP` (restarts at frame 0), `ONCE` (stops on the final frame), or `BOOMERANG` (plays forward then reverses). These modes are typically defined in [movement-sequences.h](movement-sequences.h) when triggerring a pose.

### Macro System

The `FACE_LIST` macro uses X-Macros to automatically generate variable declarations and registration objects:

```cpp
#define FACE_LIST \
    X(walk) \
    X(rest) \
    X(my_new_face) // Just add this line!
```

This eliminates the need to manually update multiple switch statements or arrays when adding new assets.

## Execution & Deployment

1. **Toolchain**: Configure your IDE for `ESP32 Dev Module` or `Lolin S2 Mini`.
2. **Calibration**: Use the Serial Monitor (115200) to send manual step commands (e.g., `rn wf`).
3. **Power Management**: If the robot brownouts during movement, increase `motorCurrentDelay` in the web settings to further stagger servo bursts.
