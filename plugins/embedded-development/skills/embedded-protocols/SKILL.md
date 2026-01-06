---
name: embedded-protocols
description: >
  Communication protocols for embedded systems including CAN bus, Modbus, I2C, SPI, UART advanced patterns,
  and IoT protocols (MQTT, CoAP). Use when implementing industrial protocols or embedded communication.
---

# Embedded Communication Protocols

Comprehensive guide to implementing communication protocols in embedded C.

## When to Use

- Implementing CAN bus communication
- Industrial Modbus RTU/TCP integration
- Advanced I2C/SPI driver development
- MQTT/CoAP for IoT applications
- Multi-protocol gateway development

## CAN Bus

### CAN 2.0B Implementation

```c
#include <stdint.h>
#include <stdbool.h>

// CAN message structure
typedef struct {
    uint32_t id;           // 11-bit or 29-bit identifier
    bool extended;         // Extended (29-bit) ID flag
    bool remote;           // Remote frame flag
    uint8_t dlc;           // Data length code (0-8)
    uint8_t data[8];       // Data bytes
    uint32_t timestamp;    // Reception timestamp
} can_message_t;

// CAN filter configuration
typedef struct {
    uint32_t id;
    uint32_t mask;
    bool extended;
} can_filter_t;

// CAN driver interface
typedef struct {
    int (*init)(uint32_t bitrate);
    int (*send)(const can_message_t *msg, uint32_t timeout_ms);
    int (*receive)(can_message_t *msg, uint32_t timeout_ms);
    int (*set_filter)(uint8_t bank, const can_filter_t *filter);
    int (*get_error_state)(void);
} can_driver_t;

// Standard CAN bitrates
#define CAN_BITRATE_125K   125000
#define CAN_BITRATE_250K   250000
#define CAN_BITRATE_500K   500000
#define CAN_BITRATE_1M     1000000

// Example: CANopen-style message handling
#define CANOPEN_NMT_ID        0x000
#define CANOPEN_SYNC_ID       0x080
#define CANOPEN_EMCY_BASE     0x080
#define CANOPEN_PDO1_TX_BASE  0x180
#define CANOPEN_PDO1_RX_BASE  0x200
#define CANOPEN_SDO_TX_BASE   0x580
#define CANOPEN_SDO_RX_BASE   0x600
#define CANOPEN_HEARTBEAT     0x700

void can_process_message(const can_message_t *msg) {
    uint16_t cob_id = msg->id & 0x780;  // Function code
    uint8_t node_id = msg->id & 0x7F;    // Node ID

    switch (cob_id) {
    case CANOPEN_NMT_ID:
        handle_nmt_command(msg->data[0], msg->data[1]);
        break;
    case CANOPEN_SYNC_ID:
        handle_sync();
        break;
    case CANOPEN_PDO1_RX_BASE:
        handle_pdo_rx(1, node_id, msg->data, msg->dlc);
        break;
    case CANOPEN_SDO_RX_BASE:
        handle_sdo_request(node_id, msg->data);
        break;
    default:
        // Unknown message type
        break;
    }
}
```

### CAN Error Handling

```c
typedef enum {
    CAN_ERROR_NONE = 0,
    CAN_ERROR_WARNING,      // Error counter > 96
    CAN_ERROR_PASSIVE,      // Error counter > 127
    CAN_ERROR_BUS_OFF       // Error counter > 255
} can_error_state_t;

typedef struct {
    uint8_t tx_error_count;
    uint8_t rx_error_count;
    can_error_state_t state;
    uint32_t last_error_code;
    uint32_t error_count;
} can_error_info_t;

// Bus-off recovery
void can_bus_off_recovery(can_driver_t *drv) {
    // Automatic recovery after 128 * 11 recessive bits
    // Or manual re-initialization
    log_error("CAN bus-off detected, attempting recovery");
    drv->init(CAN_BITRATE_500K);
}
```

## Modbus

### Modbus RTU Implementation

