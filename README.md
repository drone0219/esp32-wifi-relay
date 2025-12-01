# ESP32 WiFi Relay Control System

ESP32-based intelligent relay controller with WiFi connectivity, web interface, MQTT support, and advanced scheduling capabilities.

## System Overview

This project implements a comprehensive relay control system using an ESP32 microcontroller. The device can operate as both a WiFi access point and station, providing multiple interfaces for control including HTTP web pages, TCP socket connections, and MQTT messaging.

## Hardware Configuration

### GPIO Pin Assignments
- **GPIO2 (ESP32_GPIO2)**: Blue LED indicator
- **GPIO4 (ESP32_GPIO4)**: Relay control output
- **GPIO5 (ESP32_GPIO5)**: Optocoupler input

## Core System Architecture

### 1. Network Connectivity

#### Dual WiFi Mode Operation
The system operates in **WIFI_AP_STA** mode, functioning simultaneously as:

- **Access Point (AP)**: Creates a local WiFi network for direct device access
  - Default SSID: `Litmath_[MAC Address]`
  - Default Password: `Litmath123`
  - Users can connect directly to configure the device

- **Station (STA)**: Connects to an existing WiFi network
  - Configurable SSID and password
  - Enables remote access and internet connectivity
  - Automatically attempts reconnection on startup

### 2. Control Interfaces

#### A. HTTP Web Server
Runs on configurable port (default: 80) with session-based authentication:

**Public Endpoints:**
- `/login` - Authentication page with password protection
  - Session cookie: ESPSESSIONID
  - Password hint display
  - Hardware reset instructions

**Authenticated Endpoints:**
- `/` - Main control dashboard
  - Real-time relay status display
  - I/O status monitoring
  - Device information (IP, MAC address)
  - Quick relay control buttons

- `/Settings` - Comprehensive configuration page
  - Device name customization
  - Password management
  - Port configuration
  - Operation mode selection
  - MQTT settings
  - Time zone and location settings

- `/WifiSetting` - Network configuration
  - WiFi network scanner
  - Station mode credentials
  - AP mode settings

**API Endpoints (Password Required):**
- `/OpenRelay?t=[seconds]&pp=[password]` - Open relay for specified duration (0=permanent)
- `/CloseRelay?pp=[password]` - Close relay immediately
- `/status?pp=[password]` - Query relay state
- `/iostatus?pp=[password]` - Query input pin state
- `/MAC?pp=[password]` - Retrieve device MAC address
- `/timers` - Configure schedule timers
- `/Mode` - Change operation mode
- `/Port` - Modify HTTP/TCP ports

#### B. TCP Socket Server
Runs on configurable port (default: 8000):
- Supports multiple simultaneous clients (configurable, default: 1)
- Command protocol identical to HTTP API
- Real-time status notifications pushed to connected clients
- Format: `Command?param1=value1,param2=value2,...`

#### C. MQTT Client
Configurable MQTT broker connection:
- **Subscribe Topic** (Input): Receives commands
  - Default: `Home/Litmath/WifiRelay/Input`
- **Publish Topic** (Output): Sends status updates
  - Default: `Home/Litmath/WifiRelay/Output`
- Automatic reconnection with random client ID
- Real-time event notifications (relay state changes, I/O changes)

#### D. TCP Client Mode
Optional outbound connection to remote server:
- Configurable server IP and port
- Automatic connection on startup
- Sends status updates to remote system

### 3. Operation Modes

The system supports six distinct operation modes:

#### Mode 1: Normal Mode
**Manual Control with Timer**
- Relay controlled via HTTP/TCP/MQTT commands
- Supports timed activation (auto-close after N seconds)
- Permanent activation option (timer=0)
- **Parameters:**
  - P1 (Pos/Neg): Input polarity for overlay
  - P2 (OVR/None): Enable/disable input overlay
- **Overlay Function**: When enabled, input signal can override relay state

#### Mode 2: I/O Control Mode
**Direct Input-to-Output Mapping**
- Relay state mirrors input pin state
- **Parameter P1:**
  - **Positive (0)**: HIGH input → Relay ON, LOW input → Relay OFF
  - **Negative (1)**: LOW input → Relay ON, HIGH input → Relay OFF
- Ideal for automated switching based on external sensors

#### Mode 3: Schedule1 Control Mode
**Time-Based Automation (Primary Schedule)**
- 6 independent timer units
- Each unit configurable with:
  - Start time and end time
  - Day-of-week selection (7 bits: Sun-Sat)
  - Month selection (12 bits: Jan-Dec)
- **Solar-Aware Timing:**
  - Support for sunrise/sunset relative times
  - Format: `SS[P/M]mmm` (sunset ± minutes) or `SR[P/M]mmm` (sunrise ± minutes)
  - Automatically calculates based on configured location/timezone
- **Overlay Support**: P2=OVR enables input override

#### Mode 4: Schedule2 Control Mode
**Time-Based Automation (Secondary Schedule)**
- Identical functionality to Schedule1
- Independent 6-timer configuration
- Allows for seasonal or backup scheduling
- Full solar calculation support

#### Mode 5: Schedule3 Control Mode
**Time-Based Automation (Tertiary Schedule)**
- Third independent scheduling system
- Same features as Schedule1 and Schedule2
- Enables complex multi-schedule scenarios

### 4. Time Management

#### NTP Synchronization
- Automatic time sync from `us.pool.ntp.org`
- Syncs on startup and periodically
- Falls back to manual time entry if NTP unavailable

#### Timezone Support
Supports 6 US timezones with automatic DST adjustment:
- Eastern (UTC-5/-4)
- Central (UTC-6/-5)
- Mountain (UTC-7/-6)
- Pacific (UTC-8/-7)
- Alaska (UTC-9/-8)
- Hawaii (UTC-10/-9)

#### Solar Calculations
- Integrated `Solarlib` for sunrise/sunset calculations
- Latitude/longitude automatically set per timezone
- Enables astronomical timer scheduling
- Daily recalculation for accuracy

### 5. Data Persistence

#### EEPROM Storage
Configuration saved to EEPROM includes:
- Device name and password
- Network credentials (AP and STA)
- HTTP/TCP/MQTT ports
- Operation mode and parameters
- All three schedule configurations (18 timer units total)
- MQTT broker settings
- TCP client server settings
- Timezone selection

**Configuration Version**: `ls6` - Validates stored settings on boot

### 6. Security Features

#### Authentication System
- Password-protected web interface
- Session cookie management (ESPSESSIONID)
- Password hint customization
- All API endpoints require password parameter

#### Hardware Password Reset
**Emergency Recovery Procedure:**
1. Hold I/O pin active during power-up
2. Wait for blue LED to blink (4 seconds)
3. Continue holding for 4 seconds after LED stops
4. Releases device with factory defaults:
   - Password: `WifiRelay123456`
   - AP Password: `Litmath123`

### 7. Control Logic Flow

#### Main Loop Processing
```
1. Handle HTTP requests
2. Process TCP socket clients
3. Execute startup delay timer (8 seconds)
4. Monitor hardware reset button sequence
5. Check for new TCP client connections
6. Read incoming TCP/MQTT commands
7. Execute current operation mode logic
8. Update relay output state
9. Send status change notifications
10. Process MQTT keep-alive
```

#### Schedule Evaluation Algorithm
For Schedule modes (3/4/5):
```
For each of 6 timer units:
  Check if current month is enabled
    Check if current day-of-week is enabled
      Convert times to 24-bit values (HH:MM:SS)
      If current time >= start AND current time <= end
        Set relay = ON
  
Apply overlay if enabled (P2=OVR):
  Check input pin state
  Override relay based on P1 polarity
  
Write relay state to GPIO4
```

### 8. Command Protocol

#### Command Format
All commands follow pattern: `Command?Name=[device]&param=value&pp=[password]`

#### Key Commands:
- **OpenRelay**: `OpenRelay?Name=device&t=30&pp=pass` (open for 30 seconds)
- **CloseRelay**: `CloseRelay?Name=device&pp=pass`
- **Mode**: `Mode?Name=device&Choice=3&P1=0&P2=1&pp=pass`
- **Timers**: `timers?Name=device&sch=1&u=1&st=06:00:00AM&en=10:00:00PM&d=1111111&m=111111111111&pp=pass`
- **Solar Timers**: `timers?Name=device&sch=1&u=2&st=SSM030&en=SRP060&d=1111111&m=111111111111&pp=pass`
  - SSM030 = 30 minutes before sunset
  - SRP060 = 60 minutes after sunrise

### 9. Status Notifications

#### Automatic Broadcasts
System automatically sends notifications via TCP/MQTT when:
- Relay state changes (ON/OFF)
- Input pin state changes
- Configuration updates
- Mode changes

