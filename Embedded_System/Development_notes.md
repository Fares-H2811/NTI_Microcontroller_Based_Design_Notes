# ATmega32 Development Notes & Best Practices

## Quick Reference & Tips

### Essential Development Setup
```c
/* Standard Include Order */
#include <avr/io.h>          // Hardware registers
#include <avr/interrupt.h>   // Interrupt handling
#include <util/delay.h>      // Delay functions
#include <avr/pgmspace.h>    // Program memory access
#include <avr/sleep.h>       // Power management
#include <avr/wdt.h>         // Watchdog timer

/* Custom includes */
#include "STD_TYPES.h"
#include "BIT_MATH.h"
/* Driver includes */
```

### ATmega32 Hardware Constraints
- **Flash Memory**: 32KB (program storage)
- **SRAM**: 2KB (variables, stack)
- **EEPROM**: 1KB (non-volatile data)
- **Clock Speed**: Up to 16MHz external crystal
- **I/O Pins**: 32 (4 ports × 8 pins each)
- **ADC**: 8-channel, 10-bit resolution
- **Timers**: 3 (Timer0: 8-bit, Timer1: 16-bit, Timer2: 8-bit)

### Memory Management Tips

#### Flash Memory Optimization
```c
/* Store strings in program memory to save RAM */
const char PROGMEM string_table[][16] = {
    "System Ready",
    "Error Occurred",
    "Please Wait"
};

void print_message(uint8_t msg_id) {
    char buffer[16];
    strcpy_P(buffer, (char*)pgm_read_word(&string_table[msg_id]));
    LCD_voidSendString(buffer);
}
```

#### RAM Conservation Strategies
```c
/* Use bit fields for multiple boolean flags */
struct {
    uint8_t system_ready    : 1;
    uint8_t error_flag      : 1;
    uint8_t data_available  : 1;
    uint8_t reserved        : 5;
} system_flags;

/* Prefer local variables over global when possible */
void process_data(void) {
    uint8_t temp_data;  // Automatically freed when function exits
    /* Processing code */
}

/* Use appropriate data types */
uint8_t counter;        // For values 0-255
uint16_t timer_value;   // For values up to 65535
boolean flag;           // For true/false values
```

### Timing & Delay Best Practices

#### Accurate Delays
```c
/* Configure F_CPU before including delay.h */
#define F_CPU 16000000UL  // 16MHz crystal
#include <util/delay.h>

/* Use appropriate delay functions */
_delay_us(50);    // Microsecond delays (compile-time constant)
_delay_ms(100);   // Millisecond delays (compile-time constant)

/* For variable delays, use timer-based approach */
void variable_delay_ms(uint16_t delay_ms) {
    for(uint16_t i = 0; i < delay_ms; i++) {
        _delay_ms(1);
    }
}
```

#### Timer-Based Non-Blocking Delays
```c
/* Non-blocking delay implementation */
typedef struct {
    uint32_t start_time;
    uint32_t duration;
    boolean active;
} SoftTimer_t;

static volatile uint32_t system_tick = 0;

ISR(TIMER0_OVF_vect) {
    system_tick++;
}

void SoftTimer_Start(SoftTimer_t* timer, uint32_t duration_ms) {
    timer->start_time = system_tick;
    timer->duration = duration_ms;
    timer->active = TRUE;
}

boolean SoftTimer_IsExpired(SoftTimer_t* timer) {
    if (!timer->active) return FALSE;
    
    if ((system_tick - timer->start_time) >= timer->duration) {
        timer->active = FALSE;
        return TRUE;
    }
    return FALSE;
}
```

### Interrupt Programming Guidelines

#### ISR Best Practices
```c
/* Keep ISRs short and simple */
volatile uint8_t button_pressed = 0;

ISR(INT0_vect) {
    button_pressed = 1;  // Set flag only
    // No delays, no complex processing!
}

/* Process flag in main loop */
int main(void) {
    while(1) {
        if (button_pressed) {
            button_pressed = 0;  // Clear flag
            handle_button_press();  // Complex processing here
        }
    }
}
```

#### Shared Variable Protection
```c
/* Protect multi-byte variables accessed from ISR */
volatile uint16_t adc_result;

uint16_t get_adc_result(void) {
    uint16_t temp;
    cli();  // Disable interrupts
    temp = adc_result;
    sei();  // Re-enable interrupts
    return temp;
}

/* Use atomic operations for single-byte variables */
volatile uint8_t status_flag;
// Direct access is safe for 8-bit variables on 8-bit MCU
```

### ADC Programming Notes

