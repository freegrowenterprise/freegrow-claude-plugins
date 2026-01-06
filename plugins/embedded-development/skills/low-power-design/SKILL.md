---
name: low-power-design
description: >
  Low-power embedded system design patterns including sleep modes, power domains, wake sources,
  and battery life optimization. Use when designing battery-powered devices or optimizing power consumption.
---

# Low-Power Embedded Design

Design patterns and techniques for ultra-low-power embedded systems.

## When to Use

- Designing battery-powered devices
- Optimizing power consumption
- Implementing sleep modes
- Managing power domains
- Estimating battery life

## Power Management Architecture

### System Power States

```c
typedef enum {
    POWER_STATE_ACTIVE,      // Full operation
    POWER_STATE_IDLE,        // CPU idle, peripherals active
    POWER_STATE_SLEEP,       // Low-power sleep, fast wake
    POWER_STATE_DEEP_SLEEP,  // Ultra-low power, slow wake
    POWER_STATE_SHUTDOWN     // Minimal power, full restart
} power_state_t;

typedef struct {
    const char *name;
    uint32_t current_ua;     // Typical current draw
    uint32_t wake_time_us;   // Time to full operation
    uint32_t retention;      // RAM/register retention flags
} power_mode_info_t;

// Example: STM32L4 power modes
static const power_mode_info_t stm32l4_modes[] = {
    {"Run",        5000,    0,      0xFFFFFFFF},  // 5mA @ 48MHz
    {"Low-Power Run", 100,  0,      0xFFFFFFFF},  // 100uA @ 2MHz
    {"Sleep",      1000,    1,      0xFFFFFFFF},  // 1mA
    {"Low-Power Sleep", 30, 1,      0xFFFFFFFF},  // 30uA
    {"Stop 0",     1.4,     35,     0xFFFFFFFF},  // 1.4uA, RAM retained
    {"Stop 1",     0.8,     35,     0xFFFFFFFF},  // 0.8uA, RAM retained
    {"Stop 2",     0.3,     35,     0xFFFFFFFF},  // 0.3uA, RAM retained
    {"Standby",    0.2,     50,     0x00000000},  // 0.2uA, no retention
    {"Shutdown",   0.03,    200,    0x00000000},  // 30nA
};
```

### Power Manager Implementation

```c
typedef struct {
    power_state_t current_state;
    uint32_t activity_mask;         // Active peripheral mask
    uint32_t wake_sources;          // Enabled wake sources
    uint32_t sleep_blocked;         // Sleep blockers
    TickType_t last_activity;       // For idle detection
    void (*pre_sleep_callback)(power_state_t);
    void (*post_wake_callback)(power_state_t, uint32_t wake_reason);
} power_manager_t;

static power_manager_t pm;

// Register activity to prevent sleep
void pm_register_activity(uint32_t activity_id) {
    taskENTER_CRITICAL();
    pm.activity_mask |= (1 << activity_id);
    pm.last_activity = xTaskGetTickCount();
    taskEXIT_CRITICAL();
}

void pm_unregister_activity(uint32_t activity_id) {
    taskENTER_CRITICAL();
    pm.activity_mask &= ~(1 << activity_id);
    taskEXIT_CRITICAL();
}

// Block sleep (e.g., during critical operation)
void pm_block_sleep(uint32_t reason) {
    pm.sleep_blocked |= reason;
}

void pm_unblock_sleep(uint32_t reason) {
    pm.sleep_blocked &= ~reason;
}

// Determine optimal power state
power_state_t pm_get_target_state(void) {
    if (pm.sleep_blocked) {
        return POWER_STATE_ACTIVE;
    }

    if (pm.activity_mask) {
        return POWER_STATE_IDLE;
    }

    TickType_t idle_time = xTaskGetTickCount() - pm.last_activity;

    if (idle_time > pdMS_TO_TICKS(30000)) {  // 30s idle
        return POWER_STATE_DEEP_SLEEP;
    } else if (idle_time > pdMS_TO_TICKS(1000)) {  // 1s idle
        return POWER_STATE_SLEEP;
    }

    return POWER_STATE_IDLE;
}
```

## Sleep Mode Implementation

### Cortex-M Sleep Entry