```c
#include <stdint.h>

// Modbus function codes
#define MODBUS_FC_READ_COILS              0x01
#define MODBUS_FC_READ_DISCRETE_INPUTS    0x02
#define MODBUS_FC_READ_HOLDING_REGISTERS  0x03
#define MODBUS_FC_READ_INPUT_REGISTERS    0x04
#define MODBUS_FC_WRITE_SINGLE_COIL       0x05
#define MODBUS_FC_WRITE_SINGLE_REGISTER   0x06
#define MODBUS_FC_WRITE_MULTIPLE_COILS    0x0F
#define MODBUS_FC_WRITE_MULTIPLE_REGISTERS 0x10

// Modbus exception codes
#define MODBUS_EX_ILLEGAL_FUNCTION        0x01
#define MODBUS_EX_ILLEGAL_DATA_ADDRESS    0x02
#define MODBUS_EX_ILLEGAL_DATA_VALUE      0x03
#define MODBUS_EX_SLAVE_DEVICE_FAILURE    0x04

// RTU frame structure
typedef struct {
    uint8_t slave_addr;
    uint8_t function_code;
    uint8_t data[252];      // Max Modbus data size
    uint16_t data_len;
    uint16_t crc;
} modbus_frame_t;

// CRC-16 calculation (Modbus polynomial 0x8005)
uint16_t modbus_crc16(const uint8_t *data, uint16_t len) {
    uint16_t crc = 0xFFFF;

    for (uint16_t i = 0; i < len; i++) {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++) {
            if (crc & 0x0001) {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    return crc;
}

// Modbus slave handler
typedef struct {
    uint8_t address;
    uint16_t *holding_registers;
    uint16_t holding_reg_count;
    uint16_t *input_registers;
    uint16_t input_reg_count;
    uint8_t *coils;
    uint16_t coil_count;
} modbus_slave_t;

int modbus_process_request(modbus_slave_t *slave,
                           const uint8_t *request, uint16_t req_len,
                           uint8_t *response, uint16_t *resp_len) {
    if (req_len < 4) return -1;

    // Verify CRC
    uint16_t crc_received = (request[req_len-1] << 8) | request[req_len-2];
    uint16_t crc_calc = modbus_crc16(request, req_len - 2);
    if (crc_received != crc_calc) {
        return -1;  // CRC error
    }

    // Check address
    if (request[0] != slave->address && request[0] != 0) {
        return 0;  // Not for us
    }

    uint8_t fc = request[1];
    response[0] = slave->address;
    response[1] = fc;

    switch (fc) {
    case MODBUS_FC_READ_HOLDING_REGISTERS: {
        uint16_t start = (request[2] << 8) | request[3];
        uint16_t count = (request[4] << 8) | request[5];

        if (start + count > slave->holding_reg_count) {
            response[1] |= 0x80;
            response[2] = MODBUS_EX_ILLEGAL_DATA_ADDRESS;
            *resp_len = 5;  // addr + fc + exception + crc
        } else {
            response[2] = count * 2;  // Byte count
            for (uint16_t i = 0; i < count; i++) {
                response[3 + i*2] = slave->holding_registers[start + i] >> 8;
                response[4 + i*2] = slave->holding_registers[start + i] & 0xFF;
            }
            *resp_len = 3 + count * 2 + 2;
        }
        break;
    }
    case MODBUS_FC_WRITE_SINGLE_REGISTER: {
        uint16_t addr = (request[2] << 8) | request[3];
        uint16_t value = (request[4] << 8) | request[5];

        if (addr >= slave->holding_reg_count) {
            response[1] |= 0x80;
            response[2] = MODBUS_EX_ILLEGAL_DATA_ADDRESS;
            *resp_len = 5;
        } else {
            slave->holding_registers[addr] = value;
            // Echo back request
            memcpy(response + 2, request + 2, 4);
            *resp_len = 8;
        }
        break;
    }
    default:
        response[1] |= 0x80;
        response[2] = MODBUS_EX_ILLEGAL_FUNCTION;
        *resp_len = 5;
    }

    // Add CRC
    uint16_t crc = modbus_crc16(response, *resp_len - 2);
    response[*resp_len - 2] = crc & 0xFF;
    response[*resp_len - 1] = crc >> 8;

    return *resp_len;
}
```

## I2C Advanced Patterns

### I2C Bus Recovery