#### ADC Configuration Tips
```c
/* ADC initialization with proper settling time */
void ADC_Init(void) {
    // Select AVCC as reference voltage
    ADMUX = (1 << REFS0);
    
    // Enable ADC, set prescaler to 128 for 16MHz -> 125kHz ADC clock
    ADCSRA = (1 << ADEN) | (1 << ADPS2) | (1 << ADPS1) | (1 << ADPS0);
    
    // Dummy conversion to initialize ADC
    ADCSRA |= (1 << ADSC);
    while (ADCSRA & (1 << ADSC));
}

/* Temperature calculation from internal sensor */
uint16_t read_internal_temperature(void) {
    ADMUX = (1 << REFS1) | (1 << REFS0) | (1 << MUX3);  // Internal temp sensor
    _delay_us(200);  // Allow voltage to settle
    
    ADCSRA |= (1 << ADSC);
    while (ADCSRA & (1 << ADSC));
    
    // Temperature = (ADC - 273) * 1.22  (approximate)
    return ADC;
}
```

### UART Communication Tips

#### Baud Rate Calculation
```c
/* Baud rate calculation for 16MHz */
#define F_CPU 16000000UL
#define BAUD 9600
#define UBRR_VALUE ((F_CPU/16/BAUD) - 1)

void UART_Init(void) {
    UBRRH = (UBRR_VALUE >> 8);
    UBRRL = UBRR_VALUE;
    
    UCSRB = (1 << RXEN) | (1 << TXEN);  // Enable RX and TX
    UCSRC = (1 << URSEL) | (1 << UCSZ0) | (1 << UCSZ1);  // 8-bit data
}
```

#### Robust UART Communication
```c
/* Non-blocking UART transmit */
boolean UART_TransmitByte(uint8_t data) {
    if (UCSRA & (1 << UDRE)) {  // Check if transmit buffer is empty
        UDR = data;
        return TRUE;
    }
    return FALSE;  // Buffer full, try again later
}

/* UART receive with timeout */
boolean UART_ReceiveByte(uint8_t* data, uint16_t timeout_ms) {
    uint16_t timeout = 0;
    
    while (!(UCSRA & (1 << RXC)) && timeout < timeout_ms) {
        _delay_ms(1);
        timeout++;
    }
    
    if (timeout >= timeout_ms) {
        return FALSE;  // Timeout
    }
    
    *data = UDR;
    return TRUE;
}
```

### I2C (TWI) Implementation Notes

#### I2C Master Mode Setup
```c
/* I2C initialization for 100kHz at 16MHz */
void I2C_Init(void) {
    // Set bit rate: 100kHz = F_CPU / (16 + 2*TWBR*Prescaler)
    TWBR = 72;  // For 100kHz at 16MHz with prescaler = 1
    TWSR = 0;   // Prescaler = 1
}

/* I2C start condition */
uint8_t I2C_Start(void) {
    TWCR = (1 << TWINT) | (1 << TWSTA) | (1 << TWEN);
    while (!(TWCR & (1 << TWINT)));
    return (TWSR & 0xF8);  // Return status
}
```

### SPI Communication Guidelines

#### SPI Master Configuration
```c
void SPI_MasterInit(void) {
    // Set MOSI, SCK, and SS as outputs
    DDRB |= (1 << DDB5) | (1 << DDB7) | (1 << DDB4);
    
    // Enable SPI, Master mode, set clock rate fck/16
    SPCR = (1 << SPE) | (1 << MSTR) | (1 << SPR0);
}

uint8_t SPI_Transceive(uint8_t data) {
    SPDR = data;                        // Start transmission
    while(!(SPSR & (1 << SPIF)));      // Wait for transmission complete
    return SPDR;                       // Return received data
}
```

### Power Management Strategies

#### Sleep Mode Implementation
```c
#include <avr/sleep.h>

void enter_sleep_mode(void) {
    set_sleep_mode(SLEEP_MODE_PWR_DOWN);  // Lowest power mode
    cli();
    
    // Disable unnecessary peripherals
    ADCSRA &= ~(1 << ADEN);  // Disable ADC
    PRR = 0xFF;              // Power reduction register - disable all
    
    sleep_enable();
    sei();
    sleep_cpu();
    sleep_disable();
    
    // Re-enable peripherals after wake-up
    PRR = 0x00;
    ADCSRA |= (1 << ADEN);
}
```

### Common Debugging Techniques

#### LED-Based Debugging
```c
/* Use LEDs for visual debugging */
#define DEBUG_LED_PORT  PORTC
#define DEBUG_LED_DDR   DDRC
#define DEBUG_LED_PIN   PC0

void debug_init(void) {
    DEBUG_LED_DDR |= (1 << DEBUG_LED_PIN);
}

void debug_blink_pattern(uint8_t pattern) {
    for(uint8_t i = 0; i < 8; i++) {
        if(pattern & (1 << i)) {
            DEBUG_LED_PORT |= (1 << DEBUG_LED_PIN);
            _delay_ms(200);
        } else {
            DEBUG_LED_PORT &= ~(1 << DEBUG_LED_PIN);
            _delay_ms(100);
        }
    }
}
```