```c
#include "stm32l4xx.h"

void enter_sleep_mode(void) {
    // Clear SLEEPDEEP bit for normal sleep
    SCB->SCR &= ~SCB_SCR_SLEEPDEEP_Msk;

    // Wait for interrupt
    __WFI();
}

void enter_stop_mode(uint8_t stop_level) {
    // Set SLEEPDEEP bit
    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;

    // Configure Stop mode
    PWR->CR1 &= ~PWR_CR1_LPMS_Msk;
    PWR->CR1 |= (stop_level & 0x07) << PWR_CR1_LPMS_Pos;

    // Enter Stop mode
    __WFI();

    // After wake: restore clocks
    SystemClock_Config();
}

void enter_standby_mode(void) {
    // Enable wakeup pins before entering
    PWR->CR3 |= PWR_CR3_EWUP1;

    // Set SLEEPDEEP
    SCB->SCR |= SCB_SCR_SLEEPDEEP_Msk;

    // Select Standby mode
    PWR->CR1 |= PWR_CR1_LPMS_STANDBY;

    // Enter Standby
    __WFI();

    // Should not reach here - reset on wake
}
```

### Wake Source Configuration

```c
typedef enum {
    WAKE_SOURCE_RTC_ALARM   = (1 << 0),
    WAKE_SOURCE_RTC_WAKEUP  = (1 << 1),
    WAKE_SOURCE_GPIO        = (1 << 2),
    WAKE_SOURCE_UART        = (1 << 3),
    WAKE_SOURCE_I2C         = (1 << 4),
    WAKE_SOURCE_LPUART      = (1 << 5),
    WAKE_SOURCE_LPTIM       = (1 << 6),
    WAKE_SOURCE_COMP        = (1 << 7),
} wake_source_t;

void configure_wake_sources(uint32_t sources) {
    // RTC Wakeup Timer
    if (sources & WAKE_SOURCE_RTC_WAKEUP) {
        RTC->CR |= RTC_CR_WUTIE;  // Enable wakeup interrupt
        RTC->CR |= RTC_CR_WUTE;   // Enable wakeup timer
    }

    // GPIO wake (EXTI)
    if (sources & WAKE_SOURCE_GPIO) {
        // Configure EXTI for button pin
        EXTI->IMR1 |= EXTI_IMR1_IM0;   // Unmask line 0
        EXTI->RTSR1 |= EXTI_RTSR1_RT0; // Rising edge
        EXTI->EMR1 |= EXTI_EMR1_EM0;   // Event mask for wake

        // Enable in PWR for Stop/Standby
        PWR->CR3 |= PWR_CR3_EWUP1;
    }

    // LPUART wake (for BLE module, etc.)
    if (sources & WAKE_SOURCE_LPUART) {
        LPUART1->CR1 |= USART_CR1_UESM;  // Enable in Stop mode
        LPUART1->CR3 |= USART_CR3_WUS_1; // Wake on start bit
    }
}

uint32_t get_wake_reason(void) {
    uint32_t reason = 0;

    if (RTC->ISR & RTC_ISR_WUTF) {
        reason |= WAKE_SOURCE_RTC_WAKEUP;
        RTC->ISR &= ~RTC_ISR_WUTF;
    }

    if (EXTI->PR1 & EXTI_PR1_PIF0) {
        reason |= WAKE_SOURCE_GPIO;
        EXTI->PR1 = EXTI_PR1_PIF0;  // Clear pending
    }

    if (PWR->SR1 & PWR_SR1_WUF1) {
        reason |= WAKE_SOURCE_GPIO;
        PWR->SCR = PWR_SCR_CWUF1;   // Clear flag
    }

    return reason;
}
```

## Peripheral Power Management

### Clock Gating