```c
// I2C bus recovery - toggle SCL to release stuck SDA
void i2c_bus_recovery(gpio_pin_t scl, gpio_pin_t sda) {
    // Configure as GPIO
    gpio_set_mode(scl, GPIO_MODE_OUTPUT_OD);
    gpio_set_mode(sda, GPIO_MODE_INPUT);

    // Clock out up to 9 bits to release slave
    for (int i = 0; i < 9; i++) {
        if (gpio_read(sda)) {
            break;  // SDA is high, bus is free
        }
        gpio_write(scl, 0);
        delay_us(5);
        gpio_write(scl, 1);
        delay_us(5);
    }

    // Generate STOP condition
    gpio_set_mode(sda, GPIO_MODE_OUTPUT_OD);
    gpio_write(sda, 0);
    delay_us(5);
    gpio_write(scl, 1);
    delay_us(5);
    gpio_write(sda, 1);
    delay_us(5);

    // Reconfigure for I2C
    gpio_set_mode(scl, GPIO_MODE_AF_OD);
    gpio_set_mode(sda, GPIO_MODE_AF_OD);
}

// Safe I2C transfer with retry and recovery
int i2c_transfer_safe(i2c_handle_t *i2c, uint8_t addr,
                      const uint8_t *tx, size_t tx_len,
                      uint8_t *rx, size_t rx_len,
                      int max_retries) {
    int ret;

    for (int attempt = 0; attempt <= max_retries; attempt++) {
        ret = i2c_transfer(i2c, addr, tx, tx_len, rx, rx_len);

        if (ret == 0) {
            return 0;  // Success
        }

        if (attempt < max_retries) {
            log_warning("I2C transfer failed, attempt %d/%d", attempt + 1, max_retries);
            i2c_bus_recovery(i2c->scl_pin, i2c->sda_pin);
            delay_ms(10);
        }
    }

    log_error("I2C transfer failed after %d attempts", max_retries + 1);
    return ret;
}
```

### I2C Device Scanning

```c
void i2c_scan(i2c_handle_t *i2c) {
    printf("Scanning I2C bus...\n");
    printf("     0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F\n");

    for (int addr = 0; addr < 128; addr++) {
        if (addr % 16 == 0) {
            printf("%02X: ", addr);
        }

        // Skip reserved addresses
        if (addr < 0x08 || addr > 0x77) {
            printf("   ");
        } else {
            // Try to read one byte
            uint8_t dummy;
            if (i2c_read(i2c, addr, &dummy, 1, 10) == 0) {
                printf("%02X ", addr);
            } else {
                printf("-- ");
            }
        }

        if ((addr + 1) % 16 == 0) {
            printf("\n");
        }
    }
}
```

## SPI Advanced Patterns

### SPI with DMA

```c
typedef struct {
    SPI_TypeDef *instance;
    DMA_Channel_TypeDef *tx_dma;
    DMA_Channel_TypeDef *rx_dma;
    gpio_pin_t cs_pin;
    volatile bool transfer_complete;
    void (*callback)(void *context);
    void *callback_context;
} spi_dma_handle_t;

// Asynchronous SPI transfer
int spi_transfer_dma(spi_dma_handle_t *spi,
                     const uint8_t *tx_buf, uint8_t *rx_buf, size_t len,
                     void (*callback)(void *), void *context) {
    if (len == 0 || len > 65535) return -1;

    spi->transfer_complete = false;
    spi->callback = callback;
    spi->callback_context = context;

    // Assert CS
    gpio_write(spi->cs_pin, 0);

    // Configure RX DMA
    spi->rx_dma->CNDTR = len;
    spi->rx_dma->CMAR = (uint32_t)rx_buf;
    spi->rx_dma->CCR |= DMA_CCR_EN;

    // Configure TX DMA
    spi->tx_dma->CNDTR = len;
    spi->tx_dma->CMAR = (uint32_t)tx_buf;
    spi->tx_dma->CCR |= DMA_CCR_EN;

    // Enable SPI DMA requests
    spi->instance->CR2 |= SPI_CR2_TXDMAEN | SPI_CR2_RXDMAEN;

    return 0;  // Transfer started
}

// DMA complete ISR
void DMA_Channel_IRQHandler(spi_dma_handle_t *spi) {
    // Disable DMA
    spi->tx_dma->CCR &= ~DMA_CCR_EN;
    spi->rx_dma->CCR &= ~DMA_CCR_EN;

    // Wait for SPI to finish
    while (spi->instance->SR & SPI_SR_BSY);

    // Deassert CS
    gpio_write(spi->cs_pin, 1);

    spi->transfer_complete = true;

    if (spi->callback) {
        spi->callback(spi->callback_context);
    }
}
```

## MQTT for IoT

### Lightweight MQTT Client