#### UART-Based Debugging
```c
/* Debug output over UART */
void debug_print_hex(uint8_t value) {
    char hex_chars[] = "0123456789ABCDEF";
    UART_TransmitByte('0');
    UART_TransmitByte('x');
    UART_TransmitByte(hex_chars[(value >> 4) & 0x0F]);
    UART_TransmitByte(hex_chars[value & 0x0F]);
    UART_TransmitByte('\n');
}

void debug_print_binary(uint8_t value) {
    UART_TransmitByte('0');
    UART_TransmitByte('b');
    for(int8_t i = 7; i >= 0; i--) {
        UART_TransmitByte((value & (1 << i)) ? '1' : '0');
    }
    UART_TransmitByte('\n');
}
```

### Performance Optimization Tips

#### Loop Optimization
```c
/* Efficient loop structures */
// Instead of:
for(uint8_t i = 0; i < 100; i++) { /* code */ }

// Use countdown when possible (faster comparison with zero):
for(uint8_t i = 100; i > 0; i--) { /* code */ }

/* Unroll small loops for speed */
// Instead of:
for(uint8_t i = 0; i < 4; i++) {
    process_data(i);
}

// Use:
process_data(0);
process_data(1);
process_data(2);
process_data(3);
```

#### Register Access Optimization
```c
/* Cache register values when accessing multiple times */
// Instead of:
if(PINB & (1 << PB0)) { /* */ }
if(PINB & (1 << PB1)) { /* */ }
if(PINB & (1 << PB2)) { /* */ }

// Use:
uint8_t pin_state = PINB;
if(pin_state & (1 << PB0)) { /* */ }
if(pin_state & (1 << PB1)) { /* */ }
if(pin_state & (1 << PB2)) { /* */ }
```

### Common Pitfalls & Solutions

#### Watchdog Timer Issues
```c
/* Disable watchdog on startup (in case it was enabled) */
void disable_watchdog(void) {
    wdt_reset();
    MCUSR &= ~(1 << WDRF);
    WDTCR |= (1 << WDCE) | (1 << WDE);
    WDTCR = 0x00;
}
```

#### Brown-out Detection
```c
/* Check for brown-out reset */
void check_reset_source(void) {
    if(MCUSR & (1 << BORF)) {
        // Brown-out reset occurred
        // Take appropriate action
        MCUSR &= ~(1 << BORF);  // Clear flag
    }
}
```

### Development Environment Setup

#### Makefile Template
```makefile
# ATmega32 Makefile Template
MCU = atmega32
F_CPU = 16000000UL
CC = avr-gcc
OBJCOPY = avr-objcopy
OBJDUMP = avr-objdump
SIZE = avr-size
PROGRAMMER = usbasp

CFLAGS = -Wall -g -Os -mmcu=$(MCU) -DF_CPU=$(F_CPU)
TARGET = main
SOURCES = main.c DIO_program.c LCD_program.c

all: $(TARGET).hex

$(TARGET).hex: $(TARGET).elf
	$(OBJCOPY) -O ihex -R .eeprom $< $@

$(TARGET).elf: $(SOURCES)
	$(CC) $(CFLAGS) -o $@ $^
	$(SIZE) $@

flash: $(TARGET).hex
	avrdude -c $(PROGRAMMER) -p $(MCU) -U flash:w:$<

clean:
	rm -f $(TARGET).elf $(TARGET).hex

.PHONY: all flash clean
```

#### Eclipse CDT Configuration
```
Project Properties → C/C++ Build → Settings:
- Tool Settings → AVR Compiler → Optimization: -Os
- Tool Settings → AVR Compiler → Language: C99
- Tool Settings → AVR Compiler → Preprocessor:
  Add: F_CPU=16000000UL, __AVR_ATmega32__
- Tool Settings → AVR Linker → General:
  No startup files, No default libraries if using custom startup
```

### Testing Strategies

#### Hardware-in-the-Loop Testing
```c
/* Automated test framework */
typedef struct {
    char test_name[32];
    boolean (*test_function)(void);
    boolean result;
} TestCase_t;

boolean test_gpio_toggle(void) {
    DIO_voidSetPinDirection(DIO_PORTA, DIO_PIN0, DIO_PIN_OUTPUT);
    DIO_voidSetPinValue(DIO_PORTA, DIO_PIN0, DIO_PIN_HIGH);
    _delay_ms(1);
    uint8_t high_state = DIO_u8GetPinValue(DIO_PORTA, DIO_PIN0);
    
    DIO_voidSetPinValue(DIO_PORTA, DIO_PIN0, DIO_PIN_LOW);
    _delay_ms(1);
    uint8_t low_state = DIO_u8GetPinValue(DIO_PORTA, DIO_PIN0);
    
    return (high_state == 1) && (low_state == 0);
}

TestCase_t test_cases[] = {
    {"GPIO Toggle Test", test_gpio_toggle, FALSE},
    // Add more tests
};

void run_all_tests(void) {
    uint8_t passed = 0, total = sizeof(test_cases) / sizeof(TestCase_t);
    
    for(uint8_t i = 0; i < total; i++) {
        test_cases[i].result = test_cases[i].test_function();
        if(test_cases[i].result) passed++;
    }
    
    // Report results via UART or LCD
}
```

These notes provide practical, tested solutions for common ATmega32 development challenges and should serve as your quick reference during development.