#### Message Format
- Relay change: `[DeviceName] Opened!` or `[DeviceName] Closed!`
- I/O change: `[DeviceName],change,IO=[0/1]`
- Status query: `[DeviceName],status=[0/1]`

## Web Interface Architecture

### SPIFFS File System Resources

The system includes HTML templates stored in the `data/` directory for ESP32 SPIFFS (SPI Flash File System). These templates can be uploaded to the device's flash memory for serving static web pages.

#### Available HTML Templates (`data/` directory):

1. **Root_NoneAuth.html** - Login/Authentication Page
2. **Root.html** - Main Dashboard (Authenticated)
3. **SetupPage.html** - Comprehensive Settings Page
4. **WifiSetup.html** - WiFi Configuration Page
5. **UtilityPage.html** - Action Confirmation/Feedback Page
6. **Notify.html** - Settings Change Notification Page

### Web Page Layout Details

#### 1. Login Page (`/login` - Root_NoneAuth.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
├─────────────────────────────────────────┤
│                                         │
│  Enter Password: [___________]          │
│  [Login Button]                         │
│                                         │
│  Password Hint: [hint text]             │
│  Error Message (if login failed)        │
│                                         │
│  Hardware Reset Instructions:           │
│  - Hold IO port active during power-up  │
│  - Watch blue LED blink (4 seconds)     │
│  - Default password: WifiRelay123456    │
│  - Default AP password: Litmath123      │
│                                         │
│  Copyright Litmath, LLC, 2017           │
└─────────────────────────────────────────┘
```

**Features:**
- Password input field (POST to `/login`)
- Password hint display
- Detailed hardware reset instructions
- Error message display for invalid credentials
- Link to Litmath website

#### 2. Main Dashboard (`/` - Root.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
│  [Logoff link]                          │
├─────────────────────────────────────────┤
│  Device Name: [device_name]             │
│                                         │
│  Relay Status: On/Off                   │
│  I/O Status: On/Off                     │
│                                         │
│  Local IP: [192.168.x.x]                │
│  MAC Address: [XX:XX:XX:XX:XX:XX]       │
│                                         │
│  Relay Operation                        │
│  [Open Relay] [Close Relay]             │
│                                         │
│  Settings                               │
│  [Relay Setup] [Wifi Setup]             │
│                                         │
│  Copyright Litmath, LLC, 2017           │
└─────────────────────────────────────────┘
```

**Features:**
- Real-time status display (relay and I/O)
- Network information (IP and MAC address)
- Quick action buttons for relay control
- Navigation to configuration pages
- Session logoff link

#### 3. Settings Page (`/Settings` - SetupPage.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
├─────────────────────────────────────────┤
│  Current Relay Name: [name]             │
│  Change Name: [________] [Save]         │
├─────────────────────────────────────────┤
│  Password Setting                       │
│  Current Password: [_______]            │
│  New Password: [_______]                │
│  Confirm Password: [_______]            │
│  Password Hint: [_______]               │
│  [Save Password]                        │
├─────────────────────────────────────────┤
│  Port Setting                           │
│  HTTP Port: [80___]                     │
│  TCP Port: [8000_]                      │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  TCP Connection Setting                 │
│  Server IP: [192.168.1.138]             │
│  Server Port: [7000]                    │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  Mode of Operation                      │
│  Control Mode: [Dropdown Menu]          │
│    - Normal Mode                        │
│    - IO Control Mode                    │
│    - Schedule1 Mode                     │
│    - Schedule2 Mode                     │
│    - Schedule3 Mode                     │
│  Parameter1: [Pos/Neg]                  │
│  Parameter2: [None/On/Off]              │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  MQTT Setting                           │
│  MQTT Server: [___________]             │
│  Port: [1883]                           │
│  Input Topic: [___________]             │
│  Output Topic: [___________]            │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  Location and DateTime Setting          │
│  Current DateTime: [datetime picker]    │
│  Location: [Timezone Dropdown]          │
│    - Eastern, Central, Mountain,        │
│      Pacific, Alaska, Hawaii            │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  [Back to Main] | Copyright 2017        │
└─────────────────────────────────────────┘
```

**Features:**
- Device name customization form
- Password change with confirmation
- HTTP/TCP port configuration
- TCP client connection settings
- Operation mode selector with parameters
- MQTT broker configuration
- Timezone/location settings with datetime picker
- Individual save buttons for each section

#### 4. WiFi Setup Page (`/WifiSetting` - WifiSetup.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
├─────────────────────────────────────────┤
│  Select WiFi Network to Connect to      │
│                                         │
│  Usable APs: [Dropdown List]            │
│    (Scanned networks)                   │
│  Password: [________]                   │
│  [Connect]                              │
├─────────────────────────────────────────┤
│  AdHoc Network                          │
│                                         │
│  AP SSID: [current_ssid]                │
│  Password: [________]                   │
│  [Save]                                 │
├─────────────────────────────────────────┤
│  [Back to Main] | Copyright 2017        │
└─────────────────────────────────────────┘
```

**Features:**
- Live WiFi network scanner (populates dropdown)
- Station mode configuration (connect to router)
- Access Point mode configuration
- Separate forms for STA and AP settings
- Automatic device restart after changes

#### 5. Utility Page (`/` after actions - UtilityPage.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
├─────────────────────────────────────────┤
│  You have changed settings              │
│                                         │
│  [Detailed feedback message]            │
│                                         │
│  [Back Link]                            │
│                                         │
│  Copyright Litmath, LLC, 2017           │
└─────────────────────────────────────────┘
```

**Features:**
- Dynamic feedback message display
- Confirmation of configuration changes
- JavaScript history back navigation
- Used after relay operations or settings updates

#### 6. Notification Page (Notify.html)

**Layout Structure:**
```
┌─────────────────────────────────────────┐
│  Welcome to Litmath, WiFi Relay         │
├─────────────────────────────────────────┤
│  Device Setting is changed!             │
│                                         │
│  Password recovery reminder:            │
│  Press button during power-up           │
│  Default: WifiRelay123456               │
│                                         │
│  [Back to Settings]                     │
│  Copyright Litmath, LLC, 2017           │
└─────────────────────────────────────────┘
```

**Features:**
- Settings change confirmation
- Password recovery reminder
- Navigation back to settings

### UI Design System

#### Color Scheme
- **Background**: Light cyan (`#E8F8F5`)
- **Header**: Golden yellow (`#D4AC0D`)
- **Success Text**: Green (`#2ecc71`)
- **Error/Warning Text**: Red (`#E74C3C`)
- **Button Styles**: Bootstrap 3.1.1 default

#### Typography
- **H1 Headers**: Large, golden yellow, page titles
- **H4 Text**: 22px, green, labels and section headers
- **H5 Text**: 16px, red, error messages and input fields
- **H6 Text**: 16px, green, information and links

#### Responsive Design
- Bootstrap 3.1.1 grid system
- Mobile-friendly viewport meta tag
- Centered container layout
- Single-column responsive layout

#### Form Behavior
- All forms use POST method
- Action attributes point to specific endpoints
- Password fields masked
- Dropdown selects for predefined options
- Datetime-local input for time configuration
- Immediate form submission (no JavaScript validation)

### Page Flow Navigation

```
Login Page → Main Dashboard ⟷ Settings Page
              ↓                     ↓
         Close Relay           WiFi Setup
              ↓                     ↓
         Open Relay            Utility Page
              ↓
         Utility Page
```

### Dynamic Content Integration

The ESP32 dynamically generates page content by:
1. Loading HTML template structure (from SPIFFS or embedded strings)
2. Replacing placeholder values with current system state
3. Populating dropdowns (WiFi networks, timezones)
4. Updating status indicators (relay state, I/O state)
5. Pre-filling form fields with current configuration

**Note**: The actual implementation uses embedded HTML strings in the .ino file rather than loading from SPIFFS, but the `data/` directory templates serve as reference designs for the page structure.

## Configuration Storage

All settings persist across power cycles and include:
- Network configurations
- Security credentials  
- Operational parameters
- 18 independent timer schedules (3 groups × 6 units)
- MQTT and TCP settings
- Timezone and location data

## Use Cases

1. **Smart Home Lighting**: Schedule-based outdoor lighting with sunset/sunrise automation
2. **Remote Power Control**: HTTP/MQTT control of appliances
3. **Industrial Automation**: I/O mode for sensor-driven switching
4. **Irrigation Systems**: Multi-schedule timer control
5. **Security Systems**: Remote monitoring and control via MQTT

## Technical Specifications

- **Microcontroller**: ESP32
- **Flash Memory**: EEPROM for persistent storage
- **Network**: Dual-mode WiFi (AP + STA)
- **Protocols**: HTTP, TCP sockets, MQTT
- **Time Sync**: NTP with automatic DST
- **Scheduling**: 18 independent timers with solar awareness
- **Authentication**: Session-based with password protection