```c
#include <stdint.h>
#include <string.h>

// MQTT packet types
#define MQTT_CONNECT     0x10
#define MQTT_CONNACK     0x20
#define MQTT_PUBLISH     0x30
#define MQTT_PUBACK      0x40
#define MQTT_SUBSCRIBE   0x82
#define MQTT_SUBACK      0x90
#define MQTT_PINGREQ     0xC0
#define MQTT_PINGRESP    0xD0
#define MQTT_DISCONNECT  0xE0

typedef struct {
    int (*send)(const uint8_t *data, size_t len);
    int (*recv)(uint8_t *data, size_t len, uint32_t timeout_ms);
    void (*on_message)(const char *topic, const uint8_t *payload, size_t len);
    uint16_t packet_id;
    bool connected;
} mqtt_client_t;

// Encode remaining length (variable length encoding)
static size_t mqtt_encode_length(uint8_t *buf, size_t len) {
    size_t pos = 0;
    do {
        uint8_t byte = len % 128;
        len /= 128;
        if (len > 0) byte |= 0x80;
        buf[pos++] = byte;
    } while (len > 0);
    return pos;
}

// Build CONNECT packet
int mqtt_connect(mqtt_client_t *client, const char *client_id,
                 const char *username, const char *password,
                 uint16_t keepalive) {
    uint8_t packet[256];
    size_t pos = 0;

    // Variable header
    uint8_t var_header[10];
    size_t vh_len = 0;

    // Protocol name
    var_header[vh_len++] = 0x00; var_header[vh_len++] = 0x04;
    var_header[vh_len++] = 'M'; var_header[vh_len++] = 'Q';
    var_header[vh_len++] = 'T'; var_header[vh_len++] = 'T';

    // Protocol level (4 = 3.1.1)
    var_header[vh_len++] = 0x04;

    // Connect flags
    uint8_t flags = 0x02;  // Clean session
    if (username) flags |= 0x80;
    if (password) flags |= 0x40;
    var_header[vh_len++] = flags;

    // Keepalive
    var_header[vh_len++] = keepalive >> 8;
    var_header[vh_len++] = keepalive & 0xFF;

    // Calculate payload length
    size_t client_id_len = strlen(client_id);
    size_t payload_len = 2 + client_id_len;
    if (username) payload_len += 2 + strlen(username);
    if (password) payload_len += 2 + strlen(password);

    // Build packet
    packet[pos++] = MQTT_CONNECT;
    pos += mqtt_encode_length(&packet[pos], vh_len + payload_len);

    memcpy(&packet[pos], var_header, vh_len);
    pos += vh_len;

    // Client ID
    packet[pos++] = client_id_len >> 8;
    packet[pos++] = client_id_len & 0xFF;
    memcpy(&packet[pos], client_id, client_id_len);
    pos += client_id_len;

    // Username/Password
    if (username) {
        size_t len = strlen(username);
        packet[pos++] = len >> 8;
        packet[pos++] = len & 0xFF;
        memcpy(&packet[pos], username, len);
        pos += len;
    }
    if (password) {
        size_t len = strlen(password);
        packet[pos++] = len >> 8;
        packet[pos++] = len & 0xFF;
        memcpy(&packet[pos], password, len);
        pos += len;
    }

    return client->send(packet, pos);
}

// Publish message
int mqtt_publish(mqtt_client_t *client, const char *topic,
                 const uint8_t *payload, size_t payload_len,
                 uint8_t qos, bool retain) {
    uint8_t packet[512];
    size_t pos = 0;

    uint8_t flags = MQTT_PUBLISH;
    if (retain) flags |= 0x01;
    if (qos) flags |= (qos << 1);

    packet[pos++] = flags;

    // Calculate remaining length
    size_t topic_len = strlen(topic);
    size_t rem_len = 2 + topic_len + payload_len;
    if (qos > 0) rem_len += 2;  // Packet ID

    pos += mqtt_encode_length(&packet[pos], rem_len);

    // Topic
    packet[pos++] = topic_len >> 8;
    packet[pos++] = topic_len & 0xFF;
    memcpy(&packet[pos], topic, topic_len);
    pos += topic_len;

    // Packet ID (QoS 1/2)
    if (qos > 0) {
        client->packet_id++;
        packet[pos++] = client->packet_id >> 8;
        packet[pos++] = client->packet_id & 0xFF;
    }

    // Payload
    memcpy(&packet[pos], payload, payload_len);
    pos += payload_len;

    return client->send(packet, pos);
}
```

## Best Practices

### Protocol Selection Guide

| Protocol | Use Case | Bandwidth | Complexity |
|----------|----------|-----------|------------|
| CAN | Automotive, industrial | Medium | High |
| Modbus | Industrial automation | Low | Low |
| I2C | Sensors, EEPROMs | Low | Low |
| SPI | High-speed peripherals | High | Low |
| MQTT | IoT cloud connectivity | Variable | Medium |
| CoAP | Constrained IoT | Low | Medium |

### Error Handling

- Always implement timeouts on communication
- Add CRC/checksum validation
- Implement retry mechanisms with backoff
- Log communication errors for diagnostics
- Handle bus errors and recovery gracefully

## Resources

- [CAN in Automation](https://www.can-cia.org/)
- [Modbus Organization](https://modbus.org/)
- [MQTT Specification](https://mqtt.org/mqtt-specification/)
- [I2C Specification](https://www.i2c-bus.org/)
