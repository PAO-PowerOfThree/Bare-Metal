# Bare-Metal
Bare-metal embedded implementation for microcontroller development.

# STM32 CAN Controller Documentation

## Project Structure

```
STM32_CAN_Controller/
├── Core/
│   ├── Src/
│   │   ├── main.c                   # Main application logic
│   │   ├── stm32f1xx_hal_msp.c      # Hardware abstraction layer
│   │   ├── stm32f1xx_it.c           # Interrupt service routines
│   │   └── system_stm32f1xx.c       # System initialization
│   └── Inc/
│       ├── main.h                   # Main header definitions
│       ├── stm32f1xx_hal_conf.h     # HAL configuration
│       ├── stm32f1xx_it.h           # Interrupt declarations
│       └── system_stm32f1xx.h       # System header
├── Drivers/                         # STM32 HAL drivers
├── [ProjectName].ioc               # STM32CubeMX configuration file
└── README.md                       # This documentation
```

## Overview

This project implements a dual-channel CAN controller using STM32F1 microcontroller with STM32CubeMX and HAL libraries. The system provides interactive control of LEDs, PWM outputs, and CAN communication through physical inputs (buttons and potentiometers) with UART debugging capability.

## Hardware Configuration

### Microcontroller
- **MCU**: STM32F1 series (configured via STM32CubeMX)
- **Clock**: HSI (High Speed Internal) oscillator
- **Voltage**: 3.3V operation

### Pin Assignments

#### Digital I/O
- **LED (PA5)**: Primary status LED with blinking control
- **LED2 (PB0)**: Secondary status LED with blinking control
- **BUTTON (PC13)**: Primary control button with pull-up resistor
- **BUTTON2**: Secondary control button with pull-up resistor

#### Analog Inputs
- **PA0 (ADC1_IN0)**: First potentiometer input for PWM control
- **PA1 (ADC1_IN1)**: Second potentiometer input for PWM control

#### PWM Outputs
- **PA8 (TIM1_CH1)**: PWM output controlled by first potentiometer
- **PA2 (TIM2_CH3)**: PWM output controlled by second potentiometer

#### Communication
- **USART1**: Debug output and system monitoring (115200 baud)
- **CAN1**: Automotive CAN bus communication interface

## Peripheral Configuration

### ADC (Analog-to-Digital Converter)
**Purpose**: Converts analog potentiometer voltages to digital values
- **Instance**: ADC1
- **Mode**: Single conversion, software triggered
- **Resolution**: 12-bit (0-4095 range)
- **Channels**: Dynamically switched between CH0 (PA0) and CH1 (PA1)
- **Sampling**: Minimal sampling time for fastest conversion

### Timer/PWM Configuration

#### TIM1 (Advanced Timer)
**Purpose**: Generates PWM signal for first channel
- **Prescaler**: 83 (reduces 8MHz HSI to ~96kHz timer clock)
- **Period**: 1000 counts (provides ~96Hz PWM frequency)
- **Channel**: TIM1_CH1 on PA8
- **PWM Range**: 0-1000 duty cycle steps
- **Dead Time**: Disabled (not needed for simple PWM)

#### TIM2 (General Purpose Timer)  
**Purpose**: Generates PWM signal for second channel
- **Prescaler**: 83 (matches TIM1 for consistent frequency)
- **Period**: 1000 counts 
- **Channel**: TIM2_CH3 on PA2
- **PWM Range**: 0-1000 duty cycle steps

### CAN Bus Configuration
**Purpose**: Automotive communication protocol implementation
- **Instance**: CAN1
- **Mode**: Normal operation mode
- **Baud Rate**: ~125 kbps (calculated from prescaler and timing segments)
- **Bit Timing**:
  - Prescaler: 8
  - Time Segment 1: 6TQ
  - Time Segment 2: 1TQ
  - Sync Jump Width: 1TQ
- **Features**: Standard 11-bit identifiers, no automatic retransmission

### UART Configuration
**Purpose**: Debug output and system monitoring
- **Instance**: USART1
- **Baud Rate**: 115200 bps
- **Data Format**: 8 data bits, 1 stop bit, no parity
- **Flow Control**: None
- **Mode**: Transmit and receive enabled

## Software Architecture

### Main Application Loop
The system operates in a continuous polling loop with these main functions:

1. **Button State Management**: Debounced button reading with toggle functionality
2. **LED Control**: Blinking pattern generation with configurable timing
3. **ADC Reading**: Sequential sampling of two potentiometer channels
4. **PWM Generation**: Real-time duty cycle adjustment based on ADC values
5. **CAN Communication**: Transmission of system state and sensor data
6. **Debug Output**: UART transmission of system status

### Control Flow

#### Dual LED Control System
Each LED channel operates independently:

**First Channel (LED on PA5)**:
- **Trigger**: Button press on BUTTON pin
- **Behavior**: Toggle between OFF and BLINKING states
- **Blink Rate**: 500ms on/off cycle when enabled
- **CAN Message**: Sends state (0=OFF, 1=BLINK) on ID 0x101
- **Debug**: UART output shows "Button: OFF/BLINK"

**Second Channel (LED2 on PB0)**:
- **Trigger**: Button press on BUTTON2 pin
- **Behavior**: Identical to first channel but independent
- **CAN Message**: Sends state (0=OFF, 1=BLINK) on ID 0x103
- **Debug**: UART output shows "Button2: OFF/BLINK"

