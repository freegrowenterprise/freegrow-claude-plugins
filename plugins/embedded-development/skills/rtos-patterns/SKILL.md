---
name: rtos-patterns
description: >
  Real-time operating system design patterns for FreeRTOS, Zephyr, and ThreadX. Task design,
  synchronization primitives, inter-task communication, and deterministic scheduling.
  Use when implementing RTOS-based applications or debugging concurrency issues.
---

# RTOS Design Patterns

Master real-time operating system patterns for reliable embedded applications.

## When to Use

- Designing multi-task embedded applications
- Choosing synchronization primitives
- Debugging deadlocks or race conditions
- Optimizing task priorities and scheduling
- Migrating between RTOS platforms

## Task Design Patterns

### Producer-Consumer Pattern

The most common RTOS pattern for data flow between tasks.

```c
// FreeRTOS Implementation
#define QUEUE_LENGTH 10
#define ITEM_SIZE sizeof(SensorData_t)

typedef struct {
    uint32_t timestamp;
    int16_t temperature;
    uint16_t humidity;
} SensorData_t;

static QueueHandle_t sensor_queue;

// Producer Task - Reads sensors
void producer_task(void *pvParameters) {
    SensorData_t data;

    while (1) {
        data.timestamp = xTaskGetTickCount();
        data.temperature = read_temperature_sensor();
        data.humidity = read_humidity_sensor();

        // Block if queue full (with timeout for safety)
        if (xQueueSend(sensor_queue, &data, pdMS_TO_TICKS(100)) != pdTRUE) {
            // Handle queue full - log, drop, or overwrite oldest
            log_warning("Sensor queue full, dropping sample");
        }

        vTaskDelay(pdMS_TO_TICKS(100)); // 10 Hz sampling
    }
}

// Consumer Task - Processes data
void consumer_task(void *pvParameters) {
    SensorData_t data;

    while (1) {
        // Block until data available
        if (xQueueReceive(sensor_queue, &data, portMAX_DELAY) == pdTRUE) {
            process_sensor_data(&data);
            store_to_flash(&data);
        }
    }
}
```

### State Machine Pattern

Clean state machine implementation with RTOS.

```c
typedef enum {
    STATE_IDLE,
    STATE_INITIALIZING,
    STATE_RUNNING,
    STATE_ERROR,
    STATE_SHUTDOWN
} SystemState_t;

typedef enum {
    EVENT_START,
    EVENT_STOP,
    EVENT_ERROR,
    EVENT_RESET,
    EVENT_TIMEOUT
} SystemEvent_t;

typedef struct {
    SystemEvent_t event;
    void *data;
} EventMessage_t;

static QueueHandle_t event_queue;
static SystemState_t current_state = STATE_IDLE;

// State handlers
static SystemState_t handle_idle(EventMessage_t *msg);
static SystemState_t handle_initializing(EventMessage_t *msg);
static SystemState_t handle_running(EventMessage_t *msg);
static SystemState_t handle_error(EventMessage_t *msg);

typedef SystemState_t (*StateHandler_t)(EventMessage_t *msg);

static const StateHandler_t state_handlers[] = {
    [STATE_IDLE] = handle_idle,
    [STATE_INITIALIZING] = handle_initializing,
    [STATE_RUNNING] = handle_running,
    [STATE_ERROR] = handle_error,
};

void state_machine_task(void *pvParameters) {
    EventMessage_t msg;

    while (1) {
        if (xQueueReceive(event_queue, &msg, portMAX_DELAY) == pdTRUE) {
            SystemState_t next_state = state_handlers[current_state](&msg);

            if (next_state != current_state) {
                on_state_exit(current_state);
                current_state = next_state;
                on_state_enter(current_state);
            }
        }
    }
}

// Post event from anywhere (including ISR)
BaseType_t post_event(SystemEvent_t event, void *data) {
    EventMessage_t msg = { .event = event, .data = data };
    return xQueueSend(event_queue, &msg, 0);
}

BaseType_t post_event_from_isr(SystemEvent_t event, void *data) {
    EventMessage_t msg = { .event = event, .data = data };
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(event_queue, &msg, &woken);
    return woken;
}
```

### Resource Guard Pattern

Safe resource access with automatic cleanup.