```c
// Peripheral clock management
typedef struct {
    volatile uint32_t *enable_reg;
    uint32_t enable_bit;
    const char *name;
} periph_clock_t;

static const periph_clock_t periph_clocks[] = {
    {&RCC->AHB2ENR, RCC_AHB2ENR_GPIOAEN, "GPIOA"},
    {&RCC->AHB2ENR, RCC_AHB2ENR_GPIOBEN, "GPIOB"},
    {&RCC->APB1ENR1, RCC_APB1ENR1_USART2EN, "USART2"},
    {&RCC->APB1ENR1, RCC_APB1ENR1_I2C1EN, "I2C1"},
    {&RCC->APB2ENR, RCC_APB2ENR_SPI1EN, "SPI1"},
    // ... more peripherals
};

void periph_clock_enable(uint8_t index) {
    *periph_clocks[index].enable_reg |= periph_clocks[index].enable_bit;
    // Required delay after enable
    __DSB();
    __NOP(); __NOP();
}

void periph_clock_disable(uint8_t index) {
    *periph_clocks[index].enable_reg &= ~periph_clocks[index].enable_bit;
}

// Disable all unused clocks before sleep
void disable_unused_clocks(void) {
    // Keep only essential clocks
    RCC->AHB2ENR = RCC_AHB2ENR_GPIOAEN;  // Only GPIOA for wake pin
    RCC->APB1ENR1 = RCC_APB1ENR1_RTCAPBEN;
    RCC->APB1ENR2 = 0;
    RCC->APB2ENR = 0;
}
```

### Power Domain Control

```c
// External peripheral power control
typedef struct {
    gpio_pin_t power_pin;
    uint32_t startup_delay_ms;
    bool is_enabled;
} power_domain_t;

static power_domain_t power_domains[] = {
    {.power_pin = {GPIOB, 5}, .startup_delay_ms = 10, .is_enabled = false},  // Sensor
    {.power_pin = {GPIOB, 6}, .startup_delay_ms = 100, .is_enabled = false}, // Radio
    {.power_pin = {GPIOB, 7}, .startup_delay_ms = 50, .is_enabled = false},  // Flash
};

void power_domain_enable(uint8_t domain_id) {
    power_domain_t *pd = &power_domains[domain_id];

    if (!pd->is_enabled) {
        gpio_write(pd->power_pin, 1);
        delay_ms(pd->startup_delay_ms);
        pd->is_enabled = true;
    }
}

void power_domain_disable(uint8_t domain_id) {
    power_domain_t *pd = &power_domains[domain_id];

    if (pd->is_enabled) {
        gpio_write(pd->power_pin, 0);
        pd->is_enabled = false;
    }
}

// Automatic power-down on idle
void power_domain_check_idle(void) {
    for (int i = 0; i < ARRAY_SIZE(power_domains); i++) {
        if (power_domains[i].is_enabled &&
            power_domain_idle_time[i] > POWER_DOWN_TIMEOUT) {
            power_domain_disable(i);
        }
    }
}
```

## Battery Life Estimation

### Current Profiling

```c
typedef struct {
    const char *state_name;
    uint32_t current_ua;
    uint32_t duration_ms;
} power_profile_entry_t;

// Example: BLE sensor device profile
static const power_profile_entry_t ble_sensor_profile[] = {
    {"Deep Sleep",      5,      9900},   // 9.9s in deep sleep
    {"Wake + Init",     5000,   5},      // 5ms wake
    {"Sensor Read",     2000,   10},     // 10ms sensor
    {"BLE Advertise",   15000,  3},      // 3ms BLE TX
    {"BLE Connect",     12000,  30},     // 30ms connection
    {"Data TX",         10000,  5},      // 5ms data
    {"BLE Disconnect",  8000,   2},      // 2ms disconnect
    {"Shutdown",        1000,   5},      // 5ms shutdown
};

// Calculate average current
uint32_t calculate_average_current(const power_profile_entry_t *profile,
                                   size_t entries) {
    uint64_t total_charge_uas = 0;  // uA * ms = uA-ms
    uint32_t total_time_ms = 0;

    for (size_t i = 0; i < entries; i++) {
        total_charge_uas += (uint64_t)profile[i].current_ua * profile[i].duration_ms;
        total_time_ms += profile[i].duration_ms;
    }

    return (uint32_t)(total_charge_uas / total_time_ms);
}

// Estimate battery life
float estimate_battery_life_days(uint32_t battery_mah, uint32_t avg_current_ua) {
    // Battery life (hours) = Capacity (mAh) / Current (mA)
    // Convert uA to mA: divide by 1000
    float current_ma = avg_current_ua / 1000.0f;
    float life_hours = battery_mah / current_ma;
    return life_hours / 24.0f;
}

// Example usage
void print_battery_estimate(void) {
    uint32_t avg_ua = calculate_average_current(ble_sensor_profile,
                                                ARRAY_SIZE(ble_sensor_profile));
    float days = estimate_battery_life_days(230, avg_ua);  // CR2032: 230mAh

    printf("Average current: %lu uA\n", avg_ua);
    printf("Estimated battery life: %.1f days\n", days);
}
```

## Low-Power Coding Techniques

### Efficient Polling

```c
// WRONG: Busy polling
while (!sensor_data_ready()) {
    // Wastes power
}

// BETTER: Polling with sleep
while (!sensor_data_ready()) {
    enter_sleep_mode();  // WFI until any interrupt
}

// BEST: Interrupt-driven
volatile bool data_ready = false;

void EXTI_IRQHandler(void) {
    data_ready = true;
    EXTI->PR1 = EXTI_PR1_PIF0;
}

void wait_for_sensor_data(void) {
    while (!data_ready) {
        enter_sleep_mode();
    }
    data_ready = false;
}
```

### Timer Consolidation

```c
// WRONG: Multiple timers
void init_multiple_timers(void) {
    // Timer 1: 100ms periodic
    // Timer 2: 1s periodic
    // Timer 3: 5s periodic
    // Each timer costs power
}

// RIGHT: Single timer with software timers
typedef struct {
    uint32_t period_ticks;
    uint32_t remaining;
    void (*callback)(void);
} soft_timer_t;

#define MAX_SOFT_TIMERS 8
static soft_timer_t soft_timers[MAX_SOFT_TIMERS];
static uint32_t base_timer_period = 100;  // 100ms base

void base_timer_callback(void) {
    for (int i = 0; i < MAX_SOFT_TIMERS; i++) {
        if (soft_timers[i].callback && soft_timers[i].remaining > 0) {
            soft_timers[i].remaining -= base_timer_period;
            if (soft_timers[i].remaining == 0) {
                soft_timers[i].remaining = soft_timers[i].period_ticks;
                soft_timers[i].callback();
            }
        }
    }
}
```

### Voltage Scaling

```c
// Dynamic voltage and frequency scaling
typedef struct {
    uint32_t frequency_hz;
    uint8_t voltage_range;  // 1=1.0V, 2=1.2V (STM32L4)
    uint32_t flash_latency;
} dvfs_profile_t;

static const dvfs_profile_t dvfs_profiles[] = {
    {2000000,   1, 0},  // 2MHz @ 1.0V - lowest power
    {16000000,  1, 1},  // 16MHz @ 1.0V
    {48000000,  2, 2},  // 48MHz @ 1.2V
    {80000000,  2, 4},  // 80MHz @ 1.2V - highest performance
};

void set_performance_level(uint8_t level) {
    const dvfs_profile_t *profile = &dvfs_profiles[level];

    // Must reduce frequency before reducing voltage
    // Must increase voltage before increasing frequency

    if (profile->voltage_range > current_voltage_range) {
        set_voltage_range(profile->voltage_range);
    }

    set_system_clock(profile->frequency_hz);
    set_flash_latency(profile->flash_latency);

    if (profile->voltage_range < current_voltage_range) {
        set_voltage_range(profile->voltage_range);
    }
}
```

## Best Practices

### DO
- Measure actual current consumption with multimeter/power analyzer
- Use sleep modes aggressively
- Disable unused peripherals and their clocks
- Use DMA to allow CPU sleep during transfers
- Implement tickless idle in RTOS

### DON'T
- Leave GPIOs floating (configure as input with pull-up/down or output)
- Keep debug interfaces enabled in production
- Use busy-wait polling
- Leave LEDs on unnecessarily
- Ignore internal pull-up/pull-down power consumption

## Resources

- [ARM Low Power Design](https://developer.arm.com/documentation/100699/latest)
- [STM32 Low Power Modes](https://www.st.com/resource/en/application_note/an4621.pdf)
- [Nordic Power Profiler](https://www.nordicsemi.com/Products/Development-tools/Power-Profiler-Kit-2)
- "Designing Embedded Hardware" by John Catsoulis