#### Dual PWM Control System
Two independent PWM channels controlled by potentiometers:

**First Channel (PWM on PA8)**:
- **Input Source**: ADC1_IN0 (PA0 potentiometer)
- **ADC Range**: 0-4095 (12-bit resolution)
- **PWM Range**: 0-1000 duty cycle steps
- **Update Threshold**: Changes > 10 steps to reduce noise
- **CAN Message**: ID 0x102, 4 bytes [ADC_H, ADC_L, PWM_H, PWM_L]
- **Debug**: Shows "ADC:xxxx PWM:xxxx"

**Second Channel (PWM on PA2)**:
- **Input Source**: ADC1_IN1 (PA1 potentiometer)  
- **Operation**: Identical to first channel but independent
- **CAN Message**: ID 0x104, same 4-byte format
- **Debug**: Shows "ADC2:xxxx PWM2:xxxx"

### CAN Communication Protocol

#### Message Format
All CAN messages use standard 11-bit identifiers:

- **0x101**: LED1 blink state (1 byte: 0=OFF, 1=BLINK)
- **0x102**: Potentiometer 1 data (4 bytes: ADC high, ADC low, PWM high, PWM low)
- **0x103**: LED2 blink state (1 byte: 0=OFF, 1=BLINK) 
- **0x104**: Potentiometer 2 data (4 bytes: same format as 0x102)

#### Transmission Function
```c
void send_can_message(uint32_t id, uint8_t *data, uint8_t len)
```
- **Purpose**: Sends formatted CAN messages with error handling
- **Parameters**: Message ID, data pointer, data length (1-8 bytes)
- **Error Handling**: Calls Error_Handler() on transmission failure
- **Debug**: Optional error message output via UART

## System Integration

### Timing and Synchronization
- **Main Loop**: 50ms cycle time for system responsiveness
- **Button Debounce**: 200ms minimum between presses
- **LED Blink**: 500ms on/off intervals
- **ADC Sampling**: Continuous polling with channel switching
- **PWM Updates**: Real-time response to potentiometer changes

### ADC Channel Management
The system uses a single ADC instance to read two channels:
1. Read and process PA0 (first potentiometer)
2. Dynamically reconfigure ADC to PA1 channel
3. Read and process PA1 (second potentiometer)  
4. Reconfigure ADC back to PA0 for next cycle

This approach shares ADC hardware while maintaining independent control loops.

### Error Handling
- **CAN Transmission Errors**: System halts and enters infinite loop
- **HAL Function Failures**: Calls Error_Handler() which disables interrupts
- **ADC Configuration Errors**: System halt on channel switching failure
- **Hardware Initialization**: Any peripheral init failure stops execution

## Development Workflow

### STM32CubeMX Configuration
1. **Clock Configuration**: HSI selected, peripheral clocks enabled
2. **Pin Assignment**: Set pin functions for GPIO, ADC, PWM, UART, CAN
3. **Peripheral Parameters**: Configure baud rates, frequencies, modes
4. **Code Generation**: Generate HAL initialization code
5. **User Code**: Add application logic in USER CODE sections

### Build and Debug Process
1. **Compile**: Use STM32CubeIDE or compatible ARM GCC toolchain
2. **Flash**: Program via ST-Link debugger interface
3. **Debug**: UART output at 115200 baud for real-time monitoring
4. **CAN Testing**: Use CAN analyzer to verify message transmission
5. **Hardware Testing**: Verify LED blinking, PWM output, button response

### Customization Points
- **Blink Rate**: Modify `BLINK_DELAY` constant (default 500ms)
- **PWM Frequency**: Adjust timer prescaler and period values
- **CAN Baud Rate**: Modify prescaler and timing segment configuration
- **Debug Level**: Add/remove UART debug messages as needed
- **ADC Sampling**: Adjust sampling time and conversion triggers

## Hardware Testing

### Functional Verification
1. **LED Control**: Press buttons to verify blinking toggle functionality
2. **PWM Output**: Rotate potentiometers to observe duty cycle changes
3. **UART Debug**: Monitor serial output for system status messages
4. **CAN Messages**: Use CAN analyzer to verify transmitted data
5. **System Integration**: Confirm all subsystems work simultaneously

### Performance Characteristics
- **Response Time**: ~50ms maximum latency for input changes
- **PWM Resolution**: 1000 steps (0.1% duty cycle granularity)
- **ADC Resolution**: 12-bit (4096 levels, ~0.8mV per step at 3.3V)
- **CAN Throughput**: Up to 4 messages per control cycle
- **Button Debounce**: 200ms minimum prevents false triggers

## Troubleshooting

### Common Issues
- **No CAN Messages**: Check CAN transceiver wiring and termination
- **Erratic PWM**: Verify stable power supply and potentiometer connections
- **Missing Debug Output**: Confirm UART baud rate and pin connections
- **Buttons Not Working**: Check pull-up resistors and GPIO configuration
- **System Hang**: Usually indicates HAL error - check peripheral initialization

### Debug Strategies
- **UART Monitoring**: Primary debugging method for real-time status
- **LED Behavior**: Visual indication of button press detection
- **PWM Measurement**: Use oscilloscope to verify output waveforms
- **CAN Analysis**: External tools required for protocol verification
- **Code Stepping**: Use debugger to trace execution flow