```c
// Resource guard using mutex
typedef struct {
    SemaphoreHandle_t mutex;
    void *resource;
    bool acquired;
} ResourceGuard_t;

static inline bool resource_acquire(ResourceGuard_t *guard,
                                    SemaphoreHandle_t mutex,
                                    void *resource,
                                    TickType_t timeout) {
    guard->mutex = mutex;
    guard->resource = resource;
    guard->acquired = false;

    if (xSemaphoreTake(mutex, timeout) == pdTRUE) {
        guard->acquired = true;
        return true;
    }
    return false;
}

static inline void resource_release(ResourceGuard_t *guard) {
    if (guard->acquired) {
        xSemaphoreGive(guard->mutex);
        guard->acquired = false;
    }
}

// Usage
void safe_resource_access(void) {
    ResourceGuard_t guard;

    if (resource_acquire(&guard, spi_mutex, &spi_bus, pdMS_TO_TICKS(100))) {
        // Safe to use SPI
        spi_transfer(data, len);
        resource_release(&guard);
    } else {
        handle_resource_timeout();
    }
}
```

## Synchronization Primitives

### Mutex vs Binary Semaphore

| Feature | Mutex | Binary Semaphore |
|---------|-------|------------------|
| Priority Inheritance | Yes | No |
| Ownership | Yes (task must give) | No |
| Recursive Taking | Optional | No |
| ISR Give | No | Yes |
| Use Case | Resource protection | Event signaling |

```c
// Mutex - for resource protection
SemaphoreHandle_t i2c_mutex = xSemaphoreCreateMutex();

void i2c_write_protected(uint8_t addr, uint8_t *data, size_t len) {
    xSemaphoreTake(i2c_mutex, portMAX_DELAY);
    i2c_write(addr, data, len);
    xSemaphoreGive(i2c_mutex);
}

// Binary Semaphore - for ISR-to-task signaling
SemaphoreHandle_t uart_rx_sem = xSemaphoreCreateBinary();

void UART_IRQHandler(void) {
    // Handle UART RX
    BaseType_t woken = pdFALSE;
    xSemaphoreGiveFromISR(uart_rx_sem, &woken);
    portYIELD_FROM_ISR(woken);
}

void uart_task(void *pvParameters) {
    while (1) {
        xSemaphoreTake(uart_rx_sem, portMAX_DELAY);
        process_uart_data();
    }
}
```

### Event Groups for Multiple Events

```c
#define EVENT_BUTTON_PRESSED  (1 << 0)
#define EVENT_TIMER_EXPIRED   (1 << 1)
#define EVENT_DATA_READY      (1 << 2)
#define EVENT_ERROR_OCCURRED  (1 << 3)

static EventGroupHandle_t system_events;

// Wait for multiple events
void event_handler_task(void *pvParameters) {
    while (1) {
        EventBits_t bits = xEventGroupWaitBits(
            system_events,
            EVENT_BUTTON_PRESSED | EVENT_DATA_READY,  // Wait for these
            pdTRUE,   // Clear on exit
            pdFALSE,  // Wait for ANY (pdTRUE for ALL)
            portMAX_DELAY
        );

        if (bits & EVENT_BUTTON_PRESSED) {
            handle_button();
        }
        if (bits & EVENT_DATA_READY) {
            handle_data();
        }
    }
}

// Set events from ISR
void GPIO_IRQHandler(void) {
    BaseType_t woken = pdFALSE;
    xEventGroupSetBitsFromISR(system_events, EVENT_BUTTON_PRESSED, &woken);
    portYIELD_FROM_ISR(woken);
}
```

## Priority Design

### Rate Monotonic Scheduling (RMS)

For periodic tasks, shorter period = higher priority.

```c
// Task Configuration based on RMS
typedef struct {
    const char *name;
    uint32_t period_ms;
    uint16_t stack_size;
    UBaseType_t priority;  // Calculated from period
} TaskConfig_t;

static const TaskConfig_t task_configs[] = {
    // Shorter period = higher priority
    {"Motor_Control",  1,    256, configMAX_PRIORITIES - 1},  // 1ms
    {"Sensor_Read",    10,   512, configMAX_PRIORITIES - 2},  // 10ms
    {"Display_Update", 100,  1024, configMAX_PRIORITIES - 3}, // 100ms
    {"Logging",        1000, 512, tskIDLE_PRIORITY + 1},      // 1s
};

// Utilization check (must be < 69% for schedulability)
// U = Σ(Ci/Ti) where Ci=execution time, Ti=period
```

### Priority Inversion Prevention

```c
// WRONG: Priority inversion possible
SemaphoreHandle_t sem = xSemaphoreCreateBinary();

// RIGHT: Use mutex with priority inheritance
SemaphoreHandle_t mutex = xSemaphoreCreateMutex();

// Priority inheritance automatically raises low-priority
// task's priority when high-priority task is blocked
```

## Memory Management

### Static Allocation (Recommended)

```c
// FreeRTOS static allocation
static StaticTask_t task_tcb;
static StackType_t task_stack[configMINIMAL_STACK_SIZE];

static StaticQueue_t queue_struct;
static uint8_t queue_storage[10 * sizeof(Message_t)];

static StaticSemaphore_t mutex_struct;

void create_resources(void) {
    // Static task
    TaskHandle_t task = xTaskCreateStatic(
        task_function, "Task",
        configMINIMAL_STACK_SIZE, NULL, 1,
        task_stack, &task_tcb
    );

    // Static queue
    QueueHandle_t queue = xQueueCreateStatic(
        10, sizeof(Message_t),
        queue_storage, &queue_struct
    );

    // Static mutex
    SemaphoreHandle_t mutex = xSemaphoreCreateMutexStatic(&mutex_struct);
}
```

### Memory Pool Pattern

```c
// Fixed-size block allocator for deterministic allocation
#define POOL_BLOCK_SIZE  64
#define POOL_BLOCK_COUNT 32

typedef struct PoolBlock {
    struct PoolBlock *next;
    uint8_t data[POOL_BLOCK_SIZE];
} PoolBlock_t;

static PoolBlock_t pool_memory[POOL_BLOCK_COUNT];
static PoolBlock_t *free_list;
static SemaphoreHandle_t pool_mutex;

void pool_init(void) {
    pool_mutex = xSemaphoreCreateMutex();
    free_list = &pool_memory[0];

    for (int i = 0; i < POOL_BLOCK_COUNT - 1; i++) {
        pool_memory[i].next = &pool_memory[i + 1];
    }
    pool_memory[POOL_BLOCK_COUNT - 1].next = NULL;
}

void *pool_alloc(void) {
    void *block = NULL;

    xSemaphoreTake(pool_mutex, portMAX_DELAY);
    if (free_list) {
        block = free_list->data;
        free_list = free_list->next;
    }
    xSemaphoreGive(pool_mutex);

    return block;
}

void pool_free(void *ptr) {
    if (!ptr) return;

    PoolBlock_t *block = container_of(ptr, PoolBlock_t, data);

    xSemaphoreTake(pool_mutex, portMAX_DELAY);
    block->next = free_list;
    free_list = block;
    xSemaphoreGive(pool_mutex);
}
```

## Debugging Patterns

### Stack Overflow Detection

```c
// FreeRTOS hook
void vApplicationStackOverflowHook(TaskHandle_t task, char *name) {
    // Log and halt - stack is already corrupted
    volatile bool halt = true;
    (void)task;
    (void)name;
    while (halt) { }
}

// Runtime monitoring
void monitor_task(void *pvParameters) {
    while (1) {
        TaskStatus_t status;
        vTaskGetInfo(NULL, &status, pdTRUE, eRunning);

        UBaseType_t high_water = status.usStackHighWaterMark;
        if (high_water < 50) {
            log_warning("Stack low: %s has %u words", status.pcTaskName, high_water);
        }

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### Deadlock Detection

```c
// Simple timeout-based deadlock detection
#define MUTEX_TIMEOUT_MS 5000

bool safe_mutex_take(SemaphoreHandle_t mutex, const char *name) {
    if (xSemaphoreTake(mutex, pdMS_TO_TICKS(MUTEX_TIMEOUT_MS)) != pdTRUE) {
        log_error("Potential deadlock acquiring %s", name);
        // Dump task states
        char buffer[512];
        vTaskList(buffer);
        log_error("Task states:\n%s", buffer);
        return false;
    }
    return true;
}
```

## Best Practices

### DO
- Use static allocation for all RTOS objects
- Set appropriate stack sizes (measure with high water mark)
- Use timeouts on all blocking calls
- Document task priorities and dependencies
- Keep critical sections short

### DON'T
- Call blocking functions from ISR
- Use recursive mutex unless absolutely necessary
- Assume task execution order at same priority
- Allocate large buffers on stack
- Forget to release mutexes (use guard pattern)

## Resources

- [FreeRTOS API Reference](https://www.freertos.org/a00106.html)
- [Zephyr Kernel Services](https://docs.zephyrproject.org/latest/kernel/services/index.html)
- [Azure RTOS ThreadX](https://docs.microsoft.com/en-us/azure/rtos/threadx/)
- "Making Embedded Systems" by Elecia